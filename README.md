# OpenStack Instance Configuration Guide

# Preface

This configuration guide served as basis for the Red Hat blog post at
https://redhatstackblog.redhat.com/2017/01/18/9-tips-to-properly-configure-your-openstack-instance/.

**_NB. This guide was written during Mitaka/Newton era and parts of it
may be obsolete or even incorrect with later releases._**

# Introduction

This document provides OpenStack instance (=VM/Guest) configuration and
optimization related information. Many of these configurations and
optimizations are relevant for every instance running on OpenStack.

Most of the configurations can be done inside guests but some require
OpenStack administrator assistance or the host side software to be on a
certain patch level and/or configured in a certain manner so the steps
depending on administrator or host side configurations are not available
for all OpenStack tenants.

For related OpenStack upstream hints, see
http://docs.openstack.org/image-guide/openstack-images.html.

# Generic Guest Configuration and Optimization

The following configurations should be evaluated for any VM running on
any OpenStack environment. These changes have no side-effects and are
typically safe to enable even if unused.

## Image Format

OpenStack storage configuration is an implementation choice by the Cloud
Provider, often not fully visible to the Cloud Consumer tenant. Storage
configuration may also change over the time without explicit
notification by the Cloud Provider. Thus, for storage the Requirements
provided by the Cloud Consumer for the Cloud Provider for the needed
features, functionality, and performance characteristics are the most
crucial aspect of storage related definitions. However, the below tips
may also be helpful in some cases.

When creating a new instance on OpenStack, it is based on a
[Glance](https://docs.openstack.org/glance/latest/) image. The two most
prevalent and recommended image formats are _Qcow2_ and _raw_. Qcow2 and
raw image files may have radical size difference as Qcow2 images
typically grow on demand meaning that a disk image for a 100 GB disk
might be only a few GBs of size as Qcow2 but 100 GB as unprocessed raw
image. Regardless of the format, it is recommended to process images
before uploading them to Glance with
[virt-sysprep(1)](http://libguestfs.org/virt-sysprep.1.html) and,
especially,
[virt-sparsify(1)](http://libguestfs.org/virt-sparsify.1.html). Sparse
images can save considerable amounts of disk space also for raw images,
use [du(1)](https://www.mankier.com/1/du) to see the actual space usage
of an image file (and create a sparse tar file or compress with
[xz(1)](https://www.mankier.com/1/xz) if it needs to be transferred to
another system). For in-depth discussion on disk image sizes, see
https://www.berrange.com/posts/2017/02/10/the-surprisingly-complicated-world-of-disk-image-sizes/.

Performance-wise Qcow2 has different compatibility versions, latest
being Qcow2v3 (sometimes referred to as Qcow3) which has performance
advantage over the original Qcow2. Qcow2v3 is considered to offer near
raw image performance. Latest tools on recent distributions (for
instance, on RHEL 7) automatically use the newer format and it is
possible to check and also convert between raw and older/newer Qcow2
images with [qemu-img(1)](https://www.mankier.com/1/qemu-img).

Image-backed instances (i.e., the image is being transferred to the Nova
compute node where the instance is running) are directly affected by the
Qcow2 vs Qcow2v3 vs raw performance difference (unless the Cloud
Provider is forcing Nova compute to use raw images).

Volume-backed instances can be created either from Qcow2 or raw Glance
images. However, as Cinder backends are vendor-specific (Ceph, 3PAR,
EMC, etc) they may not all use Qcow2 or raw. Instead, some may have
their own mechanisms, like dedup, thin provisioning, or copy-on-write.
This means that in many cases the main differences are the manageability
of different sized images and the time required for volume creation
(which, for example, on Ceph with raw images is almost instant due to
its copy-on-write approach). (See also
https://bugzilla.redhat.com/show_bug.cgi?id=1383587.) Of particular
note, using Qcow2 in Glance with Ceph is not supported (see Ceph
documentation and https://bugzilla.redhat.com/show_bug.cgi?id=1383014).

In conclusion, in many cases Qcow2v3 might be the most suitable format
but in some cases, for example when an image which is constantly used to
create new volume-backed instances, raw image format might be
considered. In any case, the outcome is largely subject to the OpenStack
storage side configurations done by the Cloud Provider.

## Image Properties

Latest OpenStack releases allow Nova to optimize certain libvirt VM
properties on the Compute host based on the available guest OS
information. To provide information about the guest OS for Nova, define
the following Glance image properties:

* **os_type=linux** # Generic name, like _linux_ or _windows_,
  needed with Windows
* **os_distro=rhel7.1** # Use **osinfo-query os** to list supported
  variants

Additionally, at least for the time being (see
https://bugzilla.redhat.com/show_bug.cgi?id=1397120) in order to make
sure the newer and more scalable _virtio-scsi_ para-virtualized SCSI
controller is used instead of the older _virtio-blk_, the following
properties need to be set explicitly:

* **hw_scsi_model=virtio-scsi**
* **hw_disk_bus=scsi**

All the supported image properties are listed at
https://access.redhat.com/documentation/en/red-hat-openstack-platform/9/paged/command-line-interface-reference/chapter-8-image-service-command-line-client#glance-property.

## cloud-init

cloud-init is a package used for early initialization of cloud
instances, to configure basics like partition / filesystem size and SSH
keys.

Install the _cloud-init_ and _cloud-utils-growpart_ packages and enable
the included services to allow cloud-init based configuration for a VM.

In many cases the default configuration is acceptable but there are lots
of customization options available, for details please refer to
cloud-init documentation at https://cloudinit.readthedocs.io/en/latest/.

On a related note, this blog post explains different ways to inject
files into a VM:
https://kimizhang.wordpress.com/2014/03/18/how-to-inject-filemetassh-keyroot-passworduserdataconfig-drive-to-a-vm-during-nova-boot/.

## QEMU Guest Agent

Install and enable QEMU guest agent which allows graceful guest shutdown
and freezing guest filesystems, most often a must-have operation for
consistent backups (but see also
https://bugzilla.redhat.com/show_bug.cgi?id=1385748):

* **yum install qemu-guest-agent**
* **systemctl enable qemu-guest-agent**

In order to provide the needed virtual devices and use the filesystem
freezing functionality when needed, the following properties need to be
defined for Glance images (see also
https://bugzilla.redhat.com/show_bug.cgi?id=1391992):

* **hw_qemu_guest_agent=yes** # Creates the necessary device files for
  the VM to allow QEMU guest agent to run
* **os_require_quiesce=yes** # Accept requests to freeze/thaw
  filesystems, https://bugzilla.redhat.com/show_bug.cgi?id=1385748

## Guest Fault Recovery

Comprehensive instance fault recovery, high availability, and service
monitoring requires a layered approach which as a whole is out of scope
for this document, below we iterate an alternative applicable entirely
inside a guest (which can be thought as being the inner-most layer).

The most often used fault recovery mechanisms for an instance are to
recover from kernel crashes and to recover from guest hangs which do not
necessarily involve kernel panic.

In the rare case the guest kernel crashes, _kdump_ will capture a kernel
vmcore for further analysis and reboot the guest. For configuration
details on this, see
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Kernel_Crash_Dump_Guide/chap-installing-configuring-kdump.html.
In case the vmcore is not wanted, kernel can be instructed to reboot
after a kernel crash by disabling the kdump service and setting the
panic kernel parameter, for example _panic=1_.

In order to reboot an instance after unexpected behavior, for example
load being over a high threshold for a certain period of time or a
complete system lockup without kernel panic, the watchdog service can be
utilized. The following property needs to be defined for Glance images
(or Nova flavors) to create the needed guest device and reset the guest
on hang (other available actions are listed in
https://docs.openstack.org/nova/latest/user/flavors.html):

* **hw_watchdog_action=reset**

Then, the watchdog package needs to be installed, the watchdog device
configured, and the service enabled:

* **yum install watchdog**
* **vi /etc/watchdog.conf**
* **systemctl enable watchdog**

By default watchdog detects kernel crashes and complete system lockups.
See the [watchdog.conf(5)](https://www.mankier.com/5/watchdog.conf) man
page for more information, e.g., how to add guest health-monitoring
scripts as part of watchdog functionality checks.

## Guest Memory Dump Collection

Under some rare circumstances an instance may get completely stuck
without anything triggering kdump and a memory dump of the instance will
be needed for further investigation. There is an open
[RFE](https://bugzilla.redhat.com/show_bug.cgi?id=1498380) to provide
Web UI for capturing instance memory dumps but before that is available
the following manual steps by the Cloud Provider are needed:

* Locate the compute node where the VM instance is running
  * **openstack server list**
  * **openstack server show _VMID_**
  * Note the _OS-EXT-SRV-ATTR_ lines containing information about the
    host/hypervisor name and instance internal name
    * The VM name will be something like _instance-000014a3_
  * Collect the memory dump on the compute node running the instance
    * Login to the compute node
    * Use **virsh list** to verify the VM is present on the node
    * Get the memory dump with
      **virsh dump instance-000014a3 /tmp/dumpfile --memory-only
      --verbose**
  * Compress the dump file and share it with the team wanting to
    investigate it

## Guest Kernel Parameters

_tuned_ is a service which configures system parameters according to the
selected profile. On OpenStack instances the most suitable profile
usually is _virtual-guest_.

It is recommended to install the required package, enable the service on
boot, and activate the preferred profile:

* **yum install tuned**
* **systemctl enable tuned**
* **tuned-adm profile virtual-guest**

The default IO scheduler of RHEL 7 (_deadline_) is most often suitable
but in some cases _noop_ might also perform well.

## Guest Kernel virtio Drivers

Guest kernel _virtio_ drivers are part of the standard kernel package
and enabled automatically without any further configuration as needed.

For an illustration of the virtio device model, see
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Tuning_and_Optimization_Guide/sect-Virtualization_Tuning_Optimization_Guide-Networking-Virtio_and_vhostnet.html.

## Host Kernel vhost-net Drivers

Host kernel _vhost-net_ drivers are part of the standard kernel package
and enabled automatically without any further configuration as needed.

For an illustration of the vhost-net device model, see
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Tuning_and_Optimization_Guide/sect-Virtualization_Tuning_Optimization_Guide-Networking-Virtio_and_vhostnet.html.

For more detailed explanation of virtio-net / vhost-net / _vhost-user_
and differences between these, see for example
http://www.virtualopensystems.com/en/solutions/guides/snabbswitch-qemu/.
Also see the DPDK section below.

## Guest Miscellaneous Tuning

It should go without saying that right-sized instances should contain
only the minimum amount of installed packages and run only the services
needed. Of particular note, it is probably a good idea to install and
enable the _irqbalance_ service as, although not absolutely necessary in
all scenarios, its overhead is minimal and it should be used for example
in SR-IOV setups (this way the same image can be used regardless of such
lower level details).

Even though implicitly set on KVM, it is a good idea to explicitly add
the kernel parameter _no_timer_check_ to prevent issues with timing
devices. Enabling persistent DHCP client and disabling zeroconf route in
network configuration with _PERSISTENT_DHCLIENT=yes_ and
_NOZEROCONF=yes_, respectively, helps to avoid networking corner case
issues.

Guest MTU settings are usually adjusted correctly by default, but having
a proper MTU in use on all levels of the stack is crucial to achieve
maximum network performance. In environments with 10G (and faster) NICs
this typically means the use of Jumbo Frames with MTU up to 9000, taking
possible VXLAN encapsulation into account. For further MTU discussion,
see http://docs.openstack.org/newton/networking-guide/config-mtu.html.

## Guest Access Methods

Instances of cloud-enabled workloads may never need to be accessed so
the following considerations might not be applicable for such cases,
especially in production.

The SSH daemon should avoid DNS lookups to speed up establishing SSH
connections. For this, consider using _UseDNS no_ in
_/etc/ssh/sshd_config_ and adding _OPTIONS=-u0_ to _/etc/sysconfig/sshd_
(see [sshd_config(5)](https://www.mankier.com/5/sshd_config and
[sshd(8)](https://www.mankier.com/8/sshd)) for details on these).
Setting _GSSAPIAuthentication no_ could be considered if Kerberos is not
in use. In case instances frequently connect to each other over SSH, the
_ControlPersist_ / _ControlMaster_ options might be considered as well.

Typically remote SSH access and console access via Horizon are enough
for most use cases. During development phase direct console access from
the Nova compute host may also be helpful, for this to work enable the
_serial-getty@ttyS1.service_, allow root access via ttyS1 if needed by
adding _ttyS1_ to _/etc/securetty_ and then access the guest console
from the Nova compute with **virsh console _id_ --devname serial1**.

# Host/Guest Configuration and Optimization

The following configurations are possible in recent OpenStack
environments and rely on host-level access. These changes cannot enabled
by the Cloud Consumer without help from the administrator and the Cloud
Provider.

## CPU Pinning / NUMA Topology Awareness

It is possible to dedicate host CPUs for guests running on the compute
node and then pin the guest vCPUs to pCPUs. The goal is to increase
performance by avoiding overcommitment and to avoid latencies which
might otherwise occur, especially on NUMA nodes. Please note that for
the time being CPU pinning may prevent guest migration, for details see
https://access.redhat.com/solutions/2191071 and
https://bugzilla.redhat.com/show_bug.cgi?id=1222414.

This Red Hat Stack blog posting gives comprehensive introduction and
step-by-step instructions how to achieve CPU pinning with NUMA
awareness:
http://redhatstackblog.redhat.com/2015/05/05/cpu-pinning-and-numa-topology-awareness-in-openstack-compute/.
For additional options, see also the upstream documents
https://docs.openstack.org/nova/latest/user/flavors.html and
https://docs.openstack.org/nova/latest/admin/cpu-topologies.html.

**Note!** The _isolcpus_ method described in the blog post is not valid
any more (on RHEL 7.2+) for non-real-time use cases, a systemd based
approach should be used instead. To separate CPUs, systemd must be
configured with proper CPU affinity rules to handle placement of host
service processes by listing the system-reserved CPUs in
_/etc/systemd/system.conf_:

* **CPUAffinity = 0,4**

To activate this change, the host must be rebooted. Use for example the
[top(1)](https://www.mankier.com/1/top) utility to verify that only the
selected CPUs are used for system services after the reboot.

Then, on each compute node's _nova.conf_ something like below needs to
be added for CPU pinning (systemd and Nova CPU parameters should be
consistent) and to enable NUMA aware scheduling:

* **vcpu_pin_set = 1,2,3,5,6,7**

* **scheduler_default_filters = ...,NUMATopologyFilter,AggregateInstanceExtraSpecsFilter**

The _openstack-nova-compute_ service needs to be restarted after these
changes. To separate dedicated performance compute nodes and generic
compute nodes, host aggregates can be used, please refer to the blog
post for details on this.

Real-time and near-real-time considerations are out of scope for this
document as most higher level applications and VNFs are expected achieve
the needed level of performance with the approaches described here.

## Huge Pages / Memory Reservation

It is possible to provide Huge Pages to applications running on
OpenStack instances to increase efficiency of memory operations. This
may be used in conjunction with reserving (some of the) host memory for
system use, not guests.

This Red Hat Stack blog posting gives comprehensive introduction and
step-by-step instructions how to configure Huge Pages:
http://redhatstackblog.redhat.com/2015/09/15/driving-in-the-fast-lane-huge-page-support-in-openstack-compute/.

The blog post contains an example how to reserve Huge Pages on the host
kernel. In case some of the host memory and its Huge Pages should be
reserved for system processes, this can be achieved with the following
kind _nova.conf_ parameters:

* **reserved_host_memory_mb = 2048**
* **reserved_huge_pages = node=0, size=2048, count=64**
* **reserved_huge_pages = node=1, size=1GB, count=2**

The _openstack-nova-compute_ service needs to be restarted after this
change.

The blog post provides additional examples how to configure flavor and
image properties for use with Huge Pages.

## Storage Configuration

OpenStack storage configuration is an implementation choice by the Cloud
Provider, often not fully visible to the Cloud Consumer tenant. Storage
configuration may also change over the time without explicit
notification by the Cloud Provider. Thus, for storage the Requirements
provided by the Cloud Consumer for the Cloud Provider for the needed
features, functionality and performance characteristics are the most
crucial aspect of storage related definitions. However, the below tips
may also be helpful in some cases.

As of OpenStack Ocata, there are no notable block/storage IO related
configurations and optimizations which would involve a guest apart from
the earlier mentioned virtio-scsi image property, a tuned profile (which
sets most relevant kernel parameters inside the guest), and possibly the
choice of an IO scheduler.

There are some upstream efforts (like
https://blueprints.launchpad.net/nova/+spec/libvirt-iothreads) ongoing
which aim to provide disk IO improvements for instances in a later
OpenStack release, most likely without additional guest configuration.

On the host level to achieve the needed level of instance disk IO,
_virtual-host_ tuned profile, SSD disks, well-tuned Ceph cluster, and/or
proper Cinder backend drivers are some of the alternatives to consider.

## Device Assignment / PCI Passthrough / SR-IOV

Device assignment, or PCI passthrough, and SR-IOV can be used to provide
direct hardware access from a host to a guest. This provides near
bare-metal performance but can eliminate some crucial
cloud/virtualization functionality, like live migration, and in general
needs to be coordinated with the Cloud Provider. Typically needed by
some of the most performance-critical applications, like core network
routing functionality, it is expected that many higher level
applications and VNFs do not benefit enough of these setups to justify
the added complexity.

For an illustration of the PCI passthrough and SR-IOV architectures, see
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Tuning_and_Optimization_Guide/sect-Virtualization_Tuning_Optimization_Guide-Networking-Device_Assignment_and_SRIOV.html.

PCI passthough / SR-IOV should be considered for generic workloads and
higher-level VNFs if other options like OVS-DPDK (see below) are not
suitable when trying to achieve the required level of performance.
However, under certain circumstances a Cloud Provider may urge to
support SR-IOV due to resource policies on compute nodes so although not
applicable for most use cases this might still be something that should
be prepared for in VNF Cloud Deliveries. On recent RHEL versions, the
CPU partitioning / NFV tuned profiles should be evaluated as well.

### PCI Passthrough

For configuration details, see
https://docs.openstack.org/nova/latest/admin/pci-passthrough.html.

### SR-IOV

For configuration details, see
https://access.redhat.com/documentation/en/red-hat-openstack-platform/9/paged/networking-guide/chapter-20-sr-iov-support-for-virtual-networking.
For SR-IOV VF bonding instructions, see
https://access.redhat.com/node/355853.

## DPDK

Starting with RHEL 7.2 / RH OSP 8, OVS with DPDK, or OVS-DPDK, is
available on Compute as a Tech Preview to provide much improved network
performance for virtual guest instances. OVS-DPDK utilizes PMD
(poll-mode drivers) on vhost-user with a zero-copy technique to expose
raw network frames from a host NIC to guests bypassing the hypervisor.
It is then the responsibility of the guest _kernel_ to handle these raw
frames, the guest user space sees incoming traffic as before and no
application changes are required in this scenario. With OVS-DPDK
cloud/virtualization operations like live migration are still possible
(with recent enough versions, meaning OVS-DPDK 2.6 / RH OSP 11 / RHEL
7.4 or newer).

Starting with RH OSP 10 and newer, DPDK should be configured with
Director, as per https://access.redhat.com/solutions/2930291.

3rd party solutions exist to handle more complex (telecom) protocols and
advanced routing scenarios with DPDK PMD used by a guest _application_
to bypass both the hypervisor and the guest OS. Since this scenario
requires application level support, this kind of solution is out of the
scope for less specialized applications and VNFs.

For more detailed explanation of virtio-net / vhost-net / vhost-user and
differences between these, see for example
http://www.virtualopensystems.com/en/solutions/guides/snabbswitch-qemu/.
For OVS-DPDK configuration details, see
https://access.redhat.com/documentation/en/red-hat-openstack-platform/9/single/configure-dpdk-for-openstack-networking.

Although there are no mandatory changes inside a guest with OVS-DPDK,
enabling multi-queue is likely to be beneficial with vhost-user, see
below. On recent RHEL versions, the CPU partitioning / NFV tuned
profiles should be evaluated as well.

## Network Multiqueuing

Network multiqueuing, or virtio-net multi-queue, is an approach that
enables packet processing to scale with the number of available vCPUs of
a guest, often providing improved transfer speeds especially with
vhost-user.

Provided that the host has recent enough components available (most
importantly, OVS 2.5 / DPDK 2.2 and
https://blueprints.launchpad.net/nova/+spec/libvirt-virtio-net-multiqueue
is implemented), this functionality can be enabled on per image basis.

The following property must be set for the Glance image where
multiqueuing is to be enabled:

* **hw_vif_multiqueue_enabled=true**

On a guest instantiated from such an image the NIC channel setup can be
checked and changed as needed with the commands below:

* **ethtool -l eth0**
* **ethtool -L eth0 combined _nr-of-queues_** # Should match the number
  of available CPUs

Note that on recent kernels with [this RFE
implemented](https://bugzilla.redhat.com/show_bug.cgi?id=1396578) the
above is configured automatically and no manual steps should be needed.

# References

* https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Tuning_and_Optimization_Guide/sect-Virtualization_Tuning_Optimization_Guide-Networking-Virtio_and_vhostnet.html
* http://blog.allenx.org/2015/07/13/the-new-feature-vhost-user-in-qemu
* http://www.virtualopensystems.com/en/solutions/guides/snabbswitch-qemu/
* https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Tuning_and_Optimization_Guide/sect-Virtualization_Tuning_Optimization_Guide-Networking-Device_Assignment_and_SRIOV.html
