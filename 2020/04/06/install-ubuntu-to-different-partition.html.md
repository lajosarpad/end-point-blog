---
author: Bharathi Ponnusamy
title: Installing Ubuntu 18.04 to a different partition from an existing Ubuntu installation
tags: linux, ubuntu, update, open-source, sysadmin, devops, scalability, chef
gh_issue_number: 1617
---

![Clean setup](/blog/2020/04/06/install-ubuntu-to-different-partition/banner.jpg)

[Photo](https://unsplash.com/photos/4pPzKfd6BEg) by [Patryk Grądys](https://unsplash.com/@patrykgradyscom) on [Unsplash](https://unsplash.com)

Our Liquid Galaxy systems are running on Ubuntu 14.04 LTS (Trusty). We decided to upgrade them to Ubuntu 18.04 LTS (Bionic) since Ubuntu 14.04 LTS reached its end of life on April 30, 2019.

### Upgrading from Ubuntu 14.04 LTS

The recommended way to upgrade from Ubuntu 14.04 LTS is to first upgrade to 16.04 LTS, then to 18.04 LTS, which will continue to receive support until April 2023. Ubuntu has LTS -> LTS upgrades, allowing you to skip intermediate non-LTS releases, but we can’t skip intermediate LTS releases; we have to go via 16.04, unless we want to do a fresh install of 18.04 LTS.

14.04 LTS -> 16.04 LTS -> 18.04 LTS

For a little more longevity, we decided to do a fresh install of Ubuntu 18.04 LTS. Not only is this release supported into 2023 but it will offer a direct upgrade route to Ubuntu 20.04 LTS when it’s released in April 2020.

### Installing Clean Ubuntu 18.04 LTS from Ubuntu 14.04 LTS

#### Install debootstrap

The [debootstrap](https://linux.die.net/man/8/debootstrap) utility installs a very minimal Debian system. Debootstrap will install a Debian-based OS into a sub-directory. You don’t need an installation CD for this. However, you need to have access to the corresponding Linux distribution repository (e.g. Debian or Ubuntu).

```bash
/usr/bin/apt-get update
/usr/bin/apt-get -y install debootstrap
```

#### Creating a new root partition

Create a logical volume with size 12G and format the filesystem to ext4:

```bash
/sbin/lvcreate -L12G -n ROOT_VG/ROOT_VOLUME
/sbin/mkfs.ext4 /dev/ROOT_VG/ROOT_VOLUME
```

#### Mounting the new root partition

Mount the partition at `/mnt/root18`. This will be the root (/) of your new system.

```bash
/bin/mkdir -p "/mnt/root18"
/bin/mount /dev/ROOT_VG/ROOT_VOLUME /mnt/root18
```

#### Bootstrapping the new root partition

Debootstrap can download the necessary files directly from the repository. You can substitute any Ubuntu archive mirror for `ports.ubuntu.com/ubuntu-ports` in the command example below. Mirrors are listed [here](https://wiki.ubuntu.com/Mirrors).

Replace $ARCH below with your architecture: amd64, arm64, armhf, i386, powerpc, ppc64el, or s390x.

```bash
/usr/sbin/debootstrap --arch "$ARCH" "$DISTRO" "$ROOT_MOUNTPOINT”
/usr/sbin/debootstrap --arch "amd64" "bionic" "/mnt/root18"
```

#### Installing fstab

This just changes the root (/) partition path in the new installation while keeping the `/boot` partition intact. For example, `/dev/mapper/headVG-root /` -> `/dev/mapper/headVG-root18 /`. Since device names are not guaranteed to be the same after rebooting or when a new device is connected, we use UUIDs (Universally Unique Identifiers) to refer to partitions in fstab. We don’t need to use UUIDs for logical volumes since they can’t be duplicated.

```bash
OLD_ROOT_PATH="$(awk '$2 == "/" { print $1 }' /etc/fstab)"
/bin/sed "s:^${OLD_ROOT_PATH}\s:/dev/mapper/headVG-root18 :" /etc/fstab > "/mnt/root18/etc/fstab"
```

#### Mounting things in the new root partition

Bind `/dev` to the new location, then mount `/sys`, `/proc`, and `/dev/pts` from your host system to the target system.

```
/bin/mount --bind /dev "/mnt/root18/dev"
/bin/mount -t sysfs none "/mnt/root18/sys"
/bin/mount -t proc none "/mnt/root18/proc"
/bin/mount -t devpts none "/mnt/root18/dev/pts"
```

#### Configuring apt

Debootstrap will have created a very basic `/mnt/root18/etc/apt/sources.list` that will allow installing additional packages. However, I suggest that you add some additional sources, such as the following, for source packages and security updates:

```
/bin/echo "deb http://us.archive.ubuntu.com/ubuntu bionic main universe
deb-src http://us.archive.ubuntu.com/ubuntu bionic main universe
deb http://security.ubuntu.com/ubuntu bionic-security main universe
deb-src http://security.ubuntu.com/ubuntu bionic-security main universe" > /mnt/root18/etc/apt/sources.list
```

Make sure to run `apt update` with `chroot` after you have made changes to the target system sources list.

Now we’ve got a real Ubuntu system, if a rather small one, on disk. `chroot` into it to set up the base configurations.

```
LANG=C.UTF-8 chroot /mnt/root18 /bin/bash
```

### Installing required packages and running chef-client

As we are maintaining most of the Liquid Galaxy configuration and packages with [Chef](https://www.chef.io/), we need to install chef-client, configure it on the new target system, and run chef-client to complete the setup.

Copy the chef configuration and persistent net udev rules into place:

```
cp -a /etc/chef "/mnt/root18/etc/"
cp /etc/udev/rules.d/70-persistent-net.rules /mnt/root18/etc/udev/rules.d/
```

Install and run chef-client and let it set up our user login:

```
/bin/cat << EOF | chroot "/mnt/root18"
/usr/bin/apt-get update && /usr/bin/apt-get install -y curl wget
/usr/bin/curl -L https://omnitruck.chef.io/install.sh | /bin/bash -s -- -v 12.5.1
/usr/bin/chef-client -E production_trusty -o 'recipe[users]'
EOF
```

Next, chroot and install the required packages:

```
cat << EOF | chroot "$ROOT_MOUNTPOINT"
/bin/mount /boot
/usr/bin/apt-get update && /usr/bin/apt-get install -y --no-install-recommends linux-image-generic lvm2 openssh-server ifupdown net-tools
/usr/sbin/locale-gen en_US.UTF-8
EOF
```

### Set Ubuntu 14.04 to boot default

Back up the current trusty kernel files into `/boot/trusty` and create a custom menu entry configuration for Ubuntu 14.04 on 42_custom_trusty. Update `/etc/default/grub` to set Ubuntu 14.04 as the default menu entry and run `update-grub` to apply it to the current system. This will be used as a fail-safe method to run Trusty again if there is a problem with the new installation.

```bash
mkdir -vp /boot/trusty
cp -v /boot/*-generic /boot/trusty/
sed -i 's/GRUB_DEFAULT=.*/GRUB_DEFAULT="TRUSTY"/' /etc/default/grub
update-grub
```

Create the custom menu entry for Ubuntu 14.04 and Ubuntu 18.04 on the target system.

```bash
mkdir -p /mnt/root18/etc/grub.d
cat 42_custom_template > /mnt/root18/etc/grub.d/42_custom_menu_entry
```

`chroot` into the target system and run `update-grub`. This will also update the GRUB configuration to boot Ubuntu 14.04 as default and update the 0th menu entry to Ubuntu 18.04 (Bionic)

```bash
cat << EOF | chroot "/mnt/root18"
update-grub
EOF
```

### Boot into Bionic

To boot into Ubuntu 18.04 (Bionic), reboot the system after `grub-reboot bionic` and test if the bionic system is working as expected.

```bash
$ grub-reboot bionic
$ reboot
```

Reboot and test our new 0th GRUB entry:

```bash
$ grub-reboot 0
$ reboot
```

A normal reboot returns to Ubuntu 14.04 (Trusty) since the default menu entry is still set to Ubuntu 14.04 (Trusty).

### Set Ubuntu 18.04 to boot default

To set our new Ubuntu 18.04 installation as the default menu entry, change GRUB_DEFAULT to 0 in `/etc/default/grub` and run `update-grub` to apply it. The next reboot will boot into Ubuntu 18.04.

```
sed -i 's/GRUB_DEFAULT=.*/GRUB_DEFAULT=0/‘ /etc/default/grub
update-grub
```

Congratulations! You now have a freshly installed Ubuntu 18.04 system.