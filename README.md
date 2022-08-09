# UEFI + Secure Boot testbed with nftables routing

I use these scripts to test any boot process modifications before applying them
to my physical machine. To get secure boot working requires some
poorly-documented QEMU magic, and
[OVMF](https://github.com/tianocore/edk2/tree/master/OvmfPkg) firmware images
from the [EDK II Project](https://github.com/tianocore/edk2).

On servers where resilience is a concern, I use [`libvirt`](https://libvirt.org/)
to manage my VMs. This is a quick-and-dirty collection of scripts for rapid prototyping.

### Requirements
- x86-64 QEMU
- kvm (optional)
- iproute2
- nftables (kernel + userspace tools)

If you're using KVM make sure you add your user to the `kvm` group or equivalent on your distribution.

### Obtaining OVMF firmware images
The `OVMF_CODE.secboot.4m.fd` and `OVMF_VARS.4m.fd` files are distribution-agnostic
flash images. On Arch, there is an [`edk2-ovmf`](https://archlinux.org/packages/extra/any/edk2-ovmf/) package with these binaries precompiled, located in
`/usr/share/edk2/x64`. You should copy at least the variables image to your `$vmdir`, as its contents will change. If your distribution does not have a similar package, you can download the package from the Arch package page and extract those files from the archive.


## Running
After grabbing the OVMF images and creating a disk image, read the `launch`, `ifup`, and `ifdown` scripts and modify them to your liking. Then
``` sh
$ ./launch
```
The network scripts don't set up a DHCP server, so you'll have to manually configure the network interface of the guest VM.

#### Preallocating a raw disk image
QEMU supports several disk image formats; I use a raw file here because it is
simple and straightforward. You can quickly allocate a large empty file with:
``` sh
$ fallocate -l 8GiB disk.img
```

#### Generating a MAC
``` sh
$ od /dev/urandom -N6 -tx1 -An | sed 's/ /:/g;s/^://'
22:38:76:06:77:f6
```
If you're worried about collisions or conformance, make sure you set the 2nd LSB of the first octet to indicate local administration.

#### Entering Firmware Configuration
Press F2 at the initial boot screen to enter the OVMF firmware configuration menu.
If you're booting off media that has GRUB installed, it will likely provide an
option to enter firmware configuration as well.

#### Configuring Secure Boot
If you've produced custom secure boot keys, you can provision them in the OVMF configuration interface by entering the
<kbd>Device Manager</kbd>&rarr;<kbd>Secure Boot Configuration</kbd> submenu,
setting `Secure Boot Mode` to `custom`, then enrolling your generated `PK`,
`KEK`, `DB`, and `DBX` keys.

Rod Smith has an excellent writeup on generating and using these
[here](https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html).
<center>
  <img src=ovmf-secureboot-config.png>
</center>

## Network Scripts
The network scripts passed to QEMU enable forwarding on the created network interface and the host interface associated with the default route.
``` sh
sysctl -w net.ipv4.conf.$tapdev.forwarding=1
sysctl -w net.ipv4.conf.$gwdev.forwarding=1
```

Next, an `nftables` table is installed which implements a dynamic source NAT rule. `masquerade` will use the ip address of the outgoing interface no matter what it is, whereas an `snat` statement requires manual specification of source IP address. This configuration will apply SNAT to any packets from interfaces for which forwarding is enabled. To narrow down the affected packets, you could add an `iif $tapdev` or `ip saddr $hostaddr` expression to the masquerade rule.
```
table ip kvm-testbed {
	chain nat-postrouting {
		type nat hook postrouting priority srcnat;
		oif $gwdev masquerade
	}
}
```

## QEMU UEFI options
I don't like to copy and paste options without understanding what they actually do. It took a little digging for me to understand the following options.
```
-global driver=cfi.pflash01,property=secure,value=on
```

This option enables the `secure` feature of QEMU's CFI
([Common Flash Memory Interface](https://en.wikipedia.org/wiki/Common_Flash_Memory_Interface)). When this feature is enabled for x86 targets, any attempts to write to the device outside of [SMM](https://en.wikipedia.org/wiki/System_Management_Mode) result in an error.[^1] This is appropriate for UEFI and secure boot, which executes in SMM and must be able to modify flash contents. Making the flash readonly is not an option for EFI variables, but lower-privilege execution contexts should not be able to modify it.

`cfi.pflash01` emulates the Intel/Sharp extended CFI command set (SCS), while `cfi.pflash02` emulates the AMD/Fujitsu standard command set.

[^1]: [hw/block/pflash_cfi01.c](https://github.com/qemu/qemu/blob/848a6caa88b9f082c89c9b41afa975761262981d/hw/block/pflash_cfi01.c#L678), [include/exec/memattrs.h](https://github.com/qemu/qemu/blob/848a6caa88b9f082c89c9b41afa975761262981d/include/exec/memattrs.h#L35), [target/i386/cpu.h](https://github.com/qemu/qemu/blob/848a6caa88b9f082c89c9b41afa975761262981d/target/i386/cpu.h#L2310)

```
-drive if=pflash,format=raw,unit=0,file="$vmdir"/OVMF_CODE.secboot.4m.fd,readonly=on
-drive if=pflash,format=raw,unit=1,file="$vmdir"/OVMF_VARS.4m.fd
```
These options configure the `pflash` backing files. The first unit is the OVMF firmware
code, and should not be writable. The second hosts mutable EFI data referenced by the
firmware code.

[More info on the QEMU mailing lists](https://lists.gnu.org/archive/html/qemu-block/2019-01/msg00917.html)

## More links
- [SecureBoot/VirtualMachine](https://wiki.debian.org/SecureBoot/VirtualMachine) - Debian Wiki
- [Controlling Secure Boot](https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html) - Rod Smith
- [nft man page](https://manpages.debian.org/unstable/nftables/nft.8.en.html)