# Podman 

- an open source tool that you can use to manage your containers locally.
- is daemonless ; interacts directly with containers, images, and registries without a daemon.
  - removes **single point of failure**.  
  - removes the need of elevated privileges to run a daemon
- Similar to other common Container Engines (Docker, CRI-O, containerd), Podman relies on an OCI compliant Container Runtime (runc, crun, runv, etc) to interface with the operating system and create the running containers. 
- Containers can either be run by root or by a non-privileged user. 
- manages the entire container ecosystem which includes pods, containers, container images, and container volumes using the **libpod** library.

## Interaction

-  CLI
- the RESTful API 
- desktop application called Podman Desktop.


# Basic Networking Guide for Podman

## Is the container run by a root user ?

### YES
- The default networking mode for rootful containers is `netavark`, which allows a container to have a routable IP address.
- will create a `bridged` network. Bridge networking creates an interface for the container on an internal bridge network, which is then connected to the internet via Network Address Translation(NAT). 
- Some use `macvlan` for networking as well. The `macvlan` plugin forwards an entire network interface from the host into the container, allowing it access to the network the host is connected to.
- Containers can be joined to a network when they are created with the `--network` flag, or after they are created via the `podman network connect` and `podman network disconnect` commands.

### NO

- unprivileged users cannot create networking interfaces on the host. 
- for rootless containers, the default network mode is `slirp4netns`. 
- `slirp4netns` cannot give containers a routable IP address. 
- It creates a tunnel from the host into the container to forward traffic.
- As of Podman version 4.0, rootless users can also use `netavark` and `aardvark` DNS server; there is no default network configuration provided. You simply need to create a network, and the one will be created as a bridge network. If you would like to switch from CNI networking to netavark, you must issue the podman system reset `--force` command. This will delete all of your images, containers, and custom networks.


How to connect containers to multiple networks ?

```
podman run -d --name double-connector \
--net postgres-net,redis-net \
container-image:latest
```
## DNS

When you use the default Podman network, the domain name system (DNS) for other containers in that network is disabled. To enable DNS resolution between containers, create a Podman network and connect your containers to that network.

When using a network with DNS enabled, a container's hostname is the name assigned to the container. For example, if a container is started with the following command, then the other containers on the `test-net` network can make requests to the first container by using the `basic-container` hostname. The `basic-container` hostname resolves to the current IP address of the `basic-container` container.

```
podman run --net test-net --name basic-container example-image
```

## Port Forwarding

- maps a port from the host machine where the container runs to a port inside of a container.

- The -p option of the podman run command forwards a port `HOST_PORT:CONTAINER_PORT`

For example, the following command maps port 80 inside the container to port 8075 on the host machine.

`podman run -p 8075:80 my-app`

- the container is assigned the broadcast address (0.0.0.0). This means that the container is accessible from all networks on the host machine.

To publish a container to a specific host and to limit the networks it is accessible from:

`podman run -p 127.0.0.1:8075:80 my-app`

## List Port Mappings

```
podman port my-app
podman port --all
```

## Inspect Network

- A container called `my-app` is attached to the `apps` network. 

`podman inspect my-app -f '{{.NetworkSettings.Networks.apps.IPAddress}}'`

# Rootless Containers

A container is rootless only when it meets the following conditions:

- The containerized process does not use the `root` user, which is a special privileged user in Linux and UNIX systems. Such a user is for administrative purposes and has the ID 0.

- The `root` user inside of the container is not the `root` user outside of the container.

- The container runtime does not use the `root` user. For example, if your container runtime runs as the `root` user, then containers managed by such runtime are not rootless containers.

Because Podman starts each container as a new process, the runtime does not require elevated privileges.

The Podman process exits after creating a container. Then the container process ID attaches to the systemd parent process ID.


## Prerequisites for Rootless Containers

### cgroup v2

- is a Linux kernel feature that Podman uses to limit container resource use without requiring elevating privileges.

Podman on Red Hat Enterprise Linux 9 (RHEL 9) uses cgroup v2 by default, with the runc container runtime implementation.

### slirp4netns

- Podman uses the slirp4netns package to implement user-mode networking for unprivileged network namespaces.

### fuse-overlayfs

Podman uses the `fuse-overlayfs` package to manage the Copy-On-Write (COW) file system.

## Changing the Container User

```
FROM registry.access.redhat.com/ubi9/ubi

RUN adduser \
   --no-create-home \
   --system \
   --shell /usr/sbin/nologin \
   python-server

USER python-server

CMD ["python3", "-m", "http.server"]
```
- The RUN instruction runs as the root user, because it precedes the USER instruction.

## Explaining User Mapping

- Traditionally, when an attacker gains access to the container file system by using an exploit, the `root` user inside the container corresponds to the `root` user outside of the container. This means that if an attacker escapes the container isolation, then they have elevated privileges on the host system.

Podman maps users inside of the container to unprivileged users on the host system by using subordinate ID ranges. Podman defines the allowed ID ranges in the /etc/subuid and /etc/subgid files.

```
cat /etc/subuid /etc/subgid
student:100000:65536
student:100000:65536
```

- student user can allocate 65536 user IDs starting with the ID 100000. 
This leads to the HostUserID = 100000 + ContainerUserID - 1 ID mapping. Consequently, user ID 1 in a container maps to the host user ID 100000, and so on.

The user ID 0 (root) is an exception, because the root user maps to the user ID that started the container. For example, if a user with the ID 1000 starts a container that uses the root user, then the root user maps to the host user ID 1000.

To generate the subordinate ID ranges, use the usermod command.

```
sudo usermod --add-subuids 100000-165535 \
  --add-subgids 100000-165535 student
  
grep student /etc/subuid /etc/subgid
/etc/subuid:student:100000:65536
/etc/subgid:student:100000:65536
```

- The `/etc/subuid` and `/etc/subgid` files must exist before you define the subordinate ID ranges with the `usermod` command. After you define the ranges, you must execute the `podman system migrate` for the new subordinate ID ranges to take effect.

You can verify the mapped user by using the podman top command. For example, the following command starts a container and uses the id command to verify that the user inside of the container is root.

```
podman run -it registry.access.redhat.com/ubi9/ubi bash
[root@e6116477c5c9 /]# id
uid=0(root) gid=0(root) groups=0(root)
```

Then, you must verify the mapping of host user (huser) to the container user (user). 
The following example uses the container ID e6116477c5c9:

```
podman top e6116477c5c9 huser user
HUSER       USER
1000        root
```

The preceding example shows that the user inside the container, root, is mapped to a user with ID 1000 on the host system.

Alternatively, you can verify the same ID mapping by printing the /proc/self/uid_map and /proc/self/gid_map files inside of the container:

```
[root@e6116477c5c9 /]# cat /proc/self/uid_map /proc/self/gid_map
         0       1000          1
         0       1000          1
```

- When you execute a container with elevated privileges on the host machine, the root mapping does not take place even when you define subordinate ID ranges, for example:

```
sudo podman run -it registry.access.redhat.com/ubi9/ubi bash
[root@4746207beab7 /]# id
uid=0(root) gid=0(root) groups=0(root)
```

In a new terminal, verify the podman top output:

```
sudo podman top 4746207beab7 huser user
HUSER       USER
root        root
```

## Limitations of Rootless Containers

### Non-trivial Containerization

Some applications might require the root user.

For example, applications such as HTTPd and Nginx start a bootstrap process and then spawn a new process with a non-privileged user, which interacts with external users. Such applications are non-trivial to containerize for rootless use.

Red Hat provides containerized versions of HTTPd and Nginx that do not require root privileges for production usage. You can find the containers in the Red Hat container registry.

### Required Use of Privileged Ports or Utilities

Rootless containers cannot bind to privileged ports, such as ports 80 or 443. Red Hat recommends that you do not use privileged ports, and use port forwarding instead. However, if you require the use of privileged ports, then you can configure the unprivileged port range:

`sudo sysctl -w "net.ipv4.ip_unprivileged_port_start=79"`

- This means rootless containers can bind to port 80 and higher.

- Similarly, rootless containers cannot use utilities that require the root user, such as the ping utility. This is because the ping utility requires elevated privileges to establish raw sockets, which is an action that requires the `cap_net_raw` privilege.

To solve such a requirement, verify whether you can grant the privilege to a non-root user. For example, you can specify a range of group IDs that are allowed to use the ping utility by using the net.ipv4.ping_group_range kernel parameter:

`sudo sysctl -w "net.ipv4.ping_group_range=0 2000000"`