# Linux on the Google Pixelbook
Installing and using Linux on the Pixelbook (internally called "eve") comes with some challenges. For the most part, things "work".. However, I ran into issues with the backlight control and audio. I was able to get them to work by doing some hacky stuff, which may or may not be a "good" solution depending on your views. This repository contains scripts and files I used, and this file explains how to use them.

You don't need to be a Linux guru to get through this, but I assume you are familiar with basic terminal use and an understanding of the Linux file system hierarchy. 

I am running MrChromebox's full UEFI firmware and Fedora Linux (though it should work on any distribution).

# The Kernel
I decided to try running the same kernel as ChromeOS uses, so I could be sure that everything "should" work if all the other files could be set up correctly.

While we can't directly copy and use the kernel from a ChromeOS image on a standard Linux distro (it won't boot because of assumptions made for ChromeOS), we can compile our own using the ChromiumOS source.

Compiling the kernel sounds like a scary process, but it isn't too difficult.

First, we need to get a copy of the source. Google maintains several versions of the kernel and uses specific ones for each device they support. As far as I understand, they backport features and security patches from the newer versions into each of the older versions they maintain. Certain device-specific patches/hacks are only available on certain kernel versions.

When Google is ready to a release an update, they create release branches for the kernels. We'll compile and run the latest release of the kernel version that has our eve-specific hacks.

At the time of this writing we need kernel version 4.4, and the latest release is on branch ```release-R72-11316.B-chromeos-4.4```.

Clone the repository:
```
git clone -b release-R72-11316.B-chromeos-4.4 https://chromium.googlesource.com/chromiumos/third_party/kernel
```

Next, we need to set up the configuration file.

The compilation process uses a file at the root of the source directory named ```.config``` to control hundreds of options, such as which drivers and functionalities to include.

Most Linux distributions compile the kernel with a generic ```.config``` file set to include a wide range of drivers and functionality. However, we can start with the settings that ChromeOS was compiled with, and just change the things required to boot a normal Linux setup.

To do this, I downloaded the recovery USB image for eve and extracted the configuration from the ```configs.ko``` module using the ```extract-ikconfig``` script from the mainline kernel repository.

I have included the stock config extracted from recovery image ```chromeos_11021.81.0_eve_recovery_stable-channel_mp.bin```.

If you would like to do this yourself, it can be accomplished like this-
```
wget https://dl.google.com/dl/edgedl/chromeos/recovery/chromeos_11021.81.0_eve_recovery_stable-channel_mp.bin.zip
unzip chromeos_11021.81.0_eve_recovery_stable-channel_mp.bin.zip
kpartx -av chromeos_11021.81.0_eve_recovery_stable-channel_mp.bin
mkdir /mnt/tmp
mount -t ext2 /dev/mapper/loop0p3 -o ro /mnt/tmp
cp /mnt/tmp/lib/modules/4.4.153-15155-g656ec00c3ce2/kernel/kernel/configs.ko configs.ko
./extract-ikconfig configs.ko > stock.config
```
You may need to change some paths depending on your system or the ChromeOS version you want to use.

This config can't be used as-is. We need to change a few settings for boot to work on a standard setup. I won't detail every change here, but to do this I compared the ChromeOS config with a Fedora config from kernel 4.4. I then went through each difference and selectively chose the side that made the most sense- mostly choosing the ChromeOS settings unless it involved EFI or something that sounded important for Linux to work.

The config I created and used for compiling ```release-R72-11316.B-chromeos-4.4``` is in this repository, named ```eve-custom.config```. You can just download this and save it as ```kernel/.config```.

Being almost entirely based on the ChromeOS config, this one does not include many drivers and features that wouldn't be used by it. It should be possible to enable any of the features that are needed, and most likely it would work fine. I just wanted my kernel to be as similar as the stock one as possible, so everything would be more likely to work.

After creating a ```.config``` file, compile and install the kernel.
```
make oldconfig
make -j15
make -j15 modules
make modules_install
make install
```

Be sure to update your bootloader configuration so that you can select and boot this kernel.

Boot, and you should now be able to adjust the brightness. Audio still won't work yet, but we'll fix that next.

# Audio
After getting the kernel set up, there are two things to do to get audio working.

First, blacklist the ```snd_hda_intel``` module. You can create a file at ```/etc/modprobe.d/blacklist.conf``` containing the following line:
```
blacklist snd_hda_intel
```

Next, copy the firmware from the recovery image to your firmware directory. There is a repository for firmware on the ChromiumOS git, but it doesn't seem to contain what we need. Maybe there is a licensing issue preventing them from distributing it there- which is why I don't include it in this repository.
```
wget https://dl.google.com/dl/edgedl/chromeos/recovery/chromeos_11021.81.0_eve_recovery_stable-channel_mp.bin.zip
unzip chromeos_11021.81.0_eve_recovery_stable-channel_mp.bin.zip
kpartx -av chromeos_11021.81.0_eve_recovery_stable-channel_mp.bin
mkdir /mnt/tmp
mount -t ext2 /dev/mapper/loop0p3 -o ro /mnt/tmp
cp /mnt/tmp/lib/firmware/* /lib/firmware/*
```
