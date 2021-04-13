# Configuring Libvirt with OVS (Open Virtual Switch) and VirtualBMC (Bare Metal Control) 

## Introduction

This configuration allows you to setup a virtual environment that mimics a group of bare metal servers. The main purpose I have been doing this was for setting up an OpenStack 13 lab living all on one large baremetal server, on CentOS 7.9. Most this information was provided to me Nick Satsia, and I am documenting it via this markdown document.

# CentOS 7 Steps

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

## Configure Libvirt OVS Networks

Here we configure the xml files to define the networks for libvirt ovs integration. When you attach VMs to these networks libvirt will automatically do the appropriate ovs changes.

### Disable the default network (unless you really want it)
```
virsh net-destroy default
virsh net-autostart --network default --disable
```

### libvirt trunk interface to bond bridge (ovs-trunk)
Create xml for a libvirt trunk interface to the bond bridge. This trunk will have all OSP infra and Provider vlans:

```
[root@cargoship ovs]# cat ovs-port-VLAN.xml
<network>
   <name>ovs-port-VLAN</name>
   <forward mode='bridge'/>
   <bridge name='ovs-trunk'/>
   <virtualport type='openvswitch'/>
   <portgroup name='lab-port-VLAN'>
     <vlan>
       <tag id='VLAN'/>
     </vlan>
   </portgroup>
</network>
```
The public network (where all your VMs will be on the public network) is `0`.

So if you want to create VLANs with IDs 10, 20, 30, 40, 50 and 60:

```
for i in 10 20 30 40 50 60; do \
     rm -f ovs-port-$i.xml; \
     cp ovs-port-VLAN.xml ovs-port-$i.xml; \
     sed -i s/VLAN/$i/g ovs-port-$i.xml; \
     virsh net-define ovs-port-$i.xml; \
     virsh net-autostart ovs-port-$i; \
     virsh net-start ovs-port-$i; \
done
```
Check status of the networks:

```
[root@dell ovs]# virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 ovs-port-10          active     yes           yes
 ovs-port-20          active     yes           yes
 ovs-port-30          active     yes           yes
 ovs-port-40          active     yes           yes
 ovs-port-50          active     yes           yes
 ovs-port-60          active     yes           yes
```

To create a trunk port of all the vlans:
```
cat << EOF > ovs-trunk2.xml
<network connections='1'>
   <name>ovs-trunk2</name>
   <forward mode='bridge'/>
   <bridge name='ovs-trunk'/>
   <virtualport type='openvswitch'/>
   <portgroup name='lab-trunk2'>
     <vlan trunk='yes'>
       <tag id='10'/>
       <tag id='20'/>
       <tag id='30'/>
       <tag id='40'/>
       <tag id='50'/>
       <tag id='60'/>
     </vlan>
   </portgroup>
</network>
EOF

virsh net-define ovs-trunk2.xml
virsh net-autostart ovs-trunk2
virsh net-start ovs-trunk2

```
## Setup Virtual Bare Metal Control (virtualBMC)

Install the software:

```
yum install epel-release
yum install python2-pip
yum install -y python-devel
yum groupinstall -y 'development tools'
```

### install virtualbmc

```
pip install --upgrade pip
pip install -U setuptools

#Install VBMC
pip install virtualbmc
pip install configparser
```

**Note: **The above vbmc didn’t work for me. I installed this rpm (python2-virtualbmc) and it worked and impitool.


```
cat << EOF > /etc/virtualbmc/virtualbmc.conf
[default]
config_dir=/root/.vbmc
[log]
logfile=/var/log/virtualbmc/virtualbmc.log
debug=True
[ipmi]
session_timout=20
EOF

cat << EOF > /usr/lib/systemd/system/virtualbmc@.service
[Unit]
Description=VirtualBMC %i service
After=network.target

[Service]
Type=forking
PIDFile=/root/.vbmc/%i/pid
ExecStart=/bin/vbmc start %i
ExecStop=/bin/vbmc stop %i
Restart=always

[Install]
WantedBy=multi-user.target
EOF


chmod 0644 /usr/lib/systemd/system/virtualbmc@.service
```


**The instructions state to disable firewalld, which I’m reluctant to do.**

Instead, once services are running, I’ll keep note of the ports.

```
[root@dell ~]# /usr/bin/vbmc list

/usr/lib/systemd/system/virtualbmc@.service

[root@dell ~]# systemctl start virtualbmc@osp-compute01.service
[root@dell ~]# systemctl start virtualbmc@osp-compute02.service
[root@dell ~]# systemctl start virtualbmc@osp-controller.service
[root@dell ~]# systemctl enable virtualbmc@osp-controller.service
Created symlink from /etc/systemd/system/multi-user.target.wants/virtualbmc@osp-controller.service to /usr/lib/systemd/system/virtualbmc@.service.
[root@dell ~]# systemctl enable virtualbmc@osp-compute01.service
Created symlink from /etc/systemd/system/multi-user.target.wants/virtualbmc@osp-compute01.service to /usr/lib/systemd/system/virtualbmc@.service.
[root@dell ~]# systemctl enable virtualbmc@osp-compute02.service
Created symlink from /etc/systemd/system/multi-user.target.wants/virtualbmc@osp-compute02.service to /usr/lib/systemd/system/virtualbmc@.service.
[root@dell ~]# systemctl list-unit-files |grep bmc
virtualbmc@.service                           enabled
[root@dell ~]# systemctl status | grep bmc
           │ │ │ ├─10087 /usr/bin/python2 /usr/bin/vbmc start osp-controller
           │ │ │ ├─10097 /usr/bin/python2 /usr/bin/vbmc start osp-compute01
           │ │ │ ├─10105 /usr/bin/python2 /usr/bin/vbmc start osp-compute02
           │ │ │ └─10462 grep --color=auto bmc
[root@dell ~]# vbmc list
+----------------+---------+---------------+------+
|  Domain name   |  Status |    Address    | Port |
+----------------+---------+---------------+------+
| osp-compute01  | running | 192.168.1.220 | 8223 |
| osp-compute02  | running | 192.168.1.220 | 8224 |
| osp-controller | running | 192.168.1.220 | 8222 |
+----------------+---------+---------------+------+
[root@dell ~]# systemctl status virtualbmc@osp-compute01.service
● virtualbmc@osp-compute01.service - VirtualBMC osp-compute01 service
   Loaded: loaded (/usr/lib/systemd/system/virtualbmc@.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2020-03-20 16:53:34 AEST; 2min 0s ago
 Main PID: 10097 (vbmc)
   CGroup: /system.slice/system-virtualbmc.slice/virtualbmc@osp-compute01.service
           ‣ 10097 /usr/bin/python2 /usr/bin/vbmc start osp-compute01

Mar 20 16:53:33 dell.momolab.io systemd[1]: Starting VirtualBMC osp-compute01 service...
Mar 20 16:53:34 dell.momolab.io vbmc[10321]: Error starting a Virtual BMC for domain osp-compute01. Error: [Errno 98] Address...in use
Mar 20 16:53:34 dell.momolab.io systemd[1]: Started VirtualBMC osp-compute01 service.
Hint: Some lines were ellipsized, use -l to show in full.
```
Apply firewall rules:

```
firewall-cmd --add-port={8222,8223,8224}/tcp
firewall-cmd --add-port={8222,8223,8224}/tcp --permanent
```

Test:

```
[root@dell ~]# ipmitool -I lanplus -U stack -P password -H 192.168.1.220 -p 8222 power status
Chassis Power is on
[root@dell ~]# ipmitool -I lanplus -U stack -P password -H 192.168.1.220 -p 8222 power help
chassis power Commands: status, on, off, cycle, reset, diag, soft
[root@dell ~]# ipmitool -I lanplus -U stack -P password -H 192.168.1.220 -p 8222 power off
Chassis Power Control: Down/Off
```


# CentOS 8 Steps
I've tested these steps on CentOS 8 and found things behave slightly differently. The steps below are what worked for me.


## Packages

```
yum install -y centos-release-openstack-victoria
 
yum install -y python3-openstackclient

yum install -y wget libguestfs-tools libguestfs-xfs  net-tools bind-utils lshw libvirt qemu-kvm virt-manager virt-install xauth virt-viewer tcpdump  numactl lm_sensors firefox openvswitch tmux
```

## Config

```
modprobe --first-time bonding
modprobe --first-time 8021q
systemctl enable libvirtd
systemctl start libvirtd

cat << EOF > /etc/modprobe.d/kvm_intel.conf
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1
EOF

cat << EOF > /etc/sysctl.d/98-rp-filter.conf
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
EOF

systemctl start ksm
systemctl enable ksm
systemctl start ksmtuned
systemctl enable ksmtuned

systemctl enable openvswitch
systemctl start openvswitch
```

The network scripts below are customised to my environment, please adjust accordingly:

```
cat << EOF >  /etc/sysconfig/network-scripts/ifcfg-br-ex
DEVICE=br-ex
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
OVSBOOTPROTO="none"
OVSDHCPINTERFACES=eno1
IPADDR=192.168.1.220
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=192.168.1.150
NM_CONTROLLED=no
EOF

cat << EOF >  /etc/sysconfig/network-scripts/ifcfg-eno1
DEVICE=eno1
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-ex
BOOTPROTO=none
NM_CONTROLLED=no
EOF

systemctl enable network
systemctl start network

virsh net-list
virsh net-destroy default
virsh net-undefine default



cat << EOF > kvm-ovs.xml
<network>
  <name>ovs-bridge</name>
  <forward mode='bridge'/>
  <bridge name='br-ex'/>
  <virtualport type='openvswitch'/>
</network>
EOF

virsh net-define  kvm-ovs.xml
virsh net-start ovs-bridge
virsh net-autostart  ovs-bridge
virsh net-list --all
```

## References:

1. https://computingforgeeks.com/how-to-install-open-vswitch-on-centos-rhel/

2. https://computingforgeeks.com/configuring-open-vswitch-on-centos-rhel-fedora/

 
