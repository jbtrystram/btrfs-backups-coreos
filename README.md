# CoreOS simple setup for a btrfs time-machine

The aim of this repo is to have a raspberry pi create bi-weekly snapshots of a distant BTRFS subvolume,
for a nice off site backup.

## Prerequisites

### SSH setup

If you want to get the snapshots from a distant sources over SSH some setup is required:

- Get the fingerprint of the remote host : `ssh-keyscan $hostname >> known_hosts`. Note that this could be done in the ignition automatically but blindly accepting the fingerprint is a security risk, do it beforehand and verify it.
- Create an ssh key : `ssh-keygen -t ecdsa -f id-ecdsa`
- Upload the public key to the remote host to backup: `ssh-copy-id root@$hostname -i id-ecdsa`

### Build the ignitionc config

If you have butane installed :  `butane --pretty --strict --files-dir . config.bu > config.ign`

Or with an ephemeral container: `podman run --rm -v .:/config/:z quay.io/coreos/butane:release --pretty --strict --files-dir=/config /config/config.bu > transpiled_config.ign`

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