# Configuring Libvirt with OVS (Open Virtual Switch) and VirtualBMC (Bare Metal Control)

## Introduction

This configuration allows you to setup a virtual environment that mimics a group of bare metal servers. The main purpose I have been doing this was for setting up an OpenStack 13 lab living all on one large baremetal server, on CentOS 7.9.


## Packages to Install

```
yum install centos-release-openstack-queens
yum install python2-openstackclient

yum install -y wget libguestfs-tools libguestfs-xfs screen net-tools ntp bind-utils lshw libvirt qemu-kvm virt-manager virt-install xorg-x11-apps xauth virt-viewer tcpdump dejavu-fonts-common dejavu-sans-fonts dejavu-sans-mono-fonts dejavu-serif-fonts numactl lm_sensors firefox openvswitch tmux

```

## Setup/Start Libvirt

```
modprobe --first-time bonding
modprobe --first-time 8021q
systemctl enable libvirtd
systemctl start libvirtd
```

## Nested Virtualisation

```
##Enable nested KVM so that we can have accelerated nested virtualisation:
cat << EOF > /etc/modprobe.d/kvm_intel.conf
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1
EOF

##Disable the rp_filter to allow our virtual machines to communicate
with the underlying host for later tasks:
cat << EOF > /etc/sysctl.d/98-rp-filter.conf
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
EOF

##Enable KSM
systemctl start ksm
systemctl enable ksm
systemctl start ksmtuned
systemctl enable ksmtuned
```
### Configure OVS Bridge

Reference: https://www.itzgeek.com/how-tos/linux/centos-how-tos/configure-openstack-networking-enable-access-vm-instances.html

This is what a working config looks like:

```
[root@dell ~]# cat /etc/sysconfig/network-scripts/ifcfg-ovs-trunk
DEVICE=ovs-trunk
NAME=ovs-trunk
DEVICETYPE=ovs
TYPE=OVSBridge
OVSBOOTPROTO="none"
OVSDHCPINTERFACES=em1 # Physical Interface Name
BOOTPROTO=static
IPADDR=192.168.1.220 # Your Control Node IP (SingleNode)
NETMASK=255.255.255.0 # Your Netmask
GATEWAY=192.168.1.1 # Your Gateway
DNS1=192.168.1.150 # Your Name Server
ONBOOT=yes


[root@dell ~]# cat /etc/sysconfig/network-scripts/ifcfg-em1
DEVICE=em1
NAME=em1
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=ovs-trunk
ONBOOT=yes
BOOTPROTO=none
```

Run this script in a tmux session:

```
[root@dell ~]# cat network.sh
systemctl stop NetworkManager
systemctl disable NetworkManager
ovs-vsctl add-port ovs-trunk em1
systemctl restart network
```

If it comes back, then good, if not, goto your console and try to fix it.




 
