# Encrypted Hybrid USB Arch Linux Installation
- Encrypted Root partition with LUKS
- Removed Journaling
- Works on any boot method (UEFI or BIOS)
- Works on both Intel and AMD
- Works on (almost) any GPU drivers

HUGE NOTE: This is a guide for myself, with a little added explaination so you can install and do whatver you want while reading this. 

### Step 1: Connecting to Wifi
```# iwctl```
Showing devices
```[iwd]# device list```
Scanning the device
```[iwd]# station <device_name> scan```
Fetching network list
```[iwd]# station <device_name> get-networks```
Connecting to WiFi
```[iwd]# station <device_name> connect <wifi_name>```
After successful connection, exit.
```[iwd]# exit```
Check connection with
```# ping archlinux.org```

### Step 2: Updating databases and setting mirrors
Setting up mirror with reflector
```# reflector -c India -a 6 --sort rate --save /etc/pacman.d/mirrorlist```

Refreshing servers
```pacman -Syy```

### Step 3: Partitioning Disks
Partition disks with `gdisk`:

```# gdisk /dev/sdX```

Setting up partitions: (Please research on this.. the partitions below are examples you can use as default.)
```
o
y
n
hit enter
hit enter
+10M
ef02
n
hit enter
hit enter
+250M
ef00
n
hit enter
hit enter
hit enter
hit enter
w
y
```
Partition has been written to disk. Check by typing `lsblk`.

### Step 4: Setting up Filesystem types

Setting up FAT32 for EFI boot.
```# mkfs.fat -F32 /dev/sdX2```

Setting up **encrypted** root.
```# cryptsetup luksFormat /dev/sdaX3```
then follow the prompts that follow.  

Opening the partition.. (NOTE: you can replace `cryptroot` with any name you want. eg: encdisk, donkey, etc.)
```# cryptsetup open /dev/sdaX3 cryptroot```

Formatting Filesystem..
```# mkfs.ext4 -O "^has_journal" /dev/mapper/cryptroot```

### Step 5: Mounting Filesystems
Mounting root filesystem
```# mount /dev/mapper/cryptroot /mnt```
Creating boot directory for efi
```# mkdir /mnt/boot```
Mounting efi filesystem
```# mount /dev/sdX2 /mnt/boot```

### Step 6: Installing base packages
```# pacstrap /mnt base base-devel linux linux-firmware linux-headers vim nano git```

### Step 7: Generating Filesystem Table
```# genfstab -U /mnt >> /mnt/etc/fstab```

--

Exiting the installer iso and chrooting into the system:
```# arch-chroot /mnt```

### Step 8: Essential Housekeeping
**Localization:** Find your timezone  
Note: If your city doesnt show up, try your continent. Eg: for India, the timezone will be Asia/Kolkata
```# timedatectl list-timezones | grep <your_city_name> ```

Localization: setting up the timezone
```
# ln -sf /usr/share/zoneinfo/your/timezone /etc/localtime
# hwclock --systohc
```

Setting up your Locale
```# vim /etc/locale.gen```
Then, uncomment the line with `en_US.UTF-8 UTF-8`.  
Create locale with `# locale-gen`

Put your language in with..
```# vim /etc/locale.conf```
then type `LANG=en_US.UTF-8`  

**Hostname:**
Directly enter your hostname in ... (eg: archusb)
```# vim /etc/hostname```

**Hosts:**
```# vim /etc/hosts```
At the end of the file, type the following
```
127.0.0.1       localhost
::1             localhost
127.0.1.1       yourhostname.localdomain       yourhostname
```

**Setting root password:**
```# passwd```

### Step 9: Installing packages and GRUB
Installing essential packages
```# pacman -S grub efibootmgr dosfstools networkmanager network-manager-applet mtools reflector ntfs-3g```

Optional packages I install:
**For bluetooth:** `bluez bluez-utils`
**XDG Utilities:** `xdg-utils xdg-user-firs`
**For battery (on laptop):** `acpi acpid tlp`
**For printing:** `cups` and for HP Printers, I also recommend `hplip`
**For security: (firewalls, etc)** `firewalld apparmor`
**Other network utilities:** `wpa_supplicant dnsutils dnsmasq`
**For linux audio:** `alsa-utils pipewire pipewire-alsa pipewire-pulse pipewire-jack`
**For ssh:** `openssh`

Working on the mkinitcpio file.
```# vim /etc/mkinitcpio.conf```

Scroll down to the `HOOKS=(..)` line. Make sure that the line looks exactly like the one below (yes, even the order of elements). Then save and exit the file.
```HOOKS=(base udev block encrypt filesystems keyboard fsck)```

Then regenerate the file.
```# mkinitcpio -p linux```

Then installing **Grub for Legacy Systems (MBR)**
```grub-install --target=i386-pc --boot-directory=/boot /dev/sdX --recheck```

Installing **Grub for EFI Systems (UEFI)**
```grub-install --target=x86_64-efi --efi-directory=/boot --boot-directory=/boot --removable --recheck```

### Step 10: Final steps with GRUB
We need the UUID for the root partition. To grab that, type the following command.

```# blkid | tee -a uuidfile```

Then,
```# vim uuidfile```

After the file opens, move your cursor to the beginning of the line with the UUID of the root partition.In my case, it was something like
```/dev/sda3: UUID="..." TYPE=".." ...```
After your cursor is at the start of this line, press `<ESC>` on your keyboard. Then, press `yy`. This will copy the file on your keyboard.

Nano users, sorry. Switch to vim :)

Now, Editing the GRUB file for encyption
```# vim /etc/default/grub```

Then, go to the end of the line GRUB_CMDLINE_LINUX_DEFAULT (before the `"`). Once you are there, type the following..

```GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID="```
after the `=` sign of the UUID, press `<ESC>` on your keyboard and paste the line you had copied by pressing `p` on your keyboard.

Once that line has been pasted after the `=`, erase everything before and after the UUID itself (ie, erase all keywords and quotation marks of the UUID itself. The UUID should be somehting like 8b00sdf-XXXX-XXXX-XXX). Then, after that, type
`:cryptroot root=/dev/mapper/<your_mapper_name>`. Your GRUB line should look something like this..

```GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=8b8dxxx-XXX-XXX-XXXX:cryptroot root=/dev/mapper/<your_mapper_name>"```

**Setting your GRUB Resolution:**
Sometimes, GRUB doesn't want to use the best resolution for your display. To fix that, scroll to the line with `GRUB_GFX_MODE=auto`. Replace the word auto or whatever may be there with your proper resolution. In my case, the line was:
```GRUB_GFX_MODE=1920x1080```

**Generate the GRUB Configuration file:**
Type:
```# grub-mkconfig -o /boot/grub/grub.cfg```

### Step 11: Enabling Services
Enabling Network..
```# systemctl enable NetworkManager```
Enabling Optional services (if you installed them):
**Bluetooth**
```# systemctl enable bluetooth```
**CUPS (Printing)**
```# systemctl enable cups.service```
**SSH**
```# systemctl enable sshd```
**tlp**
```# systemctl enable tlp```
**ACPI Services**
```# systemctl enable acpid```
**Firewall**
Note: you should research into firewalld and set up some commands after enabling this. Simply enabling firewall won't protect your system. You'll have to set it up. Same with AppArmor.
```# systemctl enable firewalld```
**AppArmor**
```# systemctl enable apparmor```

### Step 12: Creating users.
Creating the user..
```# useradd -mG wheel <username>```
Giving password to the user..
```# passwd <username>```
To execute commands with sudo, do the following:
```EDITOR=vim visudo```
then, scroll to the line
```# %wheel ALL=(ALL) ALL```
and uncomment it.  

Adding users to the sudoers file..
```# echo "<username> ALL=(ALL) ALL" >> /etc/sudoers.d/<username>```

### Step 13: Disabling Journaling
We're installing this on a USB, which has limited write cycles. To stop them from being wasted on Journaling, we're going to edit some lines in a file.
NOTE: If a system is shutdown incorrectly, files added or changes made during the session may not save. You will have to properly shut down the system before unplugging the USB everytime.  

Go to:
```# vim /etc/systemd/journald.conf```

Then, uncomment the line `#Storage=auto` and change it to say `Storage=volatile`. Similarly, uncomment the line `#RuntimeMaxUse=` and change it to say `RuntimeMaxUse=50M`.

### Step 14: Exiting the chroot and other stuff..
To exit the chroot type
```exit```
Then, unmount all partitions
```umount -a```

Then, finally shutdown the system by typing `shutdown now`. If that doesnt work for some reason, then type `poweroff`.

Remove the USB with the Live ISO. Then boot into the USB with your Arch Linux install. You should see the GRUB screen if everything went correctly!
Congrats! You now have a base-install of Arch Linux on the USB!

## The Graphical Environment

If you have followed so far and wish to deviate from the guide from this point out, you can. You can install your own set of packages and drivers to make your own system. In this guide however, i'm going to follow some general recommendations with a portable USB and install `LeftWM`, A Window Manager written in Rust. You can install your own DE or WM, obviously. There are plenty of guides out there to support you. 

### Step 1: The Drivers
According to the Wiki, its better to not use any proprietary drivers for the portable system. Also, its of good practice to have as many drivers as possible so you can plug this USB onto any machine you find and have it work properly (obviously has to be of x86_64 architecture).  

If you want to have this system running on machines of both CPU vendors (Intel and AMD, install microcode for both of them.)  

Installing Microcode for both vendors:
```$ sudo pacman -S intel-ucode amd-ucode```

Installing Video Drivers:
```$ sudo pacman -S xf86-video-intel xf86-video-amdgpu xf86-video-noveau xf86-video-ati xf86-video-vesa xf86-video-fbdev```

### Step 2: Installing the Graphical Display Environment
This is where you can deviate healthily into your own stuff. As an example, this will be my setup. Note that I obviously have more applications with me, this is for example.

```$ sudo pacman -S xorg xf86-input-libinput libinput lightdm leftwm alacritty firefox feh```

I encourage you to look into graphics yourself. Just running the above command won't get you a working graphical session. There are a plethora of articles on the Wiki as well as tutorials on YouTube and Odysee.

# Final Congrats!
Finally! Congratulations on installing everything you need! You now should have a working USB with a graphical system. Now, enjoy your journey Young Jedi!