---
layout: post
title: Manual Fusion Drive Configuration
tags:
- apple
- diy
- fusiondrive
- mac
- osx
status: publish
type: post
published: true
---
Apple is now selling Macs with a feature called Fusion Drive, which merges a SSD and a HDD and keeps your most used files on the SSD. This sounds a lot like tiering in enterprise storage solutions, [and indeed it is](http://arstechnica.com/apple/2012/10/more-on-fusion-drive-how-it-works-and-how-to-roll-your-own/). It turns out this is a built-in feature of CoreStorage (Apple's logical volume manager), and that means we can configure this setup on any Mac that runs OS X with Fusion Drive support (10.8.2 and onward guaranteed). I tried this successfully on my iMac with an Intel 520 120 GB SSD and a factory installed 1 TB HDD. I based my test on [the excellent work](http://jollyjinx.tumblr.com/post/34638496292/fusion-drive-on-older-macs-yes-since-apple-has) done by [Patrick Stein aka jollyjinx](http://jollyjinx.tumblr.com/). This is what I did:

Backup! This procedure will destroy all existing data, so a full backup is required. A Time Machine backup will simplify the installation process.

To create the CoreStorage volume we'll need to boot from a 10.8.2 system. Download the OS X Mountain Lion installation app from the App Store. When the installation dialog pops up just quit it. Mount the installation disk image located here:

```
/Applications/Install OS X Mountain Lion.app/Contents/SharedSupport/InstallESD.dmg
```

Now restore the partition in this image to a USB stick. Insert the USB stick and open Disk Utility. Create a partition on the stick big enough to hold the install image. Select the partition contained in the mounted InstallESD image and choose the Restore tab. The source should already be filled with the correct information but there is no destination specified, so drag the partition created on the USB stick to this field. Press the Restore button to transfer the contents of the install image to the partition on the USB stick.

Leave USB stick inserted and reboot Mac. When you hear the startup sound press and hold the Option (Alt) key. Select the install partition on the USB as boot drive. When the OS X installer has loaded open Disk Utility.

We need partitions to include in the CoreStorage volume, one on the SSD and one on the HDD. Create a single partition on the HDD. As for the SSD, if you are using a non-Apple drive you won't have TRIM support in OS X, and one way to mitigate potential performance issues when the drive gets full (and it will at some point with this setup) is to overprovision free space so that there always will be a substantial pool of available cleared cells. If the SSD already has been filled up you can [secure erase](http://www.thomas-krenn.com/en/wiki/SSD_Secure_Erase) it to recover from performance degradation.

[Overprovisioning can also improve performance](http://www.storagereview.com/intel_ssd_520_enterprise_review) in general, but that depends on the workload and usage patterns. Keep in mind that SandForce equipped drives like the Intel 520 already are overprovisioned and hence might not require extra overprovisioning for normal use cases. I like the idea of overprovisioning so let's select 2 partitions for the SSD. The first is a normal partition 104 GB in size, the second is configured as free space.

These partitions will now be joined as a logical volume in Core Storage. Quit Disk Utility, and at the installer dialog select the Tools menu and open Terminal. Let's take a look at the current partition layout.

```
$ diskutil list
/dev/disk0
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *120.0 GB   disk0
   1:                        EFI                         209.7 MB   disk0s1
   2:                  Apple_HFS                         104.0 GB   disk0s2
/dev/disk1
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *1.0 TB     disk1
   1:                        EFI                         209.7 MB   disk1s1
   2:                  Apple_HFS                         999.9 GB   disk1s2
```

Based on this we want to join disk0s2 and disk1s2. If we wanted to use all of the drives we could simply refer to disk0 and disk1, but since the SSD is now overprovisioned we must specify the partition(s) to use. First we need to create a logical volume group with CoreStorage.

```
$ diskutil cs create FusionVG disk0s2 disk1s2
```

Take note of the logical volume group UUID from that command (or use `diskutil cs list` to find it) as it's needed in order to create the logical volume.

```
$ diskutil cs createVolume 35C7C1AD-463E-4D94-9FD4-C107D660643B jhfs+ Fusion 100%
$ diskutil cs list
CoreStorage logical volume groups (1 found)
|
+-- Logical Volume Group 35C7C1AD-463E-4D94-9FD4-C107D660643B
    =========================================================
    Name:         FusionVG
    Size:         1103345127424 B (1.1 TB)
    Free Space:   28672 B (28.7 KB)
    |
    +-< Physical Volume FDFE44CC-3151-4F68-AEEE-3BD3D1915C2A
    |   ----------------------------------------------------
    |   Index:    0
    |   Disk:     disk0s2
    |   Status:   Online
    |   Size:     104000000000 B (104.0 GB)
    |
    +-< Physical Volume C6DB4E59-5499-47F6-81C7-3B71F9CAD769
    |   ----------------------------------------------------
    |   Index:    1
    |   Disk:     disk1s2
    |   Status:   Online
    |   Size:     999345127424 B (999.3 GB)
    |
    +-> Logical Volume Family 11D7E8D0-195C-4658-B38B-9B3D9E48CCF4
        ----------------------------------------------------------
        Encryption Status:       Unlocked
        Encryption Type:         None
        Conversion Status:       NoConversion
        Conversion Direction:    -none-
        Has Encrypted Extents:   No
        Fully Secure:            No
        Passphrase Required:     No
        |
        +-> Logical Volume 1CDA3D65-96DF-4F9B-A374-D08D26AF87EA
            ---------------------------------------------------
            Disk:               disk2
            Status:             Online
            Size (Total):       1098913529856 B (1.1 TB)
            Size (Converted):   -none-
            Revertible:         No
            LV Name:            Fusion
            Volume Name:        Fusion
            Content Hint:       Apple_HFS
```

The logical volume is now ready for installation. Quit Terminal to go back to the installation dialog. In order to recreate the Recovery partition, do not restore directly from a Time Machine backup. Instead do a normal reinstallation of OS X and migrate data from the backup when presented with the option to do so.

Enjoy your very own Fusion Drive!


## References

- [Fusion Drive on older Macs? YES!](http://jollyjinx.tumblr.com/post/34638496292/fusion-drive-on-older-macs-yes-since-apple-has)
- [DIY Fusion Drive: an attempt to retrofit a pre-fall 2012 Mac with an SSD and a traditional hard disk](http://www.petralli.net/2012/10/analyzing-apples-fusion-drive-in-an-attempt-to-retrofit-an-existing-macs-with-an-ssd-and-a-traditional-hard-disk/)
- [What happens to Fusion Drive when you use Boot Camp](http://www.petralli.net/2012/10/what-happens-to-fusion-drive-when-you-use-boot-camp/)
- [Intel SSD 520 Enterprise Review](http://www.storagereview.com/intel_ssd_520_enterprise_review)
- [Understanding the SSD Performance Degradation Problem](http://www.anandtech.com/show/2738/8)
- [SSD Secure Erase](http://www.thomas-krenn.com/en/wiki/SSD_Secure_Erase)
