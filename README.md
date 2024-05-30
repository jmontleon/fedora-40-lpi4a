# Fedora 40 can more or less work on the Lichee Pi 4A.

The big component missing is working EFI support in either the vendor or upstream u-boot.

- Because of this we cannot use grub2-efi
- extlinux also does not support
- some kernel patches which have not yet made it upstream are required to make the lpi4a more or less functional

It is therefore easiest to update the image using a VM and then copying it to an SD Card to boot the Lichee Pi 4A.

1. Follow the [instructions](https://fedoraproject.org/wiki/Architectures/RISC-V/Installing) for launching a Fedora 40 VM.
1. Once it has booted login
1. Update `dnf -y update`
1. Install git meson ninja-build `dnf -y install gcc meson`
1. Clone unzboot
1. Get extlinux
1. Install an updated kernel
1. Extract the kernel
1. Create a an extlinux.conf
1. Shutdown the VM
1. DD the image to an SD Card
1. Eject it safely, insert it in your Lichee Pi 4a, and boot.
