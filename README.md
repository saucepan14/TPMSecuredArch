# TPM Secured Arch
This repo contains various files used in Arch Linux root partition to be securley encrypted and unlocked without requiring a password. It uses a luks based encryption and adds a systemd unit to the initramfs to get the key from the tpm. This page assumes you have installed Arch before and are comforartable installing it on your own. It also assumes you have a tpm in your machine, you have access to controlling secure boot keys and you can boot with UEFI mode as I am only using UEFI for this. I am also using systemd boot instead of grub in my case however, if you are using grub you may still be able to apply some of this to your setup.

## Important notes to consider
I am working on having /boot be part of the encrypted file system and only having a seperate partition for /EFI conataning only the EFI stub so that you will not need to worry about someone modify the initramfs, vmlinuz-linux and cmdline files that would reside on the noramlly unencrypted boot partition. Should my theory work you may not need to install a boot loader at all! this will also allow for securley regenerating the EFI stub should it fail the secure boot check. I have not made these changes yet but feel free to try this on your own just understand that some of the provided files will need to be modified in order to get it working. I have also not yet experimented with an encrypted swap partition while this might not be a concern to a desktop with large ammounts of memory this is a concern if you do not have a lot of spare memory or if you are using a laptop and want to safley suspend to disk. I will be working on both of these issues and will post my findings. I also wish to explore setting up tpm based decryption on other distros mainly debian based distros and opensuse.

## Creating an encrpyted root partition
When first beging the install process for Arch you will need an encrpyted root partition this can be done with the following command:
```
cryptsetup luksFormat /dev/[the partiion you wish to encrypt]
```
Please note this will erase your data on that partition so be sure to copy and imporatant data somewhere.

## Mounting the root partition and generating some of the imporant files
You will next need to decrpyt the now encrypred partition so you can mount it and install arch onto it
You can decrypt the partition with the follwing
```
cryptsetup open --type luks /dev/[the name of you encrypted partion i.e. /dev/sda2, /dev/sdb3, /dev/nvme0n1p2, etc]
```
Then mount the now decrypted partion and continue installing Arch as normal
Mount with this command
```
mount /dev/mapper/root /mnt
```
After you have arch-chrooted into the now mounted partion you will need to be sure to generate /etc/fstab using the following:
```
genfstab -U -p /mnt /mnt/etc/fstab
```
Next you will need to create the file /etc/crypttab.initramfs and this line:
```
root    UUID=[the uuid of the encrypted partition]     /crypto_keyfile.bin    luks
```
Before you finish the install you will need to make some changes to /etc/mkinitcpio.conf
You will need to change the the Hooks() to be like this
```
HOOKS(base systemd autodetect sd-vconsole modconf block sd-encrpyt filesystem keyboard fsck)
```

We will still need to modify the config file again but this is all the is needed for now.

## Installing the boot loader and creating the EFI stub
Depending on how you choose to have your system configured you may neeed to install and configure systemd boot (i.e bootctl) I will not be going over this here but I will link some articles to help point you in the right direction. Just be sure to set the kernel paramters to set use the /dev/mapper/root as the root file system
https://wiki.archlinux.org/index.php/Systemd-boot#Installation
https://wiki.archlinux.org/index.php/Systemd-boot#Configuration

However I belive it is not entierly nessacry to use a bootloader if you do not wish to have multiple operating systems you want to switch between on the same compuer. If you are soley wanting to you use Arch as your os and do no plan on having other operating systems installed on your computer you can just boot the EFI stub directly.

After this we create and efi stub which bundles the initramfs, vmlinuz-linux, kernel paramaters and information about the os into a single EFI file.
You can do this by simply running the GenStub shell script, this will assume that the kernel paramters are stored in a file under /boot/cmdline, it also assumes that the initramfs and vmlinuz-linux are under /boot/ as well. It will output an EFI file under /boot/EFI/Linux/Linux.efi. If you wish to have only the stub exposed unencrypted then you will need /boot/ to be with in the root partition. You will additionally need to modify the final line and change if from 
```
"/usr/lib/systemd/boot/efi/linuxx64.efi.stub" "/boot/EFI/Linux/Linux.efi"
```
To
```
"/usr/lib/systemd/boot/efi/linuxx64.efi.stub" "/EFI/Linux/Linux.efi"
```

After that you can boot in to your new install and get started working and adding the key file and adding policies to the tpm

## Creating keys to be used in secure boot and signing the EFI stub

Once you have craeted an EFI stub you will need to sign it so secure boot can verify that it is trusted by you.
The first step is to run the mkkeeys.sh script this will create a total of 17 files with 3 of them being secret keys that you need to keep somewhere secure. If they are not secured properly then someone and sign an os boot into it and get the keyfile to your drive which is obviously bad. 
Of note is that mkkeys.sh will create the keys in the directory that the script is run so if you run the script in ~/ then it will generate all the keys and associtated files and place them in your home directory. While not necsacerally a bad thing you might want to make a new directory and cd into it and run the script from there to avoid unnecassary clutter. 

Once you have your keys and the cert files you next need to sign the EFI stub, this can be done with the following command:
```
sbsign --key ~/SecurebootKeys/DB.key --cert ~/SecurebootKeys/DB.crt \
--output /boot/EFI/Linux/Linux.efi /boot/EFI/Linux/Linux.efi
```
Please note the location of where the keys are being pulled from in my case they are in the ~/Secureboot/ folder, you may have yours in a different place. Also if you wish to have the signed EFI file seperate from the non signed one and the --output paramater to what you want the signed version to be called i.e.
```
--output /boot/EFI/Linux/Linux-signed.efi
```

After this command is run you may get a warning along the lines of 
```
warning: data remaining[1231832 vs 1357089]: gaps between PE/COFF sections?
```
I do not know what these warnings are but they do not seem to cause issue. If you do understand these warning and know how to fix them or clearify what they please let me know and I will be sure to add it and credit you.

After you have created your keys and signed the EFI file you need to load the files with the *.cer, *.esl and *.auth extentions onto an unencrypted flash drive prefereably with FAT32 file system (you may be able to use a different file system but I have not tried to so test at your own risk). Once this is done you will need to get into your computers bios and first clear the existing keys. You want to clear the existing keys other wise and keys already trusted in secure boot will be able to be used to verify a binary that isn't signed by you and get your keyfile from the tpm. Once you have cleared the keys you need to add your personal keys. It is preferable to add the keys starting with the DB then the KEK and then the PK. I my case I used the *.esl files however your scenario may be different. After you have added the necascary files ensure secure boot is enabled and boot back into your Arch install. If you are unable to boot back into your Arch install your bios may have disabled non secure boot verified images from booting you can change this but be sure to doulbe check if you signed the EFI correctly and if you added the right files to secure boot.

Also to note the exact process of adding secure boot keys is bios specific and I can not provided a comprehensive guide for every system if you are unsure or are having trouble google is your friend.

## Creating the tpm policiy and adding a keyfile to luks

Next we will need to add a keyfile to the luks encrypted partition and store it in the tpm. We will be creating a 256 bit (i.e. 32 byte) keyfile using data from /dev/random this makes the keyfile very hard to brute force and takes away the trouble of having to create a strong keyfile on your own. A copy of the keyfile will be stored within the encrypted partition which is not a security concern as someone could only get a copy of the keyfile if they have access to your encrpyed partition in which case you have bigger problems. 
To create the keyfile use the follwing command:
```
dd if=/dev/random of=/root/crypto_keyfile.bin bs=32 count=1
```
This creates the key file in /root/. If you are worried about accidently overwriting it use:
```
chmod 0400 /root/crypto_keyfile.bin
```
To set it to read only.
The we will be adding the key to luks with the follwing command:
```
cryptsetup luksAddKey /dev/[name of the encrpyed partion i.e. /dev/sd1, /dev/nvme0n1p2, etc]
```
Next we need to store the keyfile in the tpm and seal it against the desiered pcrs, in our case pcrs 0 and 7. The GenTPMPolicy script will create the tpm policy and store the keyfile against the current pcrs. So you need to ensure secure boot is working properly and only trusts images singed by you. As the tpm policy takes the current values of the pcrs and uses them as rules to seal the keyfile. So if your secure boot is not identifing your current boot images as valied against the current keys then all other images that are not valid against the keys can also acess the key from the tpm. Once you have run the GenTPMPolicy script check that the key has been stored and can be read from.
To see if the keys has been stored:
```
tpm2_getcap handles-persistent
tpm2_readpublic -c 0x81000000
```
Note the paramter of -c may be different for you just take the value give from the first command and use it for -c

To read the keyfile:
```
tpm2_unseal -c 0x81000000 -p pcr:sha1:0,7 | hexdump
```
This will print the hex values of the keyfile which is a more readable form
You can then compare this to the original key file with
```
hexdump /root/crypto_keyfile.bin
```
If both of the outputed values are the same then you have sucessfully added the key to the tpm!

## Adding the mkinitcpio hooks and decrypting at boot
In order to have to system decrypt the root partition on it's own we need to add a custom hook to mkinitcpio. The hook and install are included under that initcpio folder. You will need to take the files from within them and place them in /etc/initcpio/hooks/ and /etc/initcpio/install respectively. You will then need to modify 2 lines in /etc/mkinitcpio.conf
You need to ensure the tpm related modules are loaders by modifying the MODULES() line to look like so.
```
MODUELS(tpm tpm_tis)
```
You will then need to modify the HOOKS() line again to add the custom hook. It should look something along the lines of this:
```
HOOKS(base systemd autodetect sd-vconsole modconf block encrypt-tpm sd-encrpyt filesystem keyboard fsck)
```

After you have done this regenearte the boot related images with
```
mkinitcpio -p linux
```

You will need to recreate the stub with GenStub script and of course sign it as well so secure boot will trust it. To sign it just follow the steps previously used to sign the EFI stub. After that you should be able to reboot and have you drive decrypted for you automatically!
If the system fails to decrpyt itself using the command:
```
journalctl -b | grep -i tpm
```
Should help point you in the right direction.

If in the future after you have had this setup working if the auto decryption does not work and prompts you for a password you should consider how much you trust the image. As someone could have modified it without you knowing and could have something malicious within it. If you do not trust the image completetly it would be best to load to a live usb, decrypt the drive and regenerate the EFI stub and sign it again and see if that fixes the issue. (This step assumes you have /boot under the encrypted partition and only /EFI is unencrypted, conatining only the EFI stub).

## Further reading
This section will go into more detail about each of the files and how exaclty they work with the system it is not currently filled out yet but I have plans to create largers write ups for each of the files. It will also includes links to a lot of sources where I got a majority of the code from alone with more helpful reading you can go through if you wish to learn more. If you are having trouble getting this setup you can conatact me and I will try to help, however I do not always check github so it might take me a little bit to get back. If you wish to help by clearing up anything I said here or by correcting things I may have gotten wrong please contact me so I can make those changes propmtley and avoid confusing and/or misguideing anyone else. There is still a lot more work to be done to this readme and I plan to work on that in another branch.
