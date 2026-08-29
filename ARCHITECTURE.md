# Libvirt conversion architecture

This document explains the libvirt conversion's two-network topology, forwarded
host-port behaviour and bidirectional synced-folder interface.

## Network topology

The original VirtualBox configuration gives each VM a per-VM NAT adapter for
internet access and a host-only adapter for lab traffic. vagrant-libvirt normally
connects every project to the shared `vagrant-libvirt` management network, commonly
`192.168.121.0/24`. That would place this intentionally vulnerable workshop on the
same layer-2 segment as unrelated Vagrant/libvirt guests.

The conversion creates two lab-specific networks instead:

| Adapter | Network | Mode | Purpose |
| --- | --- | --- | --- |
| 1 | `k8s-workshop-mgmt` | NAT + DHCP | SSH, provisioning, DNS, internet |
| 2 | `k8s-workshop` | isolated, static | Kubernetes, registry, C2, host access |

Adapter 1 defaults to `192.168.157.0/24`. It remains NAT-enabled, so all four VMs
retain outbound internet access. The network is shared by the four lab VMs but is
not the default management segment used by unrelated vagrant-libvirt projects.
This is closer to VirtualBox isolation, although it does not reproduce
VirtualBox's separate NAT engine for each VM.

Adapter 2 defaults to `192.168.56.0/24` and has no libvirt forwarding or DHCP.
The host-side bridge still permits host-to-guest and guest-to-guest communication.
Kubernetes node IPs, Calico autodetection, registry certificates, and Poseidon
callbacks are explicitly pinned to this adapter so a valid management address is
never selected accidentally.

### Forwarded host ports

The upstream Vagrantfile does not set `host_ip` for the teamserver ports 7443
and 8081. Under both VirtualBox and libvirt, the resulting listeners bind to all
host IPv4 and IPv6 interfaces:

```text
0.0.0.0:7443
0.0.0.0:8081
[::]:7443
[::]:8081
```

Other machines may therefore reach Mythic and the callback port when routing and
the host firewall permit it. This can be useful when the lab runs on a headless
host or must accept connections from another system.

The course URLs using `127.0.0.1` describe the normal local access path; they do
not restrict the forwarding rules to loopback. Internal lab callbacks use
`192.168.56.10` and do not depend on these host forwards.

This behaviour is inherited from the original VirtualBox configuration rather
than introduced by the libvirt conversion. The patch leaves it unchanged to
preserve upstream behaviour. Operators who require loopback-only access can add
`host_ip: "127.0.0.1"` to both forwarded-port declarations, but doing so
intentionally changes the upstream behaviour.

See the [Vagrant forwarded-port documentation](https://developer.hashicorp.com/vagrant/docs/networking/forwarded_ports)
and [VirtualBox manual](https://download.virtualbox.org/virtualbox/6.1.40/UserManual.pdf).

The ranges are runtime inputs:

```text
K8S_LAB_MGMT_NETWORK=192.168.157.0/24
K8S_LAB_PRIVATE_NETWORK=192.168.56.0/24
```

Before libvirt creates a VM, the host preflight validates both values, inspects all
defined libvirt networks and active IPv4 routes, and rejects overlaps. An existing
network is reused only when its name, CIDR, forwarding, and DHCP settings match.
The check never edits or deletes an existing network and never silently chooses a
new range. The error identifies the conflict and shows an environment override.

No runtime check can predict a VPN or route that is activated later. If the host's
networking changes, halt the lab and rerun the preflight with appropriate overrides.

## Synced-folder contract

`/vagrant` is not merely a source-code mount. It is the state channel between VMs:

- the control plane writes `kubeadm-init.out`, `admin.conf`, and registry keys;
- workers read the join command, kubeconfig, and registry certificate;
- the teamserver and lab exercises exchange generated payload and kubeconfig data.

Rsync cannot implement this contract because it synchronizes host to guest only.
Leaving the Vagrant folder type unspecified is also unsafe: Vagrant chooses a usable
backend from installed plugins, and rsync can win on a host without NFS.

The supported interface is:

```text
K8S_LAB_SYNCED_FOLDER=auto
K8S_LAB_SYNCED_FOLDER=9p
K8S_LAB_SYNCED_FOLDER=virtiofs
K8S_LAB_SYNCED_FOLDER=nfs
```

`auto` considers only bidirectional backends. It selects virtiofs when a
`virtiofsd` executable is available, otherwise an already-active NFS service, and
otherwise 9p. NFS readiness is checked without assuming systemd: the host must
provide `exportfs` and accept local TCP connections on the NFS port. The selected
backend is printed by the libvirt preflight. An explicit but unavailable backend
fails there rather than changing semantics silently; VirtualBox and VMware can
still load the Vagrantfile without satisfying libvirt-only host requirements.

| Backend | Advantages | Costs |
| --- | --- | --- |
| virtiofs | fastest; no NFS export | separate daemon; shared-memory backing; passthrough ownership can create root-owned host files |
| NFSv4 | good performance; host-user ownership mapping | host service and export privileges required |
| 9p | no host daemon; squash ownership available | slower and more sensitive to permission behaviour |

Virtiofs uses `memfd` shared memory backing. NFS uses version 4 with UDP disabled.
9p uses squash semantics. Generated host-side file ownership must still be checked
after the first provisioning run because host security policies and libvirt service
users vary between distributions.
