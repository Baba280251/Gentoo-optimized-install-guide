## Gentoo BTRFS with Optimizations and Musl Libc
This is the Gentoo Btrfs with Optimizations and Musl Libc. or GentooBOML (GBOML) :penguin:


| :warning: Disclaimer                                                                                                                |
| :---------------------------------------------------------------------------------------------------------------------------------- |
| Please keep in mind that this might not work on your machine, this guide is specified for amd64 CPUs and EFI machines.              |

## Why?
This is (going to be) a painful journy to achive maximum performance on gentoo with many unusual choices. This all began when i started having issues with replacing and matching different parts of linux in a virtual machine. I struggled with making proper alterations to common parts like glibc and gcc, so i'm making this guide to document the progress of GentooBOML.

## What Does This Contain?
1. Btrfs! an alternative file system to EXT4 which offers many features such as: Autodefrag, Compression, Subvolumes, Self-Healing and most importantly, Great Snapshotting of data instantaneously. It has got many developments over the years, and now 1 year in, i've never had a problem with using Btrfs, although many still consider it unstable, many linux distros have made the bold switch to the future file system BTRFS.
2. Musl Libc! although not a common choice, Musl has proven to be a great standard C library with great simplicity and a small code base, it can do as much as glibc. * This comes to the indivisuals' choice, if you want a stable system then prefer not to use musl libc. It can break many packages!
3. LTO! lto is very controversial in the gentoo community with many opting to just stick with use flags and masking, but lto can offer a great performace boost on many machines, for me i've used lto for many months now and never had a system breaking problem, mostly small breakages that need an hour of fixing for a month use, but be warned, you'll have to debug a lot on your own!

## Let's Start with the BTRFS basics
Btrfs can produce snapshots of the system data, and you can rollback anytime you want, to take proper advantage of these features, btrfs subvolumes must be properly setup.
For making snapshots, subvolumes are a way to exclude variable files from being snapshotted, this prevents data loss and removes big snapshots.

Here I follow the standard subvolumes model provided by [OpenSUSE Wiki](https://en.opensuse.org/SDB:BTRFS#Default_Subvolumes)
The following directories should be excluded from snapshots:

1. /home/       **Avoided if home is on a different partition.**
2. /root/       **Default root user directory.**
3. /opt/        **For 3rd party software installations.**
4. /srv/        **For data of WEB and FTP servers.**
5. /usr/local   **Containing local temperory files and caches.**
6. /var/        **Containing variable files.**
7. /.snapshots/ **This is a custom directory made specifically for snapshots. ❗ You don't want to take a snapshot of a snapshot ❗**
8. /tmp/        **For temporary files.**

Now that you know how btrfs works. Let's continue to the early setup phase:

## Phase 1: Disk Preparation
First of all. Pepare your partitions. I prefer the following setup:

|Partition |  Filesystem  |  Partition Type  |  Size  |
|----------|--------------|------------------|--------|
|/dev/sda1 |     vFat     |  EFI Partition   |  512M  |
|/dev/sda2 |     Btrfs    |  Home Partition  |  any%  |
|/dev/sda3 |     Btrfs    |  Linux Partition |  60G   |

**Home Partition is optional but preferable.** For more information about this setup see the [FAQ](#faq)

After making the partitions make sure to format them correctly:
```bash
mkfs --type vfat /dev/{efipart}    # Replace {efipart} with your EFI part.
mkfs --type btrfs /dev/{homepart}  # You can also have ext4 or vfat // Replace {homepart} with your home partition if you have one.
mkfs --type btrfs /dev/{linuxpart} # Replace {linuxpart} with your linux partition.
```

Now that you have setup disks, you're ready for Phase 2. Please notice that from now on i'll be using the setup above, you must change it according to your setup...

## Phase 2: Mouting and Setup
Mount the linux partition first
```bash
mount /dev/sda3 /mnt
cd /mnt
```

Let's make sure that the subvolumes are created properly, this is done bt creating subvolumes using the standard @ format:
```bash
btrfs subvolume create @
btrfs subvolume create @root
btrfs subvolume create @opt
btrfs subvolume create @srv
btrfs subvolume create @local
btrfs subvolume create @var
btrfs subvolume create @tmp
btrfs subvolume create @.snapshots
btrfs subvolume create @home # Not needed if you use a seperate partition  //
```

**@ is the main parent subvolume, it's like the / which is the parent directory.**

Now unmount the partition and make sure to be outside the /mnt directiry
```bash
cd /
umount -r /mnt
```

Now mount the linux partition firstly to create the specified directories:
```bash
mount -o defaults,noatime,compress=zstd,commit=120,autodefrag,subvol=@ /dev/sda3 /mnt
```
Make sure to create the following directories:
```bash
mkdir -p /mnt/{home,srv,root,tmp,opt,usr/local,var,.snapshots,efi,boot}
```
Great! Now mount the rest of the subvolumes:
```bash
mount -o defaults,noatime,compress=zstd,commit=120,autodefrag,subvol=@local /dev/sda3 /mnt/usr/local
mount -o defaults,noatime,compress=zstd,commit=120,autodefrag,subvol=@root /dev/sda3 /mnt/root
mount -o defaults,noatime,compress=zstd,commit=120,autodefrag,subvol=@opt /dev/sda3 /mnt/opt
mount -o defaults,noatime,compress=zstd,commit=120,autodefrag,subvol=@srv /dev/sda3 /mnt/srv
mount -o defaults,noatime,compress=zstd,commit=120,autodefrag,subvol=@var /dev/sda3 /mnt/var
mount -o defaults,noatime,compress=zstd,commit=120,autodefrag,subvol=@tmp /dev/sda3 /mnt/tmp
mount -o defaults,noatime,compress=zstd,commit=120,autodefrag /dev/sda2 /mnt/home
mount /dev/sda1 /mnt/efi
```
**I mounted the EFI Partition on /efi because it's better to seperate the /boot for kernels and /efi for grub and efi configurations.**

Download the latest Musl Libc in the /mnt directory. This can change so make sure to "wget" the latest version from [Gentoo Downloads](https://www.gentoo.org/downloads/)
```bash
cd /mnt
wget https://gentoo.osuosl.org//releases/amd64/autobuilds/20220220T170542Z/stage3-amd64-musl-hardened-20220220T170542Z.tar.xz
```
Now all we need is to untar the tar
```bash
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

Now this is the end of [Phase 2](#phase-2-mouting-and-setup). We can go to Phase 3.

## Phase 3
***TO BE CONTINUED... Phase 3 and 4 are under experimentation. Bulletproof methodes are not possible, although some startigies can be applied for a usable system. soon enough i'll be able to make a gentoo system that works with musl libc and clang. I hope i can get these phases finished before summer. or if i can't then next year when Musl Libc gentoo is stable. Please be patient...***

## FAQ
**1. Why put the EFI partition first?**
Because the EFI partition is a common partition between different linux distros, so it doesn't need to change according to the distro.

**2. Why put the Home partition second?**
Because the Home partition starts at a certain section in the hard disk or SSD, which when increasing or decreasing the partition size from the begining, it needs to move the partition data, increasing/decreasing the size at the ends doesn't need to move the data. So, by having it between the EFI partition and Linux partition, it fixes the issue.
**3. What is the minimum size for the partitions?**
EFI can be up to 512 Megabyes, Home partition can be whatever size you want, and the Linux partition needs to be at least 50-60 Gigabytes.
