---
layout: post
title: Setting up a ZFS disk array with RAIDZ3 and LUKS
date: 2018-05-04
tags: sysadmin zfs linux
---

Recently I have been looking for a way to get more storage space. I've been looking for

* redundancy
* ease of use
* speed

in the sense that it should be able to tolerate several concurrent disk failures without data loss, should appear a single volume comprised of multiple disks so I shouldn't have to worry about what disk to put things on, and should be fast enough to stream video from. So when I saw a HP MSA 60 on CraigsList, which included some hard drives and a PCIe SAS controller, it sounded perfect.

At first I didn't know what a disk array was, I thought it was a big server and not a bunch of disks hooked up to a backplane.

Getting it working was pretty easy, all I had to do was install the PCIe SAS controller card and then I could see all of the disks in /dev. At first I thought I had to flash the controller firmware to IT (initiator target) mode, which configures it to do simple passthrough instead of doing hardware RAID (i.e., JBOD mode) since I planned on using it with ZFS. I actually spent a lot of time on this, before realizing the previous owner had already flashed it to IT mode (oops!).

It's rather loud, not earsplitting but certainly you would be unable to sleep with it in the same room, so I decided I would place it in another room with another computer and use it as a NAS, via samba or NFS or something, I haven't decided yet. At first I wasn't sure which ZFS configuration to use - I could have two RAIDZ arrays, and then mirror them, or I could have them all in one big RAIDZ. I decided, to keep things simple and fast, that would use ZRAID, and since you also cannot expand a ZRAID pool after creation[^0], I would have to fill the entire array (12 disks) before making the array. I also decided to use RAIDZ3, which can tolerate up to three concurrent disk failures before data loss[^1], as opposed to only two with a mirroring setup (if each half of a pair dies at once). The extra redundancy of RAIDZ3 also works out well because I am a cheapskate and didn't want to pay more for non-secondhand drives while also wanting to not worry about data loss, so the disks will probably fail more often than newer disks.

Another aspect of RAIDZ is that it uses the size of the smallest disk in your array, assumes all drives to be this size, and then ignores the rest of the space - so if you have 11x2TB disks and 1x500GB disks, you have effectively 6TB of data to work with (before parity). I recieved with the array some 500GB and 1TB disks, but I didn't really want to be stuck with only 4.5TB of effective space until I could be bothered replacing the rest. So, I got some cheap secondhand disks off of eBay, with the goal in mind that the smallest disk in the array should be 1TB. They actually seem to be pretty good, some of the disks were HGST SAS drives with a manufacture date of mid-2015[^2], while some were Seagates with a manufacture date of 2011 and about 55k hours on them. I also scrounged around and found a few more 1TB and 2TB drives I could use for the array. Another feature of the array is that it will get double speed if there are only SAS disks in it (600MB/s vs 300MB/s for SATA2 or mixed SAS/SATA).

The next step was to write over all of them with random data, as I plan to encrypt them with LUKS and that is the standard best practice, with the goal being to prevent leakage of information about the plaintext - some adversary shouldn't be able to see which parts of the disk have encrypted data on them, and which simply contain random data. At first I was worried about exhausting the kernel's store of entropy (and thus having very slow generation of random data) so I tried shred, but it was far too slow - it was writing to each disk at about 7MB/s, and maxing out the four cores on the CPU. So, I decided to just try urandom anyway, and it actually was quite a bit faster, writing to each disk between about 20MB/s and 12MB/s. This is still very slow, but it has a slow CPU, so close enough. All in all, this step took roughly three days.

After all of the disks were in the array and had been overwritten with random data, the next step was to figure out disk encryption. Unfortunately, the CPU this machine is running on is quite old and slow, and lacks hardware accelerated AES:

```
# cryptsetup benchmark
# Tests are approximate using memory only (no storage IO).
PBKDF2-sha1       919803 iterations per second for 256-bit key
PBKDF2-sha256    1243862 iterations per second for 256-bit key
PBKDF2-sha512     828259 iterations per second for 256-bit key
PBKDF2-ripemd160  593085 iterations per second for 256-bit key
PBKDF2-whirlpool  389515 iterations per second for 256-bit key
#     Algorithm | Key |  Encryption |  Decryption
        aes-cbc   128b   131.1 MiB/s   159.6 MiB/s
    serpent-cbc   128b    63.3 MiB/s   267.7 MiB/s
    twofish-cbc   128b   141.0 MiB/s   208.1 MiB/s
        aes-cbc   256b   122.5 MiB/s   137.6 MiB/s
    serpent-cbc   256b    61.0 MiB/s   261.8 MiB/s
    twofish-cbc   256b   153.3 MiB/s   209.0 MiB/s
        aes-xts   256b   152.5 MiB/s   166.7 MiB/s
    serpent-xts   256b   233.9 MiB/s   245.5 MiB/s
    twofish-xts   256b   190.0 MiB/s   161.3 MiB/s
        aes-xts   512b   127.8 MiB/s   134.2 MiB/s
    serpent-xts   512b   238.1 MiB/s   254.4 MiB/s
    twofish-xts   512b   197.9 MiB/s   197.7 MiB/s
```

Since *none* of these are even fast enough to saturate the link to my SAS controller even at 300MB/s, I decided to just go with the fastest algorithm available, 512 bit serpent-xts-plain64 and sha256. What I really would have liked to use was the Threefish cipher and Skein hashing algorithm, which are both designed to be both secure and very fast by only using simple arithmetic operations which are very fast on general-purpose hardware[^3], as opposed to AES which uses more complex operations that require specialized hardware to accelerate. Unfortunately only Skein is available in the Linux kernel (in ```drivers/staging/skein```), and it wasn't possible to benchmark it in LUKS as ```cryptsetup benchmar``` only accepts a --cipher argument, so I ended up not using it. Interestingly enough, Skein uses Threefish as part of its algorithm, but unfortunately the one used to implement Linux's version of Skein isn't hooked up to the kernel's cryptography subsystem, so I can't use it here. Hopefully some day Threefish can be used with LUKS for disk encryption on slower computers.

After configuring encryption for all of the disks, making the array was quite simple:

```
# zpool create -f -o ashift=12 -o autoexpand=on reliquary raidz3 /dev/mapper/reliquary0 /dev/mapper/reliquary1 /dev/mapper/reliquary2 /dev/mapper/reliquary3 /dev/mapper/reliquary4 /dev/mapper/reliquary5 /dev/mapper/reliquary6 /dev/mapper/reliquary7 /dev/mapper/reliquary8 /dev/mapper/reliquary9 /dev/mapper/reliquary10 /dev/mapper/reliquary11
```

I am using ashift=12 because some of the disks I believe have (or might in the future have) 4k sectors, and the penalty for using a 4k sector size on a 512b disk is much smaller than the converse.

RAIDz3 seemed to be pretty much perfect for my strategy of getting a bunch of older and cheaper secondhand disks to work together with without worrying too much about data loss (I still plan to take other backups though, of course), but there are a few other steps I will take. One of them is to do a weekly ```zfs scrub``` and SMART long tests with smartd. Another is to try to keep the array at least half full at all times, because ```zfs scrub``` only checks the blocks which are known to contain data - so if part of the disk becomes damaged that is not in use, then ```zfs scrub``` will not find it. The last, and most important thing, is to replace disks as soon as possible when they have failed - thus decreasing the chance of disk failures being concurrent with each other. Indeed, by filling the array partway with random data and initiating a scrub, I already found a failing disk, not two days after array initialization:

```
# zpool status
  pool: reliquary
 state: ONLINE
status: One or more devices has experienced an unrecoverable error.  An
	attempt was made to correct the error.  Applications are unaffected.
action: Determine if the device needs to be replaced, and clear the errors
	using 'zpool clear' or replace the device with 'zpool replace'.
   see: http://zfsonlinux.org/msg/ZFS-8000-9P
  scan: scrub in progress since Thu May  3 20:57:22 2018
	151G scanned out of 3.00T at 20.5M/s, 40h28m to go
	38.0M repaired, 4.91% done
config:

	NAME             STATE     READ WRITE CKSUM
	reliquary        ONLINE       0     0     0
	  raidz3-0       ONLINE       0     0     0
	    reliquary0   ONLINE       0     0     0
	    reliquary1   ONLINE       0     0     0
	    reliquary2   ONLINE       0     0     0
	    reliquary3   ONLINE       0     0     0
	    reliquary4   ONLINE       0     0     0
	    reliquary5   ONLINE       0     0     0
	    reliquary6   ONLINE       0     0     0
	    reliquary7   ONLINE       0     0     0
	    reliquary8   ONLINE   2.93K     0     0  (repairing)
	    reliquary9   ONLINE       0     0     0
	    reliquary10  ONLINE       0     0     0
	    reliquary11  ONLINE       0     0     0
```

I noticed it had some reallocated sectors, and a single pending sector, while I was writing random data to it. The reallocated sector count was 107 during the wipe, but ```smartctl``` showed it had risen to 880, which is a sign of impending disk death (some fields redacted):
```
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  5 Reallocated_Sector_Ct   0x0033   058   058   036    Pre-fail  Always       -       880
196 Reallocated_Event_Count 0x0033   058   058   036    Pre-fail  Always       -       880
197 Current_Pending_Sector  0x0012   100   100   000    Old_age   Always       -       5
```

This is especially concerning given that those pending sectors have occurred *after* I had written random data over the entire disk - sectors are pending until either a read from them succeeds (which could either take a very long time or never happen at all, given that the whole reason a sector is reallocated at all is due to some hardware problem with that sector), or until they are written to, which results in the new data being written to the freshly reallocated sector and the data in the old, damaged sector forgotten.

I also got a pair of cheap SSDs to use for an external ZFS Intent Log, which have been working pretty well, it made the server much more responsive when doing IO-heavy things like downloading large files with bittorrent.

All in all, I am pretty pleased with my purchases so far. As long as I stay vigilant about staying on top of my disk/array health, by:

* doing weekly smart tests
* making sure there are no reallocated sectors (sign of impending drive death) or read errors in the smart reports
* doing weekly zfs scrubs to check for corruption
* frequently checking zpool status to make sure there are no checksum/read errors
* most importantly, replacing failing or failed disks as soon as possible

I hope to keep my array of old and secondhand parts in good working order for a long time to come.

[^0]: (yet...? https://twitter.com/OpenZFS/status/921042446275944448?s=09 )
[^1]: ("data loss" in this context means "complete loss of all of the data in the array", theres not really much of an in-between, unless youre okay with missing an evenly distributed 1/12th of all of your data)
[^2]: (they might even be under warranty? it would be pretty cool if they were)
[^3]: https://www.schneier.com/academic/skein/threefish.html
