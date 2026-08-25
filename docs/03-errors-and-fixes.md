# Real Errors I Hit — and How I Fixed Them

These are actual errors I got while setting up the Fedora VM on Red Hat Developer Sandbox. Not from docs. Not theoretical. I hit all three of these and had to figure out what was going on.

---

## Error 1 — CloneValidationFailed: PVC Too Small

### What happened

```
CloneValidationFailed: target PVC size (10Gi) is smaller than source image size (30Gi)
```

### Why it happened

I had set the PVC size to 10Gi in the VM manifest. The Fedora image in the openshift-virtualization-os-images namespace is around 30Gi. CDI will not clone an image into a PVC that is smaller than the source. Simple as that.

### Fix

Increase the storage in the DataVolume template to something larger than the source image. I used 35Gi to have some headroom.

```yaml
dataVolumeTemplates:
  - metadata:
      name: fedora-vm-disk
    spec:
      storage:
        resources:
          requests:
            storage: 35Gi
```

---

## Error 2 — cloudInit Webhook Rejection: No User Defined

### What happened

The VM got rejected immediately on apply. The sandbox webhook said a cloudInit user must be defined.

### Why it happened

Red Hat Developer Sandbox enforces a webhook that checks every VM manifest before accepting it. One of the rules is that you must define at least one user in the cloudInit userData section. Without this, the VM spec never even gets stored, it gets rejected at the door.

### Fix

Add a cloudInitNoCloud volume with a userData block that defines a user:

```yaml
volumes:
  - name: cloudinit
    cloudInitNoCloud:
      userData: |
        #cloud-config
        user: sourav
        password: redhat123
        chpasswd:
          expire: false
```

---

## Error 3 — WaitForFirstConsumer Deadlock

### What happened

The DataVolume stayed stuck in PendingPopulation. Nothing moved. The VM never came up.

### Why it happened

This one took me the longest to figure out. Three things combined to create a deadlock:

1. The gp3 StorageClass uses volumeBindingMode: WaitForFirstConsumer. This means the PVC will not bind to a node until a consumer pod is actually scheduled on that node.
2. The VM runStrategy was set to Manual, so no pod would be scheduled until I explicitly started the VM.
3. When I tried to start the VM via virtctl or oc, the sandbox webhook blocked it.

So the situation was: PVC is waiting for a pod, pod is waiting for PVC to bind, CLI is blocked. Nobody moves.

```
PVC waiting for consumer pod
  Pod waiting for PVC to bind
    CLI blocked by webhook
```

### Fix

Start the VM from the OCP web console instead of the CLI. The console bypasses the CLI webhook restriction. Once I did that:

- virt-launcher pod got scheduled
- PVC saw a consumer and bound to the node
- CDI populated the disk
- VM came up

```
Virtualization → VirtualMachines → fedora-vm → Actions → Start
```

### Key takeaway

WaitForFirstConsumer + runStrategy Manual + restricted sandbox = deadlock. If you are in a sandbox or restricted environment, always start your first VM from the console.

---

## Summary

| Error | Root Cause | Fix |
|-------|-----------|-----|
| CloneValidationFailed | PVC smaller than source image | Set storage to 35Gi |
| cloudInit webhook rejection | No user defined in cloudInit | Add userData with user and password |
| WaitForFirstConsumer deadlock | PVC, pod, and CLI all blocking each other | Start VM from OCP web console |
