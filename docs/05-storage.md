# VM Storage — How CDI, DataVolumes and PVCs Work Together

Before a VM can start, it needs a disk with an OS already on it. This is different from containers where you just pull an image. For VMs, the disk has to be provisioned as a PVC first. CDI handles all of this.

---

## The Flow

```
DataSource (pre-built OS image PVC in openshift-virtualization-os-images)
  └── DataVolume (you create this, it tells CDI what to clone)
        └── CDI importer pod (spins up automatically, does the actual copy)
              └── PVC (the VM's virtual hard disk, ready to boot from)
                    └── StorageClass → CSI driver → backend storage (gp3/EBS in this lab)
```

---

## What Is a DataSource?

Red Hat maintains a set of pre-built OS images as PVCs in the openshift-virtualization-os-images namespace. These are ready to use. You do not create them, you just reference them.

![OS Image Sources](../screenshots/06-os-image-sources.png)
*Available OS images in the sandbox: Fedora, RHEL 7/8/9/10, CentOS Stream 9/10, Windows 2016/2019/2022/2025, Windows 10/11. All are 30Gi PVCs on gp3 storage.*

---

## What Is a DataVolume?

A DataVolume is a CDI object that wraps a PVC with a source. When you create one pointing at a DataSource, CDI takes care of everything:

1. Creates a PVC in your namespace
2. Spins up a cloner pod
3. Copies the source OS image into the PVC
4. Updates the PHASE and PROGRESS fields so you can track it

```bash
oc get dv fedora-vm-disk -o wide
```

![DV CLI Output](../screenshots/07-dv-cli-output.png)
*fedora-vm-disk showing Succeeded and 100.0% progress. CDI has finished cloning the Fedora image into the PVC.*

Once PHASE is Succeeded, your PVC has a full copy of the OS image on it. That PVC is now the VM's virtual hard disk.

---

## The PVC in OCP Console

After the DataVolume succeeds, you can see the PVC in the console under Storage → PersistentVolumeClaims.

![PVC Bound](../screenshots/08-pvc-bound.png)
*fedora-vm-disk PVC showing Status Bound, Capacity 35 GiB, StorageClass gp3. This is the actual disk the VM boots from.*

---

## What You See Inside the VM

When you log into the VM and run lsblk or df -h, you can see the disk directly:

![VM Console](../screenshots/05-vm-console.png)
*vda is the virtual disk (35G) provisioned by CDI from the PVC. vda3 is the root partition mounted at /, /home, /boot and /var.*

![Filesystem](../screenshots/04-filesystem.png)
*The filesystem view from OCP console showing vda3 as btrfs with 34.45 GiB total, mounted at root.*

The vda disk maps directly to the PVC. The PVC maps to an AWS EBS volume via the gp3 StorageClass and CSI driver.

---

## RWO vs RWX — Important for Live Migration

In this lab I used gp3 StorageClass with ReadWriteOnce (RWO) access mode. That means the disk can only be attached to one node at a time.

This is fine for running a VM normally. But for live migration, you need ReadWriteMany (RWX) because during migration the disk has to be accessible from both the source node and the destination node at the same time.

For RWX you need a storage backend that supports it, like ODF (OpenShift Data Foundation) with Ceph. gp3 (AWS EBS) does not support RWX.

| Access Mode | When to Use |
|------------|-------------|
| RWO | Normal VM workloads, single node |
| RWX | Live migration, must use ODF or Ceph |

---

## What We Used in This Lab

```
StorageClass: gp3 (AWS EBS via CSI driver)
Access Mode: RWO
Size: 35Gi
Backend: AWS EBS volume
```

---

## Standalone DataVolume YAML

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: fedora-vm-disk
spec:
  source:
    pvc:
      namespace: openshift-virtualization-os-images
      name: fedora
  storage:
    resources:
      requests:
        storage: 35Gi
```
