# Logical Volume Management (LVM2)

## Create a new volume group (VG) and a Logical Volume (LV)

- First, ask the virtualization team to add a new disk with the desired size, for instance, 100GB
- It should be present in the block list via lsblk. If it's not present in the list, `rescan` the  SCSI devices

  ```bash
  for host in /sys/class/scsi_host/host*/scan; do
    echo "Rescanning $host"
    echo "- - -" | tee "$host"
  done
  ```

### Create partition

Lets say the scsi disk, is `/dev/sdd`

```bash
gdisk /dev/sdd
```

```txt
GPT fdisk (gdisk) version 1.0.8

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries in memory.

Command (? for help):
```

If there is need to help, type `?` , hit enter, otherwise to create a new partition, type the `n`.

```txt
n       add a new partition
```

Follow the instruction and after all , write the changes with `w`.

```txt
w       write table to disk and exit
```

### Create the PV

Create the PV from disk

```bash
pvcreate /dev/sdd1
```

### Create the VG

Create the volume group from the pv which created before step. for instance the new vg name is `vg-k8s-storage`

```bash
vgcreate vg-k8s-storage /dev/sdd1
```

### Create the LV

Create the logical volume from the volume group, with desired size , if there is no specific size needed, so create it with the vgs `100%FREE` size, for instance the LV name is `lv-k8s-storage`

```bash
lvcreate -l +100%FREE --name lv-k8s-storage vg-k8s-storage
```

Set the desired filesystem to the lv, for instance we need the `ext4`.

```bash
mkfs.ext4 /dev/mapper/vg--k8s--storage-lv--k8s--storage
```

Resize with `resize2fs` utility , this utility is for `ext4` filesystem.

```bash
resize2fs /dev/mapper/vg--k8s--storage-lv--k8s--storage
```

## Add to `fstab` and mount it

Create a target directory to mount the lv.

```bash
mkdir /srv/data.vol
```

Add to `fstab`, edit fstab file with `vim /etc/fstab` and append below to the file.

```txt
/dev/vg-k8s-storage/lv-k8s-storage /srv/data.vol ext4    defaults      0 2
```

[Man `fstab`][man-fstab]

```txt
The fifth field (fs_freq).
       This field is used by dump(8) to determine which filesystems need
       to be dumped. Defaults to zero (don’t dump) if not present.

The sixth field (fs_passno).
    This field is used by fsck(8) to determine the order in which
    filesystem checks are done at boot time. The root filesystem
    should be specified with a fs_passno of 1. Other filesystems
    should have a fs_passno of 2. Filesystems within a drive will be
    checked sequentially, but filesystems on different drives will be
    checked at the same time to utilize parallelism available in the
    hardware. Defaults to zero (don’t check the filesystem) if not
    present.
```

After the editing file you need to invoke `mount --all --verbose` to mount all filesystems mentioned in fstab.

```txt
-a, --all               mount all filesystems mentioned in fstab
-v, --verbose           say what is being done
```

[man-fstab]: https://man7.org/linux/man-pages/man5/fstab.5.html
