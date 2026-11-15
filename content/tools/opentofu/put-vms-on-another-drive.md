# Putting VMs on another drive

The fleet's VM disks go on the Proxmox storage `local-zfs` unless the tfvars file names another. This page adds a second drive to a Proxmox host as a storage of its own, and points the VMs at it: every VM on the host, one VM, or one disk of a VM. Do it before the first run creates the VMs, when the drive Proxmox is installed on is too small or too slow for them.

Status: written, not yet run.

## Prerequisites

- The drive is installed in the Proxmox host and holds nothing you want to keep.
- The VMs that should use it do not exist yet. The run creates a VM's disks once and never moves them, so a VM that exists keeps its disks where they are.
- You have the API token from [Create the API token](../../fleet-bootstrap/foundation/proxmox-and-installer.md#token).

## Placeholders

| Placeholder | Value |
| --- | --- |
| `<disk-id>` | The drive's name under `/dev/disk/by-id/`, from step 1 |
| `<pool>` | A name for the ZFS pool on the drive, such as `tank` |
| `<storage>` | A name for the Proxmox storage, such as `tank-vms` |
| `<vmid>` | The ID in Proxmox of a VM the run creates |

## 1. Find the drive {#find}

On the Proxmox host, as root, list the drives and their names by ID:

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL
ls -l /dev/disk/by-id/ | grep -v part
```

Match the size, model, and serial of the new drive to one name in the second list, such as `nvme-Samsung_SSD_990_PRO_2TB_S7KHNJ0X123456`. That name is `<disk-id>`. A name by ID stays the same when the drive moves to another port, which `/dev/sdb` does not.

## 2. Make a ZFS pool on it {#pool}

> [!WARNING]
> This erases everything on the drive. Check `<disk-id>` against step 1 before running it.

```bash
zpool create -o ashift=12 <pool> /dev/disk/by-id/<disk-id>
zpool status <pool>
```

The status shows the pool `ONLINE`. Use ZFS rather than LVM-thin, because [Rebuilding a VM](../../fleet-bootstrap/procedures/rebuild-a-vm.md#snapshot) snapshots the persistent disk with `zfs`, and its steps only work on ZFS.

## 3. Add it as a Proxmox storage {#storage}

```bash
pvesm add zfspool <storage> --pool <pool> --content images,rootdir --sparse 1
pvesm status
```

The status lists `<storage>` as `active`. `images` lets it hold VM disks, and `--sparse 1` gives a disk only the space it uses, as `local-zfs` does.

## 4. Let OpenTofu use it {#token}

Grant OpenTofu's role on the new storage, to the user and to the token, as [Create the API token](../../fleet-bootstrap/foundation/proxmox-and-installer.md#token) did for `local-zfs`:

```bash
pveum acl modify /storage/<storage> --users tofu@pve --roles TofuVM
pveum acl modify /storage/<storage> --tokens 'tofu@pve!tf' --roles TofuVM
```

Without the grant, the run stops when it creates a disk there, with a permission error for `Datastore.AllocateSpace`.

## 5. Name it in the tfvars file {#tfvars}

Open `opentofu/prod.tfvars` in the private repo. Set `datastore_id` at the level the change applies to. A lower level wins over a higher one.

| Where | Covers |
| --- | --- |
| The server's entry, under `servers` | Every VM on that Proxmox server that sets no `datastore_id` of its own |
| A VM's entry, under `vms` | That VM's OS disk, Docker disk, and cloud-init drive, and each extra disk that sets none of its own |
| A disk under a VM's `extra_disks` | That one disk, such as the persistent disk |

To put every VM on the new drive:

```hcl
servers = {
  vh01 = {
    endpoint     = "https://172.16.1.11:8006/"
    insecure     = true
    default_node = "vh01"
    datastore_id = "<storage>"
  }
}
```

To put only one host's persistent disk there, and leave its other disks where they are:

```hcl
    extra_disks = {
      persist = { interface = "scsi2", size_gb = 20, datastore_id = "<storage>" }
    }
```

The installer ISO stays on the storage `local`, whatever `datastore_id` says. See [The Proxmox servers](../../fleet-bootstrap/concepts/fleet-private.md#servers) for the other keys.

## 6. Run it {#run}

Commit the change from `~/src/fleet-ansible`:

```bash
git -C ../fleet-private add opentofu/prod.tfvars
git -C ../fleet-private commit -m "Put the VM disks on <storage>"
git -C ../fleet-private push
```

The run creates each new VM's disks on the storage its entry now names. See [Adding a host](../../fleet-bootstrap/procedures/add-a-host.md#run) and [The first run](../../fleet-bootstrap/foundation/first-run.md).

Check one VM's drives on the Proxmox host once it exists:

```bash
qm config <vmid> | grep -E '^(scsi|ide0)'
```

Each drive listed starts with `<storage>:`.

## Not yet confirmed {#unconfirmed}

No VM in the fleet has been put on a second drive by these steps.

- That a VM created after step 5 has its disks and its cloud-init drive on `<storage>`.
- The wording of the permission error in step 4.
