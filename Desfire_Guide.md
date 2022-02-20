# Desfire Introduction Guide
<a id="Top"></a>

### Based on RRG/Iceman Proxmark3 repo

### Ver.1 20 Feb 2022 - First Draft

# Table of Contents

| Contents                                                                            |
| ----------------------------------------------------------------------------------- |
| [Part 1](#part-1)                                                                   |
| [Introduction](#introduction)                                                       |
| [Desfire overview](#desfire-overview)                                               |
| [Whats on my card](#whats-on-my-card)                                               |
| [Create an application](#create-an-application)                                     |
| [Change an application key](#change-an-application-key)                             |
| [Create a file](#create-a-file)                                                     |
| [Read a file](#read-a-file)                                                         |
| [Write to a file](#write-to-a-file)                                                 |

# Part 1
^[Top](#top)

## Introduction
^[Top](#top)

Desfire is a generak purpose HF (High Frequency) RFID card that is used
in the 13.56 Mhz frequency space. It can be a complex card to use due to
the many options it supports, so we will use the proxmark3 rrg fork to 
get a start on using its commands so configure a desfire card.  

Desfire has several versions and new features are added as it evolves.
This guide will be based on the EV1 unless otherwise highlighted.

It is highly recommended that when learning about RFID that learning how
to read the data sheets be near the top of the list. It can be very hard
as the data sheet will hold the information you need, but you don’t yet
know what it means.  A lot of the detailed datasheets for Desfire is not 
available to the general public, as a result this guide is based on what 
was public and expermenting with the proxmark3 desfire commands.

This guide is not a how do I clone document. It is meant to help people
learn how to use Desfire and in the process learn about rfid and the
proxmark3.

Throughout this guide I will give examples. It is recommended that you
try these as we go. To do so, have a blank Desfire (ev1 or later) card 
that you can use for this purpose.

## Desfire overview
^[Top](#top)

Most LF cards had a fairly simple read only setup; When powered on, it 
will send its programed ID modulated and encoded as needed so the readers 
as per design.  There were some general purpose cards like the EM4x05 and 
T55xx that has user accessable blocks to read and write data as needed.
As we move into HF it is more common for cards to be read/write cards.  A 
common card is the Mifare Classic.  These cards have impoved security 
(over LF) and tend to still have a block structure. (i.e. Data is 
read/writting block by block).  Over time things are improved and new 
generation cards are developed and released, Desfire is an example.

Desfire is not broken in blocks and sectors, rather it is more like a 
directoy and file stucture (where a Application can be thought of as the 
directly, and the files inside that Application can be thought of as files 
in the directory.

Building and sending commands to a desfire card mostly involves 3 main things.
- How do we authenticate (encryption, key etc)
- How will we exchange information (plain, encrypted, mac'ed etc)
- What do we want to do (create, read, write etc)

### Note: not all modes support all options.  An incorrect or missing option can lead to the -20 error

While Application Identifiers (AID) need to be uniqure on a card, file 
identifiers (FID) only need to be unique inside an application.

### Note: normal user files need to have the native FID must be between 0x00-0x1F

Desfire supports several versions of encryption. 
- Native Des (8 byte keys)
- Native 2kTDes (16 byte keys)
- ISO 2KTDes (24 byte keys)
- ISO 2KTDes (16 byte keys)
- ISO AES (16 byte Keys)

As such commands should be sent using the same method.

### Note: Native Des is really Native 2KTDes with the 8 byte key repeated to form 16 bytes

Due to the many options and combination of ways a desfire application 
can be setup, this document is not intended to cover them all; rather 
provide a guide to get you started and help you understand why the proxmark3 
desfire commands may seem complex (when compared to other card formats).

## Whats on my card
^[Top](#top)

A good place to start to see what is on you card is using the 'hf mfdes info' command.
```
[usb] pm3 --> hf mfdes info
```
For a blacnk card, you should see something simular to the following.
Please note actual data like UID, Batch number etc will vary as expected.
```
[=] ---------------------------------- Tag Information ----------------------------------
[+]               UID: 04 49 4A 9A 15 62 80
[+]      Batch number: B9 0C 19 45 50
[+]   Production date: week 15 / 2019

[=] --- Hardware Information
[=]    raw: 04010101001605
[=]      Vendor Id: NXP Semiconductors Germany
[=]           Type: 0x01
[=]        Subtype: 0x01
[=]        Version: 1.0 ( DESFire EV1 )
[=]   Storage size: 0x16 ( 2048 bytes )
[=]       Protocol: 0x05 ( ISO 14443-2, 14443-3 )

[=] --- Software Information
[=]    raw: 04010101041605
[=]      Vendor Id: NXP Semiconductors Germany
[=]           Type: 0x01
[=]        Subtype: 0x01
[=]        Version: 1.4
[=]   Storage size: 0x16 ( 2048 bytes )
[=]       Protocol: 0x05 ( ISO 14443-3, 14443-4 )

[=] --------------------------------- Card capabilities ---------------------------------
[=]     1.4 - DESFire Ev1 MF3ICD21/41/81, EAL4+
[+] ------------------------------------ PICC level -------------------------------------
[+] Applications count: 0 free memory 2304 bytes
[+] PICC level auth commands: auth: YES auth iso: YES auth aes: NO auth ev2: NO auth iso native: YES auth lrp: NO
[+] PICC level rights:
[+] [1...] CMK Configuration changeable   : YES
[+] [.1..] CMK required for create/delete : NO
[+] [..1.] Directory list access with CMK : NO
[+] [...1] CMK is changeable              : YES
[+]
[+] Key: 2TDEA
[+] key count: 1
[+] PICC key 0 version: 0 (0x00)

[=] --- Free memory
[+]    Available free memory on card         : 2304 bytes

[=] Standalone DESFire
```
What we can see is the Card Master Key Application (CMK) overview.
This card in its default state can support severy auth methods.
It also shows "Application Count: 0", so no user applications yet and 
2304 bytes free memory, so looks like a 2K Desfire EV1 card.
We will cover looking at an application setup later in this guide.

## Create an application
^[Top](#top)

Their is a few peices of information we need to know and define before
we can create an application.
- AID : The application identifier we wish to use (unique per application) 3 bytes
- FID : The applications File Identifier - 2 Bytes
- KS1 : Key Set 1
- KS2 : Key Set 2
- CMK : The card master key type and no. (default des and 0), and key (default 0000000000000000)

For this example we will use an AID of 123456 (my first App, but does not need to start at 1)
For the FID for the application, lets set it to 3456.
We have the defaul CMK, so DES (only 1 CMK key, so key No. 0) with a key of 0000000000000000

Summary
    AID:    123456
    FID:    3456
    CMK: No.0, type des, key 0000000000000000

That leaves us to define KS1 and KS2.
KS1 and 2 define the how the application will be setup.  This can get complex, 
but if you take it step by step it should be come clearer.

Ket Set 1 contains how the application is depentent on the AMK (Application Master Key)
It is 8 bits long. Where bit 0 is the least signifent bit.

       0:   Allow change master key
       1:   Free Directory list access without master key
            0: AMK auth needed for GetFileSettings and GetKeySettings
            1: No AMK auth needed for GetFileIDs, GetISOFileIDs, GetFileSettings, GetKeySettings
       2:   Free create/delete without master key
            0:  CreateFile/DeleteFile only with AMK auth
            1:  CreateFile/DeleteFile always
       3:   Configuration changable
            0: Configuration frozen
            1: Configuration changable if authenticated with AMK (default)
       4-7: ChangeKey Access Rights
            0: Application master key needed (default)
            0x1..0xD: Auth with specific key needed to change any key
            0xE: Auth with the key to be changed (same KeyNo) is necessary to change a key
            0xF: All Keys within this application are frozen

Mapped out view
        7 6 5 4                                     3                       2                                 1
        0 0 0 0 - AMK needed to change Rights       0 - Config Frozen       0 Create/Del File only with AMK   0 - AMK needed for GetFile/Key Settings
        0 0 0 1                                     1 - Config Not Frozen   1 Create/Del Any Key              1 - Any key GetFile/Key Settings
         ....   - Auth with Key X to change Key
        1 1 0 1  
        1 1 1 1 - All Keys Are frozed

Bits 7..4 define what key and change what key.  In this example we will force all keys to be changed with the AMK, so (in binary) 0000----
Bit 3 defines if we can change the config.  Lets allow future changes to the config, so                                           ----1---
Bit 2 defines if we need the AMK to create/delete files (or if anyone can create files, so lets lock it down.                     -----0--
Bit 1 defines if we need to use the AMK to find information about the application, lets make it easy to look this up by others.   ------1-
Bit 0 defines if we are allowed to change the application master key,  lets be flexable and allow the AMK to be changed.          -------1

### Bring it all together and we have KS1 value of : 0B Hex

Moving onto KS2. 

       0..3: Number of keys stored within the application (max. 14 keys)
       4:    RFU
       5:    Use of 2 byte ISO FID, 0: No, 1: Yes
       6..7: Crypto Method 00: DES/3DES, 01: 3K3DES, 10: AES
       Example:
            2E = FID, DES, 14 keys
            6E = FID, 3K3DES, 14 keys
            AE = FID, AES, 14 keys

       7 6             5                 4   3 2 1 0
       1 0  AES        0 No 2 Byte FID   0   x x x x Number of Keys (xE Max) E = 14 keys no. 0-13
       0 1  3K3Des     1 2 Byte FID
       0 0  Des/3Des

Bits 7-6 define the encryption we wish to use. Lets go with AES                                    10------
Bit 5 defines if we wish to use 2 byte File Identifications (FIDS), let use 2 byte FIDS            --1-----
Bit 4 RFU                                                                                          ---0----
Bit 3-0 defines how many keys we wish to use.  Lets keep it sime and have 1 + AMK, 2 keys.         ----0010

### Bring it all together and we have a KS2 of : A2 (hex)

Summary
    AID:    123456
    FID:    3456
    KS1:    0B
    KS2:    A2
    CMK: No.0, type des, key 0000000000000000
    
Lest build and run the command to create this application on the proxmark3.
```
hf mfdes createapp --aid 123456 --fid 3456 --ks1 0B --ks2 A2 -t des -n 0 -k 0000000000000000 
```
If all goes well we should see a created reply
```
[+] Desfire application 123456 successfully created
```

To see if we can find it, we could use the 'lsapp' command.
```
hf mfdes lsapp
```
And that found 2 applications, the PICC Level Application and the one we just created.
```
[+] ------------------------------------ PICC level -------------------------------------
[+] Applications count: 1 free memory 2144 bytes
[+] PICC level auth commands: auth: YES auth iso: YES auth aes: NO auth ev2: NO auth iso native: YES auth lrp: NO
[+] PICC level rights:
[+] [1...] CMK Configuration changeable   : YES
[+] [.1..] CMK required for create/delete : NO
[+] [..1.] Directory list access with CMK : NO
[+] [...1] CMK is changeable              : YES
[+]
[+] Key: 2TDEA
[+] key count: 1
[+] PICC key 0 version: 0 (0x00)

[+] --------------------------------- Applications list ---------------------------------
[+] Application number: 0x123456 iso id: 0x3456 name:
[=]   DF AID Function 123456     : (unknown)
[+] Auth commands: auth: NO auth iso: NO auth aes: YES auth ev2: NO auth iso native: YES auth lrp: NO
[+]
[+] Application level rights:
[+] -- AMK authentication is necessary to change any key (default)
[+] [1...] AMK Configuration changeable   : YES
[+] [.0..] AMK required for create/delete : YES
[+] [..1.] Directory list access with AMK : NO
[+] [...1] AMK is changeable              : YES
[+]
[+] Key: AES
[+] key count: 2
[+]
[+] Key versions [0..1]:  00, 00
```
We can see it supports AES Auth, has 2 keys the Correct AID and FID we set and everything looks good so far.

## Change an application key
^[Top](#top)

In order to change an application key we need to know a few things.
- The Application ID (AID)
- The key number we want to change
- the key type
- the old key
- if key of the key used to change password (as per KS1 from the create application)

Lets change Key 1 from 00000000000000000000000000000000 to 11223344556677889900112233445566
We will use Key 0 with key 00000000000000000000000000000000 to perform this action (inline with the Application Setup.
Since this is an AES Applcaition, we will set the crypt type to AES
```
hf mfdes changekey --aid 123456 -t aes -n 0 --key 00000000000000000000000000000000 --newkeyno 1 -oldkey 00000000000000000000000000000000 --newkey 11223344556677889900112233445566
```
All going well, we should get a positive response
```
[=] Changing key for AID 123456
[=] auth key 0: aes [16] 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[=] changing key number  0x01 (1)
[=] old key: aes [16] 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[=] new key: aes [16] 11 22 33 44 55 66 77 88 99 00 11 22 33 44 55 66
[=] new key version: 0x00
[+] Change key ok
```

Now let change key 0 to aabbccddeeff0099feedbeef12345678
```
hf mfdes changekey --aid 123456 -t aes -n 0 --key 00000000000000000000000000000000 --newkeyno 0 --oldkey 00000000000000000000000000000000 --newkey aabbccddeeff0099feedbeef12345678
```
result
```
[=] Changing key for AID 123456
[=] auth key 0: aes [16] 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[=] changing key number  0x00 (0)
[=] old key: aes [16] 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[=] new key: aes [16] AA BB CC DD EE FF 00 99 FE ED BE EF 12 34 56 78
[=] new key version: 0x00
[+] Change key ok
```

### Note: there may be some special commands that can shorten the change key, see the online help for options.

Now that we have changed both keys, lets run some tests

Test 1 - Using the wrong key 0 with lsfiles
```
[usb] pm3 --> hf mfdes lsfiles --aid 123456 -t aes -n 0 -k 00000000000000000000000000000000
[!!] Desfire authenticate error. Result: [7] Sending auth command failed
[-] Select or authentication AID 123456 failed. Result [7] Sending auth command failed
```

Test 2 - Using the wrong key 1 with lsfiles
```
[usb] pm3 --> hf mfdes lsfiles --aid 123456 -t aes -n 1 -k 00000000000000000000000000000000
[!!] Desfire authenticate error. Result: [7] Sending auth command failed
[-] Select or authentication AID 123456 failed. Result [7] Sending auth command failed
```

Test 3 - Using the correct key 0 with lsfiles
```
[usb] pm3 --> hf mfdes lsfiles --aid 123456 -t aes -n 0 -k aabbccddeeff0099feedbeef12345678
[=] There is no files in the AID 123456
```

Test 4 - Using the correct key 1 with lsfiles
```
[usb] pm3 --> hf mfdes lsfiles --aid 123456 -t aes -n 1 -k 11223344556677889900112233445566
[=] There is no files in the AID 123456
```

As can be seen from the above command, when we used the wrong keys, it failed as expected.
We we used the correct key the command succeded (who reported no files in the AID).


## Create a file
^[Top](#top)

Defire supports several file types (e.g. Standard (data) file, Value file, Record file and so on.
Each file type has a purpose.  For this example we are going to create a standard data file.

In order to create a file we need to know a few things.  These can change with the type of file 
you are creating and how the application was setup (yes, it can be complex, but we should be able 
to work it all out).

- The AID under which we wish to create the file
- The key number we will use to authenticate this request
- The encryption type we need to use
- The communication mode used
- The FID (File Identifier) 1 Byte 00-1F
- The isofid (ISO File Identifier) 2 Byte (if its an ISO Application)
- The size of the file (in hex 3 bytes)
- The rights (acl) we wish to assign to the file

See the help file for the ways to provide these options and valid values.

For this example, I am going to use the long version for the ACL
Information needed
AID                 : 123456
Key to use          : 0
Encyprion           : AES
Key                 : aabbccddeeff0099feedbeef12345678
Communication mode  : mac
FID                 : 1
ISODFID             : 0001
SIZE 16 bytes       : 000010
Read Rights         : free   (anyone can read the file)
Write Rights        : key0  (Must use Key 0 to write to the file)
Read/Write Rights   : key0 
Change Rights       : key0

```
hf mfdes createfile --aid 123456 -n 0 -t aes -k aabbccddeeff0099feedbeef12345678 -m mac --fid 01 --isofid 0001 --size 000010 --rrights free --wrights key0 --rwrights key0 --chrights key0
```
Result
```
[=] ---- Create file settings ----
[+] File type        : Standard data
[+] File number      : 0x01 (1)
[+] File ISO number  : 0x0001
[+] File comm mode   : Plain
[+] Additional access: No
[+] Access rights    : e000
[+] read     : free
[+] write    : key 0x00
[+] readwrite: key 0x00
[+] change   : key 0x00
[=] File size        : 16 (0x10) bytes
[+] Standard data file 01 in the app 123456 created successfully
```

### Note: If you get an error, adding -av to a command can give more information that may help work out why it failed.
### Normally if things look correct, a failure will be linked to a paramater not matching the applicaiton comms mode or out of range

Now that we have a file create, lets rerun the lsfiles to see what it reports

```
[usb] pm3 --> hf mfdes lsfiles --aid 123456 -t aes -n 0 -k aabbccddeeff0099feedbeef12345678
[=] ------------------------------------------ File list -----------------------------------------------------
[+]  ID |ISO ID|     File type       | Mode  | Rights: raw, r w rw ch   | File settings
[+] ------------------------------------------------------------------------------------------------------------
[+]  01 | 0001 |0x00 Standard data   | Plain |e000, free key0 key0 key0 | size: 16 [0x10]
```

## Read a file
^[Top](#top)

Now that most of the hard bit is done, we are down to reading and writing to/from a file.
In the example used here we are using the standard file.
As with all desfire commands we need to select the correct options for the operation.
With different options like --offset and --length we can read just part of a file.

By new you should feel a theme of collecting the information needed and the values we are using in these examples.
Lets go with the simple command for reading a standard file using key 0
- AID           : 123456
- Encyprion     : aes
- Key Number    : 0
- Key           : aabbccddeeff0099feedbeef12345678
```
hf mfdes read --aid 123456 -t aes -n 0 -k aabbccddeeff0099feedbeef12345678 --fid 01
```
Result
```
[=] ------------------------------- File 01 data -------------------------------
[+] Read 16 bytes from file 0x01 offset 0
[=]  Offset  | Data                                            | Ascii
[=] ----------------------------------------------------------------------------
[=]   0/0x00 | 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 | ................
```

## Read a file
^[Top](#top)

By this stage we now have an appliction created, changed its keys, created a standard file, now
we can write data into that file.

Information needed
- AID           : 123456
- Encyprion     : aes
- Key Number    : 0
- Key           : aabbccddeeff0099feedbeef12345678
- type of file  : data
- the data      : 00112233445566778899AABBCCDDEEFF
- write mode    : plain

```
hf mfdes write --aid 123456 --fid 01 -t aes -n 0 -k aabbccddeeff0099feedbeef12345678 -m plain --type data -d 00112233445566778899AABBCCDDEEFF
```
result
```
[=] Write data file 01 success
```

Now lets read that back
```
[usb] pm3 --> hf mfdes read --aid 123456 -t aes -n 0 -k aabbccddeeff0099feedbeef12345678 --fid 01
[=] ------------------------------- File 01 data -------------------------------
[+] Read 16 bytes from file 0x01 offset 0
[=]  Offset  | Data                                            | Ascii
[=] ----------------------------------------------------------------------------
[=]   0/0x00 | 00 11 22 33 44 55 66 77 88 99 AA BB CC DD EE FF | .."3DUfw........
```

Now that we have some identifable data, lets try to read 7 bytes from the middle
```
[usb] pm3 --> hf mfdes read --aid 123456 -t aes -n 0 -k aabbccddeeff0099feedbeef12345678 --fid 01 -o 000007 -l 000002
[=] ------------------------------- File 01 data -------------------------------
[+] Read 2 bytes from file 0x01 offset 7
[=]  Offset  | Data                                            | Ascii
[=] ----------------------------------------------------------------------------
[=]   7/0x07 | 77 88                                           | w.
```

Next, lets change those two bytes to AA BB
```
[usb] pm3 --> hf mfdes write --aid 123456 -t aes -n 0 -k aabbccddeeff0099feedbeef12345678 --fid 01 -m plain --type data -o 000007 -d AABB
[=] Write data file 01 success
```
and verify with a read
```
[usb] pm3 --> hf mfdes read --aid 123456 -t aes -n 0 -k aabbccddeeff0099feedbeef12345678 -o 000007 -l 000002 --fid 01
[=] ------------------------------- File 01 data -------------------------------
[+] Read 2 bytes from file 0x01 offset 7
[=]  Offset  | Data                                            | Ascii
[=] ----------------------------------------------------------------------------
[=]   7/0x07 | AA BB                                           | ..
```
of read the entire file and see the changes in context.
```
[usb] pm3 --> hf mfdes read --aid 123456 -t aes -n 0 -k aabbccddeeff0099feedbeef12345678  --fid 01
[=] ------------------------------- File 01 data -------------------------------
[+] Read 16 bytes from file 0x01 offset 0
[=]  Offset  | Data                                            | Ascii
[=] ----------------------------------------------------------------------------
[=]   0/0x00 | 00 11 22 33 44 55 66 AA BB 99 AA BB CC DD EE FF | .."3DUf.........
```


