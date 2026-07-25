---
title: "Virtualizing OpenWRT Router"
last_modified_at: 2026-07-25T13:00:00+02:00
categories:
  - OpenWRT
tags:
  - virtualization
  - networking
---

## Introduction
This virtual machine replicates my five-NIC router as closely as possible. Once everything is properly set up, the physical box is mirrored almost 1:1, at least from a configuration point of view. VMware Workstation Pro 26H1 is the hypervisor in this scenario, but VirtualBox can be used as well.

### NICs
The most challenging part has always been the NIC setup. Here's the breakdown:

| OpenWRT logical interface | OpenWRT physical device | VM NIC |
| --- | --- | --- |
| wan | eth0 | NAT |
| lan | br-lan (eth0-3) | 4x LAN Segments
| - | - | 1x host-only adapter |

and the actual NICs in the VM:

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/openwrt/openwrt-vm-devices.jpg)

The NAT NIC provides the backend device for the `wan` interface. The Workstation takes care of the DHCP assignment, NAT translation, and actual Internet access over the host.

The 4 distinct LAN Segments[^1] emulate the `eth` devices - they function together as a bridge.

The host-only adapter does not have a physical counterpart. It enables easier SSH access to the VM.

### Cons of choosing the Host-only Adapter for bridging
Even though the host networking[^2] might seem a viable choice for the LAN interface, there is a better alternative.

Bridging 4 NICs within the same host-only network would create a switching loop. In the console you would see messages like:
```
received packet on ethX with own address as source
ICMPv6: NA <MAC> advertised our own address <MAC> on <interface>
```
You could bridge using 4 distinct host-only networks, but they would have to be created in advance. The LAN segment does not need to be pre-created and is contained within the VM config file (less management overhead, better portability). It also does not connect to the host network, making it a better choice overall.

## VMX config file
Point the `scsi0:0.fileName` property to your vmdk disk file. Other than that, the file should be ready to use.
<details>
<summary>openwrt.vmx</summary>

<pre style="font-size: 11px;"><code>
#!/usr/bin/vmware
.encoding = "UTF-8"
cleanShutdown = "TRUE"
config.version = "8"
cpuid.coresPerSocket = "2"
displayName = "openwrt"
ethernet0.addressType = "generated"
ethernet0.connectionType = "nat"
ethernet0.generatedAddress = "00:0c:29:79:3b:a5"
ethernet0.generatedAddressOffset = "0"
ethernet0.pciSlotNumber = "32"
ethernet0.present = "TRUE"
ethernet0.virtualDev = "e1000"
ethernet1.addressType = "generated"
ethernet1.connectionType = "pvn"
ethernet1.generatedAddress = "00:0C:29:79:3B:AF"
ethernet1.generatedAddressOffset = "10"
ethernet1.pciSlotNumber = "33"
ethernet1.present = "TRUE"
ethernet1.pvnID = "52 f4 2f 14 12 bf e9 e0-67 af 6b 46 37 65 0a b0"
ethernet1.virtualDev = "e1000"
ethernet2.addressType = "generated"
ethernet2.connectionType = "pvn"
ethernet2.generatedAddress = "00:0C:29:79:3B:B9"
ethernet2.generatedAddressOffset = "20"
ethernet2.pciSlotNumber = "34"
ethernet2.present = "TRUE"
ethernet2.pvnID = "52 1a 57 59 24 d5 e7 a4-e3 b0 48 9c 78 02 c5 3b"
ethernet2.virtualDev = "e1000"
ethernet3.addressType = "generated"
ethernet3.connectionType = "pvn"
ethernet3.generatedAddress = "00:0C:29:79:3B:C3"
ethernet3.generatedAddressOffset = "30"
ethernet3.pciSlotNumber = "35"
ethernet3.present = "TRUE"
ethernet3.pvnID = "52 9d 2a 1e 49 10 26 17-00 1e 49 9c 5b e6 72 b4"
ethernet3.virtualDev = "e1000"
ethernet4.addressType = "generated"
ethernet4.connectionType = "pvn"
ethernet4.generatedAddress = "00:0C:29:79:3B:CD"
ethernet4.generatedAddressOffset = "40"
ethernet4.pciSlotNumber = "37"
ethernet4.present = "TRUE"
ethernet4.pvnID = "52 ba 11 ba 65 08 ce 2b-93 00 ff 15 fe 6b 6f 3e"
ethernet4.virtualDev = "e1000"
ethernet5.addressType = "generated"
ethernet5.connectionType = "hostonly"
ethernet5.generatedAddress = "00:0C:29:79:3B:D7"
ethernet5.generatedAddressOffset = "50"
ethernet5.pciSlotNumber = "38"
ethernet5.present = "TRUE"
ethernet5.virtualDev = "e1000"
extendedConfigFile = "openwrt.vmxf"
floppy0.present = "FALSE"
guestOS = "other6xlinux-64"
hpet0.present = "TRUE"
mem.hotadd = "TRUE"
memsize = "512"
monitor.phys_bits_used = "45"
numvcpus = "2"
nvram = "openwrt.nvram"
pciBridge0.pciSlotNumber = "17"
pciBridge0.present = "TRUE"
pciBridge4.functions = "8"
pciBridge4.pciSlotNumber = "21"
pciBridge4.present = "TRUE"
pciBridge4.virtualDev = "pcieRootPort"
pciBridge5.functions = "8"
pciBridge5.pciSlotNumber = "22"
pciBridge5.present = "TRUE"
pciBridge5.virtualDev = "pcieRootPort"
pciBridge6.functions = "8"
pciBridge6.pciSlotNumber = "23"
pciBridge6.present = "TRUE"
pciBridge6.virtualDev = "pcieRootPort"
pciBridge7.functions = "8"
pciBridge7.pciSlotNumber = "24"
pciBridge7.present = "TRUE"
pciBridge7.virtualDev = "pcieRootPort"
powerType.powerOff = "soft"
powerType.powerOn = "soft"
powerType.reset = "soft"
powerType.suspend = "soft"
scsi0:0.fileName = ""
scsi0:0.present = "TRUE"
scsi0:0.redo = ""
scsi0.pciSlotNumber = "16"
scsi0.present = "TRUE"
scsi0.virtualDev = "lsilogic"
softPowerOff = "TRUE"
sound.autoDetect = "TRUE"
sound.fileName = "-1"
sound.pciSlotNumber = "-1"
svga.vramSize = "268435456"
tools.syncTime = "FALSE"
toolsInstallManager.updateCounter = "3"
usb:0.deviceType = "hid"
usb:0.parent = "-1"
usb:0.port = "0"
usb:0.present = "TRUE"
usb:1.deviceType = "hub"
usb:1.parent = "-1"
usb:1.port = "1"
usb:1.present = "TRUE"
usb:1.speed = "2"
usb.pciSlotNumber = "-1"
uuid.bios = "56 4d 8c d8 b0 c4 6c a1-19 da 83 f0 48 79 3b a5"
uuid.location = "56 4d 8c d8 b0 c4 6c a1-19 da 83 f0 48 79 3b a5"
vcpu.hotadd = "TRUE"
virtualHW.productCompatibility = "hosted"
virtualHW.version = "22"
vm.lastPowerRequestTimestamp = "1784884439331603"
vmci0.id = "1215904677"
vmci0.present = "TRUE"
vmotion.checkpointFBSize = "134217728"
vmotion.checkpointSVGAPrimarySize = "268435456"
vmotion.svga.graphicsMemoryKB = "262144"
vmotion.svga.mobMaxSize = "268435456"
vmxstats.filename = "openwrt.scoreboard"
</code></pre>

</details>

## Issues
The host-only NIC used for management occasionally hangs for unknown reasons:
![alt]({{ site.url }}{{ site.baseurl }}/assets/images/openwrt/openwrt-vm-nic-hang.jpg)

The NIC has to be manually reconnected using the lower toolbar from the VM window.

## References
[^1]: [Configuring LAN Segments](https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/workstation-pro/26H1/using-vmware-workstation-pro/configuring-network-connections/configuring-lan-segments.html)
[^2]: [Configuring Host-Only Networking](https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/workstation-pro/26H1/using-vmware-workstation-pro/configuring-network-connections/configuring-host-only-networking.html)