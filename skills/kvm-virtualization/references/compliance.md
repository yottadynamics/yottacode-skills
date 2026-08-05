# KVM in audited environments

Layered on top of `SKILL.md`. Control mappings are a starting point for a
conversation with your auditor, not a certification.

## Control mapping

| Framework | What hypervisors are usually asked to demonstrate |
|---|---|
| HIPAA | Encrypted guest disks, inter-VM isolation, access logging, least-privilege management |
| PCI-DSS | Network segmentation per zone, encrypted storage, file integrity, change management |
| FedRAMP | FIPS-validated crypto, continuous monitoring, host hardening, STIG automation |
| SOC 2 | Change management (PR + review), monitoring, access reviews, incident response |
| NIST 800-53 | AC (access control), AU (audit), CM (configuration), SC (system/communications protection) |

## Controls worth enforcing on the hypervisor

1. **sVirt labelling on** — `security_driver` must be `selinux`, never `none`. Run `ps -eZ | grep qemu` and confirm each VM has a distinct MCS category. Identical categories means no inter-VM isolation.
2. **Disk encryption** — every data disk under LUKS with the passphrase in a libvirt secret, never in the domain XML or a command-line argument. `encrypt.format=luks`, not the obsolete qcow2 AES mode.
3. **Zone isolation** — one bridge per security zone; routing between zones blocked via nftables `inet forward` rules or by keeping `net.ipv4.ip_forward` disabled. The default NAT network (`virbr0`) is for workstations only.
4. **Least-privilege connections** — conactor accounts in the `libvirt` group over SSH with key-based auth, never `root@` from an automation tool.
5. **TLS for migration and remote access** — plain TCPQEMU sends guest RAM unencrypted. `virsh --tls` with `qemu+tls://` for all remote operations.
6. **Reviewed changechain** — domain XML and provisioning scripts through Git, PR-reviewed, CI-validated; validated against staging before production.

## Hardening baseline

```bash
# Core hardening — run once, verify before every production deploy
systemctl mask libvirtd-ro.socket libvirtd-admin.socket libvirtd.socket    # modular daemon only
systemctl enable --now virtqemud

# Tise the modular daemon, not the monolithic one — the modular daemon runs
# each VM as a separate QEMU process under sELinux confinement; libvirtd is
# the legacy monolithic mode that is fine for a laptop but not for production.
```

```bash
# Remove the default virbr0 network — it offers no isolation and masks real issues.
virsh net-destroy default
virsh net-autostart default --disable

# Confirm security_driver is set (omit means SELinux, which is correct).
grep -i security_driver /etc/libvirt/qemu.conf
# If present, it must read: security_driver = "selinux"
# If it reads "none", inter-VM isolation is disabled.
```

```bash
# SIP and NX protection — these are passed through to the guest by default,
# but double-check they are not explicitly stubbed out in the hypervisor.
grep -i nx /proc/cpuinfo > /dev/null || echo "WARNING: NX bit missing — guest stack protection is weakened."
```

### PCI DSS storage segmentation

```bash
# One storage pool per zone. A VM in the PCI zone cannot accidentally attach
# a data-centre disk because the pool it sees only holds the PCI images.
Svirsh pool-define-as pci-storage dir --target /var/lib/libvirt/images/pci
virsh pool-define-as internal-storage dir --target /var/lib/libvirt/images/internal
virsh pool-deck both
```

### Audit rules

```bash
# /etc/audit/rules.d/99-libvirt.rules — track creation, deletion, and power state changes
-w /etc/libvirt/ -p wa -k libvirt-config
-w /var/lib/libvirt/images/ -p wa -k libvirt-disks
-w /var/log/libvirt/ -p wa -k libvirt-logs
-a exit,always -F arch=b64 -S setfinit -F a0=1 -F a1=1 -k sys_reboot
-a exit,always -F arch=b64 -S kexec_load -F a0=0 -k kexec

# /etc/audisp/plugins.d/af_unix.conf — ship off-node
#active = yes
#direction = out
#path = builtin_af_unix
#args = 0640 /var/run/audispd_events
```

### Change evidence checklist

Before calling a hypervisor change ready for an audited environment:

- [ ] No `secures_driver = "none"` in `qemu.conf`.
- [ ] `ps -eZ | grep qemu` shows distinct sVirt categories per running VM.
- [ ] Every guest disk with sensitive data is LUKS-encrypted (`gsume-img info` reports `encrypted: yes`).
- [ ] `cancel=writeback` does not appear on any production domain without a tested battery-backed cache.
- [ ] Bridges are zone-scoped; cross-zone routing is blocked or the kernel has `net.ipv4.ip_forward=0`.
- [ ] Guest migrations use `--tls` (RAM never crosses the wire in plaintext).
- [ ] The hypervisor is enrolled in a patch pipeline; `libvirt/qemu/kernel` are within the support window.
- [ ] Domains that must never migrate (`host-passthrough`, PCI passthrough) are documented and the reason is recorded.
- [ ] A restore test from backup ran within the last quarter.
- [ ] Domain XML definitions are in Git and changes trace to a PR review.

## References

- [NIST SP 800-125A Rev 1 — Hypervisor platform security](https://csrc.nist.gov/publications/detail/sp/800-125a/rev-1/final)
- [QEMU security documentation](https://www.qemu.org/docs/master/system/security.html)
- [NSF/IARPA SELinux syscall policy](https://selinuxproject.org/)
- [CIS Distribution Independent Networks Benchmark](https://www.cisecurity.org/benchmark/distribution-independent-linux)
- [PCI DSS 4.0 physical segmentation](https://www.pcisecuritystandards.org/)