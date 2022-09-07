---
title: New challenges for old tricks - RAID56 in BTRFS
author:
- Qu Wenruo <wqu@suse.com>

# Background

Although Btrfs is already the default filesystem for root for a long long time,
and also the default filesystem for Fedora recently, there is a not-so-small
chunk of feature completely truncated from SLE kernel: RAID56.

Unsurprisingly btrfs RAID56 is also marked as unsafe in the upstream
documentation:
[https://btrfs.readthedocs.io/en/latest/btrfs-man5.html#raid56-status-and-recommended-practices]

In this talk, I'll go through the rabbit holes of:

- RAID56 in generic

  Not an expert here, Neil Brown would provide more details on this topic.
  Thus this would really be a simple introduction to the RAID56 features used
  in btrfs.

- RAID56 in btrfs

  All the problems we found, and possible solutions we're discussing.

- Brand new world for RAID56

  This involves zoned devices (including the infamous consumer level SMR disks).

# RAID56 in generic (the light side)

## RAID4

Although RAID4 is not utilized by btrfs, it provides a very good basis to explain
the whole RAID456 family.

The idea itself is pretty straightforward:

```
    Disk 1 :   |  Data block 1  |  Data block 4  | ...
    Disk 2 :   |  Data block 2  |  Data block 5  | ...
    Disk 3 :   |  Data block 3  |  Data block 6  | ...
    Disk 4 :   | Parity (1+2+3) | Parity (4+5+6) | ...
```

In above example, `Disk 4` is always storing the parity, which is the XOR result
of all the data stripes in the same vertical stripe.
While for disk 1~3, it's no different than RAID0.

TLDR: RAID4 is RAID0 plus one dedicated device storing the parity.

For recovery:

If we lost one of disk 1~3, we can recover the data by XOR parity and the
remaining data stripes.
If we lost disk 4, there is no immediate data loss anyway.

Thus for RAID4, we can tolerate one missing device.

*OR DO WE?*

## RAID5

Then we come to RAID5:

```
    Disk 1 :   |  Data block 1  | Parity (4+5+6) |  Data block 9  | ...
    Disk 2 :   |  Data block 2  |  Data block 4  | Parity (7+8+9) | ...
    Disk 3 :   |  Data block 3  |  Data block 5  |  Data block 7  | ...
    Disk 4 :   | Parity (1+2+3) |  Data block 6  |  Data block 8  | ...
```

Initially for the first vertical stripe, it looks no different than RAID4.
But when comes to the next vertical stripe, it's different.
There is a *ROTATION*!


This in fact addresses one of the sin of RAID56, parity update.

Since parity is the XOR of all data blocks, if any data block gets updated,
parity stripe has to be updated too.

For RAID4, if we just touched data block 1, 2, 3 sequentially, every time we
touch the data block, parity also has to be update, meaning it's updated
way more frequently than data stripes.
This can lead to a performance bottleneck, thus RAID5 distribute the parity
stripes across all devices, to also distribute the parity update loads.

But remember this, it will come back to bite us again.

## RAID6

For RAID45, we can only tolerate one device, which is fine if one only has
a small pool of devices.
When the number of devices go up, we want more tolerance, thus here comes
RAID6.

```
    Disk 1 :   | Data block 1 |  Q (3 + 4)   |   P (5 + 6)    | ...
    Disk 2 :   | Data block 2 | Data block 3 |   Q (5 + 6)    | ...
    Disk 3 :   |  P (1 + 2)   | Data block 4 |  Data block 5  | ...
    Disk 4 :   |  Q (1 + 2)   |  P (3 + 4)   |  Data block 6  | ...
```

P is still calculated the same as RAID5.

But Q is more complex, for the details of such black magic, just go check the
wikipedia page: [https://en.wikipedia.org/wiki/Standard_RAID_levels]

# RAID56 in generic (the dark side)

It sounds pretty simple so far, then what can go wrong?

- Data synchronization

  One can not expect data and parity stripes can always reach disk at the same
  time. What if only parity is updated then a power loss happened?

  Or even worse, only parity write reached the disk, then the device containing
  the parity chose to go a strike after power is recovered?

  Thus there comes things like journal (or partial parity log for RAID5) to
  address this.

  Mostly we write the updated data into somewhere else before we submit
  the real data to disks.
  Then at recovery time we re-write the data.

  At least, we have solution for this for a long time.

- Rotten bit

  If one or more bits inside data or parity stripes is rotten?
  In theory we have the extra parity stripe(s) which can help us to recovery
  the data.

  But how to determine which one is correct/corrupted?

- Destructive read-modify-write (RMW)

  Obviously we have to update a RAID56 full stripe by reading out all the data
  (at least the untouched part), modify the data and update the parity, then
  write all the new data stripe and parity to disk.

  But what if we also have rotten bits?

```
    Disk 1 :   | Good data 1 | Good data 4 |
    Disk 2 :   | Good data 2 | Bad data  5 |
    Disk 3 :   | Good data 3 | Good data 6 |
    Disk 4 :   | G P (1+2+3) | G P (4+5+6) |
```

  If we're updating data 6, we need to read out data 4, 5 at least.

  But data 5 is corrupted, if we calculate the new parity using good 4,
  bad 5, new 6, then we lost the chance to recover data 5.

  The only chance is to read out all data 4 5 6, and parity (4+5+6).

  Then by some super dooper black magic, we find out data 5 is corrupted,
  then rebuild data 5, then go back to the old RMW path.

  But how?

- A completely new challenge that comes from the ancient day

  Hint, tape.

# RAID56 in btrfs

## How btrfs handles RAID profiles?

Btrfs handles various RAID profiles in a block group by block group base.

This means, btrfs can have one block group in RAID1 mode, while another one
in RAID5 mode, like this:

```
  Btrfs logical address space:

  0G    1G     2G     3G     4G  ...
  |     | BG A |      | BG B |
           |              \- RAID 5, using devices:
           |                  Devid 1 Physical X1
           |                  Devid 2 Physical X2
           |                  Devid 3 Physical X3
           |
           \- RAID1, using devices:
               Devid 1 Physical Y1
               Devid 2 Physical Y2
```

## Bugs related to BTRFS RAID56

This is not going to feel good...

- Write-hole

  This is exactly the same problem of data synchronization problem.

  And unfortunately we don't have any upstream solution.

  Write-intent bitmap or journal is purposed, but don't rush, there
  are new problems facing them.

  This is the problem from day 1, and still bothering us today.

- Subpage support

  Just try to mkfs on an PowerPC/Aarch64 with 64K page size, then try
  to mount it with a x86_64 system.

  Other filesystems address this by having subpage support, meaning
  they can have block size smaller than page size.
  Thus for PowerPC or 64K page sized Aarch64, they can handle block size
  (normally 4K, created by x86_64) smaller than their page size.

  (On the other hand, they can not handle blocksize larger than page size,
   just like recent btrfs).

  Thankfully RAID56 part is addressed in v5.19.

- Over-updated parity

  Parity is already updated more frequently than data stripes, what can go
  worse?

  Well, update the whole parity, even for ranges which doesn't need to be
  updated.

  Initially this sounds no big deal, right? Just more IO for a already
  bottlenecked parity stripe.

  But remember the above destructive RMW? Now it's not funny any more.

  Thankfully this is addressed in v6.0.

## A new hope

But not everything is bad, we had one thing to save our backend:

- Checksum

  If the rotten bit happens in a data stripe, and we're reading that corrupted
  sector, and we have checksum for it.

  Then we can recover!

Furthermore, since btrfs has checksum for all of its metadata and part of the data
(default to have data checksum, but can be disabled), in the future we can
have our chance to do full verification before doing RMW.

Thus this can solve the destructive RMW problem! (at least in the future, with the
extra IO as the cost)

# Tape's counterattack

## Zoned device

Tapes sound old (well it's still in-use today), but it's going to bite us now, just
in a different name: zoned devices.

Zoned devices are recently pushed by WDC, and our old friend Johannes Thumshirn
<Johannes.Thumshirn@wdc.com> actively working on that.

So what's zoned device in the first place?
[https://zonedstorage.io/docs/introduction/zoned-storage]

In short, it's kinda like an enhanced tape.

The tape part is:

- No overwrite ability

  If some data is written, one can not overwrite that data, at least not
  directly.

- There will be a write pointer indicating the next write position

  Just like the tape head.

The enhanced part is:

- We can still do random reads without any limitation

  Just like the old disks.

- Multiple zones in a zoned device

  Thus you can consider it as multiple independent tape headers for write,
  sounds awesome right?
  Well, the number of opened zones for write has limits (7 for WDC zoned devices?)

- You can still do write with queue depth larger than 1

  But this comes with a drawback, one can not really know where the
  data is going to be written.

  The zoned device will tell you the real physical position to read,
  after the write is finished.

- Some zoned device can still have conventional zones

  In those conventional zones, we can do the overwrite just like
  it's still in the 90s.

## Btrfs zoned support 

Currently btrfs (and f2fs?) are the only two filesystems supporting zoned device.

The COW nature of btrfs makes it a perfect match, CoW converts the metadata updates
into sequential write with queue depth 1.

While for cases of non-COW data writes (from prealloc to NODATACOW files), btrfs just
rejects them on zoned device.

For other CoW based data write, we go queue depth >1, and catch the returned
bytenr from the zoned device, record them into metadata, then rely on above
mentioned metadata writeback behavior.

But why this would bother RAID56?

## RAID56 and zoned device

### How to support RAID56 with zoned device?

The nature of zoned devices makes the following thing much harder

- How to do journal/write-intent to synchronize the devices?

  No overwrite thus it is not suitable to go the original write-intent
  bitmap/journal.

  There is still a small chance, going ring buffer write-intent/journal
  into two dedicated zones.

  But the real world zoned devices have more limitations, they have limited
  amount of opened zones, extra zones can be too expensive or just impossible.

- How to do proper data writes?

  Since either we don't know where the data will land in advance (using queue depth > 1),
  or the performance is as slow as nail (using queue depth = 1)

Johannes is introducing a new feature called raid-stripe-tree, introducing
a new flush-translation like layer.

So in btrfs address space, we are still thinking we're using certain device range
to build our RAID profiles.

But when we start reading some data using that physical bytenr, we refer to the
raid-stripe-tree to get the real bytenr inside the zone.

And at write time, we update the record in raid-stripe-tree to build the
mapping of real bytenr inside the zone and the traditional btrfs device bytenr.

This would work pretty well for data chunks with RAID0/1/10.

But what about RAID56?

### The parity strikes back

Remember that parity got a much frequent update?

Let's see an example using 64K stripe length (data and parity stripe length),
and doing 48K sync write with 16K block size..

We write the first 4 16K:

```
XXX: Used space.

               0     16K   32K   48K   64K
    Disk 1 :   |XXXXX|XXXXX|XXXXX|XXXXX| Data stripe 1
    Disk 2 :   |     |     |     |     | Data stripe 2
    Disk 3 :   |     |     |     |     | Data stripe 3
    Disk 4 :   |XXXXX|XXXXX|XXXXX|XXXXX| Parity stripe
```

So far so good, nothing much different than regular RAID56.

But what about the next 16K block?

```
               0     16K   32K   48K   64K
    Disk 1 :   |XXXXX|XXXXX|XXXXX|XXXXX| Data stripe 1
    Disk 2 :   |?????|     |     |     | Data stripe 2
    Disk 3 :   |     |     |     |     | Data stripe 3
    Disk 4 :   |XXXXX|XXXXX|XXXXX|XXXXX| Parity stripe
                ?????
```

Normally we should update the 16K block on Disk 2, and update overwrite
the first block of Disk 4.

But this is zoned device, we can not write before your write pointer.

Now we have to make some hard calls, by either:

- Mark the full stripe full

  As two device have exhausted their allocated space.

- Write the new block to unused space in Disk 2/3

  Since we have RST, in theory we can write the data to any disk,
  ignoring the strict RAID56 rotation.


But either way, there are obvious problems:

- Mark the full stripe full

  Super bad space efficiency.

  We can only write 64K into a full stripe which is supposed to be 3 * 64K.

- Write the new block to unused space of other disks

  Then one day we may have a data block and its corresponding parity on the
  same disk!

  Then we can no longer tolerant one missing device.


Although not all hope is lost for the first case, for btrfs zoned mode, there
are reserved zones, and automatic zones reclaim mechanism.

That means, for above block groups with parity zone already exhausted, btrfs
can read it out and write into another block group (of course, writing them
in a more RAID56 friendly way).

But there are always ways to screw it up, like filling the fs, then only delete
all the data on one data stripes, and try to write data back.

No to mention ENOSPC problems of btrfs is already somewhat infamous.

## The last call (BBS)

So there seems to be a spectrum of methods to solve RAID56 problems so far:
(Definitely unrelated to politics)

```
                        /- Will there be some good middle ground?
                        |
  The progressive       |                    The conserative
  |<--                  |                 -->|
  |                                          \- Use traditional solution
  |                                             like journal/write-intent
  |                                             to solve RAID56 problems on
  |                                             conventional disks.
  |
  \- Use the new zoned solution to solve everything
     Even if the disks are conventional we still treat them as zoned devices
     (aka, zoned emulation mode)
```

For the zoned emulation mode, it seems to be a silver bullet.
But with more hidden causes of ENOSPC, I'm not that sure if it should be the
way to go.

For the traditional method, it's tried and true, we even have initial patches
submitted for write-intent bitmap support for btrfs.
And no new cause of ENOSPC at all.
But that means, zoned devices are completely ignored.

Personally speaking, I have seen too many flops chasing one-fit-all solutions.
Like the initial GNOME 3 and Windows 8 for touchscreen.

Thus I'm willing to go split ways to solve the same problems, until we really
got a silver bullet (that a big IF though).

There is also some new ideas, like some extra encoding for the data (like
compression), but can stand a big chunk of missing data.

But no really suitable solution for now.
