OVS-DPDK Source build
=====================

CentOS and Oracle Linux currently do not provide packages
for ovs with dpdk.
The Ubuntu packages do not support UIO based drivers.
To user the uio_pci_generic driver on Ubuntu a source build is required.

Building ovs with dpdk containers from source
---------------------------------------------

- Append the following to /etc/kolla/kolla-build.conf to select the version
  of ovs and dpdk to use for your source build.

kolla-build.conf
________________

[openvswitch-base-plugin-ovs]
type = git
location = https://github.com/openvswitch/ovs.git
reference = v2.6.0

[openvswitch-base-plugin-dpdk]
type = git
location = http://dpdk.org/git/dpdk
reference = v16.07

- To build the container execute the follow command:
    tools/build.py --template-override \
    contrib/template-override/ovs-dpdk.j2 dpdk

Prepare servers for ovs-dpdk container
-----------------------------------
(this section will be automated at a later date)

Before deploying the ovs-dpdk container,
the follow steps must be performed on the host.

- Hugepages should be allocated for use by dpdk and guest instances.
  This can be done at runtime via sysfs or at kernel boot time
  via the kernel command line.
  To presistently allocate hugepages on the kernel commandline append
  "default_hugepagesz=2M hugepagesz=2M hugepages=2048"
  On ubuntu this can be done by adding
  GRUB_CMDLINE_LINUX="default_hugepagesz=2M hugepagesz=2M hugepages=2048"
  to /etc/default/grub and regenerating the grub config via
  'grub-mkconfig  -o /boot/grub/grub.cfg'

- Load kernel dpdk compatible kernel modules,
  this can be done via adding the module to /etc/modules
  for example:
    ubuntu@kolla-ovsdpdk:~/kolla$ cat /etc/modules
    # /etc/modules: kernel modules to load at boot time.
    #
    # This file contains the names of kernel modules that should be loaded
    # at boot time, one per line. Lines beginning with "#" are ignored.

    uio_pci_generic
    vfio_pci

- Before deploying ovs-dpdk containers you need to bind the
  physical nic to the kernel driver.
  The "ovs-dpdkctl.sh" tool has been introduced to perform this configuration.

  To use this tool you must first generate the config file.
  This can be done by executing ovs-dpdkctl.sh gen_config and setting the
  bridge_mappings and port_mappings variable on the command line.
  For example:
    bridge_mappings=physnet1:br-ex port_mappings=ens5:br-ex \
    tools/ovs-dpdkctl.sh gen_config  && \
    tools/ovs-dpdkctl.sh show_config

  In this example it is assuming your neutron_external_interface is "ens5".

  This will result in the creation of a config similar to this.

  [ovs]
  bridge_mappings = physnet1:br-ex
  port_mappings = ens5:br-ex
  cidr_mappings =
  pmd_coremask = 0x2
  ovs_mem_channels = 4
  ovs_socket_mem = 512
  dpdk_interface_driver = uio_pci_generic
  hugepage_mountpoint = /dev/hugepages
  pci_whitelist = '-w 0000:00:05.0'

  [ens3]
  address = 0000:00:03.0
  driver =e1000

  [ens4]
  address = 0000:00:04.0
  driver =e1000


  [ens5]
  address = 0000:00:05.0
  driver = uio_pci_generic

  To bind the NICs to the drivers specified in the config
  execute the following command:

      tools/ovs-dpdkctl.sh bind_nics

  To re-bind NICs to their original driver execute:

      tools/ovs-dpdkctl.sh unbind_nics

- To persist NIC binding across server reboots install the ovs-dpdkctl service

      tools/ovs-dpdkctl.sh install

  TODO(sean-k-mooney): install ovs-dpdkctl utility with kolla-host playbook
                       and execute on startup with systemd service file.

Deploy ovs-dpdk
---------------

To deploy ovs-dpdk instead of ovs simpley add
'ovs_datapath: "netdev"' to /etc/kollaglobal.yml

