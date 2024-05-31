# Fedora 40 on the Lichee Pi 4A

Fedora 40 can more or less work on the Lichee Pi 4A, but there are a few pieces missing.  
  
The big component missing is working EFI support in either the vendor or upstream u-boot. Because of this we cannot use grub2-efi. Fedora kernels are also built using `CONFIG_EFI_ZBOOT` and extlinux does not support zboot so we need to decompress them in order to use them with extlinux. Finally, some kernel patches that have not yet made it upstream are required to make the lpi4a more functional.  
  
It is easiest to modify the image using a VM and then copying it to an SD Card to boot the Lichee Pi 4A.

## Update Image

- Follow the [instructions](https://fedoraproject.org/wiki/Architectures/RISC-V/Installing) for launching a Fedora 40 VM.
- Once it has booted login
- Update `dnf -y update`
- Install packages `dnf -y install gcc gcc-c++ git glib2-devel meson ninja-build tar wget`
- OpenSBI
  - `wget https://github.com/riscv-software-src/opensbi/releases/download/v1.4/opensbi-1.4-rv-bin.tar.xz`
  - `tar Jxvf opensbi-1.4-rv-bin.tar.xz`
  - `cp opensbi-1.4-rv-bin/share/opensbi/lp64/generic/firmware/fw_dynamic.bin /boot`
- unzboot 
  - `git clone https://github.com/eballetbo/unzboot`
  - `cd unzboot`
  - `sed -i '293,298d' unzboot.c`
  - `sed -i '284,286d' unzboot.c`
  - `meson build`
  - `ninja -C build`
  - `cp build/unzboot /usr/local/bin/`
  - `cd`
- kernel
  ```
  dnf -y install https://github.com/jmontleon/fedora-40-lpi4a/raw/main/rpms/kernel-6.8.11-300.1.riscv64.fc40.riscv64.rpm \
    https://github.com/jmontleon/fedora-40-lpi4a/raw/main/rpms/kernel-core-6.8.11-300.1.riscv64.fc40.riscv64.rpm \
    https://github.com/jmontleon/fedora-40-lpi4a/raw/main/rpms/kernel-modules-6.8.11-300.1.riscv64.fc40.riscv64.rpm \
    https://github.com/jmontleon/fedora-40-lpi4a/raw/main/rpms/kernel-modules-core-6.8.11-300.1.riscv64.fc40.riscv64.rpm              
  ```
  - `unzboot /boot/vmlinuz-6.8.11-300.1.riscv64.fc40.riscv64 /boot/vmlinuz-6.8.11-300.1.riscv64.fc40.riscv64.unzboot`
- extlinux
  - `dnf -y install https://github.com/jmontleon/fedora-40-lpi4a/raw/main/rpms/extlinux-bootloader-1.2-16.rv64.fc40.riscv64.rpm`
  ```
  cat << EOF > /boot/extlinux/extlinux.conf
  menu title Fedora boot menu
  prompt 0
  timeout 50
  default F40S1

  label F40S1
          menu label Fedora 40 6.8.11-300.1.riscv64.fc40.riscv64
          linux /vmlinuz-6.8.11-300.1.riscv64.fc40.riscv64.unzboot
          initrd /initramfs-6.8.11-300.1.riscv64.fc40.riscv64.img
          fdt /dtb/thead/th1520-lichee-pi-4a.dtb
          append console=ttyS0,115200 root=UUID=ae525e47-51d5-4c98-8442-351d530612c3 ro rootflags=subvol=root rootwait
  EOF
  ```
  - NOTE: be sure to use the correct UUID for the image you downloaded. You can use `blkid` and/or look one of the entries in `/boot/loader/entries` to get this. 
- Shutdown the VM
- dd the image to an SD Card
- Eject it safely, insert it in your Lichee Pi 4a, and boot.

## U-Boot
The u-boot env I used to boot this:
```
aon_ram_addr=0xffffef8000
arch=riscv
audio_ram_addr=0x32000000
baudrate=115200
board=light-c910
board#=LP
board_name=light-c910
boot_conf_addr_r=0xc0000000
boot_conf_file=/extlinux/extlinux.conf
boot_efi=run bootcmd_load; run fdt_load; chk_hibernate; fixup_memory_region; bootslave; bootefi bootmgr ${fdt_addr_r}
bootcmd=run bootcmd_load; chk_hibernate; fixup_memory_region; bootslave; sysboot mmc ${mmcdev}:${mmcbootpart} any $boot_conf_addr_r $boot_conf_file;
bootcmd_load=run mmc_select; run load_aon; run load_c906_audio; run load_str; run load_opensbi
bootdelay=2
cpu=c9xx
default_mmcdev=1
dtb_addr=0x03800000
efi_8be4df61-93ca-11d2-aa0d-00e098032b8c_Boot0000={nv,boot,run}(blob)0100000086004600650064006f0072006100000001041400b9731de684a3cc4aaeab82e828f3628b031a050001031a05000104012a00010000
efi_8be4df61-93ca-11d2-aa0d-00e098032b8c_BootOrder={nv,boot,run}(blob)0000
efi_8be4df61-93ca-11d2-aa0d-00e098032b8c_OsIndicationsSupported={boot,run}(blob)0000000000000000
efi_8be4df61-93ca-11d2-aa0d-00e098032b8c_PlatformLang={nv,boot,run}(blob)656e2d555300
efi_8be4df61-93ca-11d2-aa0d-00e098032b8c_PlatformLangCodes={boot,run}(blob)656e2d555300
efi_8be4df61-93ca-11d2-aa0d-00e098032b8c_RuntimeServicesSupported={boot,run}(blob)8001
emmc_dev=0
eth1addr=4e:f4:ce:b0:ee:fc
ethaddr=4e:f4:ce:b0:ee:fb
fdt_addr_r=0x03800000
fdt_high=0xffffffffffffffff
fdt_load=load mmc 1:2 ${fdt_addr_r} dtb/${fdtfile}; fdt addr ${fdt_addr_r}
fdtcontroladdr=ffba1d80
fdtfile=thead/th1520-lichee-pi-4a.dtb
fdtoverlay_addr_r=0x03700000
fwaddr=0x10000000
gpt_partition=gpt write mmc ${emmc_dev} $partitions
kdump_buf=180M
kernel_addr_r=0x00200000
load_aon=load mmc ${mmcdev}:${mmcbootpart} $fwaddr light_aon_fpga.bin;cp.b $fwaddr $aon_ram_addr $filesize;bootaon
load_c906_audio=load mmc ${mmcdev}:${mmcbootpart} $fwaddr light_c906_audio.bin;cp.b $fwaddr $audio_ram_addr $filesize
load_opensbi=load mmc ${mmcdev}:${mmcbootpart} $opensbi_addr fw_dynamic.bin
load_str=load mmc ${mmcdev}:${mmcbootpart} $fwaddr str.bin;cp.b $fwaddr $str_ram_addr $filesize
mmc_select=if test -e mmc ${default_mmcdev}:${mmcbootpart} ${boot_conf_file}; then mmcdev=1; else mmcdev=0; fi;
mmcbootpart=2
opensbi_addr=0x0
partitions=name=table,size=2031KB;name=boot,size=500MiB,type=boot;name=swap,size=4096MiB,type=swap,uuid=${uuid_swap};name=root,size=-,type=linux,uuid=${uuid_rootfsA}
pxefile_addr_r=0x00600000
ramdisk_addr_r=0x06000000
scriptaddr=0x00500000
sdcard_dev=1
sdcard_gpt_partition=gpt write mmc ${sdcard_dev} $partitions
splashimage=0x30000000
splashpos=m,m
str_ram_addr=0xffe0000000
uuid_rootfsA=80a5a8e9-c744-491a-93c1-4f4194fd690a
uuid_swap=5ebcaaf0-e098-43b9-beef-1f8deedd135e
vendor=thead
```
