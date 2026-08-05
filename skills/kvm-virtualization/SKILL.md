---
name: kvm-virtualization
description: Provision, harden, and operate virtual machines on Linux KVM/QEMU/libvirt — domain XML, disk images, bridged and VLAN networking, snapshots, backups, live migration, and device passthrough. Use when creating or tuning a VM with virt-install or virsh, working with qemu-img, debugging a VM that will not start or migrate, setting up storage pools, or hardening a hypervisor.
license: MIT
metadata:
  author: yottacode
---

# KVM virtualization

libvirt fails quietly: a VM boots without the isolation you thought you configured, a snapshot is crash-consistent instead of application-consistent, a migration succeeds and leaves the disk behind. Verify each step against the running host rather than the XML you wrote.

## 1. Confirm the host can do what you are about to ask

```bash
grep -Ec '(vmx|svm)' /proc/cpuinfo          # >0 means hardware virt is present
lsmod | grep kvm                            # kvm_intel or kvm_amd, plus kvm
systemctl is-active virtqemud libvirtd      # modular or monolithic daemon
virsh version

# Only needed for device passthrough
grep -oE '(intel|amd)_iommu=on' /proc/cmdline
```

Hardware virtualization present in `/proc/cpuinfo` but no `/dev/kvm` usually means it is disabled in firmware, not missing.

## 2. Pick the disk format and cache mode deliberately

| Format | Snapshots | Thin | Encryption | Use for |
|---|---|---|---|---|
| qcow2 | internal + external | yes | LUKS built in | default for most workloads |
| raw | none | no | LUKS via block layer | high-IOPS databases, latency-sensitive |
| LVM LV | LVM snapshots | thin pools | LUKS via block layer | large fleets, iSCSI/Ceph backends |

`cache=none` with `io=native` is the production default: it bypasses the host page cache, so a host crash cannot lose writes the guest believes are durable. `cache=writeback` is faster and unsafe without a battery-backed controller.

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' io='native' discard='unmap'/>
  <source file='/var/lib/libvirt/images/app-01.qcow2'/>
  <target dev='vda' bus='virtio'/>
</disk>
```

`io=native` requires `cache=none`; pairing it with a caching mode silently falls back to the thread pool.

## 3. Choose the CPU model for the fleet, not the host

This is the decision that determines whether you can ever migrate.

- **`host-model`** — the default for anything in a pool. libvirt resolves it to a named model plus features at start time, so a VM can move to any host with an equal-or-newer CPU.
- **`host-passthrough`** — exposes the physical CPU exactly. Best performance and gets microcode-level features through to the guest, but the destination host must be effectively identical. Use for pinned, single-host workloads.

```xml
<cpu mode='host-model' check='partial'/>
```

Committing to `host-passthrough` on a mixed-generation cluster produces VMs that cannot be evacuated for maintenance. Decide once, per pool, and record it.

## 4. Write commands that survive being copied

libvirt commands are long and get commented. A `#` after a trailing backslash **ends the line** — the backslash escapes the space, not the newline. The remaining flags are silently dropped and then run as commands.

```bash
# WRONG — runs `virsh migrate --live` with no destination, then errors on the rest
virsh migrate \
  --live \                  # migrate while running
  --persistent \            # define on destination
  app-01 qemu+tls://host-02.example.com/system

# RIGHT — comments above the command
# --live:       migrate while the VM keeps running
# --persistent: define the domain on the destination
virsh migrate \
  --live \
  --persistent \
  app-01 qemu+tls://host-02.example.com/system
```

This matters most for `virt-install` and `virsh migrate`, where the dropped flags are the security-relevant ones.

## 5. Keep passphrases off the command line

Anything in `argv` is visible in `ps` to every user on the host. `qemu-img` takes a secret from a file:

```bash
install -m 0400 -o root -g root /dev/null /etc/libvirt/secrets/disk.pass
# write the passphrase into that file, then:

qemu-img create -f qcow2 \
  --object "secret,id=sec0,file=/etc/libvirt/secrets/disk.pass" \
  -o "encrypt.format=luks,encrypt.key-secret=sec0" \
  /var/lib/libvirt/images/db-01-data.qcow2 200G

qemu-img info \
  --object "secret,id=sec0,file=/etc/libvirt/secrets/disk.pass" \
  /var/lib/libvirt/images/db-01-data.qcow2
# expect: encrypted: yes, and an encrypt.format of luks
```

Do not add `encryption=on` — that selects the obsolete qcow2 AES format and conflicts with `encrypt.format=luks`.

## 6. Freeze before snapshotting, and know what you can revert

`virsh snapshot-create-as --disk-only` produces an **external** snapshot. libvirt cannot `snapshot-revert` those on most versions — the command errors out. External snapshots are a backup mechanism, not an undo button.

```bash
virsh domfsfreeze db-01                     # needs qemu-guest-agent in the guest
virsh snapshot-create-as db-01 \
  --name "pre-upgrade-$(date +%Y%m%d-%H%M%S)" \
  --disk-only --atomic --quiesce
virsh domfsthaw db-01
```

Without the freeze you get a crash-consistent image: it will usually boot, and a database on it will usually need recovery.

- **To roll back**, use an internal snapshot (`virsh snapshot-create-as` with no `--disk-only`, qcow2 only, VM should be off) or restore from backup.
- **To discard an external snapshot** after copying the base, merge the overlay back down: `virsh blockcommit db-01 vda --active --pivot --verbose`.

Always `domfsthaw` in a trap. A script that dies between freeze and thaw leaves the guest filesystem locked and the VM effectively hung — see `references/operations.md` for the backup script that does this correctly.

## 7. Isolate networks at the right layer

Separate bridges per zone give you separation only if nothing routes between them. The `bridge` netfilter family sees traffic **within** one bridge; traffic between two bridges on the same host is routed, so it is filtered in the `inet` family's `forward` hook.

```nft
# /etc/nftables.d/kvm-zones.nft — blocks routing between zone bridges
table inet zone_isolation {
  chain forward {
    type filter hook forward priority filter; policy accept;
    iifname "br-pci"    oifname != "br-pci"    drop
    iifname != "br-pci" oifname "br-pci"       drop
    iifname "br-hipaa"  oifname != "br-hipaa"  drop
    iifname != "br-hipaa" oifname "br-hipaa"   drop
  }
}
```

Confirm the rules load and match: `nft -c -f /etc/nftables.d/kvm-zones.nft` then `nft list ruleset | grep -A5 zone_isolation`. Stronger still is not enabling `net.ipv4.ip_forward` on the hypervisor at all when zones must never talk.

Avoid the default NAT network (`virbr0`) for anything but a workstation — it blocks inbound connections and gives no real isolation.

## 8. Verify sVirt is actually labelling

sVirt gives each QEMU process a unique MCS category, so a guest escape cannot reach another guest's disk.

```bash
ps -eZ | grep qemu | head -3
# expect distinct categories per VM: system_u:system_r:svirt_t:s0:c123,c456

grep -E '^\s*security_driver' /etc/libvirt/qemu.conf
# must be "selinux", or absent (which defaults to selinux)

ls -lZ /var/lib/libvirt/images/     # expect svirt_image_t
restorecon -Rv /var/lib/libvirt/images/   # if labels drifted
```

Identical MCS categories on two running VMs means labelling is off, even if SELinux reports enforcing.

## 9. Validate a migration before trusting it

```bash
virsh domstate app-01                       # on the destination
virsh domblklist app-01                     # storage resolved?
virsh domifaddr app-01                      # guest agent reports addresses?
ps -eZ | grep -- '-name guest=app-01'       # sVirt relabelled on the new host?
journalctl -u virtqemud --since '10 min ago' | grep -i migrat
```

Prerequisites: shared storage (or `--copy-storage-all`), compatible CPU models, matching bridge names on both hosts, and libvirt/QEMU versions where the destination is not older than the source. Use `--tls` — plain migration sends guest RAM over the network unencrypted.

## Output format for hypervisor reviews

```markdown
## KVM review

- Verdict: ready / needs changes
- Isolation: <sVirt labels, security_driver, zone bridging>
- Storage: <format, cache mode, encryption, pool backing>
- CPU/memory: <model choice, pinning, NUMA, hugepages>
- Migration: <CPU compatibility, TLS, shared storage>
- Backup: <freeze/thaw, consistency, restore tested>

## Required changes
- `path/to/domain.xml` or command — <what to change and why>
```

## Pitfalls

1. **A `#` comment after a trailing `\`** — silently truncates the command and runs the rest as shell commands.
2. **Passphrases in `argv`** — visible in `ps`. Use `--object secret,...,file=`.
3. **`snapshot-revert` on a `--disk-only` snapshot** — unsupported; you need an internal snapshot or a restore.
4. **Snapshot without `domfsfreeze`** — crash-consistent only; databases may need recovery on boot.
5. **Freeze with no `trap` to thaw** — a failed script leaves the guest filesystem locked.
6. **`security_driver = "none"`** — removes all inter-VM isolation. Fix labels with `restorecon` instead.
7. **`host-passthrough` on a mixed cluster** — the VM can never be migrated off for maintenance.
8. **`cache=writeback` without battery-backed cache** — silent data loss on host power failure.
9. **Passthrough without IOMMU** — the device can DMA into arbitrary host memory.
10. **`bridge vlan add` and `ip` commands for persistent config** — they vanish on reboot. Configure through NetworkManager or your distribution's network config.
11. **`virsh destroy` as routine shutdown** — it is a power cut; use `virsh shutdown`.
12. **KSM left on for untrusted tenants** — page deduplication across guests is a known side channel.

## References

- `references/provisioning.md` — virt-install, cloud-init, Terraform, storage pools, passthrough.
- `references/operations.md` — backup and restore scripts, migration, monitoring, performance tuning.
- `references/compliance.md` — hardening baseline, audit rules, encryption, control mappings.
- [libvirt documentation](https://libvirt.org/docs.html)
- [libvirt domain XML format](https://libvirt.org/formatdomain.html)
- [QEMU security documentation](https://www.qemu.org/docs/master/system/security.html)
- [NIST SP 800-125A Rev 1 — Hypervisor platform security](https://csrc.nist.gov/publications/detail/sp/800-125a/rev-1/final)
