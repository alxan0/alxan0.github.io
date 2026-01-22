---
title: Isolating Home Lab Traffic with a Private L2 Network
date: 2025-04-22 18:00:00 +0000
categories: [Home Lab, Networking]
tags: [truenas, docker, virtualization, security]
description: A guide on creating an isolated Layer 2 bridge to connect a Home Assistant VM and a Frigate Docker container without exposing the host.
---

## **Introduction**

When setting up your home lab it is always a good idea to keep security in mind. One way to do this is to isolate parts of your infrastructure, this will allow you to reduce the damages an attacker can cause if they gain access to one of your services. In my case frigate is running on my host in docker container because i want to use the gpu acceleration and for home assistant i'm using the Home Assistant OS. By default TrueNAS Scale doesn't allow VMs to comunicate with the host, this is easy to deal with as you can simply use a bridge network, but this gives access to all the services running on TrueNAS as well as it's GUI which is something I would like to avoid. This left me with two options: use a vpn or a private network. In this case I choose the private network as it's simple to implement and it has a lower CPU overhead since it's local switching at layer 2. 
Time to tinker.

## **Overview**

- **Goal**: Create a **private, isolated L2 network** that both the VM and the Docker container can attach to.
- **No Gateway/No Router**: This subnet is only for communication between Frigate and Home Assistant, so there’s no default route.
- **Linux Bridge**: We’ll use a system-level Linux bridge on the host to unify the VM and the container at Layer 2.

> **Note**: The specific commands or GUI steps can vary depending on your host OS (e.g., TrueNAS SCALE or a standard Linux distro). Below is the general concept.

## **1. Create a Linux Bridge on the Host**

1. **Check existing interfaces** to pick a name for your new bridge:

```bash
ip link show
```

2. **Create a new bridge** (e.g., `br20`) with no IP address or gateway:

```bash
sudo ip link add br20 type bridge
```
- This is your dedicated private bridge interface.

3. **Bring the bridge up**:

```bash
sudo ip link set br20 up
```
- At this point, `br20` is just an empty bridge with no IP.

> **Note:** If you assign an IP, then your host will also connect to the network and frigate and home assistant will be able to communicate with it, rendering our efforts useless.

##### **For TrueNAS SCALE**
1. **Go to the network tab** and click on create
2. **Create a new bridge** (e.g., `br20`) with no IP address or gateway:

![TrueNAS Network Interface](/assets/truenas-net-interface.png)

>**Note 1:** I would recommend to set a default gateway in global configuration before making the bridge and set a static IP for you TrueNAS server if you don't have one already.

## **2. Attach Your Home Assistant VM to `br20`**

#### **TrueNAS SCALE / KVM GUI Approach**

- When you edit your **VM network interface** in the TrueNAS SCALE UI (or KVM Manager),
	- Create a new network interface
	- Set the adapter type to VirtIO 
    - Select **`br20`** as the target bridge.

> **Note 1:** I would recommend to set a static IP for your Home Assistant instance before attaching it to our network.

## **3. Set a static IP for `br20` on Your Home Assistant OS VM**

1. **Set static IP** inside Home Assistant settings
> **Note:** Unfortunately HA doesn't allow us to set a static IP without also setting a gateway, so we need to remove the gateway after saving

2. **Exit the HA CLI**
```bash
ip addr add 192.168.50.10/24 dev eth0
ip link set eth0 up
ip route add 192.168.50.0/24 dev eth0
```
## **4. Create a Corresponding Docker Network on `br20`**

We want Docker containers to attach directly to `br20` instead of Docker’s default `docker0` or user-defined bridging. We’ll instruct Docker to use the **system bridge**:

```bash
docker network create \ 
	--driver bridge \ 
	--subnet=192.168.50.0/24 \ 
	-o "com.docker.network.bridge.name"="br20" \ 
	vm_bridge
```
- **`--subnet=192.168.50.0/24`**: Must match the subnet you intend to use with the VM.
- **No `--gateway`**: Because we want a completely isolated network and will assign addresses manually.
- **`-o com.docker.network.bridge.name=br20`**: Tells Docker to use the existing `br20` bridge you created.

> **Important**: If Docker complains that `br20` already exists, it might require `--force` or for you to remove any existing Docker network with the same name.

After this, Docker sees a “network” named `vm_bridge` that physically corresponds to the **system-level** `br20` interface.

## **5. Run Frigate in That Network With a Static IP**

Now that you have the `vm_bridge` network defined, run your Frigate container and specify the **IP address** in the same subnet as the VM:

**Example using `docker compose`:**

```yaml
services:
  frigate:
    container_name: frigate
    privileged: true # this may not be necessary for all setups
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:stable
    shm_size: 64mb # update for your cameras based on calculation above
    devices:
      - /dev/dri:/dev/dri # For intel hwaccel, needs to be updated for your hardware
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ${appdata}/frigate/config:/config
      - ${media}/frigate:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - 8971:8971
      - 8554:8554 # RTSP feeds
      - 8555:8555/tcp # WebRTC over tcp
      - 8555:8555/udp # WebRTC over udp
    # attaching the container to default network and our vm_bridge network
    networks:
      default: null
      vm:
        ipv4_address: 192.168.50.20
    environment:
      FRIGATE_RTSP_PASSWORD: ${FRIGATE_RTSP_PASSWORD}
    #cap_add:
    #  - CAP_PERFMON

networks:
  default:
    name: frigate_main
    external: false
  vm:
    name: vm_bridge
    external: true
```
- This assigns **`192.168.50.20`** to Frigate on the `br20` bridge.

## **Summary**

3. **Create a new system-level Linux bridge** (`br20`)
4. **Attach your VM** to that bridge (bridged mode, static IP).
5. **Tell Docker** to use that exact Linux bridge with a custom network.
6. **Assign static IPs** in the same subnet to the VM and the Frigate container.

## **Debugging**

``` shell
ls -l /sys/class/net/br20/brif

bridge link show dev br20

sudo tcpdump -i br20 -n
```
