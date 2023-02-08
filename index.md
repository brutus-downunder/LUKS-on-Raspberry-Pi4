## LUKS on Raspberry Pi with crypt unlock by Dropbear SSH

***Last updated: 08th February 2023.***


This guide explains how to encrypt the root partition of a USB SSD that has a fresh install of a 64bit OS on a Raspberry Pi 4 with LUKS. The process requires a Raspberry Pi 4 running Raspberry Pi 64bit OS on the USB SSD and a USB memory with capacity of at least 8GB (for a fresh install). It is important to have a backup of the USB SSD, in case anything goes wrong, because it will be overwritten, and also of the USB memory as well.

Full disk encryption is quite easy to perform with modern Linux distributions. Raspberry Pi is an exception because the boot partition does not include most of the needed programs and kernel modules. On the other hand, it is important to use disk encryption with a Raspberry Pi because the USB SSD can be removed from the unit and its content read quite easily.

Note: Before proceeding make sure you can type a | "pipe" with the keyboard connected to the USB port on your Raspberry Pi


Download latest Raspberry Pi4 Desktop 64bit OS from -
https://downloads.raspberrypi.org/raspios_arm64/images/

Download latest Raspberry Pi4 Server 64bit OS from - https://downloads.raspberrypi.org/raspios_lite_arm64/images/

Read Raspberry Pi4 64bit offical Forum - https://www.raspberrypi.org/forums/viewtopic.php?f=117&t=275370

Read Raspberry Pi 64bit Forum - https://www.raspberrypi.org/forums/viewtopic.php?f=63&t=275372


### Requirements

Linux kernel 5.0 or later. You can check it with this command:
```
uname -s -r
```

### Initial setup phase

Update and Upgrade
```
sudo apt update && sudo apt upgrade -y
```
Reboot
```
sudo reboot
```
Edit dhcpcd for a Static IP address
```
/etc/dhcpcd.conf
```
add your IP address, Gateway address and DNS server(s) address
```bash
interface eth0
    static ip_address=192.168.0.xxx/24
    static routers=192.168.0.xxx
    static domain_name_servers=192.168.0.xxx
```
Edit sshd_config for SSH 
```
/etc/ssh/sshd_config
```
changing Port and Authentication settings
```
Port 2122
PermitRootLogin yes
```
Copy your authorized_keys file to /home/pi/.ssh and set Permissions
```
sudo chmod 0600 /home/pi/.ssh/authorized_keys
```
Also copy it to /root
```
sudo cp -r /home/pi/.ssh /root
```
and set Permissions
```
sudo chmod 0600 /root/.ssh/authorized_keys
```
Restart the SSH Service
```
sudo service ssh restart
```
Set a root password
```
sudo passwd root
```
Reboot
```
sudo reboot
```
After rebooting, SSH into 192.168.0.xxx:2122 as pi. Make sure Pageant is loaded with your Privatekey. Only proceed if successful login with Hostkey authentication is achieved. i.e. No password is required or should be entered.

Edit sshd_config for SSH 
```
/etc/ssh/sshd_config
```
changing the Authentication setting to only allow Hostkey authentication
```
PasswordAuthentication no
```
### Starting the Install phase

Another Update
```
sudo apt update
```
You should install the programs needed:
```
sudo apt install -y busybox cryptsetup initramfs-tools dropbear-initramfs
```
Copy your authorized_keys file to /etc/dropbear-initramfs
```
sudo cp -r /home/pi/.ssh/authorized_keys /etc/dropbear-initramfs
```
and set Permissions
```
sudo chmod 0600 /etc/dropbear-initramfs/authorized_keys
```
Restart the SSH Service
```
sudo service ssh restart
```

The microprocessors of Raspberry Pi computers do not include the hardware for AES acceleration. That means that a software implementation of AES is needed, which is slower. Fortunately, there is a recent alternative, Adiantum, that runs fast in software.
Linux kernel 5.0 or later includes the cryptographic kernel modules needed for Adiantum. You can check that every module is present and loaded with this command:
```
cryptsetup benchmark -c xchacha20,aes-adiantum-plain64
```
The output, if everything is all right, will be like this:
```
# Tests are approximate using memory only (no storage IO).
#            Algorithm |       Key |      Encryption |      Decryption
xchacha20,aes-adiantum        256b       111.2 MiB/s       114.6 MiB/s
```
If the execution shows an error message complaining about the cipher not being available, either the kernel modules are not present or are not loaded. You can load the necessary kernel modules for that cipher with ‘modprobe’:
```
sudo modprobe xchacha20
sudo modprobe adiantum
sudo modprobe nhpoly1305
```
If ‘modprobe’ complains about the module not being found, the kernel module is not present. A possible cause can be that the kernel version is not 5.0 or later.

You could check AES and see the speed difference with Adiantum:
```
cryptsetup benchmark -c aes-xts-plain64
```
```
# Tests are approximate using memory only (no storage IO).
# Algorithm |   Key |      Encryption |      Decryption
aes-xts        256b        88.7 MiB/s        86.2 MiB/s
```

### Preparing Linux

‘initramfs’ has to be recreated when a new kernel is installed or just now that we are changing its configuration. We need to create a new file:
```
/etc/kernel/postinst.d/initramfs-rebuild
```
and it should have this content:
```bash
#!/bin/sh -e

# Rebuild initramfs.gz after kernel upgrade to include new kernel's modules.
# https://github.com/Robpol86/robpol86.com/blob/master/docs/_static/initramfs-rebuild.sh
# Save as (chmod +x): /etc/kernel/postinst.d/initramfs-rebuild

# Remove splash from cmdline.
if grep -q '\bsplash\b' /boot/cmdline.txt; then
  sed -i 's/ \?splash \?/ /' /boot/cmdline.txt
fi

# Exit if not building kernel for this Raspberry Pi's hardware version.
version="$1"
current_version="$(uname -r)"
case "${current_version}" in
  *-v8+)
    case "${version}" in
      *-v8+) ;;
      *) exit 0
    esac
  ;;
  *+)
    case "${version}" in
      *-v8+) exit 0 ;;
    esac
  ;;
esac

# Exit if rebuild cannot be performed or not needed.
[ -x /usr/sbin/mkinitramfs ] || exit 0
[ -f /boot/initramfs.gz ] || exit 0
lsinitramfs /boot/initramfs.gz |grep -q "/$version$" && exit 0  # Already in initramfs.

# Rebuild.
mkinitramfs -o /boot/initramfs.gz "$version"
```
The file should be made executable:
```
sudo chmod +x /etc/kernel/postinst.d/initramfs-rebuild
```
We also need to specify some programs that need to be included in ‘initramfs’. We do that with a new file at:
```
/etc/initramfs-tools/hooks/luks_hooks
```
and it should have this content:
```bash
#!/bin/sh -e
PREREQS=""
case $1 in
        prereqs) echo "${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec /usr/lib/aarch64-linux-gnu/libgcc_s.so.1 /usr/lib
copy_exec /sbin/resize2fs /sbin
copy_exec /sbin/fdisk /sbin
copy_exec /sbin/cryptsetup /sbin
```
The programs are ‘resize2fs’, ‘fdisk’ and ‘cryptsetup’.
The file should be made executable:
```
sudo chmod +x /etc/initramfs-tools/hooks/luks_hooks
```
‘initramfs’ for Raspberry Pi OS does not include kernel modules for LUKS and encryption by default. We need to configure the kernel modules to add. This file has to be edited:
```
/etc/initramfs-tools/modules
```
and the following lines with the names of kernel modules added:
```
algif_skcipher
xchacha20
adiantum
aes_arm
sha256
nhpoly1305
dm-crypt
```
Now we need to build the new ‘initramfs’:
```
sudo -E CRYPTSETUP=y mkinitramfs -o /boot/initramfs.gz
```
You can ignore the warning about ‘cryptsetup’ if the next checking is correct.

We can check that the programs are present in ‘initramfs’ with the following command:
```
lsinitramfs /boot/initramfs.gz | grep -P "sbin/(cryptsetup|resize2fs|fdisk)"
```
We can check that the kernel modules are present in ‘initramfs’ with the following command:
```
lsinitramfs /boot/initramfs.gz | grep -P "(algif_skcipher|chacha|adiantum|aes-arm|sha256|nhpoly1305|dm-crypt)"
```

### Preparing Boot
We need to modify some files before rebooting the Rasperry Pi. These changes are meant to tell the boot process to use an encrypted root filesystem. After the previous changes, the Raspberry Pi will boot correctly. After changing the following files, the Raspberry Pi will not boot until the whole process of encrypting the root partition and configuring LUKS is completed. If after modifying the next four files and before the root partition is encrypted anything goes wrong, the changes in the four files can be reverted and the Raspberry Pi would boot normally. It is a good idea to make a copy of the files before modifying them.

**File: /boot/config.txt**

The next line has to be appended at the end of the file:
```
initramfs initramfs.gz followkernel
```

**File: /boot/cmdline.txt**

It contains one line with parameters. One of them is ‘root’, that specifies the location of the root partition. For the Raspberry Pi it is usually ‘/dev/sda2’, but it can also be other device (or the same) specified as “PARTUUID=xxxxx”. The value of ‘root’ has to be changed to ‘/dev/mapper/crypted’. For example, if ‘root’ is:
```
root=/dev/sda2
```
it should be changed to:
```
root=/dev/mapper/crypted
```
also, at the end of the line, separated by a space, this text should be appended:
```
cryptdevice=/dev/sda2:crypted
```

**File: /etc/fstab**

The device for the root partition (‘/’) should be changed to the mapper.
For example, if the device for root is:
```
/dev/sda2
```
it should be changed to:
```
/dev/mapper/crypted
```

**File: /etc/crypttab**

At the end of the file a new line should be appended with this text:
```
crypted	/dev/sda2	none	luks
```
Add our "at boot time" SSH IP to the bottom of /etc/initramfs-tools/initramfs.conf
```
IP=192.168.0.xxx:::255.255.255.0::eth0:off
```
Configure dropbear with our "at boot time" settings
```
sudo chmod 646 /etc/dropbear-initramfs/config && sudo echo 'DROPBEAR_OPTIONS="-p 2222"' >> /etc/dropbear-initramfs/config && sudo chmod 644 /etc/dropbear-initramfs/config
```
and add one more dropbear setting
```
sudo chmod 646 /etc/dropbear-initramfs/config && sudo echo 'DROPBEAR=y' >> /etc/dropbear-initramfs/config && sudo chmod 644 /etc/dropbear-initramfs/config
```
We need to create a file to run at "our boot time" SSH login
```
/etc/initramfs-tools/hooks/cryptroot-unlock.sh
```
and it should have this content:
```bash
#!/bin/sh

if [ "$1" = 'prereqs' ]; then echo 'dropbear-initramfs'; exit 0; fi

. /usr/share/initramfs-tools/hook-functions

source='/tmp/cryptroot-unlock-profile'

root_home=$(echo $DESTDIR/root-*)
root_home=${root_home#$DESTDIR}

echo 'if [ "$SSH_CLIENT" ]; then /usr/bin/cryptroot-unlock; fi' > $source

copy_file ssh_login_profile $source $root_home/.profile

exit 0
```
and make it executable
```
sudo chmod +x /etc/initramfs-tools/hooks/cryptroot-unlock.sh
```

Everything is ready now to reboot. After rebooting, the Raspberry Pi OS will fail to start because we have configured a root partition that does not exist yet. After several similar messages indicating the failure, the ‘initramfs’ shell will show.

### Encrypting the root partition

In the ‘initramfs’ shell, we can check that we can use ‘cryptsetup’ with the kernel module ciphers:
```
cryptsetup benchmark -c xchacha20,aes-adiantum-plain64
```
If the test is completed successfully, everything is all right. Now we have to copy the root partition of the USB SSD to the USB memory. The idea is to have a copy of the root partition of the USB SSD in the USB memory, create an encrypted volume in the root partition (the content will be lost) and copy the root partition in the USB memory back to the encrypted partition of the USB SSD. Take into account that the previous content of the USB memory will be lost. Before doing that, we are going to check the root partition and reduce the size of its filesystem to the minimum possible, so that the copy operation takes less time. The command to check it and correct possible errors is:
```
e2fsck -f /dev/sda2
```
After finishing, we have to resize the filesystem. It is very important to take note of the size of the filesystem after resizing it, the number of blocks of 4k:
```
resize2fs -fM -p /dev/sda2
```
An example of the output of the program is:
```
Resizing the filesystem on /dev/sda2 to 2345678 (4k) blocks.
```
The number to take note of is ‘2345678’ in the example.
Once the resizing is finished we are going to get a checksum value of the whole partition. We will do the same after every copy operation between USB SSD and USB memory. If the checksum is the same, the copy will be equal. Let’s create the checksum for the root partition to copy. Remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem. It is useful to add “time ” before “dd” to obtain how long it takes the operation. When the rest of checksums are calculated, it allows us to know, more or less, how much time it will take:
```
time dd bs=4k count=XXXXX if=/dev/sda2 | sha1sum
```
Take note of the SHA1 value. 
Connect the USB memory to the Raspberry Pi now. Some information about the USB memory will be printed. If there is no other USB memory connected, the device for the memory will probably be /dev/sdb. It can be checked with:
```
fdisk -l /dev/sdb
```
We are going to use the whole USB memory, /dev/sdb.
This operation will copy the root filesystem of the USB SSD into the USB memory device, deleting all of its content (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/sda2 of=/dev/sdb
```
While copying to or from USB SSD and USB memories a message might appear indicating that task worker has been blocked for several seconds. It can be ignored if the checksums remain correct.

Now we have the original root filesystem and the copy. We also have the checksum of the original. We have to calculate the checksum of the copy and compare them. We can consider they are an exact copy if the checksums coincide. The command to calculate the checksum of the copy is (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/sdb | sha1sum
```
Assuming that the checksums are correct, now it is time to encrypt the root filesystem of the USB SSD, to create the LUKS volume using ‘cryptsetup’. There are many parameters and possible values for the encryption. These are the commands I have chosen:
```
cryptsetup --type luks2 --cipher xchacha20,aes-adiantum-plain64 --hash sha256 --iter-time 5000 –-key-size 256 --pbkdf argon2i luksFormat /dev/sda2
```
More information about the parameters can be found here:
<https://man7.org/linux/man-pages/man8/cryptsetup.8.html>

The command will ask for a passphrase twice (for confirmation). It is important that the passphrase is long and uses different character sets (letter, numbers, symbols, uppercase, lower case, etc.).
After creating the LUKS volume, we have to open it and copy the contents of the root filesystem into it. The command to open the LUKS volume is:
```
cryptsetup luksOpen /dev/sda2 crypted
```
It will ask for the passphrase chosen in the previous stage. Once opened, we can copy the root filesystem in the USB memory into the encrypted volume (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/sdb of=/dev/mapper/crypted
```
After it has copied, we have to calculate the checksum of the copy once more to validate it. If the checksums coincide, the copy is correct (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/mapper/crypted | sha1sum
```
In addition to the checksum check, we should also check the the filesystem of the LUKS volume:
```
e2fsck -f /dev/mapper/crypted
```
We copied a reduced filesystem. Now we have to expand it to the size of the USB SSD: Note: This may take a while depending on the USB SSD size.
```
resize2fs -f /dev/mapper/crypted
```
The process is nearly finished now. The USB memory can be removed because it is nolonger needed. Now we have to exit for the boot process to continue:
```
exit
```
The boot process will enter into the initramfs shell again. This time we have to open the LUKS volume we just created to make the root filesystem available:
```
cryptsetup luksOpen /dev/sda2 crypted
```
After opening the LUKS volume we have to exit again and the Raspberry Pi OS will start normally:
```
exit
```

### Booting
We don't want to enter into initramfs shell every time we switch on our Raspberry Pi. Using Dropbear we can make the Raspberry Pi OS ask for the crypt unlock passphrase via SSH. What we need for that is to build initramfs once more and then reboot:
```
sudo mkinitramfs -o /tmp/initramfs.gz
sudo cp /tmp/initramfs.gz /boot/initramfs.gz
```

After rebooting, SSH into 192.168.0.xxx:2222 as root. Make sure Pageant is loaded with your Privatekey.

A prompt message should appear, something like “Please unlock disk...”.

You should enter your crypt unlock passphrase, once it has unlocked, the SSH connection will be lost. This is normal since the OS is continuing to boot

After a short time (to allow for booting into the OS to complete) you can SSH into 192.168.0.xxx:2122 as root or pi
