# Linux full disk encryption with LUKS + LVM + TPM2.0

In this document we will explain the process to encrypt the disk partitions without boot partition. The passphrase to unlock the decryption process will be configurated on TPM2.0 chip that decrypts automatically when the boot processes requires.


## Pre-requisites
The distro was chosen is an Ubuntu 20.04 running on a Lenovo Thinkpad X270 that includes TPM 2.0 chip but any laptop that includes this chip will work as well.

We need to create the following partition schema: 
* 100 MB for efi partition
* 500 MB for boot partition
* An encrypted partition for include root and swap partitions on LVM (it's needed to able the laptop hibernate using swap partition)



## Installation

Running live CD, select the "Try Ubuntu" option and we can proceed to create the partition schema with the following steps:

Open a terminal and  change to root

Creating the partitions, for this example we will use a nvme0n1 SSD but you can change for your specific devices as a sda for example.
```bash
fdisk /dev/nvme0n1
```
Create the partitions as the picture bellow
 [ here will be a picture ] 

Change the type of the first partition to type:ef
[ here will be a picture ] 

Now we are able to create the LUKS partition.

```bash
cryptsetup luksFormat /dev/nvme0n1p3
```
Type a strong passphrase and confirm it. Don't forget to keep in a secure place this pass.

Now, we will decrypt the partition to continue the installation process.
```bash
cryptsetup luksOpen /dev/nvme0n1p3  nvme0n1p3_crypt
```

Add the partition decrypted to a LVM group

```bash
#Creating a physical LVM volume
pvcreate /dev/mapper/nvme0n1p3_crypt

#Adding the physical volume to volume group vglinux0
vgcreate vglinux0 /dev/mapper/nvme0n1p3_crypt

#Creating a logical volume for swap partition
lvcreate -L 4G -n "lv0-swap" vglinux0

#Creating a logical volume for root partition
lvcreate -l +100%FREE -n "lv1-root" vglinux0

#Confirm if the volumes are sucessfully created.
lvs
 
```

Now we can proceed with the installation process, clicking on Install Ubuntu [ here will be a picture ] 

Click on Something else when you asked about partitions and select the lv0-swap for swap partition and lv1-root for / partition

Select the efi partition and configure it type
Select the device for boot partition and configure it as a boot partition. 

[here will be a image ]

Proceed with the installation process and when you prompted about the process has been finished, you will click on "Continue Testing" to proceed with the some necessary configurations...

In Ubuntu 20.04 I had an error when the process of installation was finished, the system boot can't found the logical volumes and it didn't show the screen requesting the passphrase to decrypt the disk, showing us a error from initramfs. After looking a solutions for days I discovered that this issue happens because the initramfs doesn't have the module to decrypt the disk configured.
To fix it we will need to do the following settings: 


Still running the live CD, open a terminal and make sure that the LUKS device is decrypted.

Grep the UUID of LUKS device

```bash
root@ubuntu:~# blkid|grep LUKS
/dev/nvme0n1p3: UUID="4b206e76-1531-48ae-95be-ae0ed7a244c1" TYPE="crypto_LUKS" PARTUUID="21db499d-b87b-41c6-864f-04d1531cb083"
```
Decrypt the device
```bash
root@ubuntu:~# cryptsetup open UUID="4b206e76-1531-48ae-95be-ae0ed7a244c1" nvme0n1p3_crypt
Enter passphrase for /dev/disk/by-uuid/4b206e76-1531-48ae-95be-ae0ed7a244c1: 
```

Confirm the LUKS device is decrypted in /dev/mapper
```bash
root@ubuntu:~# ls /dev/mapper/*
/dev/mapper/control  /dev/mapper/ubuntu--vg-root  /dev/mapper/nvme0n1p3_crypt
```

Mount the logical volume that contains root partition. 
```bash
root@ubuntu:~# mount /dev/mapper/ubuntu--vg-root /mnt/ubuntu-root/
```

Mount necessary files and directories to run a chroot
```bash 
mount -o bind /sys /mnt/ubuntu-root/sys /
mount -o bind /proc /mnt/ubuntu-root/proc /
mount -o bind /dev /mnt/ubuntu-root/dev /
cp /etc/resolv.conf /mnt/ubuntu-root/etc/
```

Inicialize chroot mode
```bash
chroot /mnt/ubuntu-root/
```

Will be necessary install the package binwalk to see the content of the initramfs
```bash
apt update && apt install binwalk
```

Find offset of gzipped initramfs content
```bash
binwalk  /boot/initrd.img-5.8.0-63-generic | grep gzip
```

Add initramfs into /etc/crypttab looks like below

```bash
cat /etc/crypttab 
# <target name>	<source device>		<key file>	<options>
nvme0n1p3_crypt UUID=76fd6e8d-2bc9-4e25-aac2-30fa83fb9bce none luks,discard,initramfs
```

Add the option CRYPTSETUP=y in /etc/cryptsetup-initramfs/conf-hook, after that, update initramfs: 

```bash
update-initramfs -k 5.8.0-63-generic -c -v &> update-initramfs-5.8.0-63-generic.cryptsetup.log
```

Make sure if the modules are into initramfs with the command bellow 
```bash
# grep /sbin/cryptsetup update-initramfs-5.8.0-63-generic.cryptsetup.log
# grep dm-crypt.ko update-initramfs-5.8.0-63-generic.cryptsetup.log
```
Run again the update-initramfs without -c and -v options
```bash
#update-initramfs -k 5.8.0-63-generic -u
```
Put the swap partition in initramfs configuration as well
```bash
# blkid | grep swap
/dev/mapper/vglinux0-lv0--swap: UUID="17881a65-0753-4eac-a7ea-cbd514659c39" TYPE="swap"

# printf "RESUME=UUID=$(blkid | awk -F\" '/swap/ {print $2}')\n" | sudo tee /etc/initramfs-tools/conf.d/resume

```


# Configuring  TPM2.0 to decrypt disk automatically. 

We will need to install these packages to send the passphrase to TPM2.0 chip. 

```bash 
sudo apt update && sudo apt install clevis-luks clevis-tpm2 clevis-dracut clevis-initramfs tpm2-* -y 
 ```

Attach master key generated by TPM to the LUKS volume. We will use bellow a specific set of PCR (Platform Configurarion Registers)

Review information about the cryptographic setup of encrypted partition:
```bash
cryptsetup luksDump /dev/nvme0n1p3
```

Binding the Luks key token in TPM PCR7
```bash
clevis luks bind -d /dev/nvme0n1p3 tpm2 '{"pcr_ids":"7"}'
Enter existing LUKS password: ******
```

Review information about the crypto setup again, check if the new key has been added to the LUKS volume: 

```bash
cryptsetup luksDump /dev/nvme0n1p3
```

Now we can reboot the machine and test if the disk will be decrypted by TPM. (You will see a screen asking for the passphrase but after about 5 seconds the TPM will automatically decrypt. 


## Author
[Bruno Alves](https://github.com/balves7)

## References
https://www.mankier.com/1/clevis

[fit-pc.com](https://fit-pc.com/wiki/index.php?title=Linux:_Full_Disk_Encryption&mobileaction=toggle_view_mobile)

[askubuntu.com/questions](https://askubuntu.com/questions/1116778/how-to-set-the-resume-variable-to-override-these-issues)


[linux.com/training-tutorials](https://www.linux.com/training-tutorials/how-full-encrypt-your-linux-system-lvm-luks/_)

