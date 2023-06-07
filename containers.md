# Containers 

- an encapsulated process that includes the required runtime dependencies for the program to run. 
- application-specific libraries are independent of the host operating system libraries. 

Containers use Linux kernel features like:

- `Control Groups (cgroups)` - for resource management, such as CPU time allocation and system memory. 

- `Namespaces` - to isolate processes within containers from each other and from the host system. Within the Linux kernel, there are different types of namespaces. Each namespace has its own unique properties:
  - A `user namespace` has its own set of user IDs and group IDs for assignment to processes. 
    - process can have root privilege within its user namespace without having it in other user namespaces.
  - A `process ID (PID) namespace` assigns a set of PIDs to processes that are independent from the set of PIDs in other namespaces. 
  - A `network namespace` has an independent network stack: its own private routing table, set of IP addresses, socket listing, connection tracking table, firewall etc.
  - A `mount namespace` has an independent list of mount points seen by the processes in the namespace. This means that you can mount and unmount filesystems in a mount namespace without affecting the host filesystem.
  - An `interprocess communication (IPC) namespace` has its own IPC resources.
  - A `UNIX Time‑Sharing (UTS) namespace` allows a single system to appear to have different host and domain names to different processes.

![Containers_vs_VMs](containers_vs_vms.png)

Keywords:

- `chroot` - a method to partially or fully isolate an environment
- `Open Container Initiative (OCI)` - a governance organization that defines standards for creating and running containers. 
- Sandboxing for isolation
  - Seccomp profiles limit syscalls to kernel
  - AppArmor profile to limit access to resources (ex. deny /proc/* w)
  
  - `Container` > `syscall` > `Linux Kernel`
  - `Container` > `syscall` > `gVisor` > `Linux Kernel` (each Container has a gVisor kernel, that acts as a middle man and limit syscalls)

  - Kata Containers - Sandboxing each container in a Light Weight VM (with a VM Kernel)