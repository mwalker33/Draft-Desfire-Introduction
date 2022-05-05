# Notes on Setting up NDEF 
<a id="Top"></a>

# Table of Contents

- [Notes on Setting up NDEF](#Notes-on-Setting-up-NDEF)
- [Table of Contents](#table-of-contents)
  - [NDEF on Desfire EV1](#NDEF-on-Desfire-EV1)


## NDEF on Desfire EV1
^[Top](#top)

The follow is a guide to assist in setting up a single NDEF record on an Desfire EV1 (or later).
Please refere to the NDEF documention and standards for assistance on the actual NDEF record setup and structure.

It is assumed you are fimular with using a desfire cards and commands.

The follow notes are based on:
    NXP - AN11004 - MIFARE DESFire as Type 4 Tag
    Rev. 2.4 - 22 May 2013

In order to setup NDEF on a Mifare Desfire card you need to create an Application and two files inside that application.
The application and files have some special needs in order for the standands to work and the NDEF recrod to be found.

### Step 1 - Create Application

While what i beleive the App ID and File IDs dont matter (for EV1 and later)
I did find a reference to using the values in this example.

The Application MUST have the DFName of D2760000850101
Note: That is the hex/binary data that needs to be stored!

    DF Name         : D2760000850101            <- **Important MUST be D2760000850101**  
    AID             : 000001                    <- In EV1 and later can be any AID  
    FID             : E110                      <- In EV1 and later can be any App - FID  
    Keys            : 1                         <- Any number of keys (based on your needs)  
    Encryption      : AES                       <- Any encryption.  
    ISO 2 Byte FID  : Yes                       <- **Important MUST support 2 Byte ISO File IDs**  




### Step 2 - Create the Capability Container file (CC File)

The CC File is a standard file to store the needed NDEF information to find your NDEF records.  This example will contrain the setup for a single NDEF record.
Note: You can define more then one NDEF data file if needed (not covered in this example)

    Type            : Standard data file
    FID             : 01                        <- File ID can be any uniqure File ID for this AID
    ISO FID         : E103                      <- **Important MUST be the 2 byte ISO File of E103**
    Size            : 0F (15 bytes)           <- May need to be longer in more advanced setups.
    Comms           : Plain                     <- **Important the file MUST support plain communication mode**
    Permissions     : E000                      <- **Read Free** write change etc key 0)  
                                                    Note: To allow public update set Write to E as well.
                                                          All keys should be set as per normal desfire rules.

CC File (E103) example

    000F20003B00340406E10400FF00FF

Usefull Items in the CC File

    000F20003B00340406 E104 00FF 00 FF
                         |    |      |
                         |    |       --- 00 = Write allowed, FF = Read Only 
                         |     ---------- The maxium size of the NDEF record (set to the NDEF record size)
                          --------------- The ISO 2 byte File ID for the NDEF Record file
                          

    Note: the NDEF Record File Size should be <= the actual data file size and >= the amount of data you have. Its not how long the actual NDEF record is.


### Step 3 - Create the NDEF Record File


    Type            : Standard data file
    FID             : 02                        <- File ID can be any uniqure File ID for this AID
    ISO FID         : E104                      <- **Important MUST be the 2 byte ISO File set in the CC File**
    Size            : 00FF (255 bytes)          <- Can be as big as needed, but should not be smaller then the value in the CC File
    Comms           : Plain                     <- **Important the file MUST support plain communication mode**
    Permissions     : E000                      <- **Read Free** write change etc key 0)  
                                                    Note: To allow public update set Write to E as well.
                                                          All keys should be set as per normal desfire rules.


NDEF data file example

    000CD1010855016E78702E636F6DFE

Usefull Items in this NDEF example record

    000C D10108 55 01 6E78702E636F6D FE
      |          |  |        |
      |          |  |         ----------- nxp.com  
      |          |   -------------------- Well known record sub-type : 01 HTTP://, 02 HTTPS://
      |           ----------------------- ASCII U - URI
       ---------------------------------- Lenght of the NDEF record (not inluding the trailing FE

