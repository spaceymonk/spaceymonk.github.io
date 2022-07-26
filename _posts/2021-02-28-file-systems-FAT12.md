---
title: "File Systems: FAT12"
layout: post
categories: [File Systems, FAT]
customexcerpt: "In this post I will share how plain FAT12 file system, meaning without the long file name and directory support, works with all necessary information (equations, limits etc.)."
---

In this post I will share how plain FAT12 file system, meaning without the long
file name and directory support, works with all necessary information
(equations, limits etc.). In a later post, I will share how to write a file
explorer / image creator for FAT12 from scratch.

- table of contents    
{:toc}


# An introduction to FAT file systems

The File Allocation Table (FAT) file system was introduced with DOS v1.0 and
supposedly written by Bill Gates in 1977.

There are several FAT versions. Here is the characteristics of the most used
ones:

**FAT12** 
- designed for floppy disks
- uses 12 bit address
- can manage up to 4096 times cluster size

**FAT16**
- designed for early hard disks 
- uses 16 bit adress
- can manage up to 64 K times cluster size

**FAT32**
- increased address space
- uses **28** bit address
- can manage up to 256 M times cluster size

**ExFAT**
- improvements on FAT32
- actually uses 32 bit addressing

**VFAT**
- an ugly hack that enables long file names
- for example, FAT32 can use this to enable long file names

**Note:** The number of clusters are the total cluster number. In practice, you
will have reserved clusters which will decrease the number of free clusters.


## Why I chose FAT12?

+ Simple and robust.
+ Easy to understand.
+ It gives an insight about how file systems work in general.
+ It enable us to see some interesting concepts such as,
  + what happens to deleted files,
  + what is defragment,
  + why some files are named like `TEXTFI~1.TXT` in old DOS systems.  
+ If you know how FAT12 works, you will figured out the others.
+ Other versions of FAT are still commonly used on floppy disks, solid-state
  memory cards including USB flash drives, as well as many portable and embedded
  devices and digital cameras.
+ Can be used in [EFI
  system](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface),
  if you really want to.


# Structure of FAT12

Basically FAT12 file system consists of 4 main areas:

1. **Boot Sector (or reserved sectors):** Lies on the first sector on the
   volume. Gives information about how many copies of the FAT tables are
   present, how many sectors in a cluster, how big a sector is etc.
2. **File Allocation Table (FAT):** FAT tables contain pointers to every cluster
   on the volume and help where to find file on the volume.
3. **Root Directory:** Stores metadata information for each file. It has a
   finite size, limits the total amount of files that can be stored. For
   floppies it is 224, for harddisk is 512.
4. **Data Area:** Where the actual data is stored.

In the below figure we can see it more clearly:

<div class="img-wrapper" markdown="block">

![FAT12 layout](/assets/img/2021-02-28/fat12-layout.png)

</div>


I also added the sizes of each section in terms of sectors. That way you can
easily find the required parameters. From above figure we can deduce the start
offsets of each section (assuming boot sector is at offset `0x00000000`):

<div class="img-wrapper" markdown="block">

\\[\mathsf{address\,of\,first\,FAT = \sharp of\,reserved\,sectors \times bytes\,per\,sector}\\]
\\[\mathsf{address\,of\,root\,directory = address\,of\,first\,FAT + \sharp of\,FATs \times sectors\,per\,FAT \times bytes\,per\,sector}\\]
\\[\mathsf{address\,of\,data\,region = address\,of\,root\,directory + maximum\,\sharp of\,root\,entries \times 32}\\]

</div>

Now let's examine each section in detail.


## Boot Sector

Whether the given medium is partitioned or not, you will find MBR or VBR on the
first sector. We are interested in VBR. But I will give a little information
about the MBR as well.


### MBR

If your device is partitioned (like in harddisks), there is a special area that
stores the partition table and the boot code. This area resides in the first
sector (Logical Block Address, LBA for short, is 0) and it is called Master Boot
Record (MBR) which is 512 bytes in size with the last two bytes being signature.

The partition table has 4 entries, each of these have size of 16 bytes and
tells the computer where to find that partition and how big it is. If you
ever wondered why you can only have 4 primary partitions that's because of
the limitation on MBR.

Below table shows the structure of a partition table entry:

<div class="table-wrapper" markdown="block">

|Offset|Size|Description|
|---|---|---|
|`0x00`|1|status flag (`0x80`: bootable, `0x00`: non-bootable)|
|`0x01`|1|*CHS address of first block in partition* <br/> head number|
|`0x02`|1|*CHS address of first block in partition* <br/> sector number bits 0 to 5, cylinder number bits 7 to 6|
|`0x03`|1|*CHS address of first block in partition* <br/> cylinder number bits 0 to 7|
|`0x04`|1|partition type (`0x01` for FAT12)|
|`0x05`|1|*CHS address of last block in partition* <br/> head number|
|`0x06`|1|*CHS address of last block in partition* <br/> sector number bits 0 to 5, cylinder number bits 7 to 6|
|`0x07`|1|*CHS address of last block in partition* <br/> cylinder number bits 0 to 7|
|`0x08`|4|LBA of first sector in partition|
|`0x0C`|4|number of blocks in partition (little-endian)|

</div>

**Note:** CHS (cylinder-head-sector) addressing, *even MBR,* is not used in
modern systems. You can read more on the
[Wikipedia](https://en.wikipedia.org/wiki/Cylinder-head-sector).

As you can see, the MBR is telling where to find partitions. In order to find
that partition we look at the entries inside the partition table.

### VBR

If you follow that entry you will find the first sector of that partition being
Volume Boot Record (VBR). This partition as known as ***boot sector*** or volume
ID. After the offset `0x0B` is the BIOS Parameter Block (BPB). It describes the
general aspects of the partition. Most of the "geometric" information about the
device in here is mostly false and shouldn't be relied on. After this part, at
`0x24`; according to FAT type, Extended BPB (EBPB) comes.

There are different versions of BPB which explains different aspects of the
volume. Below table shows how VBR (boot sector) is formatted for FAT12 with
[DOS 3.31 BPB](https://en.wikipedia.org/wiki/Design_of_the_FAT_file_system#BPB331):

<div class="table-wrapper" markdown="block">

|Offset|Size|Description|
|---|---|---|
|`0x00`|3|Jump bytes. Since the first sector of the disk is loaded and executed, we need to jump over the disk format information. It can be `EB FE 90` which means infinite loop.|
|`0x03`|8|OEM identifier. It tells the formatting done in which operating system. On Linux its `mkdosfs`.|
|`0x0B`|2|Bytes for sector in two's power. Common value is 512.|
|`0x0D`|1|Number of sectors per cluster in powers of two. Common value is 4 KiB per cluster (do the math).|
|`0x0E`|2|Number of reserved sectors before the first FAT. Boot sector included. For FAT12 you can set to 1.|
|`0x10`|1|Number of FATs. Almost always 2. One for backup.|
|`0x11`|2|Maximum number of root directories. Must be set so that the root directory occupies entire sectors (i.e. the value must become a multiple of the sector size). For floppy disks it is 224.|
|`0x13`|2|Total sectors in the logical volume. If this value is 0, it means there are more than 65535 sectors and you should check the 4 byte value at `0x20`.|
|`0x15`|1|Media descriptor. `0xF8` for fixed disk, `0xF0` for 1.44 MB floppy disk. **Same value must be repeated as the first byte of each copy of FATs.** <br> IBM defined the media descriptor byte as 11111red, where r is removable, e is eight sectors/track, d is double sided.|
|`0x16`|2|Sectors per FAT. Use the formula in the note below.|
|`0x18`|2|Sectors per track.|
|`0x1A`|2|Number of heads.|
|`0x1C`|4|Hidden sectors that preceding the partition. Must be 0 on media that are not partitioned.|
|`0x20`|4|Total sectors. Should be used if the value at `0x13` is 0. Not used in FAT12 due to addressing capability.|

</div>


***Note:*** When calculating sector size for FAT, you should know the cluster
address size. It is **12 bits** which is 1.5 byte in FAT12. This means you can
address up to $$ 2^{12} = 4096 $$ clusters. So if you want to use maximum amount
of clusters then you should set this value as $$ 4096 \times 1.5 \div 512 = 12 $$
sectors according to below formula:

<div class="img-wrapper" markdown="block">

\\[\mathsf{sectors\,per\,FAT = \frac {\sharp of\,clusters \times cluster\,pointer\,size} {bytes\,per\,sector}}\\]

</div>

Other versions will have extensions over this BPB. You can find more from the
links under the [references](#references).

Some systems look at the cluster number to find out which FAT version is
present.


## File Allocation Table

The FAT can be considered as the *table of contents* of the data region. It
shows which cluster may be available for use, reserved or occupied by a file.
Since the data region is divided into equally sized clusters, a file can either
fit inside a cluster or not. If a file is small enough to fit inside a cluster,
the FAT is pretty much unnecessary. Otherwise, it helps us to track the data, as
clusters of a file are *chained* together. And when we are reading a file, we
look at the the root directory to get the first cluster of the file and then
simply find the next cluster by looking at the FAT.

Below table shows the entry values and their meanings:

<div class="table-wrapper" markdown="block">

|Value|Meaning|
|---|---|---|
|`0x000`|Free Cluster|
|`0x001`|Reserved Cluster|
|`0x002 - 0xFEF`|Data Cluster (value that represents next cluster)|
|`0xFF0 - 0xFF6`|Reserved Cluster|
|`0xFF7`|Bad Cluster|
|`0xFF8 - 0xFFF`|Last Cluster (EoC)|

</div>

**Be careful** that each value is 12 bits long and in little endian. For
example, if you read 3 bytes (24 bits), the least significant 12 bits
corresponds to first entry and the most significant 12 bits corresponds to the
second.

Because of the reserved cluster, the data region starts at the cluster #2
(numbered from 0). I think, we need this because the first byte of the FAT is
the media descriptor which is 1 byte long, since it corrupts the format we
padded with `0xF` and started with cluster #2. If we use `0xFFF` value to denote
the last cluster, then we have 4084 different value for a cluster *(why?)*. This
means that a file can span all 4084 clusters at maximum. Now we can calculate
the maximum size limit for a file, as follows:

<div class="img-wrapper" markdown="block">

\\[\mathsf{maximum\,file\,size = sectors\,per\,cluster \times bytes\,per\,sector \times \sharp of\,clusters}\\]

</div>

For 4084 clusters, if you take sector size as 512 bytes and 128 sectors (64 KiB)
per cluster then you have a maximum of ~256 MB but the slack space will be
immense. Or choose a common value, like 4 KiB cluster size and that makes ~16
MB.


### Fragmentation

The clusters of a file are not always store in adjacent to each other. If they
are not adjacent then we say there is a *fragmentation*. In theory, this makes
no difference, because we follow the next cluster in the chain no matter what.

But in real life, there is a downside: If this cluster chain has spreaded all
over the FAT, seeking from one end to the another end to get the cluster
address, slows down the process. But why is this happening? The answer is about
the design of the storage medium. Since reading speed of an harddisk is heavily
depends on the location of the data on the magnetic disks, non-adjacent readings
may be cause harddisk to do time consuming actions like seeking between magnetic
heads.

This was a big problem in hard drive and tape drive but not on USB drives or
SDDs. Because reading random cells has a constant speed on these devices, so
fragmentation is not a problem.

**Warning:** Neither Hidden nor System files should be moved during
defragmentation or any other disk service.


## The Root Directory

FAT12 has only one directory called *Root Directory*, without the directory
support. It is consists of 32 bytes long entries. The size of the root directory
is important since it limits the number of files in the partition. In the
previous section we saw that an FAT entry can have 4084 different cluster
addresses, meaning up to 4084 number of files can be stored. But it is possible
only if the number of root directory entries is 4084. This value is usually 224
for floppies and 512 for non-floppy devices.

Each entry has the following structure but keep in mind that the usage of some
bytes (especially reserved bytes) may differ from system to sytem, if you want
more detailed information, you may take a look at the [reference](#references)
links:

<div class="table-wrapper">
<table>
<tr>
  <th>Offset</th>
  <th>Size</th>
  <th>Description</th>
</tr>
<tr>
  <td><code>0x00</code></td>
  <td>11</td>
  <td>Name of the file in short format.
      Explained in the <a href="#short-file-names-sfn">next section</a>.<br>
      The first byte is significant and can have the following special values:
      <table style="width: calc(100% - 10px); margin: 10px">
      <tr>
        <th>Value</th>
        <th>Meaning</th>
      </tr>
      <tr>
        <td><code>0x00</code></td>
        <td>Entry and the remaining entries are empty.</td>
      </tr>
      <tr>
        <td><code>0x05</code></td>
        <td>Inital character is actually <code>0xE5</code></td>
      </tr>
      <tr>
        <td><code>0x2E</code></td>
        <td>'Dot' entry; either "." or ".."</td>
      </tr>
      <tr>
        <td><code>0xE5</code></td>
        <td>Entry has been deleted.</td>
      </tr>
      </table>
  </td>
</tr>
<tr>
  <td><code>0x0B</code></td>
  <td>1</td>
  <td>File attributes:
      <table style="width: calc(100% - 10px); margin: 10px">
      <tr>
        <th>Bit</th>
        <th>Meaning</th>
      </tr>
      <tr>
        <td>0</td>
        <td>Read Only</td>
      </tr>
      <tr>
        <td>1</td>
        <td>Hidden</td>
      </tr>
      <tr>
        <td>2</td>
        <td>System</td>
      </tr>
      <tr>
        <td>3</td>
        <td>Volume Label</td>
      </tr>
      <tr>
        <td>4</td>
        <td>Subdirectory</td>
      </tr>
      <tr>
        <td>5</td>
        <td>Archive</td>
      </tr>
      <tr>
        <td>6</td>
        <td>Unused, Device (never found on disk, internal use only)</td>
      </tr>
      <tr>
        <td>7</td>
        <td>Unused</td>
      </tr>
      </table>
      Note: <code>0x0F</code> value means a long file name entry.
  </td>
</tr>
<tr>
  <td><code>0x0C</code></td>
  <td>2</td>
  <td>Reserved.</td>
</tr>
<tr>
  <td><code>0x0E</code></td>
  <td>2</td>
  <td>Creation time with the following format:
    <table style="width: calc(100% - 10px); margin: 10px">
      <tr>
        <th>Bits</th>
        <th>Meaning</th>
      </tr>
      <tr>
        <td>0-4</td>
        <td>Seconds / 2 (0 to 29)</td>
      </tr>
      <tr>
        <td>5-10</td>
        <td>Minutes (0 to 59)</td>
      </tr>
      <tr>
        <td>11-15</td>
        <td>Hours (0 to 23)</td>
      </tr>
    </table>
  </td>
</tr>
<tr>
  <td><code>0x10</code></td>
  <td>2</td>
  <td>Creation date with the following format:
    <table style="width: calc(100% - 10px); margin: 10px">
      <tr>
        <th>Bits</th>
        <th>Meaning</th>
      </tr>
      <tr>
        <td>0-4</td>
        <td>Days (1 to 31)</td>
      </tr>
      <tr>
        <td>5-8</td>
        <td>Months (1 to 12)</td>
      </tr>
      <tr>
        <td>9-15</td>
        <td>Years (0 means 1980, 127 means 2107)</td>
      </tr>
    </table>
  </td>
</tr>
<tr>
  <td><code>0x12</code></td>
  <td>2</td>
  <td>Last access date. Same format at <code>0x10</code></td>
</tr>
<tr>
  <td><code>0x14</code></td>
  <td>2</td>
  <td><a href="https://en.wikipedia.org/wiki/File_Allocation_Table#Extended_Attributes">Extended Attributes Index (EA-index)</a> for FAT12.</td>
</tr>
<tr>
  <td><code>0x16</code></td>
  <td>2</td>
  <td>Last modified time. Same format at <code>0x0E</code></td>
</tr>
<tr>
  <td><code>0x18</code></td>
  <td>2</td>
  <td>Last modified date. Same format at <code>0x10</code></td>
</tr>
<tr>
  <td><code>0x1a</code></td>
  <td>2</td>
  <td>First cluster in FAT12. For empty files, entries with volume flag set, subdirectory pointing to root set the value to 0.</td>
</tr>
<tr>
  <td><code>0x1c</code></td>
  <td>4</td>
  <td>File size in bytes</td>
</tr>
</table>
</div>

The parts required for manipulating files are listed below:

1. the file name,
2. the file attributes,
3. the first cluster in FAT,
4. file size in bytes.

File name is the file's access point for the user. Format explained briefly in
the next section.  
The file attributes are important because the entry may not be a solid file
entry, such as long file name entry etc.  
In order to access the file we need to know where it lays on disk. Thus, we need
first cluster address. The formula for the start address of a file is:

<div class="img-wrapper" markdown="block">

\\[\mathsf{begin\,address = address\,of\,data\,region + (cluster \sharp - 2) \times sectors\,per\,cluster \times bytes\,per\,sector}\\]

</div>

And finally, file size. From that we can deduce where the file ends *inside* the
cluster:

<div class="img-wrapper" markdown="block">

\\[\mathsf{EOF\,offset = file\,size \bmod cluster\,size}\\]

</div>


### Short File Names (SFN)

FAT12 does not support arbitrary length file names due to constant sized root
entries. Instead it uses a format called "8.3 Filename". It is a simple format:
first eight characters refers to the file name, last three refers to the file
extension which is 11 characters in total. If the file name or extension is
shorter than 8 and 3 bytes respectively, then remaining fields are padded with
space (`0x20`) while storing. There can only be one period character. SFN
consists of characters that can only be represented in ASCII.

In summary, following characters can be used in SFN:

* A to Z, all upper case letters
* 0 to 9, all numbers
* Space (trailing spaces will interpreted as padding)
* Period (only use for seperating extension. Trailing periods will be discarded)

If you copy a file with a long name into the FAT12 partition, that name needs to
be converted into SFN format. I will explain this procedure in a later post. But
basically if it is longer than 8 bytes it will be truncated. For example,
`afilename.txt` becomes `AFILEN~1.TXT`. That last number will increase if there
are collisions. For example, `afilenamethatislonger.txt` will be `AFILEN~2.TXT`.


# Terminology

|cluster|In order to store data, the partition is divided into equally-sized memory blocks or fragments. Each of this blocks is called a cluster. Larger cluster size can produce slack space, smaller ones may cause to run out FAT addresses.|
|slack&nbsp;space|When a file has saved on disk, it is divided into clusters. Sometimes cluster size is much bigger than the actual file size. In that case the remaining size inside the cluster cannot be used. This dead memory is called slack space.|
|TFAT|Transaction-Safe FAT File System. While performing a drive operation, changes would be made to second FAT. When the operation is completed it is copied to the first FAT.|


# References

- [FAT12 Overview by Samuel Skånberg](http://fileadmin.cs.lth.se/cs/Education/EDA385/HT09/student_doc/FinalReports/FAT12_overview.pdf)
- [TU/E University - FAT](https://www.win.tue.nl/~aeb/linux/fs/fat/fat-1.html)
- [Wikipedia - Design of the FAT file system](https://en.wikipedia.org/wiki/Design_of_the_FAT_file_system)
- [Wikipedia - File Allocation Table](https://en.wikipedia.org/wiki/File_Allocation_Table)
- [Working with short 8.3 filenames](http://cis2.oc.ctc.edu/oc_apps/Westlund/xbook/xbook.php?unit=01&proc=page&numb=10)
- [Microsoft - 8.3 Filename](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-fscc/18e63b13-ba43-4f5f-a5b7-11e871b71f14)
- [Wikipedia - 8.3 Filename](https://en.wikipedia.org/wiki/8.3_filename)
- [OSDev - FAT](https://wiki.osdev.org/FAT)
