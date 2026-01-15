# OpenWrt A/B Upgrades
## Overview
A/B upgrades allow for safer OpenWrt upgrades. If the upgrade does not go as
planned, you can easily boot into your previous image.

The A/B upgrade approach functions by having two root parititons. When
executing an upgrade the new image will be written to the non-active root
partition. The grub config is updated to allow booting into either image.

At boot, you'll be able to select between either image. The most recently
installed/upgraded image will be the default (highlighted and first in the
list). Both root options also have the default failsafe options, and the
partition label. Example grub screen:

```
*OpenWrt 23.05.5 r24106-10cc5fcd00 B
 OpenWrt 23.05.5 r24106-10cc5fcd00 B (failsafe)
 OpenWrt 23.05.2 r23630-842932a63d A
 OpenWrt 23.05.2 r23630-842932a63d A (failsafe)
```

### Partition Layout
* /dev/sda1 = /boot with label kernel
* /dev/sda2 = / with label rootfs_a
* /dev/sda3 = / with label rootfs_b

Filesystem labels are used to auto-detect partitions for the upgrade. If the
system was deployed from the combined-ext4 image the setup is largely handled
automatically and only needs you to create the new root partition. This is
tested with both sata/msata and nvme devices.

Both root images have their kernels on the same boot partition. They are named
with the A/B label as a suffix. The layout above would have
`/boot/vmlinuz-A` and `/boot/vmlinuz-B` for the two rootfs's kernels.

> [!CAUTION]
> Be aware that you may run out of space with this script, as it stores 3 separate
> kernel images inside of the tiny 16MB /boot partition, and will youhave to follow the resize
> instructions at the very bottom of this guide.
> 
> NEVER use the OpenWRT documentation's script ([this one](https://openwrt.org/docs/guide-user/advanced/expand_root)) to resize your partitions
> if you decide to use A/B partitioning. It will render your drive unbootable!!

### High Level Process
1. Detect all required partitions
1. Identify the non-active root partition
2. Create sysupgrade backup
3. Format target partition
4. Mount target partition to /mnt/sysupgrade
5. Extract rootfs to /mnt/sysupgrade
6. Copy kernel to /boot/vmlinuz-ID
7. Extract openwrt-backup to /mnt/sysupgrade
8. Backup grub.cfg
9. Generate new grub.cfg
10. Reboot

## Setup for GPT partition layout
1. Install Packages: `opkg install parted coreutils-stat blkid e2fsprogs`
2. If required, resize the initial root partition (2) (naming it is shown here but
   optional)
   ```
   root@OpenWrt:~# parted --fix --script /dev/sda resizepart 2 1041MiB name 2 root_a
   Warning: Not all of the sapce available to /dev/sda appears to be used, you can fix
   the GPT to use all of the space (an extra 8142303 blocks) or continue with the current
   setting?
   Fixing, due to --fix
   root@OpenWrt:~# parted /dev/sda unit MiB print
    Model: ATA VBOX HARDDISK (scsi)
    Disk /dev/sda: 4096MiB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags:

    Number  Start    End      Size     File system  Name    Flags
    128     0.02MiB  0.25MiB  0.23MiB                       bios_grub
     1      0.25MiB  16.2MiB  16.0MiB  fat16                legacy_boot
     2      16.3MiB  1041MiB  1025MiB  ext4         root_a
   ```
2. Create partiton for second rootfs (3)
   ```
   root@OpenWrt:~# parted /dev/sda mkpart root_b ext4 1041MiB 2064MiB
   root@OpenWrt:~# parted /dev/sda unit MiB print
   Model: ATA VBOX HARDDISK (scsi)
   Disk /dev/sda: 4096MiB
   Sector size (logical/physical): 512B/512B
   Partition Table: gpt
   Disk Flags:

   Number  Start    End      Size     File system  Name    Flags
   128     0.02MiB  0.25MiB  0.23MiB                       bios_grub
    1      0.25MiB  16.2MiB  16.0MiB  fat16                legacy_boot
    2      16.3MiB  1041MiB  1025MiB  ext4         root_a
    3      1041MiB  2064MiB  1025MiB               root_b
   ```

## Setup for MBR partition layout
1. Install Packages: `opkg install parted coreutils-stat blkid e2fsprogs`
2. If required, resize the initial root partition (2)
   ```
   root@OpenWrt:~# parted --fix --script /dev/sda resizepart 2 1041MiB
   root@OpenWrt:~# parted /dev/sda unit MiB print
   Model: ATA VBOX HARDDISK (scsi)
   Disk /dev/sda: 4096MiB
   Sector size (logical/physical): 512B/512B
   Partition Table: msdos
   Disk Flags:

   Number  Start    End      Size     Type     File system  Flags
    1      0.25MiB  16.2MiB  16.0MiB  primary  ext2         boot
    2      16.3MiB  1041MiB  1025MiB  primary  ext4
   ```
2. Create partiton for second rootfs (3)
   ```
   root@OpenWrt:~# parted /dev/sda mkpart primary ext4 1041MiB 2064MiB
   root@OpenWrt:~# parted /dev/sda unit MiB print
   Model: ATA VBOX HARDDISK (scsi)
   Disk /dev/sda: 4096MiB
   Sector size (logical/physical): 512B/512B
   Partition Table: msdos
   Disk Flags:

   Number  Start    End      Size     Type     File system  Flags
    1      0.25MiB  16.2MiB  16.0MiB  primary  ext2         boot
    2      16.3MiB  1041MiB  1025MiB  primary  ext4
    3      1041MiB  2064MiB  1025MiB  primary
   ```

## Upgrading
1. Optional: Modify upgrade.sh to update default grub options
2. Install Packages: `opkg install coreutils-stat blkid`
3. Download rootfs and kernel to /root
   ```
   cd /root
   wget https://downloads.openwrt.org/snapshots/targets/x86/64/openwrt-x86-64-generic-kernel.bin
   wget https://downloads.openwrt.org/snapshots/targets/x86/64/openwrt-x86-64-rootfs.tar.gz
   ```
3. Upgrade: `./upgrade.sh openwrt-x86-64-generic-kernel.bin openwrt-x86-64-rootfs.tar.gz`
4. Reboot

### Example outputs

First upgrade, from a system built with the combined-ext4 image and the setup
instructions above. Note the warning that is presented, and the script is
guessing rootfs_b's location.
```
root@OpenWrt:~# ./upgrade.sh openwrt—23.05.5-x86-64-generic-kernel.bin openwrt—23.05.5-x86-64-generic-rootfs.tar.gz
WARNING: rootfs_a found with OpenWRT name on /dev/sda2 (this is normal for first use)
Assuming rootfs_b should be placed on /dev/sda3. Proceed with caution.
Current Root FS: /dev/sda2
Target Root FS: /dev/sda3
Kernel: openwrt-23.05.5-x86-64-generic-kernel.bin
Rootfs: openwrt-23.05.5-x86-64-generic-rootfs.tar.gz
Continue? (y/n) y
upgrade: Saving config files...
Formatting /dev/sda3...
Mounting /dev/sda3 to /mmt/sysupgrade...
Extracting rootfs.
Restoring backup...
Generating grub.cfg...
Copying kernel...
root@OpenWrt:~#
```

Sample output from the same system, but now it's running from image B, so it
will write to image A's partition.
```
root@OpenWrt:~# ./upgrade.sh openwrt—23.05.5-x86-64-generic-kernel.bin openwrt—23.05.5-x86-64-generic-rootfs.tar.gz
Current Root FS: /dev/sda3
Target Root FS: /dev/sda2
Kernel: openwrt-23.05.5-x86-64-generic-kernel.bin
Rootfs: openwrt-23.05.5-x86-64-generic-rootfs.tar.gz
Continue? (y/n) y
upgrade: Saving config files...
Formatting /dev/sda2...
/dev/sda2 contains a ext4 file system labelled 'rootfs'
last mounted on / on Sat Oct 12 11:26:16 2024
Proceed anyway? (y,N) y
Mounting /dev/sda2 to /mmt/sysupgrade...
Extracting rootfs.
Restoring backup
Generating grub.cfg
Copying kernel
root@OpenWrt:~#
```

## FAQ / Notes

* Q: Why do you use not use `root=PARTUUID=<uuid>` in grub.cfg anymore?

  A: For some reason OpenWRT uses truncated PARTUUID's that match between
  partitions and devices. For MBR this is probably expected, but with GPT
  it causes us a limitation. The truncation is the last 2 hex values of
  the UUID are modified as follows:
  * 02 (root partition PARTUUID)
  * 01 (boot partition PARTUUID)
  * 00 (device partition table id PTUUID)

  And the result is, if the running root PARTUUID truncated to 00 on the end
  is not found, /boot won't get mounted. This exhibited itself as Image A
  was mounting /boot, and Image B was not. So we use the device name instead
  which works around the issue. There is no practical way to have the same
  PARTUUID on multiple partitions, so we have to work around the issue.

  The function that is doing this is
  [export_bootdevice()](https://github.com/openwrt/openwrt/blob/fa6bd065ddad1a3e5d507a4eb9b5fd1fd4b4055a/package/base-files/files/lib/upgrade/common.sh#L186)
  in `/lib/upgrade/common.sh`. Switching to device names follows a different
  codepath avoiding the issue entirely.
  [Related issue here](https://github.com/openwrt/openwrt/issues/6206).

* Q: When it's running how can I easily tell which partition is currently
  root?

  A: Since both root's show up as `/dev/root` mounted on `/` this isn't
  obvious. There are a few ways (and many more):
  ```
  root@OpenWrt:/# cat /proc/cmdline
  BOOT_IMAGE=/boot/vmlinuz-B root=/dev/sda3 ...
  root@OpenWrt:/# ls /.i*
  /.image-B
  ```
  The first is simply checking the running kernel command line, or check
  for a `.image-*` file in the root. This script creates that, so until
  you boot a root this script has written, it wouldn't be present.

* Q: I got the `no space left on device` error when running the script, what do I do now?

  A: This typically indicates your `/boot` partition is not large enough, as mentioned
  in the warning.
  ### Fix:
  These steps assume the following for your boot drive:
   1. Partition 1 (/boot) is still 16MiB fat16
   2. Partition 2 is ext4, and you manually expanded it past the default 104MiB
      size (to something like 8GiB, using the steps listed earlier in the guide)
   3. Partition 3 is also ext4, is identical in size to P2, and exists to be used
      as the "B" slot
   4. Partition 4 is where other data for programs like Docker, etc is stored
      (again, WON'T be used by the upgrade script), or you have a completely
      separate block device (which is automounted)
   6. You have at least 2GiB of freespace remaning on the boot disk after
      configuring this scheme
  
  You will need to do the following:
   1. Obtain a liveCD of GParted, and boot your system into it. (PXE booting
      won't work, since your router isn't going to be booted into OpenWRT.)
   2. Identify the correct storage device that you flashed OpenWRT onto.
   3. Open a terminal, and backup the /boot partition using `cp`:
    
      ```
      sudo mkdir -p /mnt/bootpart
      mkdir -p /tmp/boot-bak 
      sudo mount -t vfat /dev/sda1 /mnt/bootpart
      sudo cp -va /mnt/bootpart/* /tmp/boot-bak
      umount /mnt/bootpart
      ```

      ⚠️⚠️⚠️ WARNING ⚠️⚠️⚠️
      ***
      
      If you are doing these steps on an existing install, BACK UP your active
      partition using LuCI's "Generate Archive" button, and also use `rsync` to
      copy all of the data in your userdata partition (partition 4) to another
      physically attached disk or NAS (via RSYNC/SFTP).

      ***
      ⚠️⚠️⚠️

   5. Switch back to (or reopen) GParted, and do the following for partition 4
      (you may not have one, so you can do this with 3 instead):
      Right-click the partition, then click the "Resize/Move" button.
      Drag the partition to the right by a hair, then the size for "Before" and
      "After" should change. Once they do, manually set the "Before" size to 112
      MiB (which result in giving partition 1 a total of 128MiB to expand to.) The
      after size should decrease, but not be zero, as we are just moving the
      partition JUST a little bit MORE into the freespace.
      
      Repeat the same steps for partition 3 and 2, until GParted shows that Partition
      1 is 16MiB, but has 112 (can sometimes calculate to 112.75) MiB of freespace
      between it and partition 2.

      Once this is confirmed, click the check (✔) at the top, DOUBLE-CHECK the changes,
      and let it execute.
      
   6. PRINT your current partition table first:
      `sudo fdisk -l /dev/sda`

      
      🚨🚨🚨 CAUTION 🚨🚨🚨
      ***
      
      This will WIPE the /boot partition completely. BACK UP THE FILES!
      You will ALSO need to pay attention to the partition type and sector numbers
      when formatting.
      
      If your starting sector number for partition 1 is 1024 or 2048, ensure to use
      THAT number, rather than 512. If you skip this, you WILL brick your GRUB.
      
      If you've already flashed that combined image like mentioned earlier, your disk
      should automatically be formatted to GPT. If for some God-forsaken reason you
      formatted your disk with MBR and somehow got OpenWRT to boot, you should consider
      a different script.

      ***
      🚨🚨🚨
      

      ```
      $ sudo fdisk /dev/sda
   
      d          # Delete partition
      1          # Select partition 1
      
      n          # Create new partition
      1          # Partition number 1
      512        # Start sector: 512
      +128M      # End: +128M
      y          # Yes to wipe the existing FAT signature
      
      t          # Change partition type
      1          # Select partition 1
      11         # Microsoft basic data (for GPT)
      
      x          # Enter expert mode
      A          # Set legacy BIOS bootable attribute
      1          # Select partition 1
      r          # Return to main menu
      
      p          # Print to verify everything looks correct
      
      w          # Write changes to disk
      ```
      
   4. Format the disk, then inform the OS:
   
      `sudo mkfs.vfat -F 32 -n kernel /dev/sda1`
      
      `sudo partprobe /dev/sda`
   6. Restore the contents of the /boot partition, and then use the GUI button to reboot:
      ```
      mount -t vfat /dev/sda1 /mnt/bootpart
      cp -va /tmp/boot-bak/* /mnt/bootpart/
      umount /mnt/bootpart
      ```
      
