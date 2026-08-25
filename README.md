# OpenShift Virtualization — Hands-on Lab Guide

I run a production OpenShift platform for a telecom SIP based contact center. All my workloads are containers. But KubeVirt kept coming up in architecture conversations and I had zero hands-on with it. So I set up a lab on Red Hat Developer Sandbox, went end to end, hit real errors, fixed them, and documented everything here.

This is not a copy paste from docs. This is what I actually did, what broke, and what I learned.

---

## What This Covers

- How the full stack works from bare metal to a running VM
- How to provision a Fedora VM using DataVolumes and CDI
- 3 real errors I hit and exactly how I fixed them
- How VM networking works and why the IP inside the VM is different from what OCP sees
- How VM storage works with CDI, DataVolumes and PVCs

---

## The Full Stack at a Glance

```
HP DL380 (bare metal server)
  └── RHCOS (OS installed directly on metal, no hypervisor)
        └── OpenShift (Kubernetes running on RHCOS nodes)
              └── OpenShift Virtualization Operator (adds VM capability)
                    ├── CDI — pulls the OS image and stores it as a PVC (the VM disk)
                    ├── VirtualMachine CR — the desired state, like a Deployment
                    ├── VMI — the running VM instance, like a Pod
                    └── virt-launcher pod — the actual pod running QEMU+KVM, this IS the VM
```

---

## Docs

| Doc | What it covers |
|-----|---------------|
| [01 - Architecture](docs/01-architecture.md) | Full stack explained layer by layer |
| [02 - VM Setup](docs/02-vm-setup.md) | Step by step VM creation with YAML |
| [03 - Errors and Fixes](docs/03-errors-and-fixes.md) | 3 real errors I hit and how I fixed them |
| [04 - Networking](docs/04-networking.md) | Why the VM IP looks different inside vs outside |
| [05 - Storage](docs/05-storage.md) | How CDI, DataVolumes and PVCs work together |

---

## Manifests

| File | What it is |
|------|-----------|
| [vm.yaml](manifests/vm.yaml) | VirtualMachine with cloudInit and DataVolume |
| [datavolume.yaml](manifests/datavolume.yaml) | Standalone DataVolume from Fedora DataSource |

---

## Screenshots

Real screenshots from my sandbox session.

| Screenshot | What it shows |
|-----------|--------------|
| [01-vm-running](screenshots/01-vm-running.png) | fedora-vm Running in OCP console, Fedora Linux 44 |
| [02-vm-details](screenshots/02-vm-details.png) | VMI, virt-launcher pod name, and network IP linked together |
| [03-virt-launcher-pod](screenshots/03-virt-launcher-pod.png) | The pod that IS the VM, running on EC2 worker node |
| [04-filesystem](screenshots/04-filesystem.png) | vda3 disk visible inside the VM via OCP console |
| [05-vm-console](screenshots/05-vm-console.png) | Logged in as sourav, ran ip addr, df -h and lsblk |
| [06-os-image-sources](screenshots/06-os-image-sources.png) | All OS images CDI can clone from |
| [07-dv-cli-output](screenshots/07-dv-cli-output.png) | CLI showing DataVolume Succeeded 100% |
| [08-pvc-bound](screenshots/08-pvc-bound.png) | PVC fedora-vm-disk Bound, 35 GiB, gp3 in OCP console |
| [09-vm-terminal-root](screenshots/09-vm-terminal-root.png) | Inside VM as root, ip addr, lsblk and df -h |

---

## Prerequisites

- Red Hat Developer Sandbox account (free at developers.redhat.com)
- oc CLI installed and logged in
- OpenShift Virtualization already enabled in the sandbox

---

## What I Still Want to Explore

- Live migration with RWX storage using ODF and Ceph
- MTV for migrating VMware VMs into OCP
- SR-IOV for high performance VM networking

---

## Author

Sourav Nandy — Solutions Architect, Platform and DevOps Engineer
[LinkedIn](https://linkedin.com/in/sourav-nandy) | [GitHub](https://github.com/sourav-ndx)
