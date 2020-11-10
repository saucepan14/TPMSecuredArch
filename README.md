# TPM Secured Arch

This repo contains various files used in Arch Linux root partition to be securely encrypted and unlocked without requiring a password. It uses a luks based encryption and adds a systemd unit to the initramfs to get the key from the TPM. This page assumes you have installed Arch before and are comforartable installing it on your own. It also assumes you have a TPM in your machine, you have access to controlling secure boot keys and you can boot with UEFI mode as I am only using UEFI for this. I am also booting directly to an EFI file and am not using a boot loader, if you are using one you are kind of own your own I have not tested using a boot loader with secure boot but I do not feel that it is impossible to use a boot loader and have secure boot verifiy signed images.


## Important notes to consider

Most of the shell is not mine I have modified parts of shell from other people in order to fit my needs, however I will have a list of sources of where I got code from in the further reading section. I have only tested this with arch linux, while I wish to explose implamenting this in other distro (Namely Debian based distros) I do not have the time to test every distro I would like. Should you get a setup like this working on another distro I would like to add a link to your own git repo or to add how you did it and of course give you credit.
There are 3 packages you will need for this (along with their dependencies) they are tpm2-tools, sbsign, python3 and efitools. 
Also of note is that in the current process the guide will go through I belive it is possible to intercept the keyfile if someone was sniffing the TPM bus when the keyfile is given out. A way around this would be to use an HMAC session with the TPM so the communication would be encrypted or to require a pin on top of having correct pcrs values in order to relase the key. I will work on getting around to implenting the latter at some point but given that it would require specialized tools I do not consider this but an extremely important issue but I still consider it to be an important one to be aware of.


## Creating an encrypted root and swap partition

When installing you Arch linux you will need at least 3 partitions. A 512M EFI partition, A swap partition with at least how much ram your system has, and a root partition which can whatever size you please. If you wish to have other encrypted partitions please see the further reading section.

After creating all the partitions necessary you will need to encrpyt the root and swap partitions, this can be done with the following command:
```
cryptsetup luksFormat /dev/[the partiion you wish to encrypt]
```
Be sure to choose a strong password as the encryption is only as strong as your password.

You will need to do this twice, once for the root partition and once for the swap partition.

Please note this will erase your data on that partition so be sure to copy and important data somewhere.


## Mounting the root partition and generating some of the important files

You will next need to decrpyt the now encrypted root partition so you can mount it and install Arch onto it.
It will not be necessary to decrypt to swap partition right now, it will be done later on.

You can decrypt the partition with the following
```
cryptsetup open --type luks /dev/[the name of you encrypted partion i.e. /dev/sda2, /dev/sdb3, /dev/nvme0n1p2, etc]
```
Then mount the now decrypted partion and continue installing Arch as normal

Mount with this command
```
mount /dev/mapper/root /mnt
```
After you have arch-chrooted into the now mounted partition you will need to be sure to generate /etc/fstab using the following:
```
genfstab -U -p /mnt /mnt/etc/fstab
```
Next you will need to create the file /etc/crypttab.initramfs and this line:
```
root    UUID=[the uuid of the encrypted partition]     /root_keyfile.bin    luks
```
Before you finish the install you will need to make some changes to /etc/mkinitcpio.conf
You will need to change the the Hooks() to be like this
```
HOOKS(base systemd autodetect sd-vconsole modconf block sd-encrpyt filesystem keyboard fsck)
```

We will still need to modify the config file again but this is all the is needed for now.


## Generating the EFI stub

If you wish to use a bootloader to choose between multiple os installs you are on your own here as I have not gone with a setup like that and instead boot directly into my Arch install. IF you do get an install with a boot loader and multiple installs working with secure boot please let me know and I will add it. 

After having run mkinitcpio you will need to run provided GenStub it will take the initramfs-linux.img, the vmlinuz-linux, cmdline arguments stored in /boot/cmdline and os-release and combine them into Linux.efi stored in /EFI/Linux/. if you wish to change the output name or where the image is output simply modify the last line of GenStub. After this is done you should be able to boot directly into the image. 

Once in the image we will get working on generating Secure boot and signing the image.


## Creating keys to be used in secure boot and signing the EFI stub

Once you have created an EFI stub you will need to sign it so secure boot can verify that it is trusted by you.
The first step is to run the mkkeys.sh script, this will create a total of 17 files with 3 of them being secret keys that you need to keep somewhere secure. If they are not secured properly then someone and sign an os boot into it and get the keyfile to your drive which is obviously bad. 
Of note is that mkkeys.sh will create the keys in the directory that the script is run so if you run the script in ~/ then it will generate all the keys and associated files and place them in your home directory. While not necessarily a bad thing you might want to make a new directory and cd into it and run the script from there to avoid unnecessary clutter. 

When running mkkeys.sh it will prompt you to give the gives common name it is not necessary but I don't see why not adding them.

Once you have your keys and the .cer, .auth, .esl, and .crt files you next need to sign the EFI stub, this can be done with the following command:
```
sbsign --key ~/SecurebootKeys/DB.key --cert ~/SecurebootKeys/DB.crt \
--output /boot/EFI/Linux/Linux-signed.efi /boot/EFI/Linux/Linux.efi
```

After this command is run you may get a warning along the lines of 
```
warning: data remaining[1231832 vs 1357089]: gaps between PE/COFF sections?
```
I do not know what these warnings are but they do not seem to cause issue. If you do understand these warning and know how to fix them or clarify what they please let me know and I will be sure to add it and credit you.


## Adding keys to secure boot

If you wish to still boot into windows and want it verified by secure boot you should save the pre loaded Microsoft KEK and DB key and add them after all the other keys have been added. This should work, however I still have not tested it and there will be links for more information in the further reading section.

After having optionally saved the Microsoft keys to your flashdrive you can clear the keys and then add your own. In my case I only added the .auth files but this may be different for you. You can try fiddling around with the keys guessing and checking which one is correct or maybe your mother board manual has some information.
When adding the keys it is preferable to add the keys starting with the DB then the KEK and then the PK. You need to be sure to replace the PK keys with your own and not Microsoft's or any other 3rd party's, as it is the top key in secure boot. 

Of note is that if an image boots being verified by secure boot it will generate a pcr 7 value depending on which key verified it. So even if you a Windows image signed by Microsoft it will not be able to get your tpm secrets stored against and imaged signed by you.

Also of note the exact process of adding secure boot keys is bios specific and I can not provided a comprehensive guide for every system if you are unsure or are having trouble google is your friend.

After adding all the keys boot into the signed image.


## Creating the tpm policy and adding a keyfile to luks

Next we will need to add a keyfile to the luks encrypted partition and store it in the tpm. We will be creating a 256 bit (i.e. 32 byte) keyfile using data from /dev/random this makes the keyfile very hard to brute force and takes away the trouble of having to create a strong keyfile on your own. A copy of the keyfile will be stored within the encrypted partition which is not a big security concern as long as you don't leave your computer unattended with the keyfile readable by any user, or someone having access to your encrypted partition in which case you have bigger problems. 

To create the keyfile use the following command:
```
dd if=/dev/random of=/root/root_keyfile.bin bs=1 count=32
```
Run the above command and replace 
```
/root/root_keyfile.bin
```
with 
```
/root/swap_keyfile.bin
```

This creates two keyfiles in /root/, we will only using /root/root_keyfile.bin for now, but GenTPMPolicy expects both swap and root keyfiles so we create both.

If you are worried about accidentally overwriting them use:
```
chmod 0400 /root/root_keyfile.bin && chmod 0400 /root/swap_keyfile.bin
```
To set it to read only.

The we will be adding the key to luks with the following command:
```
cryptsetup luksAddKey /dev/[name of the encrypted root partition i.e. /dev/sd1, /dev/nvme0n1p2, etc] /root/root_keyfile.bin
```

Next we need to store the keyfile in the tpm and seal it against the desired pcrs, in our case pcrs 0 and 7. Before we actually store the keys in the tpm I suggest storing a test file in the tpm. First create a test file with some simple text in it. Then seal with with the GenTPMPolicy script

To test if you can read the file from the tpm use

```
tpm2_unseal -c 0x81000000 -p pcr:sha1:0,7
```

If it returns the value of your test file then it works.

If it does not return the test file run

```
tpm2_getcap handles-persistent
```

and replace ```0x81000000``` with the value returned from the previous command.

To test if it will only give away the to images signed by you turnoff your computer and boot into the unsigned image, and run the same command again. If you receive a tpm error then it means your secure boot is working correctly, if you still get the value of the file returned then your secure boot is not working correctly and is giving away the key only to unsigned images. I recommend changing which files you uploaded to secure boot. I.E. uploading .esl files instead of .auth files. The goal is to have the tpm only give you the key to images signed by you. If you wish to remove the saved file from the tpm use:

```
tpm2_evictcontrol -C o -c 0x81000000
```

After evicting the file you can run the previous commands to re-add the file to the tpm under (hopefully) correct pcr values.

Once you have verified the tpm will only give away files when booted into images signed by you boot into a signed image and first evict control of any information already in the tpm with:

```
tpm2_evictcontrol -C o -c 0x81000000
```

Then run the GenTPMPolicy script, it will store both /root/root_keyfile.bin and /root/swap_keyfile.bin inside the tpm.
After running the script verifiy both values have been stored with 

```
tpm2_getcap handles-persistent
```

It should return two values, the first value being the location of the root keyfile and the second being the location of swap keyfile.

To verify the tpm has stored the values correctly run 

```
tpm2_unseal -c 0x81000000 -p pcr:sha1:0,7 | hexdump
```

and compare the output to 

```
hexdump /root/root_keyfile.bin
```
If done correctly they should return the same value.


## Adding the mkinitcpio hooks and decrypting at boot

In order to have to system decrypt the root partition on its own we need to add a custom hook to mkinitcpio. The hook and install are included under that initcpio folder. You will need to take the files from within them and place them in /etc/initcpio/hooks/ and /etc/initcpio/install respectively. You will then need to modify 2 lines in /etc/mkinitcpio.conf
You need to ensure the tpm related modules are loaders by modifying the MODULES() line to look like so.

```
MODUELS(tpm tpm_tis)
```

You will then need to modify the HOOKS() line again to add the custom hook. It should look something along the lines of this:

```
HOOKS(base systemd autodetect sd-vconsole modconf block encrypt-tpm sd-encrpyt filesystem keyboard fsck)
```

After you have done this regenerate the boot related images with

```
mkinitcpio -p linux
```

Then recreate the stub and sign it with GenStub and SignEFI (in the order). Ensure that the locations of keys files in SignEFI match where you have your stored, if they are stored somewhere else modify the file and then run it.

Then if all is correct you should be able to turn off your restart your system and have it decrypt itself!

If the system fails to decrypt itself using the commands:

```
journalctl -b | grep -i tpm
```

and

```
journalctl -b | grep -i keyfile
```

Should help point you in the right direction for debugging.

If in the future after you have had this setup working if the auto decryption does not work and prompts you for a password you should consider how much you trust the image. As someone could have modified it without you knowing and could have something malicious within it. If you do not trust the image completely it would be best to load to a live usb (with an image that you know is safe), decrypt the drive and regenerate the EFI stub and sign it again and see if that fixes the issue.


## Decrypting swap partition and hibernation.

Now that you have a system that can decrypt itself at boot you can now add a swap partition that will decrypt at boot too!
First you need to add the keyfile to the existing encrypted partition that you will be using as swap. You can do that with this command:

```
cryptsetup luksAddKey /dev/[name of the encrpyed swap partion i.e. /dev/sd1, /dev/nvme0n1p2, etc] /root/swap_keyfile.bin
```

Then we need to add a line to /etc/crypttab.initramfs that looks like this:

```
swap    UUID=[the uuid of the encrypted partition]     /swap_keyfile.bin    luks
```

Add a line to /etc/sftab similar to this:

```
/dev/mapper/swap    swap    swap   defaults    0    0
```

All that is left to do it is to modify the cmdline arguments telling the system where to look to resume from. To do this we need to add this argument to /boot/cmdline

```
resume=/dev/mapper/swap
```

After that you should be good to reboot and your system should now decrypt the boot partition and the swap partition on its own. You can now even hibernate your computer and pickup exactly where you left off with the command:

```
systemctl hibernate
```

To note is that ```systemctl hibernate``` might not lock you display manager, be sure you have a session manager installed to lock your system for you. I am personally using light-locker and have it set in the xfce4 power manager settings to lock whenever the computer is powered off (i.e. sleep or hibernating).

If you wish to lock the computer without suspending or hibernating it you can use ```loginctl lock-session``` to simply sign out.


## Pacman hooks for automatic stub generation and signing

I highly recommend added the included pacman hooks to automatically create and sign an EFI stub, otherwise you will have to do this manually everytime you updated or your install won't boot. you will need to modify ```pacman.d/hooks/99-secureboot.hook``` and replace:
```Exec = /usr/bin/sh -c "/path/to/GenStub; /path/to/SignEFI"```
with the actual path to the scripts. If you are using an alternative kernel such as linux-lts or linux-zen you will need to modify:
```Target = linux``` with the respective package of your kernel.


## Links to further reading + sources

If you wish to read more about the tpm2-tools package and what it can do the mankier page is a great place to start.

### General Further reading:

It can be found here:

https://www.mankier.com/package/tpm2-tools

For further reading on encrypred swap you can find information on the Arch Wiki:

https://wiki.archlinux.org/index.php/Dm-crypt/Swap_encryption

More information for understanding on suspend and hibernation in linux can be found here:

https://wiki.archlinux.org/index.php/Power_management/Suspend_and_hibernate

### Sources and more further reading

Most of the shell included is not mine and is made by other people, I have just combined various bits to fit my needs you can find links to major sources below.

mkkeys.sh + more information on secure boot:

https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html

GenStub + GenTPMPolicy:

https://medium.com/@pawitp/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704

https://medium.com/@pawitp/its-certainly-annoying-that-tpm2-tools-like-to-change-their-command-line-parameters-d5d0f4351206

https://medium.com/@pawitp/the-correct-way-to-use-secure-boot-with-linux-a0421796eade 

install/encrypt-tpm:

https://bbs.archlinux.org/viewtopic.php?id=248836
https://github.com/archont00/arch-linux-luks-tpm-boot

Longer descritptions are soon to come.
