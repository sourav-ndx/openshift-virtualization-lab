
## VM Running on OpenShift

Once the VM is up, this is what it looks like in the OCP console. Fedora Linux 44 Cloud Edition, 1 CPU, 1 GiB memory, running as a regular workload on the cluster alongside containers.

![VM Running](screenshots/01-vm-running.png)

OCP links everything together. From one screen you can see the VirtualMachineInstance, the virt-launcher pod name, and the network IP.

![VM Details](screenshots/02-vm-details.png)

## The virt-launcher Pod

This is the most important thing to understand. Every running VM has exactly one virt-launcher pod on a worker node. Inside that pod, QEMU runs the VM guest OS using the KVM kernel module that is already built into RHCOS. From OpenShift's perspective the VM is just this pod. The scheduler treats it like any other pod.

![virt-launcher Pod](screenshots/03-virt-launcher-pod.png)

## Storage — How the VM Gets Its Disk

CDI clones the Fedora OS image from the openshift-virtualization-os-images namespace into a PVC in your namespace. That PVC becomes the VM's virtual hard disk. You do not manually provision anything, CDI handles it when you create the DataVolume.

![DV CLI Output](screenshots/07-dv-cli-output.png)

Once the DataVolume shows Succeeded at 100%, the PVC is ready and the VM can boot from it.

![PVC Bound](screenshots/08-pvc-bound.png)

These are all the OS images available in the sandbox that CDI can clone from. Fedora, RHEL versions, CentOS, and even Windows are all there.

![OS Image Sources](screenshots/06-os-image-sources.png)

## Networking — Why the IP Looks Different Inside vs Outside

When I ran ip addr inside the VM I got 10.0.2.2. But OCP was showing the VM IP as 10.130.0.89. Same VM, two different IPs.

This is because OpenShift Virtualization uses masquerade networking by default. The VM has an internal IP managed by QEMU inside the virt-launcher pod. All traffic in and out gets NATed through the virt-launcher pod's real OVN-Kubernetes IP. So the cluster sees 10.130.0.89 and the VM thinks it is 10.0.2.2.

![VM Terminal Root](screenshots/09-vm-terminal-root.png)

The lsblk output here also shows vda which is the 35G virtual disk from the PVC. df -h confirms the filesystem is up and only using 2% of the 35G.

## Filesystem Inside the VM

![Filesystem](screenshots/04-filesystem.png)

vda3 is the root partition with a btrfs filesystem of 34.45 GiB. This maps directly to the PVC that CDI provisioned.

## VM Console

Logged in as sourav using the credentials set in cloudInit. Passwordless sudo works as well.

![VM Console](screenshots/05-vm-console.png)

## Docs

| Doc | What it covers |
|-----|---------------|
| [01 - Architecture](docs/01-architecture.md) | Full stack explained layer by layer |
| [02 - VM Setup](docs/02-vm-setup.md) | Step by step VM creation with YAML |
| [03 - Errors and Fixes](docs/03-errors-and-fixes.md) | 3 real errors I hit and how I fixed them |
| [04 - Networking](docs/04-networking.md) | Why the VM IP looks different inside vs outside |
| [05 - Storage](docs/05-storage.md) | How CDI, DataVolumes and PVCs work together |

## Manifests

| File | What it is |
|------|-----------|
| [vm.yaml](manifests/vm.yaml) | VirtualMachine with cloudInit and DataVolume |
| [datavolume.yaml](manifests/datavolume.yaml) | Standalone DataVolume from Fedora DataSource |

## Prerequisites

- Red Hat Developer Sandbox account (free at developers.redhat.com)
- oc CLI installed and logged in
- OpenShift Virtualization already enabled in the sandbox

## What I Still Want to Explore

- Live migration with RWX storage using ODF and Ceph
- MTV for migrating VMware VMs into OCP
- SR-IOV for high performance VM networking

## Author

Sourav Nandy — Solutions Architect, Platform and DevOps Engineer  
[LinkedIn](https://linkedin.com/in/sourav-nandy) | [GitHub](https://github.com/sourav-ndx)
