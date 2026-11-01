# Arch Linux VM Installation Guide

**Author:** Evan Boss
**Course:** Sys Admin
**Host:** VMware Workstation
**ISO:** 'archlinux-2026.11.01-x86_64.iso'
**Hostname:** 'batcomputer' | **Users:** 'evan', 'codi'

A short guide for building my Arch Linux VM from scratch, following the [Arch Installation Guide](https://wiki.archlinux.org/title/Installation_guide)(no 'archinstall') 

---

## 1. Download and verify the ISO
Run on the **host** (not in the VM) from PowerShell.

```powershell
cd $HOME\Downloads
Get-Filehash .\archlinux-2026.11.01-x86_64.iso -Algorithm SHA256 # compute the ISO's hash
```

Compare the output to hash published at archlinux.org/download:

---

## 2. Create the VM
| Setting           | Choice                                                                  |
| ----------------- | ----------------------------------------------------------------------- |
| Guest OS          | Linux, Other Linux 6.x kernel 64-bit                                    |
| Name              | Arch                                                                    |
| Processors        | 2 processors, 1 core per processor                                      |
| Memory            | 2048 MB                                                                 |
| Network type      | NAT                                                                     |
| I/O controller    | LSI Logic (default)                                                     |
| Virtual disk type | SCSI (default)                                                          |
| Disk              | New virtual disk, 20 GB, stored as a single file, default file location |
Then click Finish. The ISO is attached under **Settings -> CD/DVD -> Use ISO image file**.

---

## 3. Boot and prepare the live environment

Power on and boot the installer. Everything form here until the first reboot runs inside the VM at the 'root@archiso' prompt.

```bash
cat /sys/firmware/efi/fw_platform_size # prints 64 if booted in UEFI mode
```
>**Problem:** this printed "No such file or directory" because the VM booted in BIOS mode. See [Problem 1](#problem-1-vm-booted-in-bios-instead-of-uefi) for the fix, then rerun the command. It should print '64'.

Then check the network, clock, and disk:

```bash
ping -c 3 archlinux.org  # confirm internet connectivity
timedatectl              # verify the clock is synchronized
lsblk                    # confirm the disk shows up (sda)
```

---

## 4. Partition, format, mount

### Partition

```bash
fdisk /dev/sda
```

| Keys                     | Purpose                                                                              |
| ------------------------ | ------------------------------------------------------------------------------------ |
| 'g'                      | Create a new GPT partition table                                                     |
| 'n', Enter, Enter, '+1G' | New 1 GB partition (EFI)                                                             |
| 't', '1'                 | Change its type to EFI System                                                        |
| 'n', Enter, Enter, Enter | New partition using the rest of the disk (root)                                      |
| 'p'                      | Print the table; check for a 1 GB EFI System and a larger Linux filesystem partition |
| 'w'                      | Write changes and exit                                                               |

```bash
lsblk # verify sda1 and sda2 exist
```

### Format

```bash
mkfs.fat -F 32 /dev/sda1 # FAT32 for the EFI partition (UEFI reads FAT)`

mkfs.ext4 /dev/sda2 # ext4 for the root partition`
```

### Mount

```bash
mount /dev/sda2 /mnt # mount root first

mount --mkdir /dev/sda1 /mnt/boot # create /mnt/boot and mount the EFI partition
```

Formatting builds a filesystem on a partition. Mounting attaches it to the directory tree so files can be written to it. Root must be mounted before '/boot'.

---

## 5. Install the base system

```bash
pacstrap -K /mnt base linux linux-firmware networkmanager vim sudo grub efibootmgr open-vm-tools

pacstrap -K /mnt nano # I originally installed vim, but I prefer nano, so I installed it too
```

| Package                           | Why                                               |
| --------------------------------- | ------------------------------------------------- |
| 'base', 'linux', 'linux-firmware' | Required: base system, kernel, hardware, firmware |
| 'networkmanager'                  | Networking                                        |
| 'vim', 'nano'                     | Text editors                                      |
| 'sudo'                            | Sudo access for users                             |
| 'grub', 'efibootmgr'              | UEFI bootloader                                   |
| 'open-vm-tools'                   | VMware integration (clipboard, display resizing)  |

---

## 6. Configure the system

```bash
genfstab -U /mnt >> /mnt/etc/fstab # save the mount layout by UUID (>> appends; > would overwrite)

cat /mnt/etc/fstab # check that / and /boot are listed

arch-chroot /mnt # switch into the new system
```

> **Note:** once inside the chroot, the path is '/etc/fstab', not '/mnt/etc/fstab'. See [Problem 2](#problem-2-cat-mntetcfstab-says-no-such-file-or-directory).

### Time zone

```bash
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime # set Central time

hwclock --systohc # sync the hardware clock
```

### Localization

```bash
nano /etc/locale.gen # uncomment "en_US.UTF-8 UTF-8"

locale-gen # generate the locale

echo "LANG=en_US.UTF-8" > /etc/locale.conf # set the system language
```

### Hostname

```bash
echo "batcomputer" > /etc/hostname
```

### Root password and users

```bash
passwd # set the root password

useradd -m -G wheel evan # create my user with a home directory and the wheel group

passwd evan # set my password

EDITOR=nano visudo # uncomment "%wheel ALL=(ALL:ALL) ALL" so wheel members can use sudo
```

### Bootloader and services

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB # install GRUB to the EFI partition

grub-mkconfig -o /boot/grub/grub.cfg # generate the GRUB config

systemctl enable NetworkManager # start networking at boot

systemctl enable vmtoolsd # start VMware tools at boot
```

### Finish and reboot

```bash
exit # leave the chroot

umount -R /mnt # unmount everything under /mnt

poweroff
```

Before powering on again, disconnect the ISO: **VM Settings -> CD/DVD**, then uncheck "Connect at power on".

---

## First boot

Power on and login as root.

```bash
ping -c 3 archlinux.org # confirm networking still works

pacman -Syu # update the system

id evan # confirm my user exists

sudo whoami # (as evan) should print: root
```

I took a **VMware snapshot** here as a checkpoint

---

## 8. Assignment requirements

### Desktop environment (LXQt + SDDM)

```bash
sudo pacman -S xorg-server lxqt sddm breeze-icons gtkmm3 mesa
```

At the group prompt I pressed Enter to install all 'lxqt' members. I chose LXQt because that is what was recommended.

### Boot into the GUI

```bash
sudo systemctl enable sddm # start the login manager at boot

sudo systemctl set-default graphical.target # boot into graphical mode

sudo reboot
```

### Second user: codi (with sudo)

```bash
sudo useradd -m -G wheel codi # create codi with a home directory and sudo via wheel

sudo passwd codi # set codi's password

id codi # confirm wheel appears in the groups

su - codi # switch to codi

sudo whoami # should print: root

exit # back to my user
```

### Additional shell: zsh

```bash
sudo pacman -S zsh # install zsh
```

I kept bash as my default shell.

### SSH

```bash
sudo pacman -S openssh # install OpenSSH

sudo systemctl enable --now sshd # enable at boot and start now

systemctl status sshd # should show active (running)

ssh localhost # test a login to myself
```

### Terminal colors and aliases

I configured both shells. Arch's default '.bashrc' already colors 'ls' and 'grep', so I only changed 'PS1' and added the missing aliases.

**Zsh** ('nano ~/.zshrc', which starts empty):

```zsh
# Color prompt: green user@host, blue directory

autoload -U colors && colors

PS1='%F{green}%n@%m%f %F{blue}%~%f %# '

alias ls='ls --color=auto'

alias grep='grep --color=auto'

alias diff='diff --color=auto'

alias ip='ip -color=auto'

alias ll='ls -lah'

alias update='sudo pacman -Syu'

alias ..='cd ..'

alias ports='ss -tulpn'

alias myip='ip -br a'
```

**Bash** ('nano ~/.bashrc'): replaced the existing 'PS1' line and added the rest at the bottom:

```bash
# Color prompt: green user@host, blue directory

PS1='\[\e[32m\]\u@\h\[\e[0m\] \[\e[34m\]\w\[\e[0m\] \$ '

alias diff='diff --color=auto'

alias ip='ip -color=auto'

alias ll='ls -lah'

alias update='sudo pacman -Syu'

alias ..='cd ..'

alias ports='ss -tulpn'

alias myip='ip -br a'
```

Apply and test:

```bash
source ~/.bashrc # apply without logging out

alias # list all aliases

echo $PS1 # show the prompt string

ll # colored long listing

myip # show IP address
```

To check zsh, launch it and test:

```bash
zsh

echo $0 # prints zsh

ll

myip

exit # return to bash
```

**Pacman** output color:

```bash
sudo sed -i 's/^#Color/Color/' /etc/pacman.conf # uncomment Color

grep '^Color' /etc/pacman.conf # verify
```

**Copy the configs to codi:**

```bash
sudo cp ~/.bashrc /home/codi/.bashrc

sudo cp ~/.zshrc /home/codi/.zshrc

sudo chown codi:codi /home/codi/.bashrc /home/codi/.zshrc # make codi the owner
```

---

## 9. Final verification

```bash
systemctl is-enabled sddm sshd NetworkManager # each should print enabled

systemctl get-default # graphical.target
```

| Requirement                    | How I verified                          |
| ------------------------------ | --------------------------------------- |
| Desktop environment            | LXQt session loads                      |
| Boots into GUI                 | SDDM login screen at boot               |
| Users: evan and codi with sudo | 'sudo whoami' prints 'root' for both    |
| Shell other than bash          | 'zsh --version', launch with 'zsh'      |
| SSH                            | 'ssh localhost'                         |
| Terminal color                 | Colored prompt and 'ls' in bash and zsh |
| Aliases                        | 'll', 'myip', 'ports' work              |

---

## Problems and fixes

### Problem 1: VM booted in BIOS instead of UEFI

- **Symptom:** 'cat /sys/firmware/efi/fw_platform_size' said "No such file or directory". 
- **Cause:** The VM firmware was set to BIOS.
- **Fix:** Powered off. In **VM -> Settings -> Options -> Advanced**, set Firmware type to **UEFI**, confirmed Secure Boot was unchecked, clicked OK, powered on, and picked: "Arch Linux install medium (x86_64, UEFI)" at the boot menu. The command then printed '64'.
- Make sure to check UEFI mode before partitioning. Nothing had been written to the disk yet, so nothing was lost.

### Problem 2: 'cat  /mnt/etc/fstab' says "No such file or directory"

- **Symptom:** The file was not found after running 'genfstab'.
- **Cause:** I was already inside the chroot, where the new system's root is '/', so the path is '/etc/fstab'.
- **Fix:** Used 'cat /etc/fstab', which showed both entries. I did not rerun 'genfstab', which would have added duplicate lines.
- Remember that paths differ inside and outside the chroot.

### Problem 3: Prompt showed '[e[32m' instead of color

- **Symptom:** The bash prompt printed raw characters like '[e[32m' before the username. 
- **Cause:** The escape sequences in 'PS1' were mistyped. Color codes need '\e' with a backslash, and each color block must be wrapped in '\[ ... \]'. 
- **Fix:** Replaced the line with the correct color code and ran 'source ~/.bashrc'.
- Make sure to double check for error when writing to files.