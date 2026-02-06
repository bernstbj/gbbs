# GBBS Pro / ACOS Message Extraction Tool
*February 2026, Brian J. Bernstein*

## Why this project?
This is a project that I created to help in extracting messages and emails from a BBS that I ran back in the 1980s on an Apple //e with GBBS Pro. The reason for this is because I want to look at those old messages for nostalgia purposes, but unfortunately the way messages were stored in GBBS Pro was a proprietary format and thus not easily done. As such, I needed to create a tool to extract these messages on a modern computer.

## Why was there a proprietary file format?
Unlike today where you could attribute a proprietary format to some kind of vendor lock-in, in this case actually a pretty clever thing to do. The Apple II is an 8-bit platform with limited storage; 5.25" floppy disks held 140K and if you had a lot of money to throw at it, you could get a 5, 10, or 20MB hard disk (HDD) the size of a large shoebox. Most people had one or two floppy drives (which were expensive enough as it was), but if someone could cough up several hundred dollars for a hard disk, you were in BBS-operating heaven. For perspective, in 1988 you could buy a Sider 20MB hard drive for an Apple II for about $600 ($1650 in 2026 dollars). Mind you, this was after years of cost reductions on HDDs for personal computers – if you looked at something like this a few years earlier, it was possibly double the cost for half the storage.

Most software on the Apple II didn't really know how to use a hard disk, so you were either a computer enthusiast or business person to justify such an expense. People playing games, doing homework, or small business tasks would be just fine with one or two 5.25" floppy drives. However, a BBS could easily outgrow a few floppy drives, especially if you wanted to offer files that users could download (games, text files, etc.). Running a BBS was a perfect use for a HDD since floppy disks were just too limiting.

But even if you could afford a HDD for your BBS, the low-level details of how computers store files on floppies or HDDs reveal how inefficient it can be to store small messages that are only a few dozen words long. With the ProDOS operating system on the Apple II, disks were laid out to have many “sectors” to hold data, and these were each 512 bytes in size. So when you store a file, the operating system would allocate one-to-many sectors, chain them together, and put the file contents into that chain of sectors. But most messages written by people tend to be a dozen or two words long which often doesn't even reach 512 bytes in size. As such, the shortest message being saved as an individual file would still take up one of those sectors thus wasting potentially hundreds of bytes on already small storage devices.

So the clever proprietary format that GBBS Pro used somewhat solved this inefficiency by storing multiple messages in a single file; if 4 messages were only a few dozen bytes long, they could coexist within one of those 512 byte sectors and the GBBS Pro software would know that one sector held 4 separate messages. Furthermore, it even went a step further by implementing a simple text compression algorithm which achieves a roughly 12.5% savings. So with this, running a BBS on a floppy disk was a little more feasible.

To make this proprietary format work, GBBS Pro was written in a custom BASIC-like programming language called ACOS (All-Purpose Communications Operating System) which was developed specifically for GBBS. For the parts of the BBS which worked with messages, special commands were available in the ACOS language which would transparently interact with database files in this format.

## So why this project then?
As mentioned earlier, nostalgia has driven me to want to see what nonsense my friends and I from the late 1980s were going on about on the BBS. Because of the proprietary format, the only way to look at those messages would be to spend time re-learning how to stand up an Apple II with the GBBS Pro software on it and configure everything to reference those message database files. However, I'd prefer to extract those messages on a modern platform since it's easy enough to get the message database files off of old disks. However, you can't just load them into a text editor or Word or something like that – no software understands the format. And so this project is born…

The tool is an exercise in studying available ACOS and GBBS source code, mixed in with a little reverse engineering because the message database file format wasn't really well documented. The goal was to be able to point the tool at a bulletin or email message file, analyze, and extract all messages from those files and save them as modern text files. As far as I've seen, this is the only tool of its kind and saves you from having to write custom ACOS code to do the same thing. However, it does go a step further than what an ACOS program could do, since it also attempts to extract even deleted messages.

## Project features
ACOS / GBBS Pro message database files are like small filesystems. Just like on other filesystems, you have a directory of active files and where they live on the disk. As well, files can be deleted from the directory and their storage space marked as unused, but rarely does an operating system go through and actually wipe out every sector where the file was stored. Because of this, the tool knows how to extract not only "active" messages, but it is also capable of recovering deleted messages and/or residual fragments to varying degrees.

With all this, the tool is capable of extracting:

- **Active messages:** these are messages which are not deleted and presumably complete. These are extracted as 'Msg-0001.txt' files.
- **Deleted messages:** these are messages which were deleted from the directory, but the start was discovered in the data blocks. If the subsequent chains still exist, it will reconstruct as much as it can. These are extracted as 'Deleted-0001.txt' files.
- **Orphan messages:** any data blocks and their subsequent chains, but the start of the message wasn't found. These are messages which chained multiple data blocks, but the message was deleted and the data block where the message started had been reused and overwritten for another message. These are extracted as 'Orphan-0001.txt' files.

## Special handling for email messages
In GBBS Pro, the message database files were used for two purposes: bulletins (a.k.a. boards, news groups, message bases, etc.) and email. While the format is the same, they have slightly different usage patterns (see [technical documentation](msgdb-technical.md) for details). The key difference with the email pattern is that the recipient of a message is not part of the message content, rather it uses the directory entry offset as a way to map to a user ID in the BBS. As such, a feature for email extraction allows for the use of specifying a GBBS Pro `USERS` file to determine the identity of the recipient.


## Some notes about message database files
If you want to go into the technical weeds, read my document on the file format. It will tell you probably everything you want to know, and perhaps good to read before the next paragraph if it doesn't make sense.

On a less-technical front, I will point out that there are size limitations for the databases. Specifically, when the sysop creates a new message board for their BBS, they specify a key thing: what is the maximum number of messages that a message board can hold.

As a way to prevent the disks on a BBS from filling up, GBBS Pro supported the feature of allowing up to a specified number of messages to be posted to a message board before purging the oldest one to make room. In other words, you could create a board for discussing people's favorite bands that would hold up to 10 messages. If someone tried to post an 11th message, the first (oldest) message on that board would get automatically deleted.

This auto-purge mechanism was super convenient, but if you're looking to find every message that ever existed on an old BBS, note that many sysops had this feature enabled to save disk space. As well, this is why you'll find deleted messages and orphaned fragments; probably because that (say) 10 message limit was hit multiple times and rotated the database.

## Getting your message files from your Apple II
I'm not going to go into detail on how to use the various tools that exist, but I suggest looking into [CiderPress II](https://ciderpress2.com/), and [AppleCommander](https://applecommander.github.io/).

In general, messages typically were stored as bulletin files named `B1`, `B2`, and so forth to correspond with what board number it related to.

## How to use the tool

Running the tool from the command-line without arguments will produce a help screen:
```
GBBS Pro Message Database Tool v1.0.1
2026-02-05, Brian J. Bernstein  (brian@dronefone.com)

Usage:
  gbbsmsgtool.py analyze <msgdb_file>
  gbbsmsgtool.py extract <msgdb_file> [--active] [--deleted] [--orphaned] [--all] [--output-dir <path>] [--users <users_file>] [--force]

Commands:
  analyze    Show database statistics and block map
  extract    Extract messages from database

Extract options:
  --active       Extract active messages (default)
  --deleted      Extract deleted messages
  --orphaned     Extract orphaned blocks
  --all          Extract all types
  --output-dir   Write to directory instead of stdout
  --users        Path to USERS file (for email recipient names)
  --force        Overwrite existing files (default: abort if files exist)
```

The tool features two modes of operation: analysis and extraction.

### Analysis
The `analyze` mode will report statistics about the message database file such as what you see below. It does not do any extraction nor does it modify the file.

```
bash> ./gbbsmsgtool.py analyze B1
=== Database Analysis: B1 ===

Format: BULLETIN
File size: 8200 bytes

Header (MSGINFO):
  Bitmap blocks: 2 (256 bytes)
  Directory blocks: 4 (512 bytes)
  Used data blocks: 41
  Message count: 9
  New message number: 1468

File layout:
  0x000-0x007: Header (8 bytes)
  0x008-0x107: Bitmap (2 blocks)
  0x108-0x307: Directory (4 blocks, max 128 entries)
  0x308+: Data blocks

Data area: 7424 bytes
Total blocks: 58

Active messages: 9
Deleted messages: 3
Orphaned blocks: 1

Block breakdown:
  Active header blocks: 9
  Active chain blocks: 32
  Deleted header blocks: 3
  Deleted chain blocks: 12
  Orphaned blocks: 1
  Unused blocks: 1
  Total: 58

Usage: 70.7% active

=== Block Map ===
Legend: [H]=Active header, [C]=Active chain, [D]=Deleted header, [d]=Deleted chain
        [o]=Orphaned, [ ]=Unused

[H][C][C][C][C][H][C][H][C][C][C][C][C][C][C][H][C][C][C][C]  20
[C][C][C][H][C][C][C][H][C][H][C][C][H][C][C][C][D][d][d][o]  40
[ ][D][D][d][d][d][d][d][d][d][d][d][d][H][C][C][C][C]
```

What you see here is:

- **File format:** This is what format the file is (BULLETIN vs EMAIL).
- **File size:** How large the source file is. This doesn't reflect how large the file could get if it was filled to it's block capacity, just how large the actual file is.
- **Bitmap blocks:** How many bitmap / data blocks exist for message storage.
- **Directory blocks:** How many directory blocks exist, i.e. this is how many entries it could hold.
- **Used data blocks:** How many actually have data in them relating to "active" messages.
- **Message count:** How many active messages were found.
- **New message number:** A counter that helped the BBS keep track of if there were new messages in that particular board since a caller's last visit. Has nothing to do with extraction, so this is just for information purposes.
- **File layout:** The offsets in the file where the header, bitmap, directory, and data blocks exist. These values can be different based on how large a database was created by the sysop.
- **Data area / Total blocks:** This is how large the data blocks area is (total blocks x 128 bytes).
- **Active header blocks:** How many active messages are found in the database. An "active" message is a message that was not deleted.
- **Active chain blocks:** How many chained data blocks exist for active messages. One-to-many "chained blocks" exist for messages that don't fit into a single data 128 byte block.
- **Deleted header blocks:** How many deleted messages are found in the database. These are messages that the tool was able to find at least the first data block of, but doesn't indicate if the entire message was found/recoverable.
- **Deleted chain blocks:** Similar to 'Active chain blocks', but for deleted messages. 
- **Orphaned blocks:** After accounting for active and deleted messages and their data block chains, these are data blocks which have content, but the tool cannot determine what they belong to. These are most likely remains of messages spanned multiple data blocks, but were deleted and earlier parts of the chain had been overwritten.
- **Unused blocks:** Data blocks that have no catalog reference, aren't allocated, and have no data in them.
- **Usage:** Percentage of how many data blocks are in use by *active* messages.

### Extraction
The `extract` mode performs an analysis of the data file and then extracts the contents to either `STDOUT` (default), or to a named output directory.

When specifying an output directory, a text file for each message will be created. By default, the tool will prevent you from overwriting existing files unless you use `--force`. Directories will be created if they don't already.

You need to specify at least one extraction mode option: active, deleted, orphaned, or all message types.

Output of extracted files will be to `STDOUT` unless you specify `--output-dir`.

If you have the GBBS Pro user database file (typically named `USERS`), you can specify this to assist the tool in extracting email files. Has no effect with bulletin files since user names are part of the message data. For email, however, you will only be able to identify the intended recipient of a message by including this file.

Example usage:
```
bash> ./gbbsmsgtool.py extract B1 --all --output-dir out
bash> ls -l out
total 104
-rw-r--r--@ 1 brianb  staff     98 Dec 27  1987 Deleted-0001.txt
-rw-r--r--@ 1 brianb  staff  1,478 Jan  5  1988 Deleted-0002.txt
-rw-r--r--@ 1 brianb  staff    404 Jan 26  1988 Deleted-0003.txt
-rw-r--r--@ 1 brianb  staff    434 Jan  8  1988 Msg-0001.txt
-rw-r--r--@ 1 brianb  staff    654 Jan  8  1988 Msg-0002.txt
-rw-r--r--@ 1 brianb  staff    258 Jan  9  1988 Msg-0003.txt
-rw-r--r--@ 1 brianb  staff  1,127 Jan 13  1988 Msg-0004.txt
-rw-r--r--@ 1 brianb  staff  1,066 Jan 13  1988 Msg-0005.txt
-rw-r--r--@ 1 brianb  staff    220 Feb  6 13:49 Orphan-0040.txt
```

Note that the numbers used for active messages (`Msg-0000.txt`) correspond to their message number as they would exist in the BBS (since messages were referenced by number). Deleted messages, however, are simply a sequence in order of their discovery, but with an attempt to be chronological. Orphaned blocks' number refers to the block number it was found in.

The timestamps of the extracted files are determined by the timestamp found within the message.

## Questions, Comments, Support

If you find this tool helpful, let me know! I am happy to help you with recovering your BBS nostalgia. I only briefly ran GBBS Pro while I was developing my own software (Compunet BBS), but I helped a number of people code in ACOS back in the day.

If there is something you need fixed, added, or whatever, raise a ticket on the [github page](https://github.com/bernstbj/gbbs) or email me at brian-removethispart *at* dronefone.com . You can also find information about this on my website at [http://dronefone.com/gbbs](http://dronefone.com/gbbs)

## License
I'm releasing this as GPL 3 (see LICENSE file). If you want to include it or my documentation, I prefer if you reference my site, but at a minimum please retain my name in credits.

