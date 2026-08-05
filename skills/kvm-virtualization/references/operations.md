# KVM operations

Backup and restore, live migration, monitoring, resource control, and
performance tuning.

## Backup

The dangerous part of a live backup is the window between `domfsfreeze` and
`domfsthaw`. A script that exits in that window leaves the guest filesystem
locked and the VM unusable until someone notices. Thaw from a trap, not from
the happy path.

```bash
#!/usr/bin/env bash
# Live VM backup: quiesced external snapshot, copy base, commit overlay back.
set -euo pipefail

VM_NAME="${1:?usage: backup-vm <domain>}"
BACKUP_ROOT="/backup/vms/${VM_NAME}"
STAMP="$(date +%Y%m%d-%H%M%S)"
WORK_DIR="${BACKUP_ROOT}/${STAMP}"
PASS_FILE="/etc/libvirt/secrets/backup.pass"
SNAP_NAME="backup-${STAMP}"

frozen=0
snapshotted=0

cleanup() {
  # Thaw first and unconditionally — a frozen guest is an outage.
  if [ "${frozen}" -eq 1 ]; then
    virsh domfsthaw "${VM_NAME}" || echo "WARNING: thaw failed for ${VM_NAME}" >&2
    frozen=0
  fi
  # Merge the overlay back so the domain stops running on a snapshot chain.
  if [ "${snapshotted}" -eq 1 ]; then
    virsh blockcommit "${VM_NAME}" "${TARGET_DEV}" --active --pivot --verbose \
      || echo "WARNING: blockcommit failed for ${VM_NAME}; overlay still in use" >&2
    snapshotted=0
  fi
}
trap cleanup EXIT INT TERM

mkdir -p "${WORK_DIR}"
chmod 0700 "${WORK_DIR}"

# Domain definition — without this the disk image is not a restorable backup.
virsh dumpxml "${VM_NAME}" > "${WORK_DIR}/${VM_NAME}.xml"

# First writable disk: columns are Type Device Target Source.
read -r TARGET_DEV DISK_PATH < <(
  virsh domblklist "${VM_NAME}" --details \
    | awk '$1=="file" && $2=="disk" {print $3, $4; exit}'
)
[ -n "${DISK_PATH:-}" ] || { echo "no file-backed disk found" >&2; exit 1; }

virsh domfsfreeze "${VM_NAME}"
frozen=1

virsh snapshot-create-as "${VM_NAME}" \
  --name "${SNAP_NAME}" \
  --disk-only --atomic --quiesce --no-metadata
snapshotted=1

virsh domfsthaw "${VM_NAME}"
frozen=0

# The base is read-only now; new writes land in the overlay.
cp --sparse=always "${DISK_PATH}" "${WORK_DIR}/"

# NVRAM, for UEFI domains
NVRAM_PATH="$(virsh dumpxml "${VM_NAME}" \
  | sed -n 's|.*<nvram[^>]*>\(.*\)</nvram>.*|\1|p' | head -1)"
if [ -n "${NVRAM_PATH}" ] && [ -f "${NVRAM_PATH}" ]; then
  cp "${NVRAM_PATH}" "${WORK_DIR}/"
fi

# Encrypt, then remove the plaintext copy. Leaving both defeats the exercise.
tar -C "${WORK_DIR}" -czf - . \
  | openssl enc -aes-256-cbc -salt -pbkdf2 -iter 600000 \
      -pass "file:${PASS_FILE}" \
      -out "${WORK_DIR}.tar.gz.enc"

sha256sum "${WORK_DIR}.tar.gz.enc" > "${WORK_DIR}.tar.gz.enc.sha256"
rm -rf "${WORK_DIR}"

echo "Backup complete: ${WORK_DIR}.tar.gz.enc"
```

`openssl enc` gives confidentiality but no authentication — a corrupted or
tampered archive fails at decrypt time with garbage rather than a clear error.
The `sha256sum` covers accidental corruption. Where integrity matters against a
motivated attacker, sign the archive or use `age`/`gpg` instead.

Restore path, and test it on a schedule:

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -iter 600000 \
  -pass "file:/etc/libvirt/secrets/backup.pass" \
  -in /backup/vms/app-01/<stamp>.tar.gz.enc \
  | tar -C /var/lib/libvirt/restore -xzf -

virsh define /var/lib/libvirt/restore/app-01.xml
virsh start app-01
```

## Snapshots

```bash
# External, quiesced — a backup mechanism
virsh domfsfreeze db-01
virsh snapshot-create-as db-01 --name "pre-upgrade-$(date +%Y%m%d)" \
  --disk-only --atomic --quiesce
virsh domfsthaw db-01

virsh snapshot-list db-01 --tree

# Discard an external snapshot: merge the overlay into the base
virsh blockcommit db-01 vda --active --pivot --verbose
```

`snapshot-revert` does not work on external (`--disk-only`) snapshots on most
libvirt versions — it errors rather than reverting. For an undo button you need
an **internal** snapshot, which is qcow2-only and best taken with the VM shut
off:

```bash
virsh shutdown db-01
virsh snapshot-create-as db-01 --name pre-upgrade --description "before v5.2"
virsh start db-01

# roll back
virsh shutdown db-01
virsh snapshot-revert db-01 --snapshotname pre-upgrade
virsh start db-01
```

Internal snapshots grow the qcow2 file and slow it down as the chain deepens.
Delete them once the change is confirmed: `virsh snapshot-delete db-01 --snapshotname pre-upgrade`.

## Live migration

Prerequisites:

1. Shared storage (NFS/Ceph/iSCSI), or `--copy-storage-all`
2. Compatible CPU models — see the `host-model` vs `host-passthrough` decision in `SKILL.md`
3. Matching bridge and network names on the destination
4. Destination libvirt/QEMU not older than the source
5. TLS certificates in place on both hosts for `--tls`

```bash
# CA:     /etc/pki/CA/cacert.pem
# server: /etc/pki/libvirt/servercert.pem
#         /etc/pki/libvirt/private/serverkey.pem
# client: /etc/pki/libvirt/clientcert.pem
#         /etc/pki/libvirt/private/clientkey.pem

# --tls encrypts the migration stream; without it guest RAM crosses the
# network in plaintext, including anything in page cache.
virsh migrate \
  --live \
  --persistent \
  --undefinesource \
  --verbose \
  --timeout 300 \
  --tls \
  app-01 qemu+tls://kvm-host-02.example.com/system
```

`--timeout 300` suspends the guest after 5 minutes to force convergence — a
busy VM that dirties memory faster than the link can copy will otherwise
migrate forever. `--auto-converge` throttles the guest CPU instead, which is
gentler on latency-sensitive workloads.

Without shared storage:

```bash
virsh migrate \
  --live \
  --persistent \
  --undefinesource \
  --copy-storage-all \
  --verbose \
  --timeout 600 \
  --tls \
  app-01 qemu+tls://kvm-host-02.example.com/system
```

The destination needs pre-created disk images of the right size —
`--copy-storage-all` copies content, not allocation.

### Post-migration validation

```bash
virsh domstate app-01
virsh domblklist app-01
virsh domifaddr app-01
ps -eZ | grep -- '-name guest=app-01'    # sVirt relabelled on the new host
journalctl -u virtqemud --since '10 min ago' | grep -i migrat
```

A VM with a passthrough device or a host-passthrough CPU on a mismatched
destination cannot migrate. Check before scheduling maintenance, not during it.

## Resource control

```bash
# CPU cap: quota/period. 400000/100000 = 400% = four cores' worth.
# --config makes it survive a restart; --live applies immediately.
virsh schedinfo app-01 --config --live \
  --set vcpu_quota=400000 --set vcpu_period=100000

# Pin vCPUs to physical cores on one NUMA node
for i in 0 1 2 3; do
  virsh vcpupin app-01 "$i" "$((i + 2))" --config --live
done

# Memory ceiling in KiB
virsh memtune app-01 --config --live \
  --hard-limit 8388608 \
  --soft-limit 7340032

# Per-device I/O ceiling
virsh blkdeviotune app-01 vda --config --live \
  --total-iops-sec 5000 \
  --total-bytes-sec 209715200
```

Omitting `--config` applies the change to the running domain only; it is lost
on the next start. Omitting `--live` defers it to the next start.

`hard-limit` is a hard memory cap enforced by cgroups — the guest is OOM-killed
if it exceeds it, including QEMU's own overhead. Set it above the guest's
configured RAM (typically +10–20%) or you will kill healthy VMs.

`virsh blkiotune --weight` maps to cgroup v1 blkio weighting and is a no-op on
cgroup v2 hosts with the default `io` controller. Use `blkdeviotune` limits
instead, which are enforced by QEMU rather than the kernel.

## Monitoring

```bash
virt-top                                  # live per-VM resource view
virsh domstats app-01                     # all counters in one call
virsh domblkstat app-01 vda
virsh domifstat app-01 vnet0
virsh cpu-stats app-01 --total
```

Lifecycle events, for an audit trail of who started and stopped what:

```bash
virsh event --all --loop --timestamp
```

Run it under systemd rather than backgrounding it from a shell, so it restarts
with the host:

```ini
# /etc/systemd/system/libvirt-events.service
[Unit]
Description=Record libvirt lifecycle events
After=virtqemud.service

[Service]
ExecStart=/usr/bin/virsh event --all --loop --timestamp
StandardOutput=append:/var/log/libvirt/lifecycle-events.log
Restart=always

[Install]
WantedBy=multi-user.target
```

For Prometheus, `libvirt-exporter` exposes per-domain CPU, memory, block, and
interface counters on `:9177/metrics`. Alert on domains that stop unexpectedly
(`libvirt_domain_info_state` leaving 1) and on block-device latency, not just
utilisation.

## Performance tuning

### CPU and NUMA

```xml
<vcpu placement='static'>4</vcpu>

<cputune>
  <vcpupin vcpu='0' cpuset='2'/>
  <vcpupin vcpu='1' cpuset='3'/>
  <vcpupin vcpu='2' cpuset='4'/>
  <vcpupin vcpu='3' cpuset='5'/>
  <!-- Keep the emulator (I/O) threads off the vCPU cores -->
  <emulatorpin cpuset='0-1'/>
</cputune>

<numatune>
  <memory mode='strict' nodeset='0'/>
</numatune>

<cpu mode='host-model' check='partial'>
  <topology sockets='1' dies='1' cores='4' threads='1'/>
</cpu>
```

Check the host's topology before choosing a cpuset — `virsh nodeinfo` and
`lscpu -e` show which cores belong to which NUMA node. Pinning vCPUs to node 0
while `numatune` points at node 1 is worse than no pinning at all.

`mode='strict'` fails the VM start rather than falling back to remote memory.
That is usually what you want for a latency-sensitive workload, and usually not
what you want for a general-purpose pool.

### Memory

```xml
<memoryBacking>
  <hugepages>
    <page size='2048' unit='KiB'/>
  </hugepages>
  <!-- Exclude this guest's pages from KSM deduplication -->
  <nosharepages/>
  <!-- Never swap this guest's memory to disk -->
  <locked/>
</memoryBacking>
```

`<locked/>` requires a `hard-limit` in `memtune` (or an unlimited memlock
rlimit for QEMU), otherwise the domain fails to start.

```bash
# Allocate 2 MiB hugepages: 4096 x 2 MiB = 8 GiB
echo 4096 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
echo 'vm.nr_hugepages = 4096' > /etc/sysctl.d/90-hugepages.conf

grep -i huge /proc/meminfo
```

Allocate hugepages early in the host's uptime. Once memory is fragmented the
kernel may not be able to satisfy the request, and a VM configured for
hugepages will refuse to start. 1 GiB pages must be reserved on the kernel
command line (`hugepagesz=1G hugepages=16`) — they cannot be allocated at
runtime.

### Storage and network

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' io='native' discard='unmap'/>
  <source file='/var/lib/libvirt/images/app-01.qcow2'/>
  <target dev='vda' bus='virtio'/>
</disk>

<interface type='bridge'>
  <source bridge='br-internal'/>
  <model type='virtio'/>
  <!-- Multiqueue: match the vCPU count so RSS spreads across them -->
  <driver name='vhost' queues='4'/>
</interface>
```

Multiqueue only helps when the guest enables it too —
`ethtool -L eth0 combined 4` inside the VM. Without that the extra queues sit
idle and you pay the memory cost for nothing.

## Command reference

| Command | Purpose | Notes |
|---|---|---|
| `virsh list --all` | All domains | `--state-running`, `--state-shutoff` |
| `virsh dominfo <vm>` | State, memory, CPU | |
| `virsh domstats <vm>` | All counters | `--block`, `--net`, `--cpu-total` |
| `virsh shutdown <vm>` | Graceful ACPI shutdown | needs ACPI in the guest |
| `virsh destroy <vm>` | Force stop | power cut — emergency only |
| `virsh dumpxml <vm>` | Export definition | `--inactive` for persistent config |
| `virsh define <xml>` | Define without starting | |
| `virsh undefine <vm>` | Remove definition | `--remove-all-storage`, `--nvram` |
| `virsh domblklist <vm>` | Disks | `--details` adds type and target |
| `virsh domfsfreeze/thaw` | Quiesce the guest FS | needs qemu-guest-agent |
| `virsh blockcommit <vm> <dev>` | Merge overlay into base | `--active --pivot` |
| `virsh migrate --live` | Live migration | `--tls`, `--copy-storage-all` |
| `virsh schedinfo/memtune/blkdeviotune` | Resource limits | add `--config --live` |
| `virsh nodeinfo` | Host CPU/NUMA topology | |
| `qemu-img info` | Image format, backing file | add `--object secret` for LUKS |
| `qemu-img convert` | Change format, flatten overlays | `-O qcow2` |
| `qemu-img check` | Verify image consistency | run on a stopped domain |
