# slurm-with-distributed-stroage
Build a slurm cluster over machines that do not have a shared file system

## conventions

1. You are on the login node of your cluster, with hostname `login-0` and IP address `192.168.100.0`.
2. There are several compute nodes with hostname `compute-1`, `compute-2`, ..., `compute-N`.
3. The local network ips of all the compute nodes are `192.168.100.X`, where X is a number from 1 to N.

## login node

TODO

## compute node

### prerequisites

1. In this document, all the compute nodes are assumed to have the same system configuration and can connect to each other within the local network.

2. You have the root privilege on all the machines, so that you can use `pdsh -l root` to run commands in parallel across multiple machines.`

3. The working directory, i.e `$PWD` is `/home/slurm_install`.

### share directory using NFS

First, we should create a directory on all nodes to store all shared data. As an example, we use `/nvme/slurm_share`.

```bash
pdsh -w "192.168.100.[0-N]" -l root 'mkdir /nvme/slurm_share'
```

To enable nfs, preparing 3 files in the working directory:
1. `/etc/hosts` : a file containing hostname and IP address mapping, so that you can use hostnames instead of IP addresses when connecting to other machines.
2. `/etc/exports`: a file used by the server to specify which directories are accessible from remote hosts.
3. `/etc/fstab`: a configuration file for the NFS client that tells it where to find the exported filesystems.

the contents of these files are as follows:

```text
# /home/slurm_install/hosts
192.168.100.0  login-0
192.168.100.1 compute-1
192.168.100.2 compute-2
...
192.168.100.N compute-N
```

```text
# /home/slurm_install/exports
/nvme/slurm_share login-*(rw,no_root_squash)
/nvme/slurm_share compute-*(rw,no_root_squash)
```

the second column of the fstab file is the mount point of the shared directory on each node. Here we use /nvme/slurm_nfs/${hostname} as the mount point of each node.

```text
# /home/slurm_install/fstab
192.168.100.0:/nvme/slurm_share  /nvme/slurm_nfs/login-0 nfs rw,bg,hard,intr,tcp,nfsvers=4,noatime,actimeo=30 0 0
192.168.100.1:/nvme/slurm_share /nvme/slurm_nfs/compute-1 nfs rw,bg,hard,intr,tcp,nfsvers=4,noatime,actimeo=30 0 0
...
192.168.100.N:/nvme/slurm_share /nvme/slurm_nfs/compute-N nfs rw,bg,hard,intr,tcp,nfsvers=4,noatime,actimeo=30 0 0
```

Then we use the following commands to apply these changes on all nodes:
```bash
# fist copy the files from login-0 to other machines
pdcp -w "192.168.100.[1-N]" /home/slurm_install/* root@compute-*:/home/slurm_install/
# then apply the changes by appending the file content to `/etc/hosts`, `/etc/exports` and `/etc/fstab` respectively.
# In case we make a mistake, backup the original files first:
pdcp -w "192.168.100.[1-N]" /etc/hosts root@compute-*:/etc/hosts.backup
pdcp -w "192.168.100.[1-N]" /etc/exports root@compute-*:/etc/exports.backup
pdcp -w "192.168.100.[1-N]" /etc/fstab root@compute-*:/etc/fstab.backup
# then append the file content to these files:
pdsh -w "192.168.100.[0-N]" -l root 'cat /home/slurm_install/hosts >> /etc/hosts'
pdsh -w "192.168.100.[0-N]" -l root 'cat /home/slurm_install/exports >> /etc/exports'
pdsh -w "192.168.100.[0-N]" -l root 'cat /home/slurm_install/fstab >> /etc/fstab'
```

Now we can mount the shared directory on all nodes:
```bash
pdsh -w "192.168.100.[0-N]" -l root 'mkdir -p /nvme/slurm_nfs/${hostname}'
pdsh -w "192.168.100.[0-N]" -l root 'mount -a'
```


### use mergerfs to combine multiple file systems into one

we use [mergerfs](https://github.com/trapexit/mergerfs) to combine all the shared directories on different nodes into a single directory, so that we can access them as if they are in the same machine.

```bash
# first install mergerfs and fuse-utils
# find the latest version of mergerfs from here: https://github.com/trapexit/mergerfs/releases
# for example, we use version 2.40.2, and our system is centos 7
wget https://github.com/trapexit/mergerfs/releases/download/2.40.2/mergerfs-2.40.2-1.el7.x86_64.rpm
```

TODO