# CoreOS simple setup for a btrfs time-machine

The aim of this repo is to have a raspberry pi create bi-weekly snapshots of a distant BTRFS subvolume,
for a nice off site backup.

## Prerequisites

### SSH setup

If you want to get the snapshots from a distant sources over SSH some setup is required:

- Get the fingerprint of the remote host : `ssh-keyscan $hostname >> known_hosts`. Note that this could be done in the ignition automatically but blindly accepting the fingerprint is a security risk, do it beforehand and verify it.
- Create an ssh key : `ssh-keygen -t ecdsa -f id-ecdsa`
- Upload the public key to the remote host to backup: `ssh-copy-id root@$hostname -i id-ecdsa`

### Build the ignition config

If you have butane installed :  `butane --pretty --strict --files-dir . config.bu > config.ign`

Or with an ephemeral container: `podman run --rm -v .:/config/:z quay.io/coreos/butane:release --pretty --strict --files-dir=/config /config/config.bu > transpiled_config.ign`

### Disk preparation

The ignition config expects a btrfs partition with a label `backup`.
Exemple from a fedora coreOS liveISO :
```
sudo fdisk /dev/sdX # create the partition
mkfs.btrfs /dev/sdXN

# create a mount point then mount the disk
mkdir /var/backup
mount /dev/sdXN /var/backup
mkfs.btrfs /dev/sdb_vg
# btrbk.conf target subvolume needs to exist
btrfs subvolume create /var/backup/mysubvol
```
For the raspberry-pi u-boot setup see below, as the disk partitioning setup is different.


## Testing

Using [cosa](https://coreos.github.io/coreos-assembler/): `cosa run -i config.ign -D "1G:serial=backup"`
If so, make sure to add the following butane bit under `storage` to format the drive:
```
  filesystems:
    - device: /dev/disk/by-id/virtio-backup
      format: btrfs
      wipe_filesystem: true
      label: backup

```

## Installation

Here we boot the raspberry-pi from a usb disk.\
Transpile the config (see above), then
install coreOS to the second partition of the disk we just created:
```
sudo coreos-installer install -s stable -i config.ign backup /dev/sdX
```
Follow the steps to install u-boot firmware: https://docs.fedoraproject.org/en-US/fedora-coreos/provisioning-raspberry-pi4/#_installing_fcos_and_booting_via_u_boot

For a raspberry-pi 3 some fixing steps are needed : https://discussion.fedoraproject.org/t/fcos-unusually-slow-on-raspberry-pi-3b/40635/16

GDisk will mention some free space, say yes, then create the btrfs partition there with a label `backup`. I left some space before
so the xfs root will be grown to ~20 GB

Be aware that the table partition must be converted to hybrid MBR once again after the first boot. I could do this from the running
coreOS system.
