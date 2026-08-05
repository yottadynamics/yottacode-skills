# KVM provisioning

Creating VMs, disks, storage pools, networks, and device passthrough.

## virt-install

Comments go above the command — a `#` after a trailing `\` truncates it.

```bash
# --cpu host-model      : migratable across a mixed-generation pool
# --machine q35         : modern chipset, PCIe
# --boot uefi           : required for Secure Boot
# --features smm=on     : required alongside Secure Boot
# --memballoon none     : predictable memory footprint
# --graphics none       : no VNC/SPICE listener at all
# --console pty         : serial console via `virsh console`
virt-install \
  --name app-01 \
  --vcpus 4 \
  --memory 8192 \
  --cpu host-model \
  --machine q35 \
  --boot uefi \
  --features smm=on \
  --memballoon model=none \
  --disk path=/var/lib/libvirt/images/app-01.qcow2,size=100,format=qcow2,bus=virtio,cache=none,io=native,discard=unmap \
  --network bridge=br-internal,model=virtio \
  --os-variant detect=on,name=rhel9.4 \
  --graphics none \
  --console pty,target_type=serial \
  --location /var/lib/libvirt/iso/rhel-9.4-x86_64-dvd.iso \
  --initrd-inject /var/lib/libvirt/kickstarts/app-01.ks \
  --extra-args "inst.ks=file:/app-01.ks console=ttyS0,115200n8" \
  --autostart \
  --noautoconsole
```

`--os-variant` drives the device model defaults. List valid values with
`osinfo-query os`; a stale variant name makes `virt-install` fall back to
generic settings and lose virtio defaults.

Add `--memorybacking hugepages=on` only after allocating hugepages on the host
(see `operations.md`) — the domain will refuse to start otherwise.

## Encrypted disk

```bash
install -m 0400 -o root -g root /dev/null /etc/libvirt/secrets/db-01.pass
# write the passphrase into that file with an editor or a secrets manager

qemu-img create -f qcow2 \
  --object "secret,id=sec0,file=/etc/libvirt/secrets/db-01.pass" \
  -o "encrypt.format=luks,encrypt.key-secret=sec0" \
  /var/lib/libvirt/images/db-01-data.qcow2 200G

qemu-img info \
  --object "secret,id=sec0,file=/etc/libvirt/secrets/db-01.pass" \
  /var/lib/libvirt/images/db-01-data.qcow2
```

`encryption=on` selects the obsolete qcow2 AES format — do not combine it with
`encrypt.format=luks`.

To attach an encrypted disk to a domain, define a libvirt secret so libvirtd
can unlock it, rather than embedding the passphrase in the XML:

```bash
cat > /tmp/secret.xml <<'EOF'
<secret ephemeral='no' private='yes'>
  <description>db-01 data disk</description>
  <usage type='volume'>
    <volume>/var/lib/libvirt/images/db-01-data.qcow2</volume>
  </usage>
</secret>
EOF
UUID=$(virsh secret-define /tmp/secret.xml | awk '{print $2}')
virsh secret-set-value --secret "$UUID" --file /etc/libvirt/secrets/db-01.pass
rm -f /tmp/secret.xml
```

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' io='native'/>
  <source file='/var/lib/libvirt/images/db-01-data.qcow2'>
    <encryption format='luks'>
      <secret type='passphrase' uuid='<uuid-from-above>'/>
    </encryption>
  </source>
  <target dev='vdb' bus='virtio'/>
</disk>
```

A LUKS-encrypted disk requires the passphrase at every VM start. Make sure the
secret survives a host reboot (`ephemeral='no'`) or autostart will fail.

## Cloud-init from a golden image

```bash
# Overlay on the golden image — inherits its virtual size
qemu-img create -f qcow2 \
  -b /var/lib/libvirt/templates/rhel9-golden.qcow2 -F qcow2 \
  /var/lib/libvirt/images/web-01.qcow2

qemu-img resize /var/lib/libvirt/images/web-01.qcow2 80G

cloud-localds /var/lib/libvirt/cloud-init/web-01-cidata.iso \
  /var/lib/libvirt/cloud-init/web-01-user-data.yml \
  /var/lib/libvirt/cloud-init/web-01-meta-data.yml

virt-install \
  --name web-01 \
  --vcpus 2 --memory 4096 \
  --cpu host-model \
  --machine q35 --boot uefi \
  --disk /var/lib/libvirt/images/web-01.qcow2,bus=virtio,cache=none,io=native \
  --disk /var/lib/libvirt/cloud-init/web-01-cidata.iso,device=cdrom \
  --network bridge=br-internal,model=virtio \
  --os-variant rhel9.4 \
  --graphics none \
  --import \
  --autostart \
  --noautoconsole
```

`qemu-img resize` grows the virtual disk only. The guest still needs
`growpart`/`resize2fs` — cloud-init's `growpart` module does this on first boot.

An overlay depends on its backing file forever. Deleting or moving the golden
image destroys every VM built on it. For independent disks, use
`qemu-img convert` instead of `-b`.

### user-data

```yaml
#cloud-config
hostname: web-01
fqdn: web-01.internal.example.com
manage_etc_hosts: true

ssh_pwauth: false
disable_root: true

users:
  - name: ops-admin
    groups: [wheel]
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 AAAA<...> ops-admin@example.com

packages:
  - aide
  - audit
  - qemu-guest-agent

# growpart/resize_rootfs expand the filesystem into the resized disk
growpart:
  mode: auto
  devices: ['/']
resize_rootfs: true

write_files:
  # setenforce is not persistent — write the config file instead
  - path: /etc/selinux/config
    permissions: '0644'
    content: |
      SELINUX=enforcing
      SELINUXTYPE=targeted

runcmd:
  - [systemctl, enable, --now, auditd]
  - [systemctl, enable, --now, qemu-guest-agent]
  - [dnf, -y, --security, update]

final_message: "cloud-init finished after $UPTIME seconds"
```

Do not `dnf remove cloud-init` in `runcmd` — cloud-init is still executing, and
removing it mid-run leaves the instance in an undefined state and breaks the
next boot's datasource handling. If you want it gone, do it in a later
configuration-management pass, or bake the removal into the golden image.

`qemu-guest-agent` is what makes `domfsfreeze` and `domifaddr` work. Install it
in every guest you intend to snapshot or back up.

## Terraform

```hcl
terraform {
  required_providers {
    libvirt = {
      source  = "dmacvicar/libvirt"
      version = "~> 0.8"
    }
  }
}

provider "libvirt" {
  uri = "qemu+ssh://virtadmin@kvm-host-01.example.com/system?keyfile=/home/ci/.ssh/id_ed25519"
}

resource "libvirt_volume" "base" {
  name   = "rhel9-base.qcow2"
  pool   = "production"
  source = "/var/lib/libvirt/templates/rhel9-golden.qcow2"
  format = "qcow2"
}

resource "libvirt_volume" "app_disk" {
  name           = "app-01.qcow2"
  pool           = "production"
  base_volume_id = libvirt_volume.base.id
  size           = 107374182400 # 100 GiB, in bytes
  format         = "qcow2"
}

resource "libvirt_domain" "app" {
  name      = "app-01"
  vcpu      = 4
  memory    = 8192 # MiB
  autostart = true

  cpu {
    mode = "host-model"
  }

  disk {
    volume_id = libvirt_volume.app_disk.id
  }

  network_interface {
    bridge = "br-internal"
  }

  console {
    type        = "pty"
    target_port = "0"
    target_type = "serial"
  }

  # UEFI needs both the firmware and a writable per-domain NVRAM copy.
  # Firmware alone leaves the domain unable to persist boot entries.
  firmware = "/usr/share/edk2/ovmf/OVMF_CODE.secboot.fd"
  nvram {
    file     = "/var/lib/libvirt/qemu/nvram/app-01_VARS.fd"
    template = "/usr/share/edk2/ovmf/OVMF_VARS.secboot.fd"
  }
}
```

Firmware paths are distribution-specific. Check what the host actually offers
with `virsh domcapabilities | grep -A20 '<os '` before hardcoding a path.

Connect as a dedicated account in the `libvirt` group, not `root@`.

## Storage pools

```bash
# Directory pool — simplest
virsh pool-define-as production dir --target /var/lib/libvirt/images/production
virsh pool-build production
virsh pool-autostart production
virsh pool-start production

# LVM pool — better throughput, no per-file overhead
virsh pool-define-as lvm-production logical --source-name vg_vms --target /dev/vg_vms
virsh pool-autostart lvm-production
virsh pool-start lvm-production

virsh pool-list --all --details
```

`pool-build` creates the target directory for a `dir` pool. Skipping it leaves
`pool-start` failing on a path that does not exist.

### Ceph RBD

```bash
# Register the Ceph key as a libvirt secret rather than putting it in pool XML
cat > /tmp/ceph-secret.xml <<'EOF'
<secret ephemeral='no' private='yes'>
  <usage type='ceph'>
    <name>client.libvirt secret</name>
  </usage>
</secret>
EOF
CEPH_UUID=$(virsh secret-define /tmp/ceph-secret.xml | awk '{print $2}')
virsh secret-set-value --secret "$CEPH_UUID" --file /etc/ceph/client.libvirt.key
rm -f /tmp/ceph-secret.xml

cat > /tmp/ceph-pool.xml <<EOF
<pool type='rbd'>
  <name>ceph-production</name>
  <source>
    <host name='ceph-mon-01.example.com' port='6789'/>
    <host name='ceph-mon-02.example.com' port='6789'/>
    <host name='ceph-mon-03.example.com' port='6789'/>
    <name>libvirt-pool</name>
    <auth type='ceph' username='libvirt'>
      <secret uuid='${CEPH_UUID}'/>
    </auth>
  </source>
</pool>
EOF
virsh pool-define /tmp/ceph-pool.xml
virsh pool-autostart ceph-production
virsh pool-start ceph-production
rm -f /tmp/ceph-pool.xml
```

## Networking

### Zone bridges

```bash
nmcli connection add type bridge \
  con-name br-pci ifname br-pci \
  ipv4.method disabled ipv6.method disabled \
  bridge.stp yes bridge.forward-delay 4

nmcli connection add type bridge-slave \
  con-name br-pci-port ifname ens224 master br-pci

nmcli connection up br-pci
```

Giving the bridge no host IP (`ipv4.method disabled`) means the hypervisor is
not reachable on that segment — one fewer path between zones.

### VLAN-aware bridge

```bash
nmcli connection add type bridge \
  con-name br-prod ifname br-prod \
  bridge.vlan-filtering yes \
  bridge.vlan-default-pvid 1 \
  ipv4.method disabled

nmcli connection add type bridge-slave \
  con-name br-prod-trunk ifname ens192 master br-prod \
  bridge-port.vlans "100,200,300"

nmcli connection up br-prod
```

Set the VLAN list through `bridge-port.vlans` so NetworkManager owns it.
`bridge vlan add vid 100 dev ens192` works but is lost at the next reboot.

Attach a guest to a specific VLAN with a portgroup on a libvirt network, or by
putting the tag directly on the interface:

```xml
<interface type='bridge'>
  <source bridge='br-prod'/>
  <vlan>
    <tag id='100'/>
  </vlan>
  <model type='virtio'/>
</interface>
```

### Zone isolation

The `bridge` netfilter family filters traffic inside a single bridge. Traffic
between two bridges on the same host is *routed*, so it is filtered in the
`inet` family:

```nft
# /etc/nftables.d/kvm-zones.nft
table inet zone_isolation {
  chain forward {
    type filter hook forward priority filter; policy accept;
    iifname "br-pci"      oifname != "br-pci"   drop
    iifname != "br-pci"   oifname "br-pci"      drop
    iifname "br-hipaa"    oifname != "br-hipaa" drop
    iifname != "br-hipaa" oifname "br-hipaa"    drop
  }
}
```

```bash
nft -c -f /etc/nftables.d/kvm-zones.nft    # syntax check first
systemctl reload nftables
nft list table inet zone_isolation
```

Best of all, leave `net.ipv4.ip_forward` off on a hypervisor that only bridges.
With forwarding disabled there is no routed path between zones to filter.

## Device passthrough

```bash
# 1. IOMMU on the kernel command line, then reboot
#    Intel: intel_iommu=on iommu=pt
#    AMD:   amd_iommu=on iommu=pt
grubby --update-kernel=ALL --args="intel_iommu=on iommu=pt"

# 2. Find the device and its vendor:device IDs
lspci -nn | grep -i nvidia
# 41:00.0 3D controller [0302]: NVIDIA Corporation ... [10de:20b2]

# 3. The whole IOMMU group must pass through together
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/}; n=${n%%/*}
  printf 'group %s: %s\n' "$n" "$(lspci -nns "${d##*/}")"
done | grep -E 'group [0-9]+:.*41:00'

# 4. Bind to vfio-pci at boot (include the audio function if present)
echo 'options vfio-pci ids=10de:20b2,10de:1db1' > /etc/modprobe.d/vfio.conf

# 5. Ensure vfio loads before the vendor driver
echo 'add_drivers+=" vfio vfio_iommu_type1 vfio_pci "' > /etc/dracut.conf.d/vfio.conf
dracut -f --regenerate-all

# 6. Reboot, then confirm
lspci -nnk -s 41:00.0 | grep -i 'kernel driver'
# expect: Kernel driver in use: vfio-pci
```

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x41' slot='0x00' function='0x0'/>
  </source>
</hostdev>
```

Notes:

- A VM with a passthrough device cannot be live-migrated.
- Passthrough pins the guest's entire memory (no ballooning, no overcommit) because the device can DMA to it at any time.
- `<kvm><hidden state='on'/></kvm>` was a workaround for the NVIDIA "Error 43" on consumer GeForce cards. NVIDIA removed that restriction in 2021, and it never applied to datacenter cards. Do not add it by default.
