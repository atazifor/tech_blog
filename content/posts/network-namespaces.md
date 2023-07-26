---
title: "Networking Concepts with Network Namespaces and Docker Containers"
date: 2023-07-24T05:20:00-05:00
tags: []
categories: [Networking]
weight: 50
show_comments: true
katex: true
draft: false
---

{{< toc >}}

In this tutorial, we will explore essential networking concepts using network namespaces and docker containers. 
By understanding these fundamentals, you will be well-prepared to dive into networking in Kubernetes and other container orchestration platforms.

<!--more-->

# Introduction to Network Namespaces

In the world of containerization and orchestration, understanding network namespaces is crucial. Namespaces are an essential Linux kernel feature that allows processes to have their isolated view of system resources. With namespaces, a process can have its private set of resources, such as network interfaces, mount points, process IDs, and more. This isolation provides a foundation for container technologies and container orchestrators like Docker and Kubernetes.

One particular namespace that plays a vital role in container networking is the network namespace. It enables containers to have their separate network stack, including network interfaces, routing tables, and firewall rules. Without network namespaces, containers would share the host's network stack, leading to potential conflicts and security concerns.

# Purpose of This Tutorial

In this lab, we will explore essential networking concepts using network namespaces and Docker containers. We'll set up two network namespaces, red and blue, and create virtual Ethernet (veth) pairs to establish communication between them. We will also introduce a network bridge and demonstrate how to enable internet access for the containers within the namespaces.

Let's get started!

# Step 1: Create Network Namespaces

We begin by creating two network namespaces, `red` and `blue`.

{{< figure src="/img/posts/networking/starter-ns.svg" width=300 title="" >}}

```shell
ip netns add red
ip netns add blue
```
The `ip netns add` command is used to create separate network namespaces for `red` and `blue`.

# Step 2: Create Virtual Ethernet (veth) Pairs

Next, we create veth pairs, `veth-red` and `veth-blue`, to connect the namespaces.

{{< figure src="/img/posts/networking/veth-link.svg" width=300 title="" >}}

```shell
ip link add veth-red type veth peer name veth-blue
```
The `ip link add` command creates two virtual Ethernet (veth) devices, `veth-red` and `veth-blue`, and establishes a connection between them through a virtual peer (`peer name`).

# Step 3: Assign Interfaces to Respective Namespaces

Now, we assign the veth interfaces to their corresponding namespaces.

{{< figure src="/img/posts/networking/netns-link.svg" width=300 title="" >}}

```shell
ip link set veth-red netns red
ip link set veth-blue netns blue
```
The `ip link set` command moves the `veth-red` interface into the `red` namespace and the `veth-blue` interface into the `blue` namespace.

# Step 4: Configure IP Addresses

We configure IP addresses for the veth interfaces inside the namespaces.

{{< figure src="/img/posts/networking/netns-ip.svg" width=300 title="" >}}

```shell
ip -n red addr add 192.168.15.1/24 dev veth-red
ip netns exec blue ip addr add 192.168.15.2/24 dev veth-blue
```
The first command assigns the IP address `192.168.15.1` to the `veth-red` interface within the `red` namespace, and the second command assigns the IP address `192.168.15.2` to the `veth-blue` interface within the `blue` namespace.

# Step 5: Enable Interfaces

Next, we enable the veth interfaces within their namespaces.

```shell
ip -n red link set veth-red up
ip -n blue link set veth-blue up
```
These commands bring the `veth-red` interface up within the `red` namespace and the `veth-blue` interface up within the `blue` namespace.

# Step 6: Test Communication

Now that the interfaces are up, let's test communication between the namespaces.

```shell
ip netns exec red ping 192.168.15.2
ip netns exec blue ping 192.168.15.1
```
These commands use the `ip netns exec` command to execute `ping` tests from the `red` namespace to the `blue` namespace and vice versa.

# Step 7: Create a Network Bridge

Let's introduce a network bridge named `v-net-0`.

```shell
ip link add v-net-0 type bridge
```
This command creates a network bridge named `v-net-0`.

# Step 8: Attach Interfaces to the Bridge

Now, we attach the veth interfaces to the network bridge.

```shell
ip link set veth-red-br master v-net-0
ip link set veth-blue-br master v-net-0
```
These commands attach the `veth-red` interface to the `v-net-0` bridge and the `veth-blue` interface to the `v-net-0` bridge.

# Step 9: Enable the Bridge

Next, we enable the bridge.

```shell
ip link set v-net-0 up
```
This command brings the `v-net-0` bridge up, enabling it to forward traffic.

# Step 10: Assign IP Address to the Bridge

We assign an IP address to the bridge so that containers connected to it can communicate with the outside world.

```shell
ip addr add 192.168.15.5/24 dev v-net-0
```
This command assigns the IP address `192.168.15.5` to the `v-net-0` bridge, making it a gateway for the connected containers.

# Step 11: Enable Forwarding and NAT

To enable internet access for the containers, we need to enable IP forwarding and set up NAT.

```shell
echo "1" > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
```
The first command enables IP forwarding, allowing the bridge (`v-net-0`) to forward packets between interfaces. The second command sets up NAT (Network Address Translation) to enable internet access for the containers within the namespaces.

# Step 12: Test Internet Access

Finally, let's test internet access from within the `red` namespace.

```shell
ip netns exec red ping 8.8.8.8
```
This command tests internet access from within the `red` namespace by pinging the Google DNS server.

Congratulations! You have successfully explored essential networking concepts using network

namespaces and Docker containers. Understanding these fundamentals will serve as a solid foundation for further exploring more complex networking scenarios and container orchestration platforms like Kubernetes.

Happy networking exploration!


# Running the Tutorial on macOS with Docker

If you are using macOS and want to follow the networking tutorial that involves network namespaces and Docker containers, you might have noticed that macOS lacks some of the essential networking commands like `ip`, `ping`, and `iptables`. To overcome this limitation and ensure you can fully participate in the tutorial, we have prepared a Docker image that provides the necessary tools.

## Using the Docker Image

To get started, make sure you have Docker installed on your macOS system. If you haven't installed Docker yet, you can download it from the official Docker website (https://www.docker.com/products/docker-desktop).

Next, we will create and use a Docker image that contains the required networking tools. The Docker image is based on the official Ubuntu base image and includes packages like `iproute2`, `ping`, `net-tools`, `dnsutils`, and `iptables`. These packages are essential for the tutorial as they allow you to perform various networking tasks within network namespaces.

### Dockerfile Contents

Below is the content of the Dockerfile used to build the Docker image:

```dockerfile
# Use the official Ubuntu base image
FROM ubuntu:latest

# Install sudo and essential networking tools
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y sudo iproute2 iputils-ping \
    net-tools dnsutils iptables && \
    rm -rf /var/lib/apt/lists/*

# Start an interactive shell by default
CMD ["/bin/bash"]
```

### Building the Docker Image

To build the Docker image, save the contents of the above Dockerfile in a file named "Dockerfile" (without any file extension) within a directory on your macOS system. Then, open the terminal, navigate to the directory containing the Dockerfile, and execute the following command:

```bash
docker build -t ubuntu_with_basic_net -f Dockerfile .
```

This command will build the Docker image and tag it with the name "ubuntu_with_basic_net". The `-f` flag specifies the path to the Dockerfile.

### Running the Docker Container

Once the Docker image is built, you can run a Docker container using the image. This container will provide you with all the necessary networking tools to perform the tutorial.

Use the following command to start the Docker container:

```bash
docker run -it --name containerA --privileged ubuntu_with_basic_net
```

The `--privileged` flag is used to grant the container extended privileges, including access to devices on the host system, which may be required for certain networking operations.

## Following the Tutorial

With the container running, you can now proceed to follow the networking tutorial using network namespaces and Docker containers. Inside the container, you have access to commands like `ip`, `ping`, `netstat`, `iptables`, and more, allowing you to perform the networking tasks within network namespaces as described in the tutorial.

If you encounter any issues or have questions during the process, feel free to ask for further assistance. Happy networking exploration on your macOS system!