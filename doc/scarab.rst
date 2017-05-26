vim /etc/default/grub
# Append to GRUB_CMDLINE_LINUX= "default_hugepagesz=2M hugepagesz=2M hugepages=2048"
grub2-mkconfig -o /boot/grub2/grub.cfg
echo "#!/bin/sh

exec /sbin/modprobe uio_pci_generic >/dev/null 2>&1
exec /sbin/modprobe vfio_pci >/dev/null 2>&1" >> /etc/sysconfig/modules/dpdk.modules
# Add the following to dpdk.modules
"#!/bin/sh

exec /sbin/modprobe uio_pci_generic >/dev/null 2>&1
exec /sbin/modprobe vfio_pci >/dev/null 2>&1"

modprobe uio_pci_generic
modprobe vfio_pci
# Reboot system

bridge_mappings=physnet1:br-ex port_mappings=p5p1:br-ex ./ovs-dpdkctl.sh gen_config && ./ovs-dpdkctl.sh show_config >> /etc/kolla/openvswitch-db-server/ovs-dpdkctl.conf
echo "uio_pci_generic" > /sys/bus/pci/devices/0000\:05\:00.1/driver_override
$DPDK_DIR/tools/dpdk-devbind.py --bind=uio_pci_generic p5p1
