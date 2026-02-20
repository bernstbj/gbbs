# GBBS Pro Message Database File Format

## Overview

This document describes the message database file format used by GBBS Pro on Apple II computers. Instead of regular text files, GBBS Pro stores bulletin board messages and private email in a compressed, block-chained format optimized for the limited storage and memory constraints of 8-bit systems. The reason for the reverse engineering of the format is to make it easy to extract messages from these files on modern machines for nostalgia and archival purposes.


**Key Features:**

- **7-bit compression**: Achieves ~12.5% space savings by packing 8 characters into 7 bytes
- **Block-chained storage**: Messages can span multiple 128-byte blocks via chain pointers
- **Two distinct formats**: Bulletin boards (public messages) and email (private messages per user)
- **Bitmap allocation**: Tracks which data blocks are in use
- **Random access**: Directory provides direct access to messages without sequential scanning

**File Types:**

- **Bulletin files** (B1, B2, B3, etc.): Public message boards, one message per directory entry
- **MAIL file**: Private email, directory entries map to user IDs, multiple messages per user

**Typical Usage:**

- Bulletin boards for public discussions, announcements, and forums
- MAIL file for private user-to-user communication
- Messages can be deleted (directory entry removed) but data remains until overwritten
- "Crunch" operation compacts directory by removing deleted entries

This format was designed for efficiency on Apple II systems with limited disk space (typically 140KB floppy disks or early hard drives). The 7-bit compression and block-chained structure allowed BBSs to store hundreds of messages while maintaining reasonable access speeds.

## File Format Variants

GBBS Pro message databases come in two formats, distinguished by byte 0 of the header:

### Bulletin Board Format (Byte 0 = 0x01 or 0x02)
Used for public bulletin boards (B1, B2, B3, etc.)

**Characteristics**:

- One message per directory entry
- Messages terminated by null byte (0x00)
- Long messages span multiple blocks via chain pointers
- Block chains link continuation blocks
- Messages shouldn't share blocks

**Structure**:

- Directory entry points to message start block
- Follow chain pointers (bytes 126-127) for multi-block messages
- Stop at null terminator or end of chain

### Email Format (Byte 0 = 0x04)
Used for private email/mail databases (MAIL file)

**Characteristics**:

- Directory entries map to User IDs (entry N = User ID N)
- Each user has a chain of blocks containing all their messages
- Messages within a user's chain are separated by EOT character (0x04)
- Messages can span blocks via chain pointers (bytes 126-127)
- 7-bit compression works the same as bulletin format

**Structure**:

- Each directory entry corresponds to a User ID
- Follow chain pointers to get all blocks for that user
- Decode the complete chain
- Split on EOT (0x04) to extract individual messages
- Each message is TO that user (implicit recipient)

**Message Format** (different from bulletins):
```
<from_user_id>
Subj : <subject>
From : <username> (#<user_id>)
Date : <timestamp>

<message body>
```
Note: No "To" line in the message data - the recipient is implicit (the user whose chain this is)

**Detection**: Check byte 0 of file header. If 0x04, use email format; otherwise use bulletin format.

## File Structure

### Header (0x00 - 0x07, 8 bytes) - MSGINFO Array

It was difficult to determine from dissecting a database file, so had to look at the ACOS source code file `DISK.S` for some of this. But what I found is below, including some references to the 6502 source's function labels:

- **Byte 0 (MSGINFO[0])**: Number of bitmap blocks
  - Each block is 128 bytes
  - Bitmap tracks which data blocks are allocated (1 bit per block)
  - Example: 2 blocks = 256 bytes = can track 2048 data blocks
  
- **Byte 1 (MSGINFO[1])**: Number of directory blocks
  - Each block is 128 bytes
  - Each directory entry is 4 bytes
  - Example: 4 blocks = 512 bytes = 128 directory entries max
  
- **Bytes 2-3 (MSGINFO[2-3])**: Number of used/allocated data blocks (16-bit little-endian)
  - Incremented when blocks are allocated (ALLOC function)
  - Decremented when blocks are deallocated (DEALLOC function)
  - Tracks current data block usage
  
- **Bytes 4-5 (MSGINFO[4-5])**: Total message count (16-bit little-endian)
  - Highest message number in the database
  - Updated when writing messages (WRTMSG function)
  - Reset during crunch operation (DO_CNCH function)
  
- **Bytes 6-7 (MSGINFO[6-7])**: Highest "new" message number (16-bit little-endian)
  - Tracks the last message with new/unread content
  - Used for message notification/tracking

### Bitmap Section

- **Offset**: 0x08 (immediately after header)
- **Size**: MSGINFO[0] × 128 bytes
- **Purpose**: Block allocation bitmap - 1 bit per data block
  - Bit set (1) = block is allocated
  - Bit clear (0) = block is free
- **Format**: Binary bitmap data

### Directory Section

- **Offset**: 0x08 + (MSGINFO[0] × 128)
- **Size**: MSGINFO[1] × 128 bytes
- **Format**: NOT compressed - plain binary data
- **Maximum entries**: (MSGINFO[1] × 128) / 4

Each directory entry is 4 bytes:

- **Bytes 0-1**: Byte offset in file where message starts (little-endian, 16-bit)
  - Absolute byte offset from beginning of file
  - Allows direct seeking to message without decoding
  - Used for random access
- **Bytes 2-3**: Starting block number (little-endian, 16-bit)
  - Block numbers are relative to data block area (block 1 = first data block)
  - Used for block-chained reading
  - Zero value (0x0000) indicates empty/unused entry
  - **For bulletin format**: Points to message start block
  - **For email format**: Points to first block of user's message chain (entry N = User ID N)

Empty entries are marked with `00 00 00 00`.

**Important**: 

- **Bulletin format**: Directory entries may point to continuation blocks or message fragments, not always complete message starts. The presence of a non-zero block number indicates the entry is "active" from the BBS perspective.
- **Email format**: Directory entry N corresponds to User ID N. Non-zero entry means that user has messages.

### Data Block Section

- **Offset**: 0x08 + (MSGINFO[0] × 128) + (MSGINFO[1] × 128)
- **Format**: 7-bit compressed data, organized in 128-byte blocks
- **Block structure**:
  - Bytes 0-125: Compressed message data (126 bytes)
  - Bytes 126-127: Next block pointer (little-endian, 16-bit)
    - 0x0000 = end of message/chain
    - Non-zero = block number of continuation block

### Example File Layout (B1)

```
MSGINFO: [02 04 29 00 09 00 BC 05]
  Byte 0: 0x02 = 2 bitmap blocks
  Byte 1: 0x04 = 4 directory blocks
  Bytes 2-3: 0x0029 = 41 used blocks
  Bytes 4-5: 0x0009 = 9 messages
  Bytes 6-7: 0x05BC = 1468 (new message number)

File structure:
  0x000-0x007: Header (8 bytes)
  0x008-0x107: Bitmap (2 × 128 = 256 bytes)
  0x108-0x507: Directory (4 × 128 = 512 bytes, max 128 entries)
  0x508-EOF:   Data blocks (128 bytes each, 7-bit compressed)

Block number translation:
  Directory says "block 1" -> file offset 0x508 + ((1-1) × 128) = 0x508
  Directory says "block 5" -> file offset 0x508 + ((5-1) × 128) = 0x708
```

### Message Data Area
**7-bit compressed** data organized in 128-byte blocks.

#### Structure

- **First message**: May or may not start at the first data block
  - Messages are accessed via directory entries
  - No guaranteed "main" message at a fixed location
  
- **All messages**: Located via directory entries
  - Directory entry points to starting block
  - Follow chain pointers to read complete message
  - Multiple messages can exist within blocks (though rare in bulletin format)

#### Block Structure (128 bytes each)

- **Bytes 0-125**: Compressed message data (126 bytes)
- **Bytes 126-127**: Next block pointer (little-endian, 16-bit)
  - Points to next block number in the chain
  - `0x0000` = end of message (no continuation)
  - **Always at bytes 126-127** - fixed position, not variable

### Message Deletion and Recovery

**Deletion Process** (from DISK.S DO_KILL):

1. Directory entry (4 bytes) is zeroed out
2. Each block in the chain is deallocated in the bitmap
3. **Data blocks are NOT modified** - chain pointers remain intact
4. Message content remains in blocks until overwritten

**Crunch Process** (from DISK.S DO_CNCH):

1. Compacts directory by removing zero entries
2. Moves valid entries forward to fill gaps
3. **Only touches directory** - never modifies data blocks
4. Writes compacted directory back to disk

**Why Deleted Messages Are Recoverable:**

- Directory entry removed, but data blocks untouched
- Chain pointers still valid and can be followed
- Message content remains until blocks are reused

**Message Reading** (from DISK.S RDMSG):

- Reads 126 bytes of data from current block
- Checks bytes 126-127 for next block pointer
- If pointer is 0x0000, end of message
- If non-zero, reads that block and continues
- **NO loop detection** - self-referencing pointers cause infinite loop/hang (a bug I found during extraction of some sample msgdb files)

**Self-Referencing Chain Pointers** (block N -> block N):

- NOT created intentionally by GBBS software
- NOT handled by GBBS - causes infinite loop/hang if encountered
- Result of **corruption or buffer reuse without proper initialization**
- BLKBUF2 write buffer is reused without zeroing
- Old chain pointers can remain if new message is shorter

**Self-Reference as Sequential Continuation Pattern:**

- Analysis of B2 database shows 35 out of 39 self-referencing pointers have valid continuations in the next sequential block (block N+1)
- Pattern: Block N -> N (self-reference) actually means "continue to block N+1"
- Likely caused by systematic bug in GBBS where current block number is written instead of next block number
- Recovery algorithm: When encountering self-reference, check if block N+1:
  - Exists and is readable
  - Has no Date header (not a new message start)
  - Has content (>10 non-null characters)
  - If yes, treat block N+1 as continuation
  - If no, mark as "[Self-referencing chain pointer detected]"
- This pattern successfully recovers ~90% of self-referencing cases
- Acts as a marker for corrupted/incomplete message chains

**Chain Pointer Corruption Causes:**

1. **Buffer reuse**: BLKBUF2 not cleared between messages, old chain data remains
2. **Incomplete writes**: Disk errors during block write operations
3. **Block reuse**: Previously used blocks allocated without initialization
4. **Software bugs**: Edge cases in message writing not properly handled

**Orphaned Blocks:**

- Continuation blocks from deleted messages
- Start block may be reused for new message, but continuations remain
- No directory entry points to them
- May contain readable fragments of old messages

#### Byte Offset vs Block Number
The directory stores both for flexibility:

- **Byte offset** (bytes 0-1): Absolute position in file for direct seeking
- **Block number** (bytes 2-3): Block-based addressing for chained reading

Example from B1:

- Directory entry 0: byte offset 1133, block 54
  - Block 54 is at file offset: 0x508 + ((54-1) × 128) = 0x1A88
  - Byte offset 1133 = 0x046D (may point mid-block for random access)
  - These may not point to the same location - byte offset is for seeking, block number is for chain following

## 7-Bit Compression Algorithm

### Overview
The compression works by storing only 7 bits per character in each byte, using the 8th bit (bit 7, the high bit) to construct an additional character. Every 7 bytes of compressed data encodes 8 characters, achieving ~12.5% compression.

### Encoding Example: "PRESUMED"

The first 7 characters are encoded with their high bit used to construct the 8th character 'D':

```
Char | ASCII | Encoded | Binary      | Bit 7
-----|-------|---------|-------------|------
 P   | 0x50  | 0xD0    | 11010000    |   1
 R   | 0x52  | 0x52    | 01010010    |   0
 E   | 0x45  | 0x45    | 01000101    |   0
 S   | 0x53  | 0x53    | 01010011    |   0
 U   | 0x55  | 0xD5    | 11010101    |   1
 M   | 0x4D  | 0x4D    | 01001101    |   0
 E   | 0x45  | 0x45    | 01000101    |   0
```

High bits collected in order: `1000100` = 0x44 = 'D'

Result: 7 bytes encode "PRESUMED" (8 characters)

(yes, this example was copied from the GBBS newsletter - see references at bottom of this document)

### Encoding Process
1. Take 8 characters to encode
2. For the first 7 characters:
   - Store bits 0-6 in the byte
   - Use bit 7 to store one bit of the 8th character
3. The 8th character is reconstructed from the 7 high bits

### Decoding Process (from ACOS assembly code)

The ACOS code (DISK.S, RDMSG function) decodes as follows:

For each byte in the compressed data:

1. `ASL` - Shift byte left, bit 7 goes to carry flag
2. `ROR CHAR8` - Rotate CHAR8 right, carry goes into bit 7 of CHAR8
3. `LSR` - Shift accumulator right (gives original byte with bit 7 cleared)
4. Output the 7-bit character (bits 0-6 of original byte)
5. `DEC BYTE8` - Decrement counter (initialized to 6, counts down to -1)
6. When counter reaches -1:
   - `LSR CHAR8` - Shift CHAR8 right once more before output
   - Output the accumulated character
   - Reset CHAR8 to 0
   - Reset counter to 6

**Note**: The assembly code uses ROR (rotate right) which accumulates bits in reverse order, then shifts right before output to correct the order.

### Python Implementation (in gbbsmsgtool.py)
```python
def decode_7bit(compressed_data, stop_at_null=True):
    """Decode 7-bit compressed data to ASCII text."""
    result = []
    i = 0
    while i + 6 < len(compressed_data):
        bytes_7 = compressed_data[i:i+7]
        char8 = 0
        chars = []
        for b in bytes_7:
            char8 = (char8 >> 1) | ((b & 0x80) >> 0)
            chars.append(b & 0x7F)
        char8 = char8 >> 1
        chars.append(char8)
        
        for c in chars:
            if stop_at_null and c == 0:
                return bytes(result).decode('ascii', errors='replace').replace('\r', '\n')
            result.append(c)
        i += 7
    return bytes(result).decode('ascii', errors='replace').replace('\r', '\n')
```

**Key points:**

- Processes 7 bytes at a time to produce 8 characters
- `stop_at_null` parameter controls whether to stop at null terminator (used for bulletin format)
- Converts carriage returns (\r) to newlines (\n) for Unix compatibility
- Handles decoding errors gracefully with 'replace' mode

### Special Characters
- `0x00`: End of message
- `0x0D` (13): Carriage return (convert to `\n` for Unix)
- Characters 32-126: Printable ASCII

## Statistics (from a bulletin file example I used to develop the tool with)
- File size: 8,072 bytes
- Header: 8 bytes
- Bitmap blocks: (from MSGINFO[0])
- Directory blocks: (from MSGINFO[1])
- Data area: Remaining bytes (blocks of 128 bytes)
- High bit percentage in data area: ~47.5% (confirms 7-bit compression)
- High bit percentage in directory/bitmap: ~7% (confirms NOT compressed)

## Block Chaining Example
Message starting at block 7:

- Block 7: Contains message data, bytes 126-127 = `09 00` (next block = 9)
- Block 9: Contains continuation, bytes 126-127 = `00 00` (end of message)

Note: Block numbers in chains are relative to the data block area, not absolute file offsets.

## Message Formats

### Bulletin Board Message Format

Each decoded bulletin message follows this structure:

```
<Subject line>
<To line: user_id,username>
<From line: user_id,username (#user_id)>
Date : MM/DD/YY  HH:MM:SS [AM/PM]

<Message body text>
```

**Example:**
```
Re: Hahahaha
0,IronKnight (#5)
6,Shortround (#6)
Date : 01/04/88  08:36:52 PM

do you have to keep on doing that?
```

**Field Descriptions:**

- **Subject**: First line of message (any text)
- **To**: Line 2 - User ID, username (0 = "All" for public messages)
- **From**: Line 3 - User ID, username with ID in parentheses
- **Date**: Line 4 - Timestamp in 12-hour format with AM/PM
- **Body**: Remaining lines - Message content (may span multiple blocks)

**Message Start Pattern** (used by gbbsmsgtool.py to identify message starts):

- Line 1: Any text (subject)
- Line 2: Matches pattern `^\d+,` (number, comma, text)
- Line 3: Matches pattern `^\d+,` (number, comma, text)
- Line 4: Contains "Date" and either ":" or "->"

### Email Message Format

Each decoded email message follows this structure:

```
<from_user_id>
Subj : <subject>
From : <username> (#<user_id>)
Date : MM/DD/YY  HH:MM:SS [AM/PM]

<message body>
```

**Example:**
```
3
Subj : Test Message
From : The Wook (#3)
Date : 01/19/88 07:23:48 PM

Hey, just testing the mail system...
```

**Field Descriptions:**

- **From User ID**: Line 1 - Numeric user ID of sender
- **Subject**: Line 2 - Subject line with "Subj :" prefix
- **From**: Line 3 - Sender's username with ID in parentheses
- **Date**: Line 4 - Timestamp in 12-hour format with AM/PM
- **Body**: Remaining lines - Message content

**Note**: Email messages do NOT have a "To" line in the stored data. The recipient is implicit - it's the user whose chain contains the message (directory entry N = User ID N). The gbbsmsgtool.py adds a "To:" line when extracting if the USERS file is provided.

## Known Issues and Limitations

1. **Character decoding**: Some characters may decode incorrectly due to:
   - Corruption in original files (disk errors, incomplete writes)
   - Special control characters not commonly used
   - Non-ASCII characters (GBBS Pro was designed for 7-bit ASCII)

2. **Bulletin format directory entries**: Not all directory entries point to message starts. Some may point to:
   - Continuation blocks (mid-message)
   - Corrupted or incomplete data
   - Blocks reused after deletion
   
   The gbbsmsgtool.py handles this by using the message start pattern to identify valid messages.

3. **Email format limitations**: 
   - Recipient information not stored in message data (implicit from directory position)
   - Requires USERS file to display recipient names
   - Multiple messages per user chain separated by EOT (0x04)

4. **Self-referencing chain pointers**: Some databases contain blocks where the chain pointer points to itself (block N -> block N). This appears to be a bug in ACOS where the current block number was written instead of the next block number. The gbbsmsgtool.py handles this by checking if the next sequential block is a valid continuation.

## Unknown / To Be Determined

### Header Fields
- Exact purpose of MSGINFO[6-7] "new message number" - how is it used by the BBS?
- Whether MSGINFO[0] and MSGINFO[1] can be different sizes in practice
- Bitmap format details - bit ordering within bytes

### Message Linking
- How messages are linked or threaded (if at all)
- Whether the "message number" in the message header relates to directory position
- How reply chains are maintained

## Message Extraction Algorithm

### Working Decoder (Verified 2026-02-04)

**Status**: Core algorithm working and tested on multiple database files. Handles both bulletin and email formats with auto-detection.

The complete message extraction process:

1. **Auto-detect Format**
   - Check byte 0 of header
   - If 0x04: Email format (EOT-separated)
   - Otherwise: Bulletin format (null-terminated)

2. **Read Directory Entries** (0x88-0x107)
   - Read max entries from header byte 2 (not hardcoded)
   - Each 4-byte entry contains block number and byte offset
   - Skip entries where block_num == 0 (empty entries)
   - Validate block_num is within file bounds before processing
   - Filter entries by checking for Date header to identify message starts

3. **Follow Block Chains**
   - Start at the block number from directory entry
   - Read 126 bytes of compressed data from block
   - Read 2-byte chain pointer at bytes 126-127
   - Continue to next block if pointer != 0
   - Stop if pointer == 0 or block already visited (prevent loops)
   - Stop if block offset exceeds file size
   - **Handle self-referencing pointers**: If next_block == current_block, try sequential continuation (block+1)

4. **Decode 7-bit Compression**
   - Process 7 bytes at a time to produce 8 characters
   - Extract bit 7 from each byte and accumulate into 8th character
   - For continuation blocks, decode without stopping at null
   - Strip leading nulls from continuation blocks
   - Convert carriage returns (\r) to newlines (\n)

5. **Handle Continuation Blocks with Date Headers**
   - If continuation block contains Date header, extract only text BEFORE the header
   - The Date header indicates start of new message (block reuse)
   - Stop chain following after extracting pre-header text

6. **Output Format**
   - Display entry number, starting block, and byte offset
   - Concatenate all decoded blocks in chain order
   - Stop at first null in concatenated message
   - Add diagnostic markers for incomplete chains:
     - `[Self-referencing chain pointer detected]` - block points to itself, sequential continuation failed
     - `[Next segment missing]` - chain points to allocated or non-existent block
     - `[Chain loop detected]` - chain forms a loop (not self-reference)

### Key Implementation Details

- **Null Termination**: Messages end with 0x00 byte in bulletin format. Decoder stops at first null in final concatenated message.
- **Block Chains**: Chain pointers link message blocks together. Some messages span multiple blocks.
- **Self-Reference Recovery**: When chain pointer equals current block, try next sequential block as continuation.
- **Directory Entry Filtering**: Not all entries are message starts - filter by Date header presence.
- **Loop Detection**: Track visited blocks to prevent infinite loops (improves on GBBS which has no loop detection).
- **Block Bounds Checking**: Essential to prevent reading beyond file size when following chains or accessing blocks.
- **Sequential Continuation**: Recovers ~90% of self-referencing cases by checking if block N+1 is valid continuation.
- **Duplicate Prevention**: Blocks used in deleted message chains (including sequential continuations) are marked to prevent output as orphans.

## Tools

### gbbsmsgtool.py - Unified Message Database Tool

Consolidated tool for analyzing and extracting messages from GBBS Pro message database files.

**Key Features:**

- Auto-detects bulletin vs email format (byte 0 of header)
- Handles both bulletin board and email (MAIL) databases
- Optional USERS file support for email recipient names
- Handles self-referencing chain pointers via sequential continuation
- Prevents duplicate output of blocks used in deleted message chains
- Reads actual directory size from header (not hardcoded)
- Provides detailed diagnostic markers for incomplete chains

#### Commands

**analyze** - Display database statistics and block allocation map
```bash
python3 gbbsmsgtool.py analyze <filename>
```

Shows:
- File size and block statistics
- Header (MSGINFO) breakdown
- File layout (bitmap, directory, data offsets)
- Block usage breakdown (allocated, deleted, orphaned, never used)
- Visual block map with status indicators (bulletin format only)

Block map legend (bulletin format):
- `[H]` = Active header (directory entry points here, message start)
- `[C]` = Active chain (continuation of active message)
- `[D]` = Deleted header (message start pattern, not in directory)
- `[d]` = Deleted chain (continuation of deleted message)
- `[o]` = Orphaned block (has data but no header or chain)
- `[ ]` = Unused (never used or zeroed out)

**Block breakdown** (bulletin format):

- Active header blocks: Directory entries that start messages
- Active chain blocks: Continuation blocks for active messages
- Deleted header blocks: Message starts not in directory
- Deleted chain blocks: Continuation blocks for deleted messages
- Orphaned blocks: Data fragments without message headers
- Unused blocks: Never written or zeroed out

**extract** - Extract messages from database
```bash
python3 gbbsmsgtool.py extract <filename> <type> [options]
```

**Required - specify extraction type:**

- `--active` - Extract active messages
- `--deleted` - Extract deleted messages
- `--orphaned` - Extract orphaned blocks
- `--all` - Extract all three types

**Optional flags:**

- `--output-dir <path>` - Write to directory instead of stdout
- `--users <users_file>` - Path to USERS file (for email recipient names and alias detection)
- `--data2 <data2_file>` - Path to DATA2 file (for board names)
- `--pretty` - Format messages with readable headers (default: raw)
- `--force` - Overwrite existing files (default: abort if files exist)

**File Protection:**
By default, the tool will abort with an error if output files already exist. Use `--force` to overwrite existing files.

Examples:
```bash
# Extract active messages to stdout
python3 gbbsmsgtool.py extract B5 --active

# Extract all types to directory
python3 gbbsmsgtool.py extract B5 --all --output-dir B5_messages

# Extract only deleted messages
python3 gbbsmsgtool.py extract B5 --deleted --output-dir B5_deleted

# Extract email with user names
python3 gbbsmsgtool.py extract MAIL --active --users USERS --output-dir MAIL_messages

# Extract with board names and pretty formatting
python3 gbbsmsgtool.py extract B1 --all --data2 DATA2 --users USERS --pretty --output-dir B1_messages

# Force overwrite existing files
python3 gbbsmsgtool.py extract B5 --active --output-dir B5_messages --force
```

#### DATA2 File Support

Optional DATA2 file can be provided to display board names for bulletin board files.

**DATA2 File Format** (standard GBBS Pro):

- 128-byte fixed-length records
- Records 0-8: Access level descriptions
- Records 9+: Message base definitions

Message base record structure:
- Board name (null-terminated, ends with \r)
- Filename in format F:B#\r (e.g., F:B1, F:B2)
- Additional fields (access levels, limits, etc.)

The tool extracts the mapping of filenames (B1, B2, etc.) to board names (System News, Public Base, etc.) and displays them in message output when using `--pretty` format.

#### USERS File Support

Optional USERS file can be provided to display recipient names for email messages and detect alias usage in bulletin messages.

**USERS File Format** (standard GBBS Pro, may vary if modified by sysops):

- Random-access file with 128-byte records
- Record N corresponds to User ID N
- Record 0 is typically unused (no user ID 0)

Record structure:

- First_name,Last_Name\r (uppercase)
- Full_name\r (proper case, preferred for display)
- City,State\r
- ... (additional fields)
- Offset 70: password (8 bytes)
- Offset 78: phone_number (12 bytes)

The tool reads the Full_name field for:
- Email recipient identification
- Bulletin message alias detection (when poster name doesn't match USERS file)

**Email Message Output with USERS file:**
```
To: Drone (#1)
3
Subj : Test Message
From : The Wook (#3)
Date : 01/19/88 07:23:48 PM

Message body...
```

**Email Message Output without USERS file:**
```
To: User ID 1 (#1)
3
Subj : Test Message
From : The Wook (#3)
Date : 01/19/88 07:23:48 PM

Message body...
```

#### Output Format

**Raw format** (default): Preserves original GBBS Pro message structure as stored in the database.

**Pretty format** (with `--pretty` flag): Reformats message headers for readability:
- Adds board name header (when DATA2 file provided)
- Converts comma-separated headers to labeled format
- Detects and displays alias usage (when USERS file provided)

**Stdout mode**: Messages are written to stdout with separators between types when using `--all`.

**Directory mode**: Messages are written as individual files:

- Active: `Msg-0001.txt`, `Msg-0002.txt`, etc. (numbered by directory entry order for bulletins, by date for email)
- Deleted: `Deleted-0001.txt`, `Deleted-0002.txt`, etc. (numbered by timestamp order, bulletin format only)
- Orphaned: `Orphan-0033.txt`, `Orphan-0034.txt`, etc. (numbered by starting block number, bulletin format only)

File timestamps are set to the 'Date:' timestamp from the message header when available.

**Pretty format examples:**

Bulletin message with board name:
```
Board: System News (B1)
Subject: ARRRGGHHH!!
To: All
From: Drone (#1)
Date : 01/08/88  05:23:05 PM

[message body]
```

Bulletin message with alias detection:
```
Board: System News (B1)
Subject: Thats it. I'm pissed
To: All
From: DRONE: THE OWNER AND SYSOP (#2-Shortround)
Date : 01/13/88  08:34:03 AM

[message body]
```

The format `(#2-Shortround)` indicates user #2 (Shortround) posted using the alias "DRONE: THE OWNER AND SYSOP".

**Customization Note:** The pretty format parser is based on standard GBBS Pro message headers. If your BBS uses customized headers, modify the `prettify_message()` function in the tool (see code comments for customization points).

#### Message Type Categories

**Active Messages**: 

- **Bulletin format**: Messages currently referenced in the directory. Extracted by following directory entries and their block chains.
- **Email format**: All messages in user chains. Directory entry N = User ID N. Messages within each user's chain are separated by EOT (0x04).

**Deleted Messages**: (Bulletin format only) Messages that have been removed from the directory but still have their header block (containing message start pattern) intact. These are complete messages that can be fully reconstructed by following their block chains through unused space.

**Orphaned Blocks**: (Bulletin format only) Data blocks that contain readable content but lack a message header. These are fragments from:

- Partially overwritten messages where the header was reused
- Messages where only continuation blocks remain
- Incomplete deletions or corruption

Orphaned blocks are extracted by following their chain pointers as far as possible through unused space.

**Never Used**: Blocks that are all nulls or contain minimal data (< 10 non-null bytes). These blocks have never been written to or were explicitly zeroed.

#### Extraction Algorithm

**Bulletin Format - Active Messages**:

1. Read directory entries
2. For each valid entry, follow block chain pointers (bytes 126-127)
3. Decode 7-bit compressed data from each block
4. Stop at null terminator or end of chain
5. Number by directory entry (preserves chronological order)

**Bulletin Format - Deleted Messages**:

1. Scan unused blocks for message start pattern (subject/to/from/date)
2. Follow block chains through unused space only
3. Stop when chain enters allocated space or hits null terminator
4. Sort by timestamp and number sequentially

**Bulletin Format - Orphaned Blocks**:

1. Find unused blocks with data but no message start pattern
2. Skip blocks already included in deleted message chains
3. Follow block chains through unused space
4. Number by starting block number
5. Stop when chain enters allocated space or hits null terminator

**Email Format - Active Messages**:

1. Read directory entries (entry N = User ID N)
2. For each non-zero entry, follow block chain pointers
3. Decode 7-bit compressed data from complete chain
4. Split on EOT (0x04) to extract individual messages
5. Each message is TO that user (implicit recipient)
6. Sort all messages by date
7. If USERS file provided, prepend "To: Name (#ID)" to each message

## References

Most of the work was doing reverse engineering based on assumptions that I had of the file format. I had learned that there was the 7-bit compression back in the late 1980s from the author himself, and doing hex dumps of the files back then I had a good idea of what was going on. However, not all of the bytes were accounted for, so I had to look at a newsletter from back-in-the-day to confirm the compression format, and also I looked at the source code of ACOS and GBBS Pro to see how everything else was being done. I did also look at some ACOS tutorials on textfiles.com, but they don't really explain what the msg() function is or how to completely use it.

* [Newsletter confirming compression technique](https://gbbs.applearchives.com/wp-content/uploads/2025/10/GBBS_Pro_News_v1n2_May-87.pdf)
* [ACOS and GBBS Pro source code](https://github.com/callapple/GBBS/tree/master/Source)
* [ACOS tutorial docs (4 of them)](http://www.textfiles.com/apple/DOCUMENTATION/)

Enjoy!

Brian J. Bernstein, February 2026

