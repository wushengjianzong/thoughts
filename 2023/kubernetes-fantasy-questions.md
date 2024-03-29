# Kubernetes Fantasy Questions

## Metadata Center

> Sorry, it is a cross-realm problem about RPC, CQRS, HTAP, etc.

## Super Cluster

+ Multi-Tenant
    - Projects
        * vcluster
        * KubeZoo
+ Virtual Node
    - Projects
        * virtual-kubelet
+ Kubernetes on Kubernetes
    - Projects
        * cluster-api

## Hybrid Deployment

+ Concept
    - Noisy Neighbors
    - Online/Offline Task Scheduling
+ Hardware
    - NUMA
    - CPU Cache
    - Hyper-Threading
+ Kernel
    - CFS
+ Kubernetes
    - CPUManager
    - PriorityClass
    - Scheduler Framework

## Hyper-Converged Infrastructure

+ Hardware
    - PCIe
    - SR-IOV/SIOV
    - VT-d
+ Software
    - Device Driver
    - CUDA/OpenCL
+ Kubernetes
    - Device Plugin
    - Resource API
    - Scheduler Framework

## Compute

+ Specification
    - CRI(Container Runtime Interface)
    - OCI(Open Container Initiative)
    - NRI(Node Resource Interface)
    - CDI(Container Device Interface)
+ Container Lifestyle Management
    - Projects
        * containerd
        * cri-o
+ Soft-Isolation
    - Kernel
        * cgroups
        * namespace
        * unionfs
    - Projects
        * runc
+ Hard-Isolation
    - Hardware
        * VT-x
        * EPT
    - Kernel
        - KVM
    - Projects
        * kata
        * qemu
        * firecracker
        * cloud-hypervisor

## Storage
+ Specification
    - CSI(Container Storage Interface)
+ Hardware
    - SCSI
    - NVMe
+ Kernel
    - nbd
    - vfs
    - nfs
+ Projects
    - Rook
        * block device
        * file system
        * object store
    - OpenEBS(CAS)
        * cstor
        * jiva

## Network
+ Specification
    - CNI(Container Network Interface)
+ Kernel Mode
    - Kernel
        * netfilter
        * eBPF
    - Projects
        * calico
        * cilium
+ User Mode
    - Projects
        * dpdk
+ HCI
    - Hardware
        * RDMA
        * P4
    

