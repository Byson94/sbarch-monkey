# sbarch-monkey

A guide on setting up secure boot on an arch linux ISO.

## Sbarch ISO

The iso used for demonstration in this repository is a very minimal version of archlinux (relang profile), 
consisting of just these core packages:

```
base
linux
linux-firmware
mkinitcpio
mkinitcpio-archiso
grub
mtools
dosfstools
sudo
vim
```

## Build ISO

The `build.sh` in the root of this repository is a custom patch of the `mkarchiso` script, which handles setting up the 
certificates and secure boot. This script can be ran to build the iso to the `outdir/`.

```bash
./build.sh
```

It automatically does every step metioned in the [Secure Boot Setup](#secure-boot-setup) section. You just have to ensure you
have these packages installed on your system:

```
sbsigntools
shim-signed (AUR)
```

## Secure Boot Setup

The secure boot is powered by the `shim-signed` package from the AUR, which provides a shim, a pre-bootloader program signed by microsoft.
Most systems come with the microsoft keys enrolled, so your device will trust this shim.

Installing this package will populate the `/usr/share/shim-signed` directory with the following files:

```
$ ls /usr/share/shim-signed/
fbaa64.efi  fbx64.efi  mmaa64.efi  mmx64.efi  shimaa64.efi  shimx64.efi
```

The important files in here are:

- shimx64.efi: The actual shim.
- mmx64.efi: The MokManager (see below).

These files should be present in your ISO at the following path:

```bash
'/EFI/BOOT/BOOTx64.EFI' # shimx64.efi renamed to BOOTx64.EFI
'/EFI/BOOT/grubx64.EFI' # Your GRUB efi file
'/EFI/BOOT/mmx64.efi'   # The MokManager
```

### MokManager

MokManager is a user interface tool in Linux used to manage Machine Owner Keys (MOK) when UEFI Secure Boot is active.
We use this tool to make the user enroll your certificate (provided in the ISO) so that the hardware is actually accept your ISO.

### Signing & Keys

Just having the said files present in your ISO is not enough. You need a *key* which you will use to sign your bootloader (grub) and 
the linux kernel with. You can see how to generate a key by examining the [genkey.sh](genkey.sh) script at the root of this repository.
You can make changes to it to suite your project's branding.

Running the script will create the following files:

- db.key: The private key you should keep a secret.
- db.crt: The public certificate which you can share.
- db.cer: The public certificate you will add in the ISO for MOK enrollment.

These keys can then be used to sign the linux kernel and grub:

```bash
# Sign Grub
sbsign --key db.key --cert db.crt \
    --output "path/to/grub" \
    "path/to/grub"

# Sign Kernel
sbsign --key db.key --cert db.crt \
    --output "path/to/linux/kernel" \
    "path/to/linux/kernel"
```

### Saving Certificate on Disk

The `db.cer` file we generated needs to be inside the ISO so that the user can enroll it in the MokManager screen.
Make sure to include this file in the FAT image (efiboot.img) which will serve as the EFI system partition.

### Recommended Implementation

I highly recommend going through the [build.sh](build.sh) file (the custom patch of mkarchiso) to find how everything
mentioned above is implemented. Similarly, you can make your own mkarchiso patch, or modify my patch and use it.

If you don't want to patch mkarchiso, then you can make a script to extract the ISO, modify it, and then repack it.

## Virtual Machine

If you want to test the ISO in virtual machine, you will need to set a few things up (assuming you are using QEMU). QEMU
does not come with secure boot with the microsoft keys pre-enrolled. If you don't set that up, QEMU will default to having
secure boot DISABLED.

First, install the necessary dependencies:

```bash
sudo pacman --needed -S edk2-ovmf virt-firmware
```

Now, you need to set up the the variables with microsoft keys enrolled:

```bash
cp /usr/share/edk2-ovmf/x64/OVMF_VARS.4m.fd /tmp/OVMF_VARS.fd
virt-fw-vars -i /tmp/OVMF_VARS.fd -o /tmp/OVMF_VARS.fd --enroll-microsoft --sb
```

Now the file at `/tmp/OVMF_VARS.fd` has the microsoft keys enrolled. Now pass this and the `OVMF_CODE.secboot.4m.fd` file
to QEMU to boot with secure boot ENABLED. (It is worth going into UEFI Firmware Settings and checking whether its enabled)

```
qemu-system-x86_64 \
  -enable-kvm \
  -m 2048 \
  -machine q35 \
  -vga std \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2-ovmf/x64/OVMF_CODE.secboot.4m.fd \
  -drive if=pflash,format=raw,file=/tmp/OVMF_VARS.fd \
  -cdrom <path/to/iso>
```

## MokManager Screen

Here are some pictures on how the final setup would look like. 

Immediately on boot, you will see this screen. Press "OK" to continue.

<img width="1272" height="674" alt="image" src="https://github.com/user-attachments/assets/932e0fcf-a513-4edc-a8e0-74e5e94200bd" />

Then select "Enroll key from disk".

<img width="1272" height="674" alt="image" src="https://github.com/user-attachments/assets/f0cfde3c-1446-4045-b2c1-9a1df7029186" />

Proceed with the ISO's EFI image.

<img width="1272" height="674" alt="image" src="https://github.com/user-attachments/assets/c078d134-6d5a-4d7a-a707-3c0dde53320a" />

And then select the certificate you put into it.

<img width="1272" height="674" alt="image" src="https://github.com/user-attachments/assets/a564dbae-f8ae-4ade-8203-d636a377b8e0" />

Then press "Continue" to continue enrolling this certificate.

<img width="1272" height="674" alt="image" src="https://github.com/user-attachments/assets/6d4827dd-74af-4ab1-b9a1-aeec2f83153a" />

Press "Yes" to enroll keys.

<img width="1272" height="674" alt="image" src="https://github.com/user-attachments/assets/3adb1e3b-8dd7-401d-8431-49da0af8326b" /> 

And finally, select reboot to reboot with the certificate properly enrolled.

<img width="1272" height="674" alt="image" src="https://github.com/user-attachments/assets/3f831762-0bfd-4efd-ac8b-3f0f95c22958" />


Voila! Now you can boot into the ISO with secure boot on! And it is persistent across reboots too!

<img width="1272" height="674" alt="image" src="https://github.com/user-attachments/assets/3adf66ec-793a-4103-a680-92523212cdca" />







