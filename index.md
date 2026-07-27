# Chimera Linux

## Goal

This document will walk you through installing a [libvirt](https://en.wikipedia.org/wiki/Libvirt) environment on top of [Chimera Linux](https://en.wikipedia.org/wiki/Chimera_Linux)
with support for [QEMU](https://en.wikipedia.org/wiki/QEMU)/[KVM](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine) and the configuration of a [Network Bridge](https://en.wikipedia.org/wiki/Network_bridge) to
connect the [Virtual Machines](https://en.wikipedia.org/wiki/Virtual_machine) to the [LAN](https://en.wikipedia.org/wiki/Local_area_network) through [NetworkManager](https://en.wikipedia.org/wiki/NetworkManager) and [virsh(1)](https://manpages.org/virsh/1)

Note: The configuration and usage of Storage Subsystems as well as more advanced Networking setups is out of scope for this guide.

## Target System

In this example our target system is a [Raspberry Pi 5 16GB](https://www.raspberrypi.com/products/raspberry-pi-5/) [SBC](https://en.wikipedia.org/wiki/Single-board_computer) running 
- CPU 4 Core Broadcom BCM2712 [Arm Cortex-A76](https://en.wikipedia.org/wiki/ARM_Cortex-A76)
- Ram 16 GB
- Drive 1TB NVME SSD
- System Language EN

Note: Architecture Specific Packages for [X86-64](https://en.wikipedia.org/wiki/X86-64) will be pointe out during this guide.

## Preparation

A fully configured and running [Chimera Linux](https://chimera-linux.org/) installation and root access is required.

Please refer to the official [Chimera Linux Documentation](https://chimera-linux.org/docs/) and my [install guide](https://theonedoc.github.io/Chimera-Linux-LUKS-LVM-BTRFS/)
for installation on configuration of [Chimera Linux](https://chimera-linux.org/).

## Setup

### Package Installation

```
doas apk add qemu qemu-tools qemu-img qemu-edk2-firmware u-boot-qemu_arm64 qemu-system-aarch64 libvirt
```

or for [X86-64](https://en.wikipedia.org/wiki/X86-64) hosts

```
doas apk add qemu qemu-tools qemu-img qemu-edk2-firmware qemu-system-x86_64 libvirt
```

### User configuration

add the user to the kvm group

```
doas usermod -a -G kvm myusername
```

### enable libvirt service [Daemons](https://en.wikipedia.org/wiki/Daemon_(computing))

Please refer to the libvirt [Documentation](https://www.libvirt.org/daemons.html#modular-driver-daemons) to understand the function of a specific [Daemon](https://en.wikipedia.org/wiki/Daemon_(computing))

```
doas -s
for drv in qemu interface network nodedev nwfilter secret storage proxy log
  do
    dinitctl enable virt${drv}d
  done
exit
```

### Bridge configuration

__Tipp:__ The Network interface Names according to the [Predictable Network Naming scheme](https://www.freedesktop.org/software/systemd/man/latest/systemd.net-naming-scheme.html) 
as well as their associated [MAC address](https://en.wikipedia.org/wiki/MAC_address) can be seen via the [ip(8)](https://manpages.org/ip/8) command.

[ip(8)](https://manpages.org/ip/8) is part of the [iproute2](https://en.wikipedia.org/wiki/Iproute2) package.

```
ip link show
```

#### [NetworkManager](https://en.wikipedia.org/wiki/NetworkManager)

Create a new [Network Bridge](https://en.wikipedia.org/wiki/Network_bridge) interface ```br0``` via [NetworkManager's](https://en.wikipedia.org/wiki/NetworkManager) [nmcli(1)](https://manpages.org/nmcli/1) command.

The [Bridge](https://en.wikipedia.org/wiki/Network_bridge) Connection will automatically be named ```bridge-br0```.

```
doas nmcli con add type bridge ifname br0
```

Connect the [NIC](https://en.wikipedia.org/wiki/Network_interface_controller) to the [Network Bridge](https://en.wikipedia.org/wiki/Network_bridge).

The on board [NIC](https://en.wikipedia.org/wiki/Network_interface_controller) of the [Raspberry Pi 5 16GB](https://www.raspberrypi.com/products/raspberry-pi-5/) is named ```end0```

You have to replace it with the name of your Network interface. 

```
doas nmcli con add type bridge-slave ifname end0 master br0
```

disable [Spanning Tree Protocol](https://en.wikipedia.org/wiki/Spanning_Tree_Protocol) on bridge ```br0```

```
doas nmcli con modify br0 bridge.stp no
```

Assign the [MAC address](https://en.wikipedia.org/wiki/MAC_address) of the [NIC](https://en.wikipedia.org/wiki/Network_interface_controller) ```end0``` to the [Network Bridge](https://en.wikipedia.org/wiki/Network_bridge) ```br0```.

This makes sure that the bridge gets the same IP as was assigned to the [NIC](https://en.wikipedia.org/wiki/Network_interface_controller) via e.g. [Dynamic Host Configuration Protocol](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol).

```
doas nmcli con modify br0 ethernet.cloned-mac-address 88:a2:9e:23:5e:d1
```

diasble the [NIC](https://en.wikipedia.org/wiki/Network_interface_controller) connection ```Wired connection 1``` and enable the bridge connection ```bridge-br0```.

Note: ```Wired connection 1``` is the default name for the first NetworkManager connction to a [LAN](https://en.wikipedia.org/wiki/Local_area_network).

__Attention:__ Any ssh sessions on this [NIC](https://en.wikipedia.org/wiki/Network_interface_controller) connection will be dropped.

```
doas nmcli con down "Wired connection 1" ; doas nmcli con up "bridge-br0"
```

Reestablish the ssh session and check if everything went well.

```
doas nmcli con show
doas nmcli con show --active
```

```
#### virsh

#Add br0 bridge to KVM/QEMU
create file /tmp/br0.xml with content:

cat << 'EOF' > /tmp/br0.xml
<network>
  <name>br0</name>
  <forward mode="bridge"/>
  <bridge name="br0" />
</network>
EOF

#let's use virsh to enable the bridge on the VM side
doas virsh net-define /tmp/br0.xml
doas virsh net-start br0
doas virsh net-autostart br0
#check if all went well
doas virsh net-list --all
```
