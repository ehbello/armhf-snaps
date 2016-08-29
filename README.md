Snappy Ubuntu Core for armhf
============================
This repository provides tools to build Ubuntu Core images for some 'armhf'
systems as well as Beagleboard and PandaBoard.

Ubuntu Core is under ongoing development and, as such, these images are meant
only to provide a basis for experimentation and a starting point for Snapd +
armhf. See [Kudos and Caveats](#kudos-and-caveats).

The Gadget Snap package
-------------------
This package includes any components such as a bootloader that are specific to
a particular device (i.e. Overo, DuoVero, Pepper, Beagle, and Panda).

As well as device-specific components, a system vendor could distribute a logo
or pre-install other useful packages into any generated images.

<!---
As these Gadget snaps are  available through the Snappy App Store, you can skip
this step and go to directly to [below](#assemble-your-image).
-->

**1. Grab some required software.**

    $ sudo apt-get install -y git crossbuild-essential-armhf make snapcraft

**2. Build the snap package.**

The included *Makefile* should automate much of this process.  Use the *-j* flag
to specify the number of concurrent make processes.

    $ make MACHINE=<machine> -j4

After some git-fetching and compiling, [fresh binaries](#u-boot) built from
the newly checked out *u-boot* repository should be packaged up in a Snappy
Gadget package, *machine-name*.snap. The [package build system][1] for Snappy
is pretty simple. Have a look at the *snap.yaml* file in the machine's *meta*
directory and then try building.

    $ snapcraft <machine>

If you make changes to any source files, just call *make* again to rebuild. We
can also clean the build... we keep checked-out source around though.

    $ make MACHINE=<machine> clean

Individual components can be also be (re-)built and cleaned e.g.

    $ make MACHINE=overo uboot
    $ make MACHINE=overo clean-uboot
    $ make MACHINE=pepper gadget
    $ make MACHINE=pepper clean-gadget

[1]: https://developer.ubuntu.com/en/snappy/build-apps/

#### U-Boot
We grab the *2016.07* version of the *u-boot* bootloader from the U-Boot
repository and store it in the *u-boot* directory.

    $ git clone git://github.com/u-boot/u-boot.git -b v2016.07 u-boot

We then configure it using the machine-specific *defconfig*:

Machine  | Machine defconfig
---------|-------------------
overo    | omap3_overo_defconfig
duovero  | duovero_defconfig
pepper   | pepper_defconfig
beagle   | omap3_beagle_defconfig
panda    | omap4_panda_defconfig

    $ cd u-boot
    $ make CROSS_COMPILE=arm-linux-gnueabihf- <machine_defconfig>

Build and install the resulting *MLO* and *u-boot.img* files.

    $ make CROSS_COMPILE=arm-linux-gnueabihf-
    $ cp MLO u-boot.img ../<machine>/


The Image File
--------------
**1. Download the only *ubuntu-device-flash* tool that works.**

    $ sudo apt-get install -y ubuntu-device-flash
    $ wget https://people.canonical.com/~mvo/all-snaps/ubuntu-device-flash
      (md5:  e3c9900a99b567862f8651d0c2fef1c2)
      (sha1: 9a19991244a02528c22fd2b7b4367673665d3ccd)
    $ chmod +x ubuntu-device-flash

**2. Assembly a flashable image.**

We'll use the *16.04* release and enable ssh as well *developer-mode* so we
can install unsigned packages. Old Snappy's rollback system is still not
supported. Instead of this, a minimal system with only one OS partition will be
defined. Image must be greather equal than 4GB. If card has free space, OS
partition will be resized on first boot.


    $ export UBUNTU_DEVICE_FLASH_IGNORE_UNSTABLE_GADGET_DEFINITION=true
    $ sudo ./ubuntu-device-flash -v core -o <machine>.img \
                                         --size 4 \
                                         --gadget <machine>_*_all.snap \
                                         --kernel linux-armhf \
                                         --os ubuntu-core \
                                         --developer-mode \
                                         --channel=edge 16

If you get the followin error:

> failed to install "[machine]" from "edge": [machine] failed to
>   install: exit status 2

Runs following [hack][2] and retry:

    $ sudo ln -s /bin/true /usr/local/bin/udevadm

[2]: https://lists.ubuntu.com/archives/snappy-devel/2016-April/001736.html

<!---

------------------------------------------------------------------------------
**Note:**

As these Gadget snaps are available through the Snappy App Store, just
replace *machine-name*.snap with *machine-name* in the *--gadget* argument
above and the Gadget snap will be automatically fetched from the store.

------------------------------------------------------------------------------
-->

**3. Burn assembled image.**

These image can be directly copied to a 4GB or greater microSD card.

------------------------------------------------------------------------------
**WARNING:**
This erases anything currently on the microSD card!  Note that any mounted
partititions of the SD card should be unmounted before writing the image. These
steps each take several minutes; your computer isn't hanging, the files are
just large.

------------------------------------------------------------------------------

Format a microSD with the newly created *img* file.  Once the card is created,
take a look at the partition structure... there should be two as described above.

    # substitute the path to the drive e.g. /dev/sdd or /dev/mmcblk0 (not the
    # path of a partition e.g. /dev/sdd1 or /dev/mmcblk0p1) in place of <drive>
    # use 'udevadm monitor' when you insert the card to determine the path
    $ sudo dd if=<machine>.img bs=4k of=<drive>
    $ sync

After the process completes, plug the microSD card into your 'armhf' (or TI)
system, boot, login as *ubuntu* with password *ubuntu* and start playing with
your shiny new Snappy Ubuntu Core!

**Technical Aside**

There is unpartitioned space at the beginning of the card in which, on some
installations, booloader components such as *MLO* and *u-boot.img* are written,
rather than storing them on the system-boot partition.  OMAP4 (DuoVero/Panda)
and AM335x (Pepper) use this but, as OMAP3 (Overo/Beagle) u-boot would require
a *raw* boot mode patch, we just place *MLO* and *u-boot.img* on the
*system-boot* partition instead.

** 4. Enjoy! **

By way of example, try installing the *xkcd-webserver* package so you can
enjoy random XKCD comics delivered to your browser by your Snappy system:

    # on your snappy system
    $ sudo snap install xkcd-webserver

On your development machine (on the same network as your snappy device),
navigate to http://*machine-name*.local


Kudos and Caveats
-----------------
In making this repository, [there][3] [were][4] [many][5] [useful][6]
[references][7]. Special thanks to [Gumstix guys][8] for their work in [previous][9]
releases.

A few known issues:

 * Standard *linux-armhf* kernel from Ubuntu Core stucks trying to find the
   initramfs. You can build your own kernel customizing the
   [Linux Kernel snap for PandaBoard][10].
 * There is a long (~5 minutes) pause before get login prompt as the system
   makes the new private keys. It is not frozen... just be patient.

The most important:

 * There is no validation that all the hardware works. I can only test
   PandaBoard image, so if you find things that are broken or have suggestions,
   leave a comment, raise an issue, or best of all, send a pull request!

[3]: https://code.launchpad.net/~mvo
[4]: https://developer.ubuntu.com/en/snappy/guides/porting/
[5]: https://wiki.ubuntu.com/SecurityTeam/PublicationNotes#Ubuntu_Core
[6]: https://lists.ubuntu.com/archives/snappy-devel/2016-January/001400.html
[7]: https://lists.ubuntu.com/mailman/listinfo/snapcraft
[8]: https://www.gumstix.com/about-us/
[9]: https://github.com/gumstix/snappy
[10]: https://github.com/ehbello/linux-panda-snap
