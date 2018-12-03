# Introduction

This is about documenting getting Linux running on the late 2016 and mid 2017 MPB's; the focus is mostly on the MacBookPro13,3 and MacBookPro14,3 (15inch models), but I try to make it relevant and provide information for MacBookPro13,1, MacBookPro13,2, MacBookPro14,1, and MacBookPro14,2 (13inch models) too. I'm currently using Fedora 27, but most the things should be valid for other recent distros even if the details differ. The kernel version is 4.14.x (after latest update).

The state of linux on the MBP (with particular focus on MacBookPro13,2) is also being tracked on https://github.com/Dunedan/mbp-2016-linux . And for Ubuntu users there are a couple tutorials ([here](https://github.com/chisNaN/ubuntu-on-macbook12) and [here](https://nixaid.com/linux-on-macbookpro/)) focused on that distro and the MacBook.

**Note**: For those who have followed these instructions ealier, and in particular for those who have had problems with the custom DSDT, modifying the DSDT is not necessary anymore - see the updated instructions below and make sure to update your clone of the roadrunner2/macbook12-spi-driver repo to get the latest drivers. 

# Summary Of Current State
## What works
* Booting (i.e Grub etc)
* Recognizes disk on all models (older kernels may need patch for some models, though)
* Keyboard, touchpad, and basic touchbar functionality
* HiDPI detection
* Accelerated video
* Screen brightness control
* Keyboard backlight
* USB
* Sensors (install ```lm_sensors``` package)
* Camera
* Bluetooth (older kernels need patches)
* WiFi on MBP13,1 and MBP14,1

## What doesn't work
* WiFi on ,2 and ,3 models (though some folks have had success with some of the workarounds)
* Suspend/Resume
* Audio (two cards show up, and intel driver is loaded, but no sound)

## Untested
* Thunderbolt
* DisplayPort

# Details

## Partitioning

If you want to keep your MacOS installation (generally a good idea if you can afford the disk space, because that's the only way to get/install firmware updates), then first boot into MacOS and resize the partition there, creating a new partition for the Linux installation. If you also want to have a Windows partition, see [this comment](#gistcomment-2164350) below.

**Warning**: If you're not going to keep MacOS, either back up the EFI System Partition (and restore its contents to the new ESP after installation) or leave it intact (i.e. don't do a full disk install, but just use the space after the ESP). This partition (it's the first one) contains drivers/firmware/etc needed by Apple's EFI loader during boot, in particular to initialize the Touchbar.

## Initial Installation

Since the internal keyboard and touchpad won't work until you have built and loaded the drivers, you'll need to plug in an external USB keyboard to do the initial setup and installation.

## Booting

If you're booting a 4.11 or later kernel, no special params or patches are needed.

If you're booting a kernel < 4.11 and have a MacBookPro13,1, MacBookPro13,2, MacBookPro14,1 or MacBookPro14,2 (13inch models), which have the Apple NVMe controller, you'll need the [kernel-nvme-controller.patch](#file-kernel-nvme-controller-patch) from this gist in order for the disk to be correctly recognized (MacBookPro13,3 uses a Samsung NVMe controller which is automatically detected correctly). Alternatively, instead of patching you can also do the following (for distros using something other than dracut to create the initrd you'll need to adjust the 2nd and 3rd lines appropriately):
```
echo 'install nvme /sbin/modprobe --ignore-install nvme $CMDLINE_OPTS; echo 106b 2003 > /sys/bus/pci/drivers/nvme/new_id' | sudo tee /etc/modprobe.d/nvme.conf
echo 'force_drivers+="nvme"' | sudo tee /etc/dracut.conf.d/disk.conf
sudo dracut --force --kver <kernel-version>
```

If you're booting a kernel < 4.10 then you'll need the following kernel param to boot properly: `intremap=nosid`. E.g.
```
sudo sed -i 's/\(GRUB_CMDLINE_LINUX=.*\)"/\1 intremap=nosid"/' /etc/default/grub
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg
```

Lastly, if you are booting a live CD or similar with a kernel < 4.9 then you will also need to add the ```nomodeset``` kernel parameter to your kernel line; you will then not have proper HiDPI detection or accelerated graphics.

## Keyboard/Touchpad/Touchbar

For this we need the drivers from https://github.com/roadrunner2/macbook12-spi-driver.git (a clone of https://github.com/cb22/macbook12-spi-driver which includes a preliminary touchbar driver and keyboard fixes). The following commands set this up.

First some extra packages:
```
sudo dnf install git kernel-devel dkms
```

Next we need to prepare for the modules to be included in the ramdisk (so they are loaded early during boot):
```
cat <<EOF | sudo tee /etc/dracut.conf.d/keyboard.conf
# load all drivers needed for the keyboard+touchpad
add_drivers+="applespi intel_lpss_pci spi_pxa2xx_platform appletb"
EOF
```
On distros using ```mkinitramfs``` instead of ```dracut``` you'll want to do the following instead:
```
cat <<EOF | sudo tee -a /etc/initramfs-tools/modules
# drivers for keyboard+touchpad
applespi
appletb
intel_lpss_pci
spi_pxa2xx_platform
EOF
```

Now get and build the drivers:
```
git clone https://github.com/roadrunner2/macbook12-spi-driver.git
pushd macbook12-spi-driver
git checkout touchbar-driver-hid-driver
sudo ln -s `pwd` /usr/src/applespi-0.1
sudo dkms install applespi/0.1
popd
```

Next we need to set the proper dpi for the touchpad and adjust the sensitivity (download the [61-evdev-local.hwdb](#file-61-evdev-local-hwdb) and [61-libinput-local.hwdb](#file-61-libinput-local-hwdb) from this gist):
```
sudo cp ...the-downloaded-61-evdev-local.hwdb... /etc/udev/hwdb.d/61-evdev-local.hwdb
sudo cp ...the-downloaded-61-libinput-local.hwdb... /etc/udev/hwdb.d/61-libinput-local.hwdb
sudo systemd-hwdb update
```

You can test the drivers by loading them and their dependencies:
```
sudo modprobe intel_lpss_pci spi_pxa2xx_platform applespi appletb
```

Finally, reboot to make sure it all works correctly:
```
sudo reboot
```

### Screen Brightness Control

Screen brightness control works out of the box on MacBookPro13,1 and MacBookPro13,2 (all kernels), and MacBookPro13,3 with recent kernels, but requires a kernel patch on MacBookPro13,3 with older kernels (see also https://github.com/Dunedan/mbp-2016-linux/issues/2). Specifically, if you have any of these kernels you need to patch: < 4.14, 4.14 - 4.14.21, 4.15 - v4.15.5 (i.e. the issue was fixed in 4.14.21, 4.15.5, and 4.16). The following will create and install the patched `apple-gmux`:
```
mkdir apple-gmux
pushd apple-gmux

curl -o apple-gmux.patch 'https://bugzilla.kernel.org/attachment.cgi?id=192601'
curl -o apple-gmux.c 'https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/plain/drivers/platform/x86/apple-gmux.c?id=refs/tags/v4.9.11'

patch < apple-gmux.patch

echo -e '
obj-m += apple-gmux.o

all:
\tmake -C /lib/modules/`uname -r`/build M=`pwd` modules
' > Makefile
make

mod=$(ls /lib/modules/`uname -r`/kernel/drivers/platform/x86/apple-gmux.ko*)
sudo mv $mod{,.orig}
sudo cp apple-gmux.ko /lib/modules/`uname -r`/kernel/drivers/platform/x86/
sudo depmod

popd

sudo reboot
```

### Other

The touchpad defaults to using the bottom-left corner for right-clicks - to get 2-finger right click, install the Gnome tweak tool and change it in there.

## WiFi

WiFi works fine on MBP13,1 and MBP14,1. But the other models use a different chipset, and while on those the ```brcmfmac``` driver is automatically loaded, there are a number of issues with it, making it for all practical purposes unusable:
* it only does 2.4GHz - no 5GHz channels are visible
* it has an extremely low sensitivity - you must be within a few feet of the base station, and even at 5 feet distance it shows a weak signal.
* it stops working after 10 or 15 or so minutes; turning WiFi off, waiting a several minutes, and then turning it back on generally gets it working again. Maybe a thermal issue?

Bug report: https://bugzilla.kernel.org/show_bug.cgi?id=193121

In the mean time some folks have that one or both of the following hacks make the WiFi work well enough them (personally, while they do improve the situation, I have not found them to be sufficient enough for actual work, i.e. I still see many packet drops and connection failures - YMMV):
* reduce the transmit power: `sudo iwconfig wlp3s0 txpower 10`
* edit the firmware blob (`/lib/firmware/brcm/brcmfmac43602-pcie.bin`) and modify the `regrev` and `ccode` values (see the above bugreport for details)

## Display

The amdgpu driver works well and is automatically loaded on MacBookPro13,3. On the 13 inch models the use of the ```intel``` needs to be forced (see first comment below).

## Camera

MacBookPro13,3/14,3: works out of the box on kernels 4.13 and later; on earlier kernels you need the following:
```
echo "options uvcvideo quirks=0x100" > /etc/modprobe.d/uvcvideo.conf
```

For MacBookPro[13,14],[12] you need the [bcwc_pcie](https://github.com/patjak/bcwc_pcie) driver (mainline branch) - see also https://github.com/Dunedan/mbp-2016-linux/issues/15.

## Bluetooth

As of kernel 4.16 bluetooth works out of the box; older kernels need patches - see https://github.com/Dunedan/mbp-2016-linux/issues/29#issuecomment-331693489 and following discussion for details. But in short you'll need to:
* Ensure your kernel is configured with `CONFIG_BT_HCIUART_BCM=y`
* apply the patches from [hci_bcm-4.13](https://github.com/roadrunner2/linux/tree/hci_bcm-4.13), [hci_bcm-4.14](https://github.com/roadrunner2/linux/tree/hci_bcm-4.14), or [hci_bcm-4.15](https://github.com/roadrunner2/linux/tree/hci_bcm-4.15), depending on whether you have a 4.13 or earlier, 4.14, or 4.15 kernel.
* build and reboot
* on 4.14 and earlier apply the service patch from the above comment and start the service as described there (not necessary on 4.15 and later).

Note that as of 4.16 there are still issues on MacBookPro13,1 and MacBookPro14,1 - see the above bug for details on what additional patches are needed.