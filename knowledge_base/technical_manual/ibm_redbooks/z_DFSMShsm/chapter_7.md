# 7. Tape considerations

This chapter describes tape usage and the necessary setup for DFSMShsm to process tape
for backup and migration. The focus in this chapter is on automatic tape management by
using IBM TotalStorage Virtual Tape Server (Virtual Tape Server) and automatic tape
libraries. Manually operated tapes are covered at an overview level because of limited usage.
The focus is on IBM hardware and related functions. However, it does not include information
about hardware installation. The information that is provided is for customization from a
DFSMShsm perspective. 

## 7.1 Basic tape setup that is needed for DFSMShsm

To define a tape environment to DFSMShsm, you have to code **SETSYS** commands and also
support these allocations in the automatic class selection (ACS) routines. Storage
management subsystem (SMS)-managed environments are the common practice today.

These settings vary depending on your environment in relationship to the following variables:

- Media types that are used
- Tape environment (virtual, automated tape library (ATL), or manually operated)
- Use of a tape management system (IBM or not IBM)
- Security

The ACS routine code ensures that your allocation points you to the correct drive, device
type, and media. For the non-SMS managed data, you use the esoteric to obtain the correct
device.

This chapter describes the tape environment that is needed to support DFSMShsm in more
detail. It also describes how to establish the DFSMShsm functions that relate to tape.

### 7.1.1 DFSMShsm single file tape operation

DFSMShsm tape processing functions support single file tapes. You cannot create either
multiple file reel-type backup or migration tapes. Support of reel-type tapes is limited to the
following functions:

- Recall or recovery of data sets that currently reside on reel-type tapes
- Creation of dump tapes
- Creation of control data set (CDS) Version Backup tapes
- Aggregate backup and recovery support (ABARS) functions

Single file format is a potentially multireel tape with only one file on it. Using the single file
format reduces I/O and system serialization because only one label is required for each
connected set (as opposed to multiple file format tapes that require a label for each data set).
The standard-label tape data set that is associated with the connected set can span up to the
allocation limit of 255 tapes. This standard-label tape data set is called the _DFSMShsm tape_
_data set_ . Each user data set is written, in 16 K logical blocks, to the DFSMShsm tape data
set. A single user data set can span up to 254 tapes, previously a single user data set only
spanned to 40 tapes.

**Tape hardware emulation**
Emulation of one tape subsystem by another tape subsystem is commonplace. Virtual tape
and new technologies emulate earlier technologies. Because the cartridges are not
interchangeable, DFSMShsm added support for several subcategories that use 3490
cartridges:

- 3590 Model E 256 track recording
- 3590 Model B 128 track recording
- 3590 Model H 384 track recording
- Virtual Tape Server
- All others whose device type is 3490

Additionally, DFSMShsm supports the following subcategories of 3590 cartridges:

- 3590 Model E 256 track recording
- 3590 Model B 128 track recording




- 3590 Model H 384 track recording
- 3592 Model J1A 512 track recording (EFMT1)
- 3592 Model E05 drive format 896 track recording (EFMT2)
- 3592 Model E06 drive format 1152 track recording (EFMT3)
- 3592 Model E07 drive format 1664/2560 track recording (EFMT4)
- 3592 Model E08 drive format 5120 track recording (EFMT5)
- All others whose device type is 3590

**Restriction:** You can use the 3592-E05 and newer tape drives in 3590 emulation mode
only, never 3490. The 3592 Model J1A can operate in 3490 emulation mode only when it
uses MEDIA5 for output.

This additional support enables the multiple use of these technologies concurrently for the
same function within the same HSMplex on condition that their output use is on different
DFSMShsm hosts. This support improves the installation of new technologies to improve
performance and capacity. When multiple technologies are used concurrently for output
outside the Data Facility Storage Management Subsystem (DFSMS) tape umbrella, an
esoteric unit name is needed to enable DFSMShsm to distinguish them. An esoteric name is
also needed to select existing (partially filled) tapes that are compatible with the allocated
drives.

**Drive compatibility**
Tape cartridges and tape formats that are compatible with the 3592 tape drives are described.
The 3592 tape cartridge has external dimensions (form factor) that allow it to be used within
existing storage cells of libraries that contain 3590 tapes. However, the 3592 tape drives must
be installed in frames separately from any 3590 drives. The 3592 tape drive cartridges are not
compatible with 3590 tape drives cartridges, and, likewise, 3590 tapes cannot be used in the
3592 drives. You need to consider the following compatibility topics:

- Supported formats

  The following formats with different recording densities yield different capacities per
cartridge type:

  - Enterprise Format 1 (EFMT1) is used by the 3592 J1A Tape Drive and the 3592 E05
Tape Drive in both native and J1A emulation mode. This format records 512 tracks on
eight channels. The 3592 E06 tape drive and 3592 E07 tape drive with code level
D3I3_5CD or higher read data that is written in EFMT1 format but do not write in
EFMT1.

  - Enterprise Format 2 (EFMT2) is used by the 3592 E05 Tape Drive and the 3592 E06
Tape Drive. This format records 896 tracks on 16 channels. When the drive operates
on encrypted data at this density, the recording format is Enterprise Encrypted
Format 2 (EEFMT2).

  - Enterprise Format 3 (EFMT3) is used by the 3592 E06 Tape Drive and the 3592 E07
Tape Drive. This format records 1,152 tracks on 16 channels. When the drive operates
on encrypted data at this density, the recording format is Enterprise Encrypted
Format 3 (EEFMT3).

  - Enterprise Format 4 (EFMT4) is used by the 3592 E07 Tape Drive. This format records
1,664 tracks on JB and JX cartridges and 2,560 tracks on JC, JK, and JY cartridges on
32 channels. When the drive operates on encrypted data at this density, the recording
format is Enterprise Encrypted Format 4 (EEFMT4).

  - Enterprise Format 5 (EFMT5) is used by the 3592 E08 Tape Drive. This format records
5,120 tracks on JC, JD, JK, JL, JY, and JZ cartridges on 32 channels. When the drive
operates on encrypted data at this density, the recording format is Enterprise
Encrypted Format 5 (EEFMT5).




- The following cartridge types are supported:

  - The 3592 E08 Tape Drive uses the Advanced Type D read/write (type JD), the
Advanced Type D WORM (type JZ), and the Advanced Type D economy (type JL)
cartridges. Both the type JD and JZ cartridges have a maximum capacity of 10 TB
(9,313.23 GiB). The Advanced Type D economy cartridges have a maximum capacity
of 2 TB (1,862.65 GiB).

  - The 3592 E07 and E08 tape drive uses the Advanced Data (type JC) and Advanced
WORM (type JY) cartridges. The type JC and JY cartridges have a maximum capacity
of 4,000 GB (3,725.3 GiB) for E07 and 7 TB (6,519.26 GiB) for E08.

  - The 3592 tape drive models E07, E06, and E05 use 3592 Extended Data (type JB) and
Extended WORM (type JX) with a maximum capacity of 1,600 GB (1,490.12 GiB) with
EFMT4 format, 1,000 GB (931.3 GiB) with EFMT3 format, and 700 GB (651.93 GiB)
with EFMT2 format. The 3592 E07 Tape Drive with code level D3I3_5CD or higher can
read format EFMT2 but does not write EFMT2 format.

  - The 3592 tape drive models E06, E05, and J1A use 3592 Standard Data (type JA) and
Standard WORM (type JW) cartridges with a maximum capacity of 640 GB (596.04
GiB) with EFMT3 format, 500 GB (465.66 GiB) with EFMT2 format, and 300 GB
(279.39 GiB) with EFMT1 format. The 3592 E07 Tape Drive with code level D3I3_5CD
or higher can read only cartridge types JA and JW.

  - The 3592 tape drive models E06, E05, and J1A use 3592 Economy Data (type JJ) and
Economy WORM (type JR) cartridges with a maximum capacity of 128 GB (119.21
GiB) with EFMT3 format, 100 GB (93.13 GiB) with EFMT2 format, and 60 GB (55.88
GiB) with EFMT1 format. The 3592 E07 Tape Drive with code level D3I3_5CD or higher
can read only cartridge types JJ and JR.

  - The 3592 E07 and E08 tape drives use the 3592 Advanced Economy (type JK)
cartridge. The maximum capacity is 500 GB (465.66 GiB) with EFMT4 format, and 900
GB (838.19 GiB) with EFMT5 format.

Table 7-1 shows the cartridge products, types, storage management subsystem (SMS)
categories, and characteristics.

_Table 7-1  Types of IBM 3592 tape cartridges_


|Product, type, and SMS category|Native capacity|Col3|Col4|Col5|Col6|Part number|
|---|---|---|---|---|---|---|
||E08|E07|E06/EU6|E05|JA1||
|Data, JA, MEDIA5, (610 m)|Not supported|640 GB (596.04 GiB) E06 format|640 GB (596.04 GiB) E06 format|500 GB (465.66 GiB) E05 format|300 GB (279.39 GiB) (J1A format)|18P7534|
|||500 GB (465.66 GiB) E05 format|500 GB (465.66 GiB) E05 format||||
|||300 GB (279.39 GiB) J1A format1|300 GB (279.39 GiB) J1A format|300 GB (279.39 GiB) J1A format|||
|Extended Data, JB, MEDIA9, (825 m)|Not supported|1,600 GB (1490.12 GiB) E07 format|1,000 GB (931.32 GiB)|700 GB (651.93 GiB)|Not supported|23R9830|
|||1,000 GB (931.32 GiB) E06 format|||||


-----

|Product, type, and SMS category|Native capacity|Col3|Col4|Col5|Col6|Part number|
|---|---|---|---|---|---|---|
||E08|E07|E06/EU6|E05|JA1||
|Advanced Data, JC, MEDIA11, (825 m)|7 TB (6.37 TiB) E08 format|4,000 GB (3,725.29 GiB) E07 format|Not supported|Not supported|Not supported|46X7452|
|Advanced Type D read/write, JD, MEDIA13 (1,032 m)|10 TB (9.1 TiB) E08 format|N/A|N/A|N/A|N/A|2727263|
|Economy, JJ, MEDIA7, (246 m)|Not supported|128 GB (119.21 GiB) E06 format1|128 GB (119.21 GiB) E06 format|100 GB (93.13 GiB) E05 format|60 GB (58.88 GiB) J1A format|24R0317|
|||100 GB (93.13 GiB) E05 format|100 GB (93.13 GiB) E05 format||||
|||60 GB (58.88 GiB) J1A format1|60 GB (58.88 GiB) J1A format|60 GB (58.88 GiB) J1A format|||
|Advanced Economy, JK, MEDIA13, (146 m)|900 GB (838.19 GiB) E08 format|500 GB (465.66 GiB) E07 format|Not supported|Not supported|Not supported|46X7453|
|Advanced Type D Economy, JL, MEDIAnn (241 m)|2 TB (1.82 TiB) E08|Not supported|Not supported|Not supported|Not supported|2727264|
|Economy WORM, JR, MEDIA8, (246 m)|Not supported|128 GB (119.21 GiB) E06 format1|128 GB (119.21 GiB) E06 format|100 GB (93.13 GiB) E05 format|60 GB (58.88 GiB) J1A format|24R0317|
|||100 GB (93.13 GiB) E05 format|100 GB (93.13 GiB) E05 format||||
|||60 GB (58.88 GiB) J1A format|60 GB (58.88 GiB) J1A format|60 GB (58.88 GiB) J1A format|||
|WORM, JW, MEDIA6, (610 m)|Not supported|640 GB (596.04 GiB) E06 format|640 GB (596.04 GiB) E06 format|500 GB (465.66 GiB) E05 format|300 GB (279.39 GiB) J1A format|18P7538|
|||500 GB (465.66 GiB) E05 format|500 GB (465.66 GiB) E05 format||||
|||300 GB (279.39 GiB) J1A format1|300 GB (279.39 GiB) J1A format|300 GB (279.39 GiB) J1A format|||


-----

|Product, type, and SMS category|Native capacity|Col3|Col4|Col5|Col6|Part number|
|---|---|---|---|---|---|---|
||E08|E07|E06/EU6|E05|JA1||
|Extended WORM, JX, MEDIA10, (825 m)|Not supported|1,600 GB (1,490.12 GiB) E07 format|1,000 GB (931.32 GiB) E06 format|700 GB (651.93 GiB)|Not supported|23R9831|
|||1,000 GB (931.32 GiB) E06 format|||||
|Advanced WORM, JY, MEDIA12, (880 m)|7 TB (6.37 TiB) E08 format|4,000 GB (3,725.29 GiB) E07 format|Not supported|Not supported|Not supported|46X7454|
|Advanced WORM, JZ, MEDIA12 (1,032 m)|7 TB (6.37 TiB) E08 format|Not supported|Not supported|Not supported|Not supported|2727265|
|Cleaning, CLNxxxJA3|N/A|N/A|N/A|N/A|N/A|18P7535|
|Notes: 1 The 3592-E07 supports reading JA, JJ, JR, and JW cartridges only with code level D3I3_5CD or later. 2 The 3592-E08 supports reading JD, JL, and JZ cartridges only with code level D3I4_460 or later.|||||||


## 7.2 Tape device allocations

Tape allocations that are invoked by DFSMShsm can be handled by dynamic allocation in the
first instance. But the WAIT and NOWAIT options of the following DFSMShsm device
allocation parameters tell DFSMShsm whether DHSM tape allocation is suspended or not.
The following parameters determine suspension:

- INPUTTAPEALLOCATION
- OUTPUTTAPEALLOCATION
- RECYCLETAPEALLOCATION

The behavior depends on the environment, such as the following factors:

- Whether the environment is a JES2 or a JES3 environment
- The number of available tape devices
- The general availability of the tape devices

### 7.2.1 Tape device selection

If one type of tape device exists at your site, no consideration for tape device selection exists.
For any tape-input or tape-output operation, only one type of tape device can be selected.

However, you have more than one type of tape device at your site, a choice must be made
about which device to select for a certain function.


-----

You can direct MVS to select a device you specify, or you can allow MVS to choose the device
for you. You can restrict MVS allocation to selecting the specific type of tape device you want
for a certain function in either an SMS-managed tape library or in a non-library environment.
Restricting device selection can improve tape processing when high-performance devices are
selected and can ensure that new devices are selected when they are introduced into your
tape environment.

### 7.2.2 Non-library tape device selection

When DFSMShsm requests that to allocate and mount a tape device, MVS allocates a tape
device according to two aspects of the request.

If the allocation request is for a specific tape or for a scratch tape, MVS first honors the device
type that DFSMShsm selects. If DFSMShsm specifies a generic unit name for a
cartridge-type device, MVS allocates a cartridge-type device, but selects a cartridge-type
device with or without a cartridge loader according to its own criteria. (If all of the devices
contain cartridge loaders, the MVS criteria are unimportant.) If DFSMShsm specifies an
esoteric unit name that is associated with devices with a cartridge loader, MVS allocates a
device from that esoteric group.

If the DFSMShsm tape-allocation request is for a cartridge-type device, MVS selects a device
by considering whether the request is for a specific tape or for a scratch tape. If the request is
for a specific tape, MVS tries to allocate a device without a cartridge loader. If the request is
for a scratch tape, MVS tries to allocate a device with a cartridge loader. If the preferred type
of device is unavailable, MVS selects any available cartridge-type device.

### 7.2.3 Library tape device selection

Tape devices that are associated with SMS-managed tape libraries are selected based on the
technologies they support. You can restrict selection of these devices by specifying the
data-class attributes that are associated with a device.

### 7.2.4 Specifying the WAIT option

When DFSMShsm initiates a tape allocation request, an exclusive enqueue is issued against
the SYSZTIOT resource. This enqueue remains until the tape allocation request is honored or
the allocation might fail. A setting of the WAIT option causes all DFSMShsm activity to go on
hold.

**Note:** Use the WAIT option only in JES3 environments or environments with a dedicated
DFSMShsm esoteric.

### 7.2.5 Specifying the NOWAIT option

When the NOWAIT option is used, dynamic allocation is not waiting for a device to become
available. Either a device is returned to DFSMShsm or a failure occurs because a device was
not available. DFSMShsm activity continues, and only the task that requests a tape device
waits and reissues the tape device request later.

The setting for a NOWAIT request is shown in Example 7-1


-----

_Example 7-1  How to specify NOWAIT options in DFSMShsm_

    SETSYS INPUTTAPEALLOCATION(NOWAIT)
    SETSYS OUTPUTTAPEALLOCATION(NOWAIT)
    SETSYS RECYCLETAPEALLOCATION(NOWAIT)

If the tape device request fails, DFSMShsm performs the following actions:

- Requests the tape device allocation six more times.

- After seven attempts to allocate a device, DFSMShsm sends a message to the operator
console. The operator response is to either repeat or cancel the request.

- A message IEF238D might be issued if DFSMShsm detects spare devices as potential
candidates for the tape mount. The operator must reply “cancel” to this request, if these
devices are unusable. Message ARC0381A is issued when the request is ended.

**Note:** The **SETSYS MOUNTWAITTIME(** **_nn_** **)** parameter in DFSMShsm specifies how long
DFSMShsm waits. The numeric value in brackets specifies the number of minutes.

### 7.2.6 Specifying esoteric tape unit names to DFSMShsm

When DFSMShsm uses the **SETSYS USERUNITTABLE** command to specify esoteric tape unit
names to DFSMShsm, DFSMShsm rejects mixed combinations of device names in an
esoteric group.

Several exceptions exist to this rule:

- DFSMShsm allows the use of both 3480 and 3480X device names in a single group.
Improved Data Recording Capability (IDRC), however, is not used with such a group
because all devices in the group are treated as 3480s. If an esoteric group that is
associated only with 3480Xs exists, 3480s must not be added to it because the data that
was already written by using this esoteric tape unit name might create IDRC
incompatibilities.

- Tape units 3592-1, 3592-2, and 3592-2E are allowed into the same esoteric because they
share common write formats. Tape units 3592-2, 3592-2E, and 3592-3E are allowed into
the same esoteric because they also share common write formats.

It is up to the user to ensure that the drives in a mixed esoteric will be used with a common
recording technology. For example, if any 3592-1 drives are included, all drives in the esoteric
must use EMFT1 for output. If the esoteric mixes 3592-2 and 3592-2E drives, all drives must
be set up to use EFMT1 or EFMT2. If the esoteric contains mixed 3592-3E and 3592-2E tape
units, all drives must be set up to use EFMT2 or EEFMT2, which are the common write
formats.

If DFSMShsm rejects an esoteric tape unit name, it does not reject the rest of the valid
esoteric names that are specified in the **USERUNITTABLE** command. Those names are now
recognized by DFSMShsm as valid esoteric tape unit names. Each time that you specify
**USERUNITTABLE** , the valid esoteric tape unit names that are identified with this parameter
replace any esoteric tape unit names that are identified through a previous **USERUNITTABLE**
parameter of the **SETSYS** command. The DFSMShsm tape environment allows the
specification of tape unit names by using either generic or esoteric names.


-----

Installations that have a mixture of non-SMS-managed 3590 devices that are defined under
the 3590-1 generic name need to perform these steps:

1. Define a unique esoteric for each recording technology.

2. Use the **SETSYS USERUNITTABLE** command to define these esoteric names to DFSMShsm.
This step also applies to mixed devices in the 3490 generic. Installations that use
SMS-managed tape devices or with a single 3590-1 recording technology do not need to
define an esoteric for those devices. However, if you have a mixed SMS-managed 3590
environment, the media type is required to be the same within a generic. The following
example shows how to code a **USERUNITTABLE** parameter:

   `USERUNITTABLE(3480 TAPEM VIRTUAL 3590-1 3590-2)`

## 7.3 Protecting DFSMShsm-owned tapes

DFSMShsm tapes can be protected with the TAPEVOL resource class in RACF. However, the
use of tape management systems replaces RACF for protecting tapes. Existing systems
might still use RACF protection, so we describe this setup. RACF protection is not
recommended for a new setup.

### 7.3.1 Protecting DFSMShsm tapes

To protect tapes that contain security-related data sets that are managed by DFSMShsm, you
can protect tapes that are managed by DFSMShsm by installing and activating RACF .

Use one of the following options:

- Define to RACF the tapes you want to protect:

  - Defining the RACF TAPEVOL resource class in the RACF descriptor table (CDT).
  - Specifying the **SETSYS TAPESECURITY(RACF|RACFINCLUDE)** command.

**Tip:** If you did not define RACF tape volume sets for DFSMShsm, but you want RACF
to protect all of the tapes through a RACF generic profile, specify the **SETSYS**
**TAPESECURITY(RACF|RACFINCLUDE)** command.

- Use RACF DATASET class protection by using one of the following options:

  - Using RACF SETROPTS TAPEDSN
  - Using DEVSUPxx TAPEAUTHDSN=YES

**Using RACF TAPEVOL resources**
You can protect volumes that are managed by DFSMShsm by using RACF TAPEVOL
resources that are defined in the RACF descriptor table.

**_Protecting up to 10,000 tapes_**
If you are protecting up to a maximum of 10,000 tapes, you define two RACF resource names
in the TAPEVOL resource class:

- HSMABR is the name for aggregate backup and recovery tapes.
- HSMHSM is the name for all other DFSMShsm tapes.

Issue the RACF commands that are shown in Example 7-2.


-----

_Example 7-2  Define HSM resources in RACF_

    RDEFINE TAPEVOL HSMABR
    RDEFINE TAPEVOL HSMHSM

You can add tapes to RACF before DFSMShsm uses them. If you choose to add tapes to
RACF, you must use the appropriate TAPEVOL. Use the command that is shown to add HSM
tapes to RACF:

`RALTER TAPEVOL HSMHSM ADDVOL( _volser_ )`

**Recommendation:** Do not add tapes to RACF, but instead let DFSMShsm add to the
TAPEVOL automatically for you.

**_Protecting more than 10,000 tapes_**
To RACF-protect more that 10,000 tapes, you define multiple RACF resource names for
DFSMShsm tape volume sets in the TAPEVOL resource class. The following resource names
apply:

- HSMHSM (must be defined)

- HSMABR for aggregate backup and recovery tapes

- DFHSM _x_

The _x_ is a non-blank character (alphanumeric, national, or the hyphen) that corresponds to
the last non-blank character of the tape volume serial number. You need to define a
DFHSM _x_ resource name for each _x_ value that exists, based on your installation naming
standards.

The RACF commands that are shown in Example 7-3 add resource names to the TAPEVOL
class for HSMHSM (required), HSMABR (for aggregate backup and recovery tapes), and
DFHSMA (for all tapes with a volume serial number that ends with the letter A ):

_Example 7-3  Define resource names to RACF TAPEVOL class_

    RDEFINE TAPEVOL HSMHSM
    RDEFINE TAPEVOL HSMABR
    RDEFINE TAPEVOL DFHSMA
    .  .  .
    RDEFINE TAPEVOL DFHSMZ
    RDEFINE TAPEVOL DFHSM@
    RDEFINE TAPEVOL DFHSM$
    RDEFINE TAPEVOL DFHSM#
    RDEFINE TAPEVOL DFHSMRDEFINE TAPEVOL DFHSM0
    .  .  .
    RDEFINE TAPEVOL DFHSM9

To activate the RACF protection of tape volumes that use the defined DFHSMx resource
names, you must issue the following RACF command on each system in the sysplex:

`RALTER TAPEVOL HSMHSM ADDVOL(DFHSMx)`

This command activates RACF TAPEVOL.


-----

You can add RACF protection to the DFSMShsm tape volumes before DFSMShsm uses
them, except for the HSMABR tapes. You must add the tape volume serial number to the
appropriate DFHSM _x_ tape volume set, which is based on the last non-blank character of the
tape volume serial number. To protect a tape with a volume serial of POK33H, you can use a
RACF command:

`RALTER TAPEVOL DFHSM **H** ADDVOL(POK33H)`

**Important:** Tapes that are already protected in the tape volume set of HSMHSM continue
to be protected.

**Using RACF DATASET class profiles**
Protecting volumes that are managed by DFSMShsm by using RACF DATASET class profiles
is described.

**_Tape data set authorization_**
You can use System Authorization Facility (SAF) to protect data sets on tape by using the
RACF DATASET class without needing to activate the TAPEDSN option or the TAPEVOL
class. In addition, you can specify that a user must have access to the first file on a tape
volume to add files on that tape volume.

For optimum tape security, use the combined capabilities of DFSMSrmm, DFSMSdfp, and
RACF. We recommend that you specify the following parameters:

- Specify these parameters in the DEVSUPxx PARMLIB member:

  - TAPEAUTHDSN=YES
  - TAPEAUTHF1=YES
  - TAPEAUTHRC4=FAIL
  - TAPEAUTHRC8=FAIL

  The parameters are described:

  - **TAPEAUTHDSN:** To enable tape authorization checks in the DATASET class.

  - **TAPEAUTHF1:** Enables additional tape authorization checks in the DATASET class
for existing files on the same tape volume when any other file on
the tape volume is opened.

  - **TAPEAUTHRC4:** Use this keyword to control PROTECTALL processing for tape data
sets. This keyword applies to the results of RACROUTE processing
when both TAPEAUTHDSN=YES and TAPEAUTHF1=YES are
specified.

  - **TAPEAUTHRC8:** Use this keyword as an aid to the implementation of
TAPEAUTHDSN and TAPEAUTHF1.

    This task provides a managed and controlled implementation of
tape authorization checks in the DATASET class, and applies only
to the results of TAPEAUTHDSN=YES and TAPEAUTHF1=YES
processing.

- Specify this parameter in the EDGRMMxx PARMLIB member:

  OPTION TPRACF(N)

  For TPRACF(NONE), DFSMSrmm does not create RACF tape profiles for any volumes in
your installation.

- Specify these parameters in RACF:

  `SETROPTS NOTAPEDSN NOCLASSACT(TAPEVOL)`


-----

You must create generic DATASET class profiles that cover each of the data set name
prefixes that DFSMShsm currently uses. Use the **RACF ADDDATASET** commands that are
shown in Example 7-4.

_Example 7-4  RACF ADDDATASET command to protect your DFSMShsm data sets_

    ADDSD 'mprefix.**'
    ADDSD 'bprefix.**'
    ADDSD 'authid.**'
    ADDSD 'bprefix.DMP.**'
    ADDSD 'outputdatasetprefix.**'

The parameters in Example 7-4 are described:

- **mprefix.\*\*:** Specifies the DFSMShsm-defined migrated data set prefix

- **bprefix.\*\*:** Specifies the DFSMShsm-defined backup and dump data set prefix

- **authid.\*\*:** Specifies the DFSMShsm prefix that is used for CDS backups

- **bprefix.DMP.\*\*:** If you want dump tapes to be protected differently than back up tapes

- **outputdatasetprefix.\*\*:**
Specifies that ABARS-created aggregates are protected

The combination of DFSMSrmm, DFSMSdfp, and RACF ensures the following conditions:

- Full 44-character data set name validation.

- Validation that the correct volume is mounted.

- Control of the overwriting of existing tape data sets.

- Management of tape data set retention.

- Control over the creation and destruction of tape volume labels.

- No limitations are caused by RACF TAPEVOL profile sizes and tape volume table of
contents (TVTOC) limitations.

- Common authorization of all tape data sets on a volume.

- Use of generic DATASET profiles, enabling common authorization with DASD data sets.

- Authorization for all tape data sets regardless of the tape label type.

- Authorization for the use of bypass label processing (BLP).

- Exploitation of RACF erase on scratch support.

- Use of DFSMSrmm FACILITY class profiles for data sets that are unprotected by RACF.

Your authorization to use a volume outside of DFSMSrmm control through ignore processing
also enables authorization to the data sets on that volume. To aid in the migration to this
environment, DFSMSrmm provides the TPRACF(CLEANUP) option, and DEVSUPxx
provides TAPEAUTHRC8(WARN) and TAPEAUTHRC4(ALLOW). The function in DFSMSdfp
does not replace all of the functional capabilities that the RACF TAPEDSN option, TAPEVOL
class, and TVTOC provide. However, together with the functions that DFSMSrmm provides,
the capability is equivalent.

The enhanced DFSMSdfp function addresses the authorization requirements for tape data
sets and relies on your use of a tape management system, such as DFSMSrmm, to perform
the following operations:

- Verify full 44-character data set names.
- Control the overwriting of existing tape files.


-----

- Handle tape data set retention.
- Control the creation and destruction of tape labels.

**Important:** With the SAF tape security implementation, as opposed to a TAPEVOL and
TAPEDSN implementation, you can perform the following tasks:

- Write more than 500 data sets to one tape or one tape volume set.
- A tape volume set can expand to more than 42 tape volumes.
- Write duplicate data set names to one tape or one tape volume set.
- The number of volumes that can be protected is not limited.

Use the MVS **SET** **DEVSUP** command to implement the new tape data set security settings.
Figure 7-1 shows the successful result of the command.

    T DEVSUP=18
    IEE252I MEMBER DEVSUP18 FOUND IN SYS1.PARMLIB
    IEE536I DEVSUP VALUE 18 NOW IN EFFECT
    IEA253I DEVSUP 3480X RECORDING MODE DEFAULT IS COMPACTION.
    IEA253I DEVSUP ISO/ANSI TAPE LABEL VERSION DEFAULT IS V3
    IEA253I DEVSUP TAPE OUTPUT DEFAULT BLOCK SIZE LIMIT IS 32760
    IEA253I DEVSUP COPYSDB DEFAULT IS INPUT
    IEA253I DEVSUP TAPEAUTHDSN: YES
    IEA253I DEVSUP TAPEAUTHF1: YES
    IEA253I DEVSUP TAPEAUTHRC4: FAIL
    IEA253I DEVSUP TAPEAUTHRC8: FAIL

_Figure 7-1  Use and result of the SET DEVSUP command_

In Figure 7-1, 18 is the suffix of the device support option member DEVSUP _nn_ that needs to
be activated.

## 7.4 DFSMShsm interaction with a tape management system

DFSMShsm communicates with other components of your z/OS environment through exits,
including your tape management system. ARCTVEXT is the installation-wide exit that allows
communication between DFSMShsm and your tape management system, except for
DFSMSrmm.

To tell DFSMShsm to enable the ARCTVEXT installation-wide, add the following **SETSYS**
command in the ARCCMDxx PARMLIB member. This command shows how to enable the
ARCTVEXT exit for non-Data Facility Removable Media (DFRMM) tape management
systems:

`SETSYS EXITON(ARCTVEXT)`

From z/OS V1.4 forward, the EDGTVEXT general-use programming interface is called by
DFSMShsm to communicate with DFSMSrmm. DFSMShsm automatically calls DFSMSrmm
to process tapes to be returned to the DFSMShsm tape pool or deleted from DFSMShsm.

The interface between DFSMShsm and DFSMSrmm is automatically created when you install
DFSMS. A benefit of this interaction is that DFSMSrmm can prevent DFSMShsm from
overwriting its own CDS backup, automatic dump, ABARS, and copies of backup or migration
tapes.


-----

You can manage the retention of valid tape volumes in two ways:

- Use DFSMSrmm vital record specifications (VRS) to retain tape volumes until DFSMShsm
finishes with them. More details about how to define a VRS are provided later.

- Set up DFSMShsm to use expiration date-protected tape volumes. Use the date 99365 to
prevent DFSMSrmm from considering the volumes for release at any time.

Use VRS because you can avoid expiration dates, operator message handling, and the need
to re-initialize tape volumes before you can reuse them. However, if you use a tape pool that
is managed by DFSMShsm, you can use expiration date protection without this overhead;
DFSMShsm can override expiration protection for its own tape volumes. DFSMShsm uses the
DFSMSrmm EDGTVEXT interface to maintain the correct volume status. DFSMSrmm
provides a programming interface so you can use the DFSMShsm exit.

The following example shows how to code the security setting for tapes, which are shown
from the DFSMShsm perspective:

    TAPESECURITY(EXPIRATION)

## 7.5 Running DFSMShsm with DFSMSrmm

DFSMSrmm can provide enhanced management functions for the tape volumes that
DFSMShsm uses for each of its tape functions. The manner in which the two products work
together depends on how you use each of them in your installation. Run DFSMSrmm with
DFSMShsm to enhance the management of the volumes that DFSMShsm uses. For
example, DFSMSrmm can manage the movement of tapes that must be sent out of the library
for disaster recovery. By using DFSMSrmm and DFSMShsm together, you can use the
scratch tape pool rather than a DFSMShsm tape pool.

DFSMSrmm provides the EDGTVEXT and EDGDFHSM programming interfaces that can be
used by products, such as DFSMShsm, object access method (OAM), and IBM Tivoli®
Storage Manager. Use these programming interfaces for DFSMSrmm tape management so
that you can maintain correct volume status.

DFSMSrmm treats DFSMShsm like any other tape volume user and retains DFSMShsm
volumes based on VRSs and retention period. DFSMShsm automatically calls EDGTVEXT
so you do not need to set up DFSMShsm to communicate with DFSMSrmm.

You can use the TVEXTPURGE PARMLIB option in the DFSMSrmm EDGRMMxx PARMLIB
member to control the action that DFSMSrmm takes when DFSMShsm calls EDGTVEXT. A
benefit of this interaction is that DFSMSrmm can prevent DFSMShsm from overwriting its
own CDS backup, automatic dump, ABARS, and copies of backup or migration tapes.
Although DFSMShsm checks its own migration and its own backup tapes, DFSMSrmm
checks them, also.

### 7.5.1 Authorizing DFSMShsm to DFSMSrmm resources

Before you can use DFSMSrmm with DFSMShsm, you are required to authorize DFSMShsm
to the RACF resource profiles STGADMIN.EDG.MASTER, STGADMIN.EDG.OWNER.user,
and STGADMIN.EDG.RELEASE.

If you have multiple DFSMShsm user IDs, for example, in a multisystem or multihost
environment, and any DFSMShsm ID can create tapes or return tapes to scratch status or
return tapes to the DFSMShsm tape pool, you must authorize each DFSMShsm user ID.


-----

Define STGADMIN.EDG.OWNER. _hsmid_ for each DFSMShsm user ID and give the other
DFSMShsm user IDs UPDATE access to it.

Table 7-2 and Table 7-3 show the necessary access level for DFSMShsm to manage scratch
tapes from a global scratch pool or a scratch pool that is managed by DFSMShsm. This
access is needed to use DFSMShsm with a global scratch pool.

_Table 7-2  Required authorization to use scratch tapes with DFSMShsm_

|Resource|Access required|
|---|---|
|STGADMIN.EDG.RELEASE|READ|
|STGADMIN.EDG.MASTER|READ|
|STGADMIN.EDG.OWNER.hsmid|UPDATE|



This access is necessary for a DFSMShsm-owned pool.

_Table 7-3  Required authorization to use DFSMShsm with a DFSMShsm scratch pool_

|Resource|Access required|
|---|---|
|STGADMIN.EDG.MASTER|UPDATE|
|STGADMIN.EDG.OWNER.hsmid|UPDATE|



### 7.5.2 Authorizing ABARS to DFSMSrmm resources

To use DFSMSrmm with DFSMShsm ABARS, you must assign ABARS IDs the correct levels
of authorization to STGADMIN.EDG.MASTER, STGADMIN.EDG.OWNER. _user_ , and
STGADMIN.EDG.RELEASE. If you have multiple ABARS user IDs, for example in a
multisystem environment, and any ABARS ID can return tapes to scratch status, you must
authorize each ABARS user ID. Define STGADMIN.EDG.OWNER. _abarsid_ for each ABARS
user ID and give the other ABARS user IDs UPDATE access to it. This method allows one
ABARS ID to release the tapes initially that are obtained from scratch by the other ABARS ID.

### 7.5.3 Setting DFSMSrmm options when using DFSMShsm

You use the DFSMSrmm PARMLIB EDGRMMxx to specify the installation options for
DFSMSrmm as described in ““EDGRMMxx PARMLIB member”.

If you use expiration dates to manage tapes, consider the values that you specify for the
**parmlib** command MAXRETPD, RETENTIONMETHOD, and TVEXTPURGE operands and
the **VLPOOL** command EXPDTCHECK operand:

- The MAXRETPD operand specifies the maximum retention period that a user can request
for data sets on volumes.

- The RETENTIONMETHOD operand specifies the retention method (EXPDT or VRSEL)
that DFSMSrmm uses for DFSMShsm volumes.

- The TVEXTPURGE operand specifies how you want to handle the release of DFSMShsm
tape volumes.

- The **VLPOOL** command EXPDTCHECK operand tells DFSMSrmm how to manage a
volume based on the expiration date field in the volume label.


-----

You can use the EXPDT retention method to avoid the processing of DFSMShsm volumes on
each inventory management VRSEL run. This retention method requires that you use
expiration date protection for DFSMShsm tape volumes, EXPDT= 99365 .

When DFSMShsm sets 99365 as the expiration date to manage its tapes, 99365 means
permanent retention or no expiration. If you choose to use DFSMShsm expiration
date-protected tape volumes, DFSMShsm sets the date 99365 to prevent DFSMSrmm from
considering the volumes for release at any time. You must specify the MAXRETPD(NOLIMIT)
operand to ensure that DFSMSrmm honors the 99365 date.

You also use the DFSMSrmm PARMLIB MAXRETPD operand value to reduce the expiration
date for all volumes, including DFSMShsm volumes. If you want to reduce the 99365
permanent retention expiration date, specify the MAXRETPD with a value 0 -  99999 days.

**Recommendation:** If you want to avoid the overhead of VRSEL processing for tape
volumes that are managed by DFSMShsm (and similar applications), use the EXPDT
retention method. If, however, you require DFSMSrmm to manage the movement of
DFSMShsm volumes, use DFSMSrmm VRS instead of the 99365 permanent retention
date to retain DFSMShsm volumes.

### 7.5.4 Setting DFSMShsm options when using DFSMSrmm

As DFSMShsm uses a tape volume, DFSMSrmm records information about data sets and
multivolume data sets at OPEN time. DFSMSrmm can use this information to manage the
volumes based on the DFSMSrmm policies that you define. DFSMShsm uses the data
control block (DCB) Open/end of volume (EOV) volume security and verification exit to ensure
that an acceptable volume is mounted for DFSMShsm use. DFSMShsm uses this exit to
reject unacceptable volumes. For example, DFSMShsm rejects volumes that are already in
use by DFSMShsm and volumes that are not authorized for use by DFSMShsm. DFSMSrmm
records information only for those volumes that are not rejected by DFSMShsm.

DFSMSrmm provides facilities so that DFSMShsm can tell DFSMSrmm when it no longer
requires a tape volume or when a tape volume changes status. The benefit is that
DFSMShsm cannot mistakenly overwrite one of its own tape volumes if an operator mounts a
tape volume in response to a request for a non-specific tape volume.

### 7.5.5 Setting DFSMShsm system options

The DFSMShsm system options that relate to the use of a tape management system other
than DFSMSrmm with global or private scratch pools are shown in Figure 7-11.

_Example 7-5  DFSMShsm system options_

    SETSYS EXITON(ARCTVEXT)/EXITOFF(ARCTVEXT)    SELECTVOLUME               TAPEDELETION               TAPESECURITY               PARTIALTAPE
   
The ARCVEXT is activated for a non-DFSMSrmm tape management system.


-----

### 7.5.6 Setting DFSMShsm dump definitions

The DFSMShsm dump definitions are shown in Example 7-6.

_Example 7-6  Setting DFSMShsm DUMP parameters_

DEFINE DUMPCLASS(class    AUTOREUSE    TAPEEXPIRATIONDATE(date)    RETENTIONPERIOD(day))

The use of **AUTOREUSE** ensures that tapes are returned to the global scratch pool when they
expire.

### 7.5.7 DFSMSrmm support for DFSMShsm naming conventions

DFSMSrmm supports all DFSMShsm options and any of the naming conventions for
DFSMShsm tape data sets, except for password security on tape volumes.

**DFSMSrmm support for retention and pooling**
With DFSMShsm, you can use DFSMSrmm system-based scratch tape pools, exit-selected
scratch pools, or tape pools that are managed by DFSMShsm.

**Recommendation:** Use a DFSMSrmm scratch pool. Use the pool that is managed by
DFSMShsm only when necessary. For example, use the pool that is managed by
DFSMShsm if you want to keep pools that are managed by DFSMShsm for Enhanced
Capacity Cartridge System Tapes because DFSMShsm fully uses a tape’s capacity. You
must let DFSMShsm decide whether a tape volume contains valid data and whether it
returns to the DFSMSrmm scratch pool or a tape pool that is managed by DFSMShsm.

Define VRSs to retain tape volumes until DFSMShsm finishes with them. DFSMSrmm uses
the retention period that is determined by the VRS to extend any expiration date or retention
period that was previously set for the volume. Additionally, you can use VRSs to identify
volumes that need to be moved out of the installation media library for safekeeping, or moved
from an ATL to a manual library. For tapes that are managed by DFSMShsm, you do not need
to respond to the IEC507D messages that are issued for expiration data-protected tapes
because DFSMShsm can override expiration for its own tapes. If you choose to use
DFSMShsm expiration date protected tape volumes, DFSMShsm sets the expiration date to
99365 , which means permanent retention to prevent DFSMSrmm from considering the
volumes for release at any time.

**Retaining DFSMShsm tapes by using expiration dates**
To use DFSMShsm expiration date protection, specify the DFSMShsm startup option:

`SETSYS TAPESECURITY(EXPIRATION)`

If you use a system scratch tape pool for DFSMShsm tapes, you need a way to manage tapes
that are protected with expiration dates that are set by DFSMShsm. To help you manage this
situation, you can automate the responses to expiration date protection messages for scratch
pool tape volumes with DFSMSrmm. Use the PARMLIB member **VLPOOL** command to set up
this automation. Set the VLPOOL EXPDTCHECK operand to EXPDTCHECK( N ). DFSMSrmm
automatically lets your users reuse the volumes in the pool without operator intervention and
without creating data integrity exposures.


-----

If you use a tape pool that is managed by DFSMShsm, DFSMShsm validates and overrides
expiration dates on its emptied, previously used tapes.

**Defining VRSs to manage DFSMShsm tapes**
Movement and retention policies are defined by using VRSs by specifying data set names or
volume serial numbers. To define VRSs, use the **RMM ADDVRS** subcommand. _DFSMSrmm_
_Managing and Using Removable Media,_ SC26-7404, offers information about the **RMM ADDVRS**
subcommand and the operands you can specify. When you specify the **RMM ADDVRS** command
operands, it might be helpful to think about the operands in three categories:

- Operands to define retention policies, including COUNT and CYCLES that are used in
these examples

- Operands to define movement policies, including DELAY, LOCATION, and
STORENUMBER, that are used in these examples

- Operands to manage the VRS itself, including DELETEDATE and OWNER that are
defaults in the **RMM ADDVRS** subcommand:

  - DELETEDATE( 1999 / 365 ) specifies the date when the VRS no longer applies. The
default value is 1999 / 365 , which means the VRS is permanent. It can be manually
deleted only if it is no longer appropriate.

  - OWNER( _owner_ ) specifies the user ID that owns the VRS.

As shown in the following examples, you can specify data set name masks in VRSs to
manage the following functions:

- Migration
- Backup
- Dumps
- TAPECOPY
- DUPLEX tape feature
- Tapes that are written by ABARS
- ABARS accompany tapes
- CDS version backups

You can tailor the examples to define policies for your DFSMShsm tapes. As you gain more
experience defining VRSs, you see several ways to define the retention and movement
policies you want.

The examples use the current DFSMShsm data set naming formats. Certain older name
formats were in use before APAR OY20664. APAR OY20664 requires you to use **PATCH**
commands to use the new format. The new format is standard in DFSMShsm. If the older
naming formats exist in your installation, you might need to define VRSs that use both naming
formats. Delete the VRSs with the old naming format when the old names no longer exist. For
more information, see _z/OS DFSMShsm Implementation and Customization Guide,_
SC35-0418. This book also provides detailed information about retaining all types of
DFSMShsm tapes by using VRS.

**Examples of DFSMSrmm panels**
The easy way to enter the DFSMSrmm panels is to select the Interactive Storage
Management Facility (ISMF) Primary Option Menu panel and enter R for the Removable
Media Manager for the DFSMSrmm option, as shown in Figure 7-2.


-----

    ISMF PRIMARY OPTION MENU - z/OS DFSMS V1 R13
    Selection or Command ===>
    
    0 ISMF Profile       - Specify ISMF User Profile
    1 Data Set         - Perform Functions Against Data Sets
    2 Volume          - Perform Functions Against Volumes
    3 Management Class     - Specify Data Set Backup and Migration Criteria
    4 Data Class        - Specify Data Set Allocation Parameters
    5 Storage Class       - Specify Data Set Performance and Availability
    6 Storage Group       - Specify Volume Names and Free Space Thresholds
    7 Automatic Class Selection - Specify ACS Routines and Test Criteria
    8 Control Data Set     - Specify System Names and Default Criteria
    9 Aggregate Group      - Specify Data Set Recovery Parameters
    10 Library Management    - Specify Library and Drive Configurations
    11 Enhanced ACS Management  - Perform Enhanced Test/Configuration Management
    C Data Collection      - Process Data Collection Function
    G Report Generation     - Create Storage Management Reports
    L List           - Perform Functions Against Saved ISMF Lists
    P Copy Pool         - Specify Pool Storage Groups for Copies
    **R Removable Media Manager  - Perform Functions Against Removable Media**
    X Exit           - Terminate ISMF
    Use HELP Command for Help; Use END Command or X to Exit.

_Figure 7-2  ISMF Primary Option Menu_

After your selection, the primary DFSMSrmm panel appears, as shown in Figure 7-3. Enter 3
to view the Administrator functions.

    REMOVABLE MEDIA MANAGER (DFSMSrmm) - z/OS V1R13
    Option ===>
    
    0 OPTIONS    - Specify dialog options and defaults
    1 USER     - General user facilities
    2 LIBRARIAN   - Librarian functions
    **3 ADMINISTRATOR - Administrator functions**
    4 SUPPORT    - System support facilities
    5 COMMANDS   - Full DFSMSrmm structured dialog
    6 LOCAL     - Installation defined dialog
    X EXIT     - Exit DFSMSrmm Dialog
    
    Enter selected option or END command. For more info., enter HELP or PF1.

_Figure 7-3  Option 3 Administrator is selected_

The panel that is shown in Figure 7-4 is displayed. First, we work with VRSs.
Enter option 3 to reach this function.


-----

    DFSMSrmm Administrator Menu - z/OS V1R13
    Option ===>
    
    0 OPTIONS  - Specify dialog options and defaults
    1 VOLUME  - Display or change volume information
    2 OWNER   - Display or change owner information
    **3 VRS    - Vital record specifications**
    
    Enter selected option or END command. For more info., enter HELP or PF1.

_Figure 7-4  Option 3 is selected to work with VRSs_

To add a VRS, enter option 2 , as shown in Figure 7-5.

     DFSMSrmm Vital Record Specification Menu
     Option ===>
     
     0 OPTIONS  - Specify dialog options and defaults
     1 DISPLAY  - Display a vital record specification
     **2 ADD    - Add a vital record specification**
     3 CHANGE  - Change a vital record specification
     4 DELETE  - Delete a vital record specification
     5 SEARCH  - Search for vital record specification
     
     Enter selected option or END command. For more info., enter HELP or PF1.

_Figure 7-5  Option 2 to add a VRS is selected_

In Figure 7-6, enter a data set mask.

    DFSMSrmm Add Vital Record Specification
    Command ===>
    
    Specify one of the following:
    Data set mask . . ' **hsm.dmp.**** '
    Job name mask . .      Add data set filter VRS . .  (Yes or No)
    
    Volume serial . .     ( May be generic )
    
    VRS name . . . .

_Figure 7-6  DFSMSrmm Add Vital Record Specification_

In Figure 7-7, enter the count, retention type, location, owner, description, and
deletion date.


-----

    DFSMSrmm Add Data Set VRS
    Command ===>
    
    Data set mask . : ' **HSM.DMP.**** '                  GDG . . **NO**
    Job name mask . :
    
    Count . . . . **99999** Retention type . . . . . . **DAYS**
    While cataloged . . . . . .
    Delay . . . .   Days       Until expired . . . . . . .
    
    Location . . . . . . **CURRENT**
    Number in location
    Priority . . . . . . 0
    Release options:
    Next VRS in chain . .        Expiry date ignore . . . .
    Chain using . . .        Scratch immediate . . . . .
    
    Owner . . . . . . **HSMTASK**
    Description . . . **RETAIN ALL DFSMSHSM DUMP DS**
    Delete date . . . **1999/365** ( YYYY/DDD )
    
    Press ENTER to ADD the VRS, or END command to CANCEL.

_Figure 7-7  DFSMSrmm Add Data Set VRS_

Table 7-4 describes the fields that are used to add a VRS.

_Table 7-4  DFSMSrmm fields to add a VRS_

|Field|Field descriptions|
|---|---|
|Data set mask|Specify the fully or partially qualified data set name.|
|Job name mask|This field can be left blank because DFSMShsm creates the tapes.|
|Generation data group (GDG)|DFSMShsm does not create GDGs, so enter NO.|
|Description|Any valid description that is meaningful for the VRS.|
|Owner|DFSMShsm is a large user of your tape resources. You need to create a unique DFSMSrmm owner. In this case, we created HSMTASK as the owner for all DFSMShsm tapes. The z/OS DFSMShsm Implementation and Customization Guide, SC35-0418, describes how to create an owner.|
|Retention type|If you want to send your tapes offsite or to the vault for a specific period, for example, 14 days for your dump tapes, specify DAYS. This value relates to the Store number in the location field.|
|While cataloged|Specify NO for DFSMShsm data sets.|
|Until expired|Specify NO for DFSMShsm data sets.|


-----

|Field|Field descriptions|
|---|---|
|Location|Valid entries are HOME, LOCAL, DISTANT, and REMOTE, but see the z/OS DFSMShsm Implementation and Customization Guide, SC35-0418.|
|Delete date|This field specifies the date when the VRS will be deleted. DFSMSrmm uses the default value of 1999/365, which means “never delete”.|
|Vital record count|99999 indicates that all vital records are kept.|
|Delay|You can delay the movement of your offsite data. A 0 indicates to move the tapes immediately. A 1 indicates to move the tapes the following day.|
|Store number in location|This value indicates the number of days that any particular tape is to stay offsite. We used DAYS in the Retention type field so our tapes will stay offsite for 14 days in location LOCAL. (LOCAL can be any storage facility in the nearby area.)|
|Next VRS name|Use this field only if you are planning to stage your tape movements, for example, DISTANT, then REMOTE, and then LOCAL. This field points to the next VRS to look at in the chain.|


To see the vital records that are already defined in DFSMSrmm, enter option 5 SEARCH to
search for vital record specifications on the DFSMSrmm VRS menu. Figure 7-8 shows a
generic search for all vital records that are defined in this DFSMSrmm.

    ssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssss
    DFSMSrmm Search VRSs
    Command ===>
    
    Optionally specify one of:
    Data set mask                          GDG . .
    Job name mask                       ( Yes or No )
    Volume serial          Retention type
    VRS name . . .         While cataloged    ( Yes or blank for all)
    Until expired     ( Yes or blank for all)
    Location . . . .        Release options:
    Next VRS in chain        Expiry date ignore    ( Yes or No )
    Chain using . .        Scratch immediate     ( Yes or No )
    Owner . . . . . . *****
    Limit . . . . . . ***** Limit search to first n VRSs. Default is *
    Dates      Start     End    Date, date range or relative value
    Reference . .       . .
    Changed . . .       . .
    Clist . . . . . .        YES to create a data set, or NO, or blank
    
    Enter HELP or PF1 for the list of available line commands

_Figure 7-8  Generic search for all vital records in DFSMSrmm_


-----

**Tip:** To see all defined VRS definitions, leave all fields blank except for asterisks in the
OWNER and LIMIT fields.

Figure 7-9 shows an example of the search results.

    DFSMSrmm VRSs (Page 1 of 4)      Row 1 to 3 of 3
    Command ===>                         Scroll ===> PAGE
    
    Enter HELP or PF1 for the list of available line commands.
    Use the LEFT and RIGHT commands to view other data columns.
    
    S Volume/Data set/Name specification      Job name Type Location Prty
    -- -------------------------------------------- -------- ---- -------- -----
    ABEND                         DSN HOME   0
    ABEND                    DFHSM*  DSN HOME   0
    ...
    HSM.**                        DSN HOME   0
    **HSM.DMP.**                      DSN CURRENT 0**
    ...
    RMM.F13002.**                     DSN HOME   0
    ...
    OPEN                         DSN HOME   0
    OPEN                     DFHSM*  DSN HOME   0

_Figure 7-9  Listing of vital records in DFSMSrmm_

Figure 7-10 shows an example of how to create a vital record to protect your
daily DFSMShsm backup tapes.


-----

    DFSMSrmm Add Data Set VRS
    Command ===>
    
    Data set mask . : ' **HSM.BACKTAPE.DATASET** '             GDG . . NO
    Job name mask . :
    
    Count . . . . 99999         Retention type . . . . . . **cycles**
    While cataloged . . . . . . NO
    Delay . . . . 0  Days       Until expired . . . . . . . NO
    
    Location . . . . . . **CURRENT**
    Number in location **99999**
    Priority . . . . . . 0
    Release options:
    Next VRS in chain . .        Expiry date ignore . . . . NO
    Chain using . . .        Scratch immediate . . . . . NO
    
    Owner . . . . . . HSMTASK
    Description . . . HSM backup tapes
    Delete date . . . 1999/365  ( YYYY/DDD )
    
    Press ENTER to ADD the VRS, or END command to CANCEL.

_Figure 7-10  Creating VRSs for DFSMShsm backup tapes_

In this example, we changed three fields:

- Retention type was changed to CYCLES because we want any tape that uses this VRS to
continue to use it until the tape is either recycled, expires naturally, or is deleted.

- Location can be changed to HOME if you do not want daily backup tapes sent offsite.

- Number in location is changed to 99999 , which is the DFSMSrmm default. This value
works with the CYCLES selection to ensure that all tapes are retained in location HOME.

A similar vital record definition must be created for your migration tapes.

**TVEXTPURGE extra days**
With z/OS V1R12 DFSMSrmm, the **TVEXTPURGE** parameter used the options RELEASE , EXPIRE ,
and NONE . With z/OS V1.13, DFSMSrmm has a new option for the EXPIRE ( _days_ ).

If tapes are expired by using the EDGTVEXT HSM exit, extra days for retention can be
defined with no additional processing.

**Use of extra days**
You can use DFSMSrmm to release DFSMShsm tapes that are requested to be purged by
DFSMShsm. You can also specify that DFSMSrmm retain a tape for a few days after its
expiration date is reached. By default, the expiration date for DFSMShsm tapes is protected
by DFSMShsm. DFSMShsm uses 1999/365 as the expiration date for permanent retention. To
enable extra days of retention for purged DFSMShsm tape volumes, use the
**TVEXTPURGE(EXPIRE(** **_days_** **))** option or set retention options in the VRSs. These specifications
are used to retain the tape volumes.


-----

**EDGRMMxx PARMLIB member**
This member specifies how DFSMSrmm processes volumes that are purged by callers of
EDGTVEXT or EDGDFHSM. The TVEXTPURGE operand specifies how you want to handle
the release of DFSMShsm tape volumes, by using the new specification that is shown in
Figure 7-11.

_Figure 7-11  EDGTVEXT TVEXTPURGE operand_

The command has the following parameters:

**EXPIRE(** **_days_** **)** Use the EXPIRE( _days_ ) option to set the volume expiration date to the
current date plus _days_ for volumes to be purged. Use the
EXPIRE( _days_ ) parameter when you use the EXPDT retention method,
by using days as a way to delay expiration of the volume. When you
use the VRSEL retention method, you can optionally use this operand
in combination with VRSs that use the UNTILEXPIRED retention type.
Apply EXPIRE( _days_ ) to set a new volume EXPDT. Then, run VRSEL
to extend retention by using the extra days retention type. For
example, specify EXPIRE(0) when you use a VRS with extra days
retention. You can also use a non-zero EXPIRE( _days_ ) value and avoid
using an extra days retention VRS. The _days_ parameter is the number
of days that DFSMSrmm retains the volume before the volume is
considered for release. The value is a 1-digit to 4-digit decimal number
and is added to today’s date to compute the new expiration date. If the
value exceeds the maximum retention period (MAXRETPD), it is
reduced to the MAXRETPD value. The default value for days is 0.
TVEXTPURGE(EXPIRE) is the same as TVEXTPURGE(EXPIRE(0)).

**NONE** DFSMSrmm takes no action for volumes to be purged.

**RELEASE** DFSMSrmm releases a volume to be purged according to the release
actions that are set for the volume. You must run expiration processing
to return a volume to scratch status. This parameter is the default. You
can specify that volumes that are purged from DFSMShsm are to be
retained for a few extra days. With this process, you can ensure that
purged volumes do not contain any data that might still be needed.
DFSMShsm migration and backup volumes can be retained by
EXPDT= 99365 . You can optionally set EXPDT to the current date or a
future date when the volume is purged from DFSMShsm with
EDGTVEXT. You can release volumes or set the volume expiration
date either to the current date or based on a number of days from the
current date. The effect of setting the expiration date depends on the
retention method (EXPDT or VRSEL) that is already specified for the
volume.


-----

**How to code expiration in DFSMShsm and DFSMSrmm**

Coding expiration in DFSMShsm and DFSMSrmm is described.

**_DFSMShsm_**
HSM supports the changes to the expiration date that were described previously. In summary,
when DFSMShsm sets 99365 as the expiration date to manage its tapes, 99365 means
permanent retention or no expiration. If you choose to use DFSMShsm expiration
date-protected tape volumes, DFSMShsm sets the date to 99365 . This setting prevents
DFSMSrmm from considering the volumes for release at any time. You must specify the
MAXRETPD(NOLIMIT) operand to ensure that DFSMSrmm recognizes the 99365 date.

**_DFSMSrmm_**
You also use the DFSMSrmm PARMLIB MAXRETPD operand value to reduce the expiration
date for all volumes that include DFSMShsm volumes. If you want to reduce the 99365
permanent retention expiration date, specify the MAXRETPD with a value of 0 -  99999 days. If
you want to avoid the processor needs of VRSEL processing for tape volumes that are
managed by DFSMShsm (and similar applications), use the EXPDT retention method. If you
require DFSMSrmm to manage the movement of DFSMShsm volumes, use DFSMSrmm
VRSs instead of the 99365 permanent retention date to retain DFSMShsm volumes.

**_Managing DFSMShsm tapes by using EDGDFHSM_**
DFSMShsm uses the EDGDFHSM programming interface to release DFSMShsm tape
volumes in DFSMSrmm. The interface between DFSMShsm and DFSMSrmm is
automatically in place when you install DFSMS, so you do not need to activate it or to call it.
Any caller of EDGDFHSM must be defined to RACF. If the caller is a started task, define the
user ID with the STARTED class. Authorize your application to release its own tape volumes.

You must also consider how volumes are retained until the application calls the EDGDFHSM
programming interface to release the volumes. You can use this program interface from
programs other than DFSMShsm to release tape volumes. For information about setting up
DFSMShsm with DFSMSrmm, see Chapter 14, “Problem determination” on page 401. You
can use this information as an example for setting other applications that manage tape. You
can retain tapes by defining VRSs such as the VRSs that are shown in this example.

**Example:** Define policies to retain all of the data until a volume is released by the
application. See Example 7-7.

_Example 7-7  Defining policies to retain data until a volume is released by the application_

    RMM ADDVRS DSN(’**’) JOBNAME(jobname) LOCATION(CURRENT) DAYS COUNT(99999)
    RMM ADDVRS DSN(’ABEND’) JOBNAME(jobname) LOCATION(CURRENT) DAYS COUNT(99999)
    RMM ADDVRS DSN(’DELETED’) JOBNAME(jobname) LOCATION(CURRENT) DAYS COUNT(99999)
    RMM ADDVRS DSN(’OPEN’) JOBNAME(jobname) LOCATION(CURRENT) DAYS COUNT(99999)
    
Another way to retain tapes is with the EXPDT retention method. You can use the
DFSMSrmm PARMLIB OPTION TVEXTPURGE operand to control the processing that
EDGDFHSM programming interface performs. You can release volumes or set the volume
expiration date either to the current date or based on a number of days from the current date.
The effect of setting the expiration date depends on the retention method (EXPDT or VRSEL)
that is already specified for the volume. For more information, see TVEXTPURGE, which is
described in ““EDGRMMxx PARMLIB member”.


-----

Callers of the EDGDFHSM programming interface can request the processing of volumes
only when their RACF user ID is authorized, as defined by the RACF FACILITY class profiles
for DFSMSrmm.

## 7.6 Using an SMS-managed tape library

A common tape setup for a DFSMShsm environment is based on SMS-managed tape
libraries. A system-managed tape library consists of tape volumes and tape devices that are
defined in the tape configuration database.

The tape configuration database is an integrated catalog facility user catalog that is marked
as a volume catalog (VOLCAT) and contains tape volumes and tape library records.

A system-managed tape library can be either automated or manual. An _automated tape_
_library_ (ATL) is a device that consists of robotic components, cartridge storage areas (or
shelves), tape subsystems, and controlling hardware and software, with the set of tape
volumes that reside in the library and can be mounted on the library tape drives. A _manual_
_tape library_ (MTL) is a set of tape drives and the set of system-managed volumes the
operator can mount manually on those drives.

If backup or migration uses a system-managed tape library, specify the **LIBRARYBACKUP** or
**LIBRARYMIGRATION** parameters with the **SETSYS TAPEUTILIZATION** command. Virtual tape
systems generally use a PERCENTFULL value of 97%, unless a larger value is needed for
virtual tapes that are larger than the nominal 400 MB standard capacity MEDIA1 or 800 MB
enhanced capacity MEDIA2 tapes.

In the case of the newer virtual tape systems (TS7700 Release 1.4 and later), where
DFSMShsm derives media capacity by checking the mounted virtual tape, DFSMShsm allows
a PERCENTFULL value up to 110%. Any larger value is reduced to 100%.

For older virtual tape systems, where DFSMShsm cannot dynamically determine virtual tape
capacity, PERCENTFULL values that are larger than 110% are honored.

### 7.6.1 Converting to a DFSMS-managed library

Consider how you will convert to an SMS-managed tape library. Is this move a physical move
of all tapes into the library? Alternatively, it might be a DFSMShsm RECYCLE into the library
from the old DFSMShsm tapes with directing new allocations to the library? It might be a
combination of these two approaches.

Information about the physical move approach is described. In this scenario, we focus only on
DFHSM.

**Considerations for moving tapes into an SMS-managed library**
The tapes that you insert in an SMS-managed tape library either contain valid data or are
empty.

Tapes that contain valid data need to be in private status. They must be associated with a
tape storage group. And, they need to be assigned the correct recording technology,
compaction, and media type through data class assignment.

Empty tapes that are not defined to DFSMShsm need to be assigned a scratch status and
associated with the global scratch pool. The use of a global scratch pool is the recommended
solution to assign scratch tapes to DFSMShsm today.


-----

Empty tapes that are defined to DFSMShsm (specific scratch pool) need to be assigned a
PRIVATE status and they need to be associated with the tape storage group for the
DFSMShsm function to which the tapes were set up with ADDVOL. After a storage group is
assigned to a tape volume, the storage group cannot be changed except with ISMF, JCL, or
by returning the tape to scratch status.

### 7.6.2 Identifying tape library tapes

First, ensure that all of the esoterics that point to the library are in place. Adjust your
USERUNITTABLE in the DFSMShsm PARMLIB to ensure that all of the esoterics that point to
the library are in place.

DFSMShsm uses connected sets. Identify these sets and ensure that you move these
connected sets together as the transition into a library occurs. You can move all of your
DFSMShsm tapes into the library. However, it might be helpful to test the connectivity and
functionality before you move all of your tapes. You can use the DFSMShsm **LIST** command
to identify a connected set.

First, look at the tapes that are connected (or for tapes that are not connected). Use the
commands that are shown in Example 7-8.

_Example 7-8  DFSMShsm list of connected or unconnected sets_

    LIST TTOC SELECT(CONNECTED)
    LIST TTOC SELECT(NOTCONNECTED)

Also, look at the filling criteria on your current tape. The commands that are shown in
Example 7-9 can be useful.

_Example 7-9  DFSMShsm list of filling criteria on the DFSMShsm tapes_

    LIST TTOC SELECT(FULL)
    LIST TTOC SELECT(NOTFULL)
    LIST TTOC SELECT(EMPTY)

You might want to stop using the empty tapes. You can set the empty tapes as DELVOL.
Specify MARKFULL on the partially filled tapes.

To identify your disaster recovery tapes (duplex or alternate volumes), issue the **LIST TTOC**
command, as shown in Example 7-10.

_Example 7-10  DFSMShsm list of DUPLEX or alternate volumes_

    LIST TTOC SELECT(ALTERNATEVOLUME)
    LIST TTOC SELECT(DISASTERALTERNATEVOLUME)

These tapes might be candidates for future ejects or they can be transferred to an offsite
library. The following command shows a DFSMShsm list of alternate tapes with errors:

`LIST TTOC SELECT(ERRORALTERNATE)`

These tapes are marked as full. Consider your action for each tape in the list.

The following example shows the **LIST AGGREGATE** command to list aggregate backups:

`LIST AGGREGATE( _agname_ )`

These backups are candidates for being moved offsite or being created in an offsite library.


-----

Next, set your ACS routines to point to the new library and set all future DFSMShsm tape
allocations to occur in the new library (if possible). You might need to look at the exceptions
where you want a tape to be allocated outside of the library (specific media type that is going
to an outside location). Ejects for specific purposes might also be needed, such as disaster
recovery tapes or data that is intended for partners that can read these tapes.

Through the ACS routines, allocations can also be made to an offsite library for disaster
recovery, if a library is available.

Also, look into your procedures, such as backup, dump, recycle, and aggregate backup, to
verify that they also occur in the new setup.

## 7.7 Using Virtual Tape Servers for DFSMShsm

The most commonly used tape setup today for a DFSMShsm environment is based on
_Virtual Tape Servers_ (VTSs), the TS7700 Virtualization Engine, or the older B10 and B20
hardware. The VTS uses the 3494 or 3584 tape library for the back end. Virtual tape is
transparent to physical tape allocation in relation to the DFSMShsm PARMLIB setup and the
ACS routines.

Running a peer-to-peer virtual tape setup might require changes that relate to the duplexing
of tapes. Consider whether to use peer-to-peer copy instead of a previous duplex copy that is
based on DFSMShsm duplication. Also, if you use duplex copy for offsite recovery as part of
your disaster recovery plan, consider whether to use the COPY EXPORT feature or whether
the peer-to-peer copy meets your disaster recovery requirements.

Using virtual tape for DFSMShsm on the TS7700 is described. VTS (B10 and B20) is
referenced only in exceptions.

### 7.7.1 Considerations for using VTS/TS7700 for DFSMShsm

DFSMShsm is a good design with physical tapes. The TS7700 does not have the same
challenge as individual tape. The TS7700 adds to the capacity with actual data only and not
to the logical capacity.

DFSMShsm benefits because of the potential of more logical tape drives (up to 256 in a
single cluster grid and 512 in a 2-cluster grid). The DFSMShsm bandwidth can be increased
by adding more drives to the DFSMShsm tasks and also by adding more DFSMShsm
auxiliary tasks (MASH).

Another good reason to run DFSMShsm in a virtual environment is that the maintenance
functions, such as AUDIT MEDIACONTROLS and TAPECOPY, can run more smoothly on
smaller media in a virtual environment.

### 7.7.2 Larger logical volume sizes in TS7700

Consider the use of larger logical volume sizes for DFSMShsm, especially for backup tapes.
These tapes will not be reread that often, compared to migration tapes that might be read
occasionally. The _Recall Takeaway_ risk is increased when you use larger volume sizes for the
ML2 volumes. Several options are described. Consider which options are the best for your
system.


-----

The TS7700 supports logical volumes with the following maximum sizes:

- 400 MB
- 800 MB
- 1,000 MB
- 2,000 MB
- 4,000 MB
- 6,000 MB
- 25,000 MB

However, effective sizes can be larger if data is compressed. For example, if your data
compresses with a 3:1 ratio, the effective maximum logical volume size for a 25,000 MB
logical volume is 75,000 MB.

Depending on the logical volume sizes that you choose, the number of required volumes to
store your data grows or shrinks, depending on the media size that you convert from. If your
data sets fill native 3590 volumes, even with 25,000 MB logical volumes, you need more
TS7700 logical volumes to store the data, which is stored as multi-volume data sets. You can
specify 400 MB CST-emulated cartridges or 800 MB ECCST-emulated cartridges when you
add volumes to the TS7700.

You can use these sizes directly or use policy management to override them to provide for the
1,000 MB, 2,000 MB, 4,000 MB, 6,000 MB, and 25,000 MB sizes. A logical volume size can
be set by VOLSER and can change dynamically by using the DFSMS Data Class storage
construct when a job requires a scratch volume. The amount of data that is copied to the
stacked cartridge is only the amount of data that was written to a logical volume. The choice
between all available logical volume sizes does not affect the real space that is used in either
the TS7700 cache or the stacked volume. Unless you need CST emulation (400 MB), specify
the ECCST media type when you insert volumes in the TS7700.

When you plan for the necessary number of logical volumes, first determine the number of
private volumes that make up the current workload you plan to migrate. Look at the amount of
data on your current volumes and then match that amount to the supported logical volume
sizes. Match the volume sizes and include the compressibility of your data. If you do not know
the average ratio, use the conservative value of 2:1.

DFSMShsm writes tapes as a single file by using connected sets. User data is written to a
tape volume in 16 K logical block increments. A user data set can span a maximum of 254
volumes. So, considering the 254 maximum volume count, DFSMShsm will assign a new
volume after 215 volumes to ensure that it is always able to write a user data set at the
maximum size.

Considerations for larger volumes can be based on the limitation of data sets to span only
254 volumes. If you work with an 800 MB virtual tape, the maximum data set size, which is
based on an expected compression factor of 2.5, is (254 x 800 x 2.5) = 508 GB. Because the
largest logical volume size is 25,000 MB, the maximum data set on a data set that spans 254
DFSMShsm volumes is (254 x 25,000 x 2.5) = 15,875 GB.



-----


### 7.7.3 The TS7700 and awareness of tape capacity

The TS7700 is aware of the virtual volume sizes from code level 1.4 and higher. Therefore,
the DFSMShsm **PERCENTFULL** parameter is less important and no longer required. In addition
to the TS7700 required code level for this support, the host support is described in APAR
OA24969.

APAR OA24969 states that virtual tape systems need to generally use a PERCENTFULL
value of 97% unless a larger value is needed to account for virtual tapes that are larger than
the nominal 400 MB standard capacity MEDIA1 or 800 MB enhanced capacity MEDIA2
tapes. In the case of the newer virtual tape systems (TS7700 R 1.4 and higher) where HSM
derives media capacity by checking the mounted virtual tape, HSM allows a PERCENTFULL
value up to 110%. Any value that is larger is reduced to 100%. For older virtual tape systems
where HSM cannot dynamically determine virtual tape capacity, PERCENTFULL values that
are larger than 110% are honored.

For a detailed description of the **SETSYS TAPEUTILIZATION** command, see _z/OS DFSMShsm_
_Storage Administration,_ SC35-0421.

The same is true for the **SETOAM TAPECAPACITY** command in OAM. With this support, the
awareness of the tape capacity is in the hardware and the command is not needed.

If you change the usage of virtual volumes to larger virtual volume sizes, ensure that
DFSMShsm uses those volumes. The following example shows a DFSMShsm command to
identify tapes that are not full:

LIST TTOC SELECT(NOTFULL)

All of the tapes in this list must be marked full with these DFSMShsm commands, as shown in
Example 7-11.

_Example 7-11  DFSMShsm command for marking volumes full_

    DELVOL _volser_ MIGRATION(MARKFULL)
    DELVOL _volser_ BACKUP(MARKFULL)

Consider using the DFSMShsm option **SETSYS PARTIALTAPE MARKFULL** . By using this option,
DFSMShsm assigns a new tape at the next operation instead of retrieving the old tape into
TS7700 cache and requiring a physical mount.


-----

### 7.7.4 The TS7700 and the MOUNTWAITTIME setting

We recommend that you set the mount wait time value in DFSMShsm by using the **SETSYS**
**MOUNTWAITTIME(12)** command. This value allows the virtual tape to be staged into cache in
time. For the setting in the IECIOSxx member in SYS1.PARMLIB, consider the MIH value:
MIH MOUNTMSG=YES, MNTS=10. Selecting this value ensures that mount pending
messages appear. This value can be adjusted to the specific needs of your environment.

### 7.7.5 The TS7700 and scratch volumes

The use of volumes from the global scratch pool is recommended. Also, assign the Fast
Ready category to this scratch pool to ensure fast mount time.

Example 7-12 shows example commands to select these DFSMShsm settings.

_Example 7-12  Selecting DFSMShsm settings for scratch pools_

    SETSYS SELECTVOLUME(SCRATCH)
    SETSYS TAPEDELETION(SCRATCHTAPE)
    SETSYS PARTIALTAPE(MARKFULL)

### 7.7.6 The TS7700 and DFSMShsm DUMP parameters

When you use the TS7700 for the AUTODUMP function, avoid using the parameter settings
that are shown in Example 7-13.

_Example 7-13  AUTODUMP previous recommendation for physical tape usage_

    DEFINE DUMPCLASS(dclass STACK(nn))
    BACKVOL SG(sgname ! VOLUMES(volser) DUMP(dclass STACK(10))

These settings were recommended previously for physical volume mounts to make these
mounts more effective. In a virtual tape environment, specify NOSTACK to prevent unnecessary
multivolume processing. This processing reduces parallelism during a restore.

### 7.7.7 The TS7700 physical limitations

The TS7700 excels at parallel mounts, especially writes. However, if your environment
requires massive parallel recalls of data from the TS7700 back end and that data is no longer
in TS7700 cache, consider your physical setup. Consider the number of available back-end
drives for reading and writing data and potentially reclaiming data at the same time. These
considerations apply to the number of parallel tasks in DFSMShsm that relate to backup,
migration, recall, and recycle.

### 7.7.8 The TS7700 and tape management systems

No special actions are required for DFSMShsm in relation to a tape management system
when you migrate to a TS7700 GRID.

For DFSMSrmm, you might need to change the VRSs and vault rules to reflect a virtual tape
environment.


-----

## 7.8 Preparing for a Virtual Tape Server or an ATL

The implementation of virtual tape and SMS-managed tape libraries involves several tasks.
These tasks are fully described in the _DFSMS Object Access Method Planning, Installation,_
_and Storage Administration Guide for Tape Libraries,_ SC35-0427. To implement DFSMShsm
functions in an SMS-managed tape environment, follow these steps:

- Determine the tape functions that you want to process in a VTS/tape library.
- Set up a global scratch pool or DFSMShsm-owned pool.
- Define or update a data class to compact tape data.
- Define or update a storage class to enable a storage group.
- Define or update a storage group to associate tape devices with the library.
- Set up or update ACS routines to filter data sets to the library.
- Prepare DFSMShsm for using tape (ARCCMDxx PARMLIB member updates).

## 7.8.1 Determine the functions to process in a VTS/tape library

VTS and tape libraries can process any DFSMShsm tape functions. You must decide which
DFSMShsm functions to process in an automated tape system. Each DFSMShsm function
uses a unique tape data set name. The ACS routine can recognize the functions that you
want to process in an SMS-managed tape library by the data set names.

The assignment of classes in the ACS routines can be based on a data set naming
convention (the DFSMShsm standard usage of data set names for backup and migration), as
shown in Table 7-5.

_Table 7-5  Overview of DFSMShsm data set naming conventions that relate to individual functions_


|DFDSMhsm function|Tape data set names|Commands with unit type restrictions|
|---|---|---|
|Back up to original|prefix.BACKTAPE.DATASET|SETSYS BACKUP(TAPE(unittype))|
|Back up to alternate|prefix.COPY.BACKTAPE.DATASET||
|RECYCLE of backup tapes to ORIGINAL|prefix.BACKTAPE.DATASET|SETSYS RECYCLEOUTPUT(BACKUP(unittype))|
|RECYCLE of backup tapes to alternate|prefix.COPY.BACKTAPE.DATASET||
|MIGRATION to original|prefix.HMIGTAPE.DATASET|SETSYS TAPEMIGRATION(- DIRECT(TAPE(unittype)) ML2TAPE(TAPE(unittype)) NONE(ROUTETOTAPE(unittype))|
|MIGRATION to alternate|prefix.COPY.HMIGTAPE.DATASET||
|RECYCLE of MIGRATION tapes to original|prefix.HMIGTAPE.DATASET|SETSYS RECYCLEOUTPUT(MIGRATION(unittype))|
|RYCLE of MIGRATION tapes to alternate|prefix.COPY.HMIGTAPE.DATASET||
|DUMP|prefix.DMP.dclassVvolser.Dyyddd.Ts smmhh|DEFINE DUMPCLASS(class UNIT(unittype))|
|SPILL|prefix.BACKTAPE.DATASET|SETSYS SPILL(TAPE(unittype))|


-----

|DFDSMhsm function|Tape data set names|Commands with unit type restrictions|
|---|---|---|
|TAPECOPY of BACKUP tapes|prefix.COPY.BACKTAPE.DATASET|TAPECOPY ALTERNATEUNITNAME(unittype1, unittype2)) TAPECOPY ALTERNATE3590UNITNAME (unittype1,unittype2)|
|TAPECOPY of MIGRATION tapes|prefix.COPY.HMIGTAPE.DATASET||
|CDS BACKUP Datamover=HSM|uid.BCDS.BACKUP.Vnnnnnn uid.MCDS.BACKUP.Vnnnnnn uid.OCDS.BACKUP.Vnnnnnn uid.JRNL.BACKUP.Vnnnnnn|SETSYS CDSVERSIONBACKUP(BACKUPDEVICEC ATEGORY(TAPE UNITNAME(unittype)))|
|CDS BACKUP Datamover=DSS|uid.BCDS.BACKUP.Dnnnnnn uid.MCDS.BACKUP.Dnnnnnn uid.OCDS.BACKUP.Dnnnnnn uid.JRNL.BACKUP.Dnnnnnn||
|ABARS processing|outputdatasetprefix.C.Go0Vnnnn outputdatasetprefix.D.Go0Vnnnn outputdatasetprefix.U.Go0Vnnnn outputdatasetprefix.O.Go0Vnnnn|ABACKUP AGNAME UNIT(unittype)|


## 7.9 Setting up a global scratch pool

The recommended setup for DFSMShsm scratch tape usage is by using a global scratch pool
that is managed by a tape management product, such as DFRMM.

The assignment of a common scratch pool from a DFSMShsm perspective is based on three
**SETSYS** parameters:

- The **SELECTVOLUME** parameter decides whether this mount is a specific or non-specific
scratch mount. This parameter is valid for backup, migration, and dump processing.

- The **PARTIALTAPE** parameter specifies whether to mark migration or backup volumes full,
and consequently if a mount occurs on a tape that is partially used or a new scratch tape.

- For the **TAPEDELETION** parameter, this setting determines whether to return the tape to a
DFSMShsm-owned pool or global scratch pool after expiration.

**Note:** For DUPLEX tapes, writes always occur to a scratch tape.

Example 7-14 shows an example of setting up this global scratch pool.

_Example 7-14  Example of setting up a global scratch pool for DFSMShsm_

    SETSYS SELECTVOLUME(SCRATCH)
    SETSYS TAPEDELETION(SCRATCHTAPE)
    SETSYS PARTIALTAPE(MARKFULL)

A _global scratch pool_ is a pool of tapes that are not defined to DFSMShsm. A global scratch
pool can be used by all tasks. When a tape is assigned by a task, it enters PRIVATE status
until it is returned to the global scratch pool.


-----

When DFSMShsm requests a scratch tape to be mounted, it creates a record in the offline
control data set (OCDS). The record is stored by DFSMShsm in the OCDS during the
records’ use by DFSMShsm. Then, the record is deleted, and the tape returns to the global
scratch pool. When a scratch tape from the global scratch pool is used by DFSMShsm, its
status is changed to PRIVATE. Global scratch pools work well with tape management
systems, such as DFSMSrmm.

## 7.10 Setting up a DFSMShsm-owned scratch pool

A specific scratch pool consists of tapes that are predefined to DFSMShsm (or another
specific user) and can in this case be used only by DFSMShsm. These tapes are manually
added to DFSMShsm with the **ADDVOL** command. For example, to add tape POK001 for use
by DFSMShsm started procedure HSM1 as a backup tape in an ATL, issue the following
command. This command adds a DFSMShsm-owned tape:

`F HSM1, ADDVOL POK001 BACKUP UNIT(3590-1)`

We recommend that you use global scratch pools. Output processing in a global scratch pool
environment calls for any scratch cartridge that reduces the mount wait time, especially when
you use devices with automatic cartridge loaders or an ATL.

If specific tasks and applications use their own pools, overhead is created and the number of
available scratch tapes that are needed increases.

To return DFSMShsm-owned tapes to a global scratch pool, the empty tapes must be
identified. (Use the **LIST TTOC SELECT(EMPTY)** command.) Next, these volumes must be
deleted from DFSMShsm through the **DELVOL** command.

## 7.11 Preparing for SMS-managed tape

The tasks to implement SMS-managed tape are described.

### 7.11.1 Defining a storage class

A storage class needs to be assigned to DFSMS-managed tape allocations. No availability
attributes are needed. In a virtual tape environment (VTS), improved cache management is
provided through your installation’s ACS routines to select a cache preference group of 0 or 1.

You can use the storage class initial access response time ( **IART** ) parameter at the host to
select the preference group. If the value that is specified in this parameter is greater than or
equal to 100, the logical volume is associated with cache preference group 0. If the value that
is specified is less than 100, the logical volume is associated with cache preference group 1,
which is also the default.

When space is needed in the cache, logical volumes that are associated with preference
group 0 will be removed from the cache before logical volumes that are associated with
preference group 1. Volumes are removed from preference group 0 based on their size, with
the largest volumes removed first.


-----

Volumes continue to be removed from preference group 1 based on a least recently used
(LRU) algorithm. Data that is written to the VTS for backup or long-term archival can assign a
storage class that specifies an initial access response time parameter that is greater than or
equal to 100.

It makes sense to use preference groups and differ between the DFSMShsm creation of
backup and migration tapes. Give DFSMShsm backup tapes a preference group of 0 because
the probability of reusing these tapes is lower than for migration tapes.

Outboard policy management on the tape library with equally named storage classes ensures
that the function is also enabled in the library. Any value in the library setting will override the
storage class setting in DFSMS.

Figure 7-13 and Figure 7-14 show a display of a storage class that is used for
tape. Two panels are involved and no definitions were added except for defaults.

    Panel Utilities Help
    ssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssss
    STORAGE CLASS APPLICATION SELECTION
    Command ===>
    
    To perform Storage Class Operations, Specify:
    CDS Name . . . . . . . 'SYS1.SMS.MHLRES3.SCDS'
    (1 to 44 character data set name or 'Active' )
    Storage Class Name . . SCLIB2  (For Storage Class List, fully or
    partially specified or * for all)
    Select one of the following options :
    3 1. List     - Generate a list of Storage Classes
    2. Display    - Display a Storage Class
    3. Define    - Define a Storage Class
    4. Alter     - Alter a Storage Class
    5. Cache Display - Display Storage Classes/Cache Sets
    6. Lock Display - Display Storage Classes/Lock Sets
    If List Option is chosen,
    Enter "/" to select option   Respecify View Criteria
    Respecify Sort Criteria
    If Cache Display is Chosen, Specify Cache Structure Name . .
    If Lock Display is Chosen, Specify Lock Structure Name . . .
    Use ENTER to Perform Selection; . . . . . . . . . . . . . .
    
_Figure 7-13  Storage class definition (Page 1 of 2)_


-----

    Panel Utilities Scroll Help
    ssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssss
    STORAGE CLASS DEFINE       Page 1 of 2
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Class Name : SCLIB3
    To DEFINE Storage Class, Specify:
    Description ==>
    ==>
    Performance Objectives
    Direct Millisecond Response . . . .       (1 to 999 or blank)
    Direct Bias . . . . . . . . . . . .       (R, W or blank)
    Sequential Millisecond Response . .       (1 to 999 or blank)
    Sequential Bias . . . . . . . . . .       (R, W or blank)
    Initial Access Response Seconds . .       (0 to 9999 or blank)
    Sustained Data Rate (MB/sec) . . .       (0 to 999 or blank)
    OAM Sublevel . . . . . . . . . . .       (1, 2 or blank)
    Availability . . . . . . . . . . . . N      (C, P ,S or N)
    Accessibility . . . . . . . . . . . N      (C, P ,S or N)
    Backup . . . . . . . . . . . . . .       (Y, N or Blank)
    Versioning . . . . . . . . . . . .       (Y, N or Blank)
    Use ENTER to Perform Verification; Use DOWN Command to View next Page;
    
_Figure 7-14  Storage class definition panel (Page 2 of 2)_

### 7.11.2 Defining a data class

The data class construct can provide the tape devices and media type information for tape
data sets. The following attributes can be specified in a data class construct:

- The type of media to use
- Whether the data is to be compacted
- Recording technology (18 track, 36 track, 128 track, or 256 track)

Different tape devices and different media types can coexist at your installation, so you might
need to assign specific data classes for the selection of specific devices and tape media. To
define a data class, go to the ISMF Primary Option menu and enter option 4 .

Figure 7-15 through Figure 7-18 show the data class definition
panels.


-----

    Panel Utilities Help
    ssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssss
    DATA CLASS APPLICATION SELECTION
    Command ===>
    
    To perform Data Class Operations, Specify:
    CDS Name . . . . . . 'SYS1.SMS.MHLRES3.SCDS'
    (1 to 44 character data set name or 'Active' )
    Data Class Name . . DCTAPE  (For Data Class List, fully or partially
    specified or * for all)
    
    Select one of the following options :
    3 1. List  - Generate a list of Data Classes
    2. Display - Display a Data Class
    3. Define - Define a Data Class
    4. Alter  - Alter a Data Class
    
    If List Option is chosen,
    Enter "/" to select option   Respecify View Criteria
    Respecify Sort Criteria
    
    Use ENTER to Perform Selection; . . . . . . .

_Figure 7-15  Data Class Application Selection panel in ISMF_

    Panel Utilities Scroll Help
    ssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssss
    DATA CLASS DEFINE         Page 1 of 5
    Command ===>
    
    SCDS Name . . . : SYS1.SMS.MHLRES3.SCDS
    Data Class Name : DCTAPE
    
    To DEFINE Data Class, Specify:
    Description ==> Data class for tape
    ==>
    Recfm . . . . . . . . . .       (any valid RECFM combination or blank)
    Lrecl . . . . . . . . . .       (1 to 32761 or blank)
    Override Space . . . . . N      (Y or N)
    Space Avgrec . . . . . .       (U, K, M or blank)
    Avg Value . . . . .       (0 to 65535 or blank)
    Primary . . . . . .       (0 to 999999 or blank)
    Secondary . . . . .       (0 to 999999 or blank)
    Directory . . . . .       (0 to 999999 or blank)
    Retpd or Expdt . . . . .       (0 to 93000, YYYY/MM/DD or blank)
    Volume Count . . . . . . 1      (1 to 255 or blank)
    Add'l Volume Amount . . .       (P=Primary, S=Secondary or blank)
    Use ENTER to Perform Verification; Use DOWN Command to View next Panel;
    
_Figure 7-16  Data Class Define panel (Page 1 of 5)_


-----

    Panel Utilities Scroll Help
    ssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssss
    DATA CLASS DEFINE         Page 2 of 5
    Command ===>   D
    
    SCDS Name . . . : SYS1.SMS.MHLRES3.SCDS
    Data Class Name : DCTAPE
    
    To DEFINE Data Class, Specify:
    
    Data Set Name Type . . . . .    (EXT, HFS, LIB, PDS, Large or blank)
    If Ext . . . . . . . . . .    (P=Preferred, R=Required or blank)
    Extended Addressability . . N   (Y or N)
    Record Access Bias . . . .    (S=System, U=User or blank)
    Space Constraint Relief . . . N   (Y or N)
    Reduce Space Up To (%) . .    (0 to 99 or blank)
    Dynamic Volume Count . . .    (1 to 59 or blank)
    Compaction . . . . . . . . .    (Y, N, T, G or blank)
    Spanned / Nonspanned . . . .    (S=Spanned, N=Nonspanned or blank)
    System Managed Buffering . .       (1K to 2048M or blank)
    System Determined Blocksize  N   (Y or N)
    EATTR . . . . . . . . . . . .    (O=Opt, N=No or blank)
    Use ENTER to Perform Verification; Use UP/DOWN Command to View other Panels;
    
_Figure 7-17  Data Class Define panel (Page 2 of 5)_

    Panel Utilities Scroll Help
    ssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssssss
    DATA CLASS DEFINE         Page 3 of 5
    Command ===>
    
    SCDS Name . . . : SYS1.SMS.MHLRES3.SCDS
    Data Class Name : DCTAPE
    
    To DEFINE Data Class, Specify:
    
    Media Interchange
    Media Type . . . . . . . 2  (1 to 13 or blank)
    Recording Technology . . 256 (18,36,128,256,384,E1,E2-E4,EE2-EE4 or ')
    Performance Scaling . .   (Y, N or blank)
    Performance Segmentation   (Y, N or blank)
    Block Size Limit . . . .       (32760 to 2GB or blank)
    Recorg . . . . . . . . .       (KS, ES, RR, LS or blank)
    Keylen . . . . . . . . .       (0 to 255 or blank)
    Keyoff . . . . . . . . .       (0 to 32760 or blank)
    CIsize Data . . . . . . .       (1 to 32768 or blank)
    % Freespace CI . . . . .       (0 to 100 or blank)
    CA . . . . .       (0 to 100 or blank)
    Use ENTER to Perform Verification; Use UP/DOWN Command to View other Panels;
    
_Figure 7-18  Data Class Define panel (Page 3 of 5)_


-----

The Data Class Define panel fields that affect tape device and media selection are explained.
_Compaction_ specifies whether data compaction needs to be selected for data sets that are
assigned to this data class. We recommend that you always set the compaction to YES , even if
you plan to direct DFSMShsm output tapes to a VTS. Media Type specifies the tape cartridge
media that is used for data sets that are associated with this data class. Capacity on the tape
depends on the tape drive that is used (how many tracks are being written). The following
media type values are valid:

- 1 For MEDIA1 (3490 standard, 400 MB physical; CST)

- 2 For MEDIA2 (3490 enhanced, 800 MB physical; ECCST)

- 3 For MEDIA3 (3590 standard, 10 GB physical; HPCT)

- 4 For MEDIA4 (3590 extended, 20 GB physical; EHPCT)

- 5 For MEDIA5 (3592 Enterprise, 300/500 GB (EFMT1/EFMT2)

- 6 For MEDIA6 (3592 WORM, 300/500 GB (EFMT1/EFMT2)

- 7 For MEDIA7 (3592 Economy, 60/100 GB (EFMT1 / EFMT2)

- 8 For MEDIA8 (3592 Economy WORM, 50/100 GB (EFMT1/EFMT2)

- 9 For MEDIA9 (3592 Extended, 700 GB (EFMT2 only)

- 10 For MEDIA 10 (3592 WORM, 700 GB (EFMT2 only)

- 11 For MEDIA 11 (3592 Enterprise, 4000 GB (EFMT4, EEFMT4)

- 12 For MEDIA 12 (3592 WORM, 4000 GB (EFMT4/EEFMT4), 10,000 GB EFMT5)

- 13 For MEDIA 13 (3592 Advanced Economy, 500 GB (EFMT4/EEFMT4), 2,000 GB
EFMT5)

- Blank cartridge type not specified

Recording Technology specifies the number of tracks on tape cartridges that are used for
data sets that are associated with this data class. Table 7-6 shows the number of tracks that
are written on recent IBM technology.

_Table 7-6  IBM tape drives and support recording technology_

|3502 drive|EFMT1 (512 track)|EFMT2 (896 track)|EFMT3 (1,152 track)|EFMT4 (2,560 track)|EFMT5 (5,120 track)|
|---|---|---|---|---|---|
|Model J1A|R/W|Not supported|Not supported|Not supported|Not supported|
|Model E05|R/W|R/W|Not supported|Nut supported|Not supported|
|Model E06|R|R/W|R/W|Not supported|Not supported|
|Model E07|R|R|R/W|R/W|Not supported|
|Model E08|Not supported|Not supported|Not supported|R/W|R/W|



Figure 7-19 on page 157 shows the panel that is used for setting the key, if encryption is
used.


-----

    DATA CLASS DEFINE         Page 4 of 5
    Command ===>
    
    SCDS Name . . . : SYS1.SMS.MHLRES3.SCDS
    Data Class Name : DCTAPE
    
    To DEFINE Data Class, Specify:
    
    Encryption Management
    Key Label 1 . . .   (1 to 64 characters or blank)
    
    Key Label 2 . . .
    
    Encoding for Key Label 1 . . . . .      (L, H or blank)
    Encoding for Key Label 2 . . . . .      (L, H or blank)
    
    Use ENTER to Perform Verification; Use UP/DOWN Command to View other Panels;
    
_Figure 7-19  Data Class Define panel (Page 4 of 5)_

Figure 7-20 shows the last panel in the Data Class Define application in ISMF. It is shown only
for reference because it contains no attributes that point to tape.

    DATA CLASS DEFINE         Page 5 of 5
    Command ===>
    
    SCDS Name . . . : SYS1.SMS.MHLRES3.SCDS
    Data Class Name : DCTAPE
    
    To DEFINE Data Class, Specify:
    
    Shareoptions Xregion . . .   (1 to 4 or blank)
    Xsystem . . .   (3, 4 or blank)
    Reuse . . . . . . . . . . . N  (Y or N)
    Initial Load . . . . . . . R  (S=Speed, R=Recovery or blank)
    BWO . . . . . . . . . . . .   (TC=TYPECICS, TI=TYPEIMS, NO or blank)
    Log . . . . . . . . . . . .   (N=NONE, U=UNDO, A=ALL or blank)
    Logstream Id . . . . . . .
    FRlog . . . . . . . . . . .   (A=ALL, N=NONE, R=REDO, U=UNDO or blank)
    RLS CF Cache Value . . . . A  (A=ALL, N=NONE, U=UPDATESONLY)
    RLS Above the 2-GB Bar . . N  (Y or N)
    Extent Constraint Removal  N  (Y or N)
    CA Reclaim . . . . . . . . Y  (Y or N)
    Use ENTER to perform Verification; Use UP Command to View previous Panel;
    
_Figure 7-20  Data Class Define panel (Page 5 of 5)_


-----

**Defining storage groups**
The _storage group_ definition in DFSMS is designed to assign tape volumes to a specific
group and a specific library.

On the Storage Group Application Select Enter Storage Group Type panel, enter 3 to create
a tape storage group, as shown in Figure 7-21.

    STORAGE GROUP APPLICATION SELECT ENTER STORAGE GROUP TYPE
    Command ===>
    
    To perform Storage Group Operations, Specify:
    CDS Name . . . . . . 'SYS1.SMS.MHLRES3.SCDS'
    (1 to 44 character data set name or 'Active' )
    Storage Group Name  SGLIB3      (For Storage Group List, fully or
    partially specified or * for all)
    Storage Group Type  tape       (VIO, POOL, DUMMY, COPY POOL BACKUP,
    OBJECT, OBJECT BACKUP, or TAPE)
    Select one of the following options :
    3 1. List  - Generate a list of Storage Groups
    2. Display - Display a Storage Group (POOL only)
    3. Define - Define a Storage Group
    4. Alter  - Alter a Storage Group
    5. Volume - Display, Define, Alter or Delete Volume Information
    
    If List Option is chosen,
    Enter "/" to select option   Respecify View Criteria
    Space Info in GB      Respecify Sort Criteria
    Use ENTER to Perform Selection;

_Figure 7-21  Storage group definition main panel_

The new storage group is related to a library, as shown in Figure 7-22.

    TAPE STORAGE GROUP DEFINE
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : SGLIB3
    
    To DEFINE Storage Group, Specify:
    
    Description ==>
    ==>
    
    Library Names (1 to 8 characters each):
    ===> SCLIB2  ===>      ===>      ===>
    ===>      ===>      ===>      ===>
    
    DEFINE  SMS Storage Group Status . . N (Y or N)
    
    Use ENTER to Perform Verification and Selection;

_Figure 7-22  Storage group definition: Tape Storage Group Define panel_


-----

If you entered Y in the Define SMS Storage Group Status field (Figure 7-22 on page 158),
you enter the information that is shown in Figure 7-23. You can set the storage group status
on individual logical partitions (LPARs).

    SMS STORAGE GROUP STATUS DEFINE     Page 1 of 2
    Command ===>
    
    SCDS Name . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : SGLIB3
    Storage Group Type : TAPE
    To DEFINE Storage Group System/          ( Possible SMS SG Status
    Sys Group Status, Specify:             for each:
    -  Pool SG Type
    System/Sys   SMS SG  System/Sys   SMS SG    NOTCON, ENABLE, DISALL
    Group Name   Status  Group Name   Status    DISNEW, QUIALL, QUINEW
    ----------   ------  ----------   ------  - Tape SG Type
    SC63    ===> ENABLE  SC64    ===> ENABLE    NOTCON, ENABLE,
    SC65    ===> ENABLE  SC70    ===> ENABLE    DISALL, DISNEW
    ===>           ===>     - Copy Pool Backup SG Type
    ===>           ===>       NOTCON, ENABLE )
    ===>           ===>     * SYS GROUP = sysplex
    ===>           ===>      minus Systems in the
    ===>           ===>      Sysplex explicitly
    ===>           ===>      defined in the SCDS
    Use ENTER to Perform Verification; Use DOWN Command to View next Panel;
    
_Figure 7-23  SMS Storage Group Status Define panel_

**ACS routines and tape allocations**
The ACS routines assign a data class to the tape allocations that match the requested device
type and media. If the correct assignment fails, the allocation fails. Example 7-15 shows a
data class ACS routine that assigns a 4 GB tape in a VTS to a DFHSM allocation request for
a backup tape.

_Example 7-15  Assignment of 4 GB DATACLAS for tape allocation_

    FILTLIST DCTAP4GB INCLUDE('JHSMB.BACKTAPE.DATASET' ....
    ----WHEN (&UNIT = &VTS_UNITS && &DSN = &DCTAP4GB) DO
    SET &DATACLAS = 'DCTAP4GB'
    WRITE 'IGDST10I DATA CLASS SET TO DCTAP4GB - VTS'
    EXIT CODE(0)
    END

The storage class assignment in the ACS routine is the point where the allocation is
determined to be SMS-managed or non-SMS-managed and whether to go to DASD or tape.


-----

In Example 7-16, the allocation is assigned to a tape storage class.

_Example 7-16  Assignment of STORCLAS for tape allocation_

    FILTLIST VTS_UNITS INCLUDE('VTS’...
    ----WHEN (&UNIT = &VTS_UNITS) DO
    SET &STORCLAS = 'SCTAPE01'
    WRITE 'IGDST10I STORAGE CLASS SET TO SCTAPE01 - VTS'
    EXIT CODE(0)
    END

The storage group with the correct tape pool must be assigned. Based on the unit name
request in this example, the allocation will target the storage group that is needed for this
allocation, as shown in Example 7-17.

_Example 7-17  Assignment of a storage group in the ACS routine_

    FILTLIST VTS_UNITS INCLUDE('TAPEV','TS77XX')
    -----WHEN (&UNIT = &VTS_UNITS) DO
    SET &STORGRP = 'SGXXXXX'
    WRITE 'IGDST10I STORAGE GROUP SET TO SGXXXXX - PEER-TO-PEER VTS'
    EXIT CODE(0)
    END

The allocation of the tape drive and media is now ongoing. Because this tape allocation is a
DFSMShsm backup tape, this allocation is for the lifetime of the task. When the task ends, the
drive is deallocated.

## 7.12 Preparing DFSMShsm for using tape

When you use DFSMShsm, certain functions, such as backup, ABARS, dump, migration, and
space management, must be activated. Certain details (for example, unit names) must also
be specified. The options that are shown in the sample PARMLIB in Example 7-18 are the
functions that specifically relate to the DFSMShsm use of TAPE.

_Example 7-18  Example of how to prepare DFSMShsm for tape use_

    /***********************************************************************/
    /* SETSYS COMMANDS IN THE ARCCMDXX PARMLIB MEMBER THAT DEFINE THE */
    /* DFSMSHSM ENVIRONMENT FOR AN SMS-MANAGED TAPE LIBRARY. */
    /***********************************************************************/
    /*
    SETSYS DUPLEX(BACKUP MIGRATION)
    SETSYS SELECTVOLUME(SCRATCH)
    SETSYS PARTIALTAPE(MARKFULL)
    SETSYS TAPEDELETION(SCRATCHTAPE)
    SETSYS BACKUP(TAPE)
    SETSYS TAPEMIGRATION(ML2TAPE)
    SETSYS TAPESECURITY(EXPIRATION)
    SETSYS DEFINE DUMPCLASS(ATLHSM NORESET AUTOREUSE NODATASETRESTORE DISPOSITION(’AUTOMATE LOCATION’))
    SETSYS INPUTTAPEALLOCATION(NOWAIT)


-----

    SETSYS OUTPUTTAPEALLOCATION(NOWAIT)
    SETSYS RECYCLETAPEALLOCATION(NOWAIT)
    SETSYS TAPEFORMAT(SINGLEFILE)
    SETSYS TAPEUTILIZATION(PERCENTFULL(97))
    SETSYS TAPEUTILIZATION(LIBRARYBACKUP PERCENTFUL(97))
    SETSYS TAPEUTILIZATION(LIBRARYMIGRATION PERCENTFUL(97))
    SETSYS MOUNTWAITTIME(10)
    SETSYS TAPEHARDWARECOMPACT

With the parameters set for tape, we are ready to begin. This environment will be a duplex
environment that uses a global scratch pool. Whenever a function ends, tapes are marked full
so that the next occurrence of this event will start by using new scratch tapes. When tapes
expire, they will return to the global scratch pool (TAPEDELETION). Backup and ML2 writes
occur on tape. TAPESECURITY is limited to DFSMShsm setting an expiration date on all
tapes, leaving it to the tape management system to handle protection and expiration. A dump
class is assigned. AUTOREUSE returns tapes to the scratch pool without requiring a **DELVOL**
command.

### 7.12.1 Usage of DFSMShsm RECYCLE

RECYCLE is an essential function in DFSMShsm tape maintenance. The constant expiration
of data leaves DFSMShsm backup and migration tapes with unused blocks. When tape
utilization falls under a limit of, for example, 50%, consider the use of RECYCLE to move the
remaining valid data to new tapes that are filled at a higher percentage. This action at the
same time frees DFSMShsm tapes, which can be returned to the global scratch pool.

To prepare for RECYCLE, your DFSMShsm PARMLIB must be updated with a statement that
is similar to the following statement:

SETSYS RECYCLEOUTPUT(BACKUP( _unittype_ ) MIGRATION( _unittype_ ))

Execute RECYCLE according to the needs of your environment, for example, once a week to
multiple times a week. This decision also depends on an adequate window of time.

Typically, RECYCLE will be executed in a batch job that is triggered by your scheduling tool.
Example 7-19 shows an example of a batch scheduled job.

_Example 7-19  Example of a batch scheduled DFSMShsm RECYCLE_

    HSEND RELEASE RECYCLE
    HSEND SETSYS MAXRECYCLETASKS(4)
    HSENDCMD WAIT RECYCLE ML2  EXECUTE PERCENTVALID(50)

RECYCLE is also often used when you move your DFSMShsm tapes to a new tape platform,
for example, from ATL-based tapes to virtual tapes in a TS7700. Example 7-20 shows sample
commands.

_Example 7-20  Using RECYCLE_

    HSEND SETSYS USERUNITTABLE(TS7740 TAPE 3590-1)
    HSEND SETSYS RECYCLEOUTPUT(BACKUP(TS7740) MIGRATION(3590-1))
    HSENDCMD WAIT RECYCLE BACKUP EXECUTE PERCENTVALID(50)


-----

If you use an esoteric that is named 3590-1 in an ATL and you want to move to your TS7740,
you can use the esoteric for the TS7740 to cover the unit addresses that are associated with
the TS7740 environment. In Example 7-20, a controlled RECYCLE from the old
ATL that is based on the esoteric 3590-1 for BACKUP now is recycled into the TS7740.

### 7.12.2 DB2 and DFSMShsm interaction

DFSMShsm can automatically migrate and recall archive log data sets and image copy data
sets. If IBM DB2® needs an archive log data set or an image copy data set that DFSMShsm
migrated, a recall begins automatically and DB2 waits for the recall to complete before DB2
continues.

For processes that read more than one archive log data set, such as the **RECOVER** utility, DB2
anticipates a DFSMShsm recall of migrated archive log data sets. When a DB2 process
finishes reading one data set, it can continue with the next data set without delay because the
data set might already be recalled by DFSMShsm.

If you accept the default value of YES for the **RECALL DATABASE** parameter on the Operator
Functions panel (DSNTIPO), DB2 also recalls migrated table spaces and index spaces. At
data set open time, DB2 waits for DFSMShsm to perform the recall. You can specify the
amount of time DB2 waits while the recall is performed with the **RECALL DELAY** parameter,
which is also on panel DSNTIPO. If RECALL DELAY is set to 0 , DB2 does not wait, and the
recall is performed asynchronously.

You can use SMS to archive DB2 subsystem data sets, including the DB2 catalog, DB2
directory, active logs, and work file databases (DSNDB07 in a non-data-sharing
environment). However, before you start DB2, recall these data sets by using DFSMShsm.
Alternatively, you can avoid migrating these data sets by assigning them to a management
class that prevents migration.

If a volume has a STOGROUP specified, you must recall that volume only to volumes of the
same device type as others in the STOGROUP . In addition, you must coordinate the
DFSMShsm automatic purge period, the DB2 log retention period, and the MODIFY utility
usage. Otherwise, the image copies or logs that you might need during a recovery might
already be deleted by DFSMShsm due to a specification in the management class. No
synchronization exists between DB2 and DFSMShsm for expiration. If a DB2 table space was
expired by DFSMShsm, it still exists in the DB2 catalog.

With the **MIGRATE** , **HMIGRATE** , or **HRECALL** commands, which can specify specific data set
names, you can move data sets from one disk device type to another disk device type within
the same DB2 subsystem. Do not migrate the DB2 directory, DB2 catalog, and the work file
database (DSNDB07). Do not migrate any data sets that are in use frequently, such as the
bootstrap data set and the active log. With the **MIGRATE VOLUME** command, you can move an
entire disk volume from one device type to another device type.

## 7.13 DFSMShsm tape functions

The ways that DFSMShsm creates multiple tape copies are described.


-----

### 7.13.1 Disaster recovery considerations

DFSMShsm includes functions that support disaster recovery. By using the DUPLEX or
TAPECOPY function, you can have your tape copies created either directly in a disaster
recovery location, or you can have your tape copies brought to a disaster recovery location
daily to protect you against a disaster. These DFSMShsm tape copies can be combined with
other backups, dumps, image copies, or ABARS backups.

Today, another popular alternative is to use two-cluster grid tape environments, where the
hardware ensures two immediate copies of all of your tapes or a part of these tapes based on
your policy setting. Additionally, a two-cluster grid also has the COPY EXPORT function to
create a physical tape for recovery in a remote site. Migrating to this type of hardware might
reduce the need for the DFSMShsm tape copy functions.

The concept of disaster recovery is that copies of all of your critical data sets are stored at a
location that is separate from your computer site and a computer system of approximately the
same capabilities will be available to use to resume operations.

You must have both of the following items to provide disaster backup:

- A way to make copies of DFSMShsm-owned tape volumes and to identify the copied
volumes to DFSMShsm.

- A way to cause DFSMShsm to use the copied volumes instead of the original volumes. To
fulfill these needs, DFSMShsm provides the duplex tape option or the **TAPECOPY** command,
and the **TAPEREPL** command.

**Duplex tape and TAPECOPY comparison**
You can use the DUPLEX or the TAPECOPY function in DFSMShsm for an additional copy of
your DFSMShsm tapes.

The major difference between the two options is that the DUPLEX function creates two tapes
concurrently. TAPECOPY runs asynchronously and can run in a window of time that is
convenient for you, based on the availability of tape drives.

**DFSMShsm duplexing**
The DUPLEX tape option provides an alternative to TAPECOPY processing for backup and
migration cartridge-type tapes. Two tapes are created concurrently with one tape that is
designated the original, and the other tape is designated as the alternate. The intent is that
the original tape is kept onsite, and the alternate tape can either be created in a remote tape
library, or taken offsite. If you have enough tape drives, the duplex tape option is
recommended over TAPECOPY.

The option to duplex backup, migration, or both is determined by the **SETSYS** parameters that
are shown in Example 7-21.

_Example 7-21  Activating the DUPLEX function in DFSMShsm_

    SETSYS DUPLEX(BACKUP(Y)
    SETSYS DUPLEX(MIGRATION(Y)
    SETSYS DUPLEX(BACKUP(Y) MIGRATION(Y)

The first command activates duplexing on backup tapes. The next command activates
duplexing for migration. The last command activates duplexing for both. One tape is
designated as the original to stay onsite, and the other tape is the alternate to take offsite or
be created in a remote location.


-----

The **ERRORALTERNATE** keyword can be used with the **DUPLEX** command. This command has two
options, as shown in Example 7-22.

_Example 7-22  Activating the DUPLEX function with the ERRORALTERNATE keyword_

    SETSYS DUPLEX MIGRATION(Y ERRORALTERNATE(CONTINUE))
    STSYS DUPLEX MIGRATION(Y ERRORALTERNATE(MARKFULL))

In the first example, the setting for ERRORALTERNATE is CONTINUE . If an error occurs when
writing to the alternate tape, writing to the primary tape continues and a TAPECOPY is issued
afterward. CONTINUE is also the default on the **ERRORALTERNATE** command.

When the MARKFULL setting is used on the **ERRORALTERNATE** keyword, the function with the
failure is terminated, both tapes are marked full, and the function continues on a new set of
tapes.

Alternate tapes with failures can be listed with the following command:

LIST TTOC SELECT(ERRORALTERNATE)

The pair of tapes maintains the original versus the alternate distinction. The original tape
volume data set names are determined by the function that created the tape.

_Example 7-23  DFSMShsm migration and backup tape data set names_

    prefix.HMIGTAPE.DATASET
    prefix.BACKTAPE.DATASET

The alternate tape volume names use the following format, as shown in Example 7-24.

_Example 7-24  DFSMShsm migration and backup tape data set names on DUPLEX tapes_

    prefix.COPY.HMIGTAPE.DATASET
    prefix.COPY.BACKTAPE.DATASET

We recommend that you use the same format for the original and alternate copy. Because
two copies are created concurrently, you will need at least two output devices for each backup
or migration task that will run, including the RECYCLE function of DFSMShsm. If DUPLEX is
in use for a function, each RECYCLE task needs three tape drives: one for input and two for
output.

The process to select tapes occurs in this order:

1. A partial tape and its alternate are selected.

2. If no partial tapes with alternates are identified, an empty ADDVOL volume and a scratch
volume as an alternate are selected.

3. If no empty ADDVOL volumes are available, two scratch volumes are selected.

**TAPECOPY function in DFSMShsm**
Use the **TAPECOPY** command to make copies of your single-file-format DFSMShsm-owned
cartridge tapes. The copies of the original tape are known as _alternates_ and the terminology
is similar to the terminology that is used in tape duplexing.

The alternate volumes start as scratch volumes to prevent backup, dump, or ML2 tapes from
being used as alternate volumes.


-----

Use the **TAPECOPY** command to copy the categories of tapes that are shown in Example 7-25.

_Example 7-25  Using TAPECOPY to copy single-file-format tapes_

    TAPECOPY BACKUP
    TAPECOPY MIGRATIONLEVEL2
    TAPECOPY ALL

These commands cause copies to be taken of full tapes that do not already have an alternate
copy. You can make copies of all volumes that are categorized as either ML2, backup, or both,
or you can make copies of individual tape volumes. You can issue one **TAPECOPY** command for
each DFSMShsm host on the condition that the requests do not overlap.

For example, you can issue a **TAPECOPY** **ML2** command on one host and a **TAPECOPY BACKUP**
command on another host if the following parameter is already specified in your ARCCMDxx
PARMLIB member:

SETSYS PARTIALTAPE(MARKFULL)

This parameter is the **SETSYS** parameter for copying partially filled tapes.

Nothing else is needed because this command instructs DFSMShsm to mark partially filled
tapes as full at the end of each task. If, instead of MARKFULL , you coded REUSE in your
ARCCMDxx PARMLIB member, you must issue the **SETSYS PARTIALTAPE(MARKFULL)**
command. After a few backup and migration cycles, the volumes will be marked as full.
Alternatively, you can issue one of the commands that are shown in Example 7-26 to mark
the volumes as full.

_Example 7-26  Alternative command to mark volumes full_

    DELVOL _volser_ BACKUP(MARKFULL)
    DELVOL _volser_ MIGRATION(MARKFULL)

The commands that are shown in Example 7-26 mark the volumes that are specified in
_volser_ as full. ML2 tape volumes are not marked as full at the end of data set migration. An
exception exists for ML2 tapes, even in a MARKFULL environment. ML2 tapes are not
marked as full after data set migration for a simple reason. If, for example, command
migration caused one data set to migrate to tape, that tape is an ML2 tape for only one data
set. This situation is not an optimum tape utilization, so ML2 tapes remain as partially filled
until a **TAPECOPY** command is issued for that volume. **TAPECOPY** , in a MARKFULL environment,
marks ML2 volumes as full and makes the copy.

**TAPECOPY** can also make copies of individual volumes, as shown in Example 7-27.

_Example 7-27  Using TAPECOPY to make copies of individual volumes_

    TAPECOPY ORIGINALVOLUMES(ovol1,ovol2,ovoln)
    TAPECOPY ORIGINALVOLUMES(ovol1,ovol2,ovoln) ALTERNATEVOLUMES(avol1,avol2,avoln)

This command causes a copy of the volume or volumes that you specify with the **ovol**
parameters. If you use the first form of the command, a default volume serial number of
PRIVAT is used for the alternate tape volumes. PRIVAT indicates to DFSMShsm that a
scratch volume must be mounted for each alternate tape volume.


-----

However, if you use the second form of the command, DFSMShsm uses the volumes that you
specify with the **ALTERNATEVOLUMES** parameter to become the alternate volumes. If you use the
**ALTERNATEVOLUMES** parameter, you must specify the same number of volumes with the
**ALTERNATEVOLUMES** parameter as you specified with the **ORIGINALVOLUMES** parameter. A
one-to-one correspondence exists between the original volumes and alternate volumes that
you specify; that is, ovol1 is copied to avol1 ; ovol2 is copied to avol2 ; and so on.

The alternate volume serial is recorded in the tape table of contents (TTOC) record of the
original tape. DFSMShsm records only the current copy of the volume in the TTOC record. An
ARC0436I message is issued each time that you make a copy when a copy exists.

Issue the **LIST TTOC** command to get a list of those volumes:

LIST TTOC SELECT(NOALTERNATEVOLUME)

If you output the list to a data set, it can be used directly in the **TAPECOPY** command, by using
the following command:

     TAPECOPY INDATASET( _volcopy.list.dsname_ )

In the previous command, _volcopy.list.dsname_ is the name of the data set, including the list
of volumes you want to create an alternate tape volume for. You can use an edited version of
the output from the **LIST** command to generate this list of volumes.

If a request for TAPECOPY is issued for a specific backup tape and the tape is in use, the
TAPECOPY operation fails. The system issues message ARC0425I, which indicates that the
tape is in use.

The TAPECOPY process with either the **ALL** or **BACKUP** keyword copies full tapes only. If you
want copies of partial backup tapes, use the following process:

- Issue one of the **HOLD** commands to stop backup for all hosts that are running backup. The
following **HOLD** commands cause the data set backup tasks with mounted tapes to
unmount the tapes at the end of the current data set:

  - HOLD BACKUP holds all backup functions.
  - HOLD BACKUP(DSCOMMAND) holds data set command backup only.
  - HOLD BACKUP(DSCOMMAND(TAPE)) holds data set command backup to tape only.

- A LIST TTOC SELECT(BACKUP NOTFULL) lists the tapes that are not full. This list also
shows which tapes do not have an alternate tape.

- A DELVOL MARKFULL is used for those tapes that you do not want extended after the
TAPECOPY is made.

- A TAPECOPY OVOL( _volser_ ) is used for those tape volsers that do not have an alternate
volume that is indicated in the LIST output.

- When the copies are complete, the corresponding **RELEASE BACKUP** command issue option
can be used (on each host that was held) to allow the backup to be _usable._

**For TAPECOPY command or DUPLEX option**
If you are copying the contents of one tape to another tape with the **TAPECOPY** command or are
using the concurrent creation option DUPLEX, you need to be aware of minor inconsistencies
that can exist in the length of cartridge-type tapes.

Because the **TAPECOPY** command copies the entire contents of one tape to another tape, it is
important that enough media is available to copy the entire source tape to its target.
Therefore, when you are copying tapes with the **TAPECOPY** command, use the default options
(the equivalent of specifying the **TAPEUTILIZATION** command with the **PERCENTFULL** option of
97%). DFSMShsm marks the end of the volume when tapes are 97% full.


-----

When you use the duplex option, it is recommended that you use the value 97% to ensure
that you can write the same amount of data to both tapes. During duplexing, the **NOLIMIT**
parameter of **TAPEUTILIZATION** will be converted to the default of 97%. If you are not copying
tapes with the **TAPECOPY** command and you are not creating two tapes by using the DUPLEX
option, you can specify the **TAPEUTILIZATION** command with a **PERCENTFULL** option of 100%.

**DFSMShsm TAPEREPL command**
The **TAPEREPL** command provides a way of replacing an original ML2 volume or a backup
volume with its alternate volume. It also is important during an actual disaster recovery or a
disaster recovery test run because you can replace original volumes with their alternate
volumes.

DFSMShsm provides a way, assuming that you created alternate tapes or duplexed alternate
tapes, to use those alternates in place of the original tapes for recalls or recovers. The
designation of “disaster alternate volumes” helps distinguish between alternate volumes that
are initially present at the recovery site and the alternate volumes that were created later at
the recovery site. You might need to use your alternate volumes on the recovery site in a real
disaster or for a disaster recovery test. To recover the volumes, run the commands that are
shown in Example 7-28.

_Example 7-28  Sample commands to recover volumes_

TAPEREPL ALL DISASTERALTERNATEVOLUMES
SETSYS DISASTERMODE

The **DISASTERALTERNATEVOLUMES** parameter causes each alternate volume to be flagged as a
disaster alternate volume. Only base DFSMShsm offline control data set (OCDS) TTOC
records of ML2 and backup tapes with alternate volumes are updated. The **SETSYS**
**DISASTERMODE** command specifies that DFSMShsm established a tape substitution mode.
While DFSMShsm is running in disaster mode, it checks the TTOC record of the original tape
volume to see whether a disaster alternate volume exists. If so, request the use of it to return
the data. When you finish the use of the disaster alternate volumes, you can reset the flag
with the following command:

`TAPEREPL ONLYDISASTERALTERNATES(RESET)`

If you are recovering from a real disaster and you want to return to your original site, you need
to convert all disaster alternate volumes to original tapes. To convert all disaster alternate
volumes to original tapes, you must issue the following command:

`TAPEREPL ONLYDISASTERALTERNATES`

**Using DFSMShsm alternate tapes with DFSMSrmm**
Previously, we showed you how to create alternate tapes for DFSMShsm-owned tape
volumes. The major use for these alternate tapes is at a remote site in a disaster. The
following process applies for using alternate volumes as disaster alternate volumes:

- Have a copy of the DFSMShsm CDSs at the disaster site.

- Perform a **TAPEREPL** command that specifies the **DISASTERALTERNATE** parameter. This
command flags each existing alternate tape as a disaster alternate.

- Place DFSMShsm in DISASTER mode. When in DISASTER mode, DFSMShsm
dynamically checks before it mounts an input tape whether the needed data resides on a
tape with a disaster alternate. If the needed data is on a tape with a disaster alternate, the
disaster alternate is requested.


-----

DFSMSrmm recognizes when DFSMShsm is opening tape data sets and tolerates the data
set names that DFSMShsm uses on condition that the last 17 characters of the data set name
match. When you perform a true replacement by using the **TAPEREPL** command without the
**DISASTERALTERNATE** keyword, DFSMShsm uses the data set name that it uses for the original
tapes.

**Set up ABARS processing**
The aggregate backup and recovery support (ABARS) function creates the following files on
tape volumes, which can then be transported to the recovery site:

- Data file
- Control file
- Instruction/activity log file (optional)

The output files are physical sequential data sets. Each of these files must be written to a
separate physical tape volume (or volumes because the files can extend over several
volumes). The size of the data file can be large, which makes it a good candidate for 3590
cartridges. The control and instruction files tend to be smaller, so you might want to write
them on smaller capacity media. However, with ABARS, you cannot specify the output unit
individually for these files. The unit name that is specified in the **ABACKUP** command applies to
all three files.

To allocate the aggregate backup and ABARS files to 3590 in 3490 emulation mode devices,
issue the following command:

ABACKUP aggregatename EXECUTE UNIT(EMUL3490)

EMUL3490 is the esoteric unit name for our IBM 3590 in 3490 emulation mode. No additional
definitions need to be included in DFSMSrmm. If volume pools are not already defined to
DFSMSrmm, you must define a new volume pool by adding the following definition in the
EGDRMMxx member of PARMLIB, as shown in Example 7-29.

_Example 7-29  Defining a new volume pool_

    VPOOL PREFIX(M*0 TYPE(SCRATCH) MEDIANAME(3590) DESCRIPTION(‘3590 SCRATCH POOL’) RACF(N) EXPDTCHECK(N)
    
Then, use the **ADDRACK** and **ADDVOLUME** DFSMSrmm subcommands to define the new volumes
in your system, as shown in Example 7-30.

_Example 7-30  Using ADDRACK and ADDVOLUME to define new volumes_

    RMM ADDRACK M00000 COUNT(100)
    RMM ADDVOLUME M00000 COUNT(1000 POOL(M*)STATUS(SCRATCH) INIT(Y) MEDIANAME(3590)

**ABARS and DFRMM**
ABARS performs data backup and recovery processes on a predefined set of data, which is
called an _aggregate_ . During backup processing, the data is packaged as a single entity in
preparation for taking it off-site, which enables the recovery of individual applications in
user-priority sequence. In most cases, the output of ABACKUP is directed to tape media
because it is easily transported from one location to another. With this ease of movement
comes the task of managing the tape contents, location, and security.


-----

Many sites manage tapes with a tape management system (DFSMSrmm, for example) and
DFSMShsm. A tape begins as a scratch tape. The tape is used by DFSMShsm to store data
and is returned to the tape management system to be reused as a scratch tape. To implement
this form of concurrent tape management, communications must be coordinated whenever
you define the environment and data sets for the use of a tape management system. The
following example shows how to create a FACILITY class profile for the ABARS user ID, which
in our case is abarsid .

RDEFINE FACILITY STGADMIN.EDG.OWNER.abarsid

Next, give user ID abarsid access to this profile and additional DFSMSrmm profiles that must
be already defined on a DFSMSrmm system, as shown in Example 7-31.

_Example 7-31  Giving the abarsid ID access to profiles_

    PE STGADMIN.EDG.OWNER.abarsid CLASS(FACILITY) ID(abarsid) ACCESS(UPDATE)
    PE STGADMIN.EDG.RELEASE CLASS(FACILITY) ID(abarsid) ACCESS(READ)
    PE STGADMIN.EDG.MASTER CLASS(FACILITY) ID(abarsid) ACCESS(READ)

**Setting up DFSMShsm to use WORM output tapes for ABACKUP**
In an SMS tape environment, and optionally in a non-SMS tape environment, the SMS data
class construct can be used to select Write Once Read Many (WORM) tapes for ABACKUP
processing. The output data set prefix that is specified in the aggregate group definition can
be used by the ACS routines to select a WORM data class.

Set up the ACS routine and the output data set name to uniquely identify the ABARS output
files that must go to WORM tape. In a non-SMS tape environment, the default allows tape
pooling to determine whether ABARS data sets go to WORM or R/W media. Optionally, if the
DEVSUP _xx_ parameter, **ENFORCE_DC_MEDIA=ALLMEDIATY** or **ENFORCE_DC_MEDIA=MEDIA5PLUS** , is
used, the data class must request the appropriate media type to be successfully mounted.

**Reducing the number of partially full tapes**
When an ML2 tape that is used as a migration or recycle target is needed as input for a recall
or ABACKUP , the requesting function can take the tape away whenever migration or recycle
completes processing its current data set. When recall or ABACKUP is finished with a partial
tape, DFSMShsm leaves it available as a migration or recycle target again, regardless of the
PARTIALTAPE setting.

During the initial selection for an ML2 tape for output from a migration or recycle task,
DFSMShsm tries to select the partial tape with the highest percentage of written data. An
empty tape is selected only if no partial tapes are available. Three options for the **LIST TTOC**
command are related:

- **SELECT(RECALLTAKEAWAY)** lists only those ML2 tapes that are taken away by recall. (If this
list is too extensive, tune your ML2 tape migration criteria.)

- **SELECT(ASSOCIATED)** lists only those ML2 partial tapes that are currently associated with
migration or recycle tasks as targets on this host or any other in the HSMplex (two or more
hosts that are running DFSMShsm that share a common migration control data set
(MCDS), OCDS, backup control data set (BCDS), and journal).


-----

- **SELECT(NOTASSOCIATED)** lists only those ML2 partial tapes that are not associated with
migration or recycle tasks as targets. The **RECYCLE** command for ML2 can recycle several
of these partial tapes. By using the **SETSYS** **ML2PARTIALSNOTASSOCIATEDGOAL** parameter,
you can control the trade-off between a few underused ML2 tapes versus the time that is
needed to recycle them. If the number of unassociated ML2 partial tapes in the HSMplex
that meets the PERCENTVALID and SELECT criteria for recycle exceeds the number that
is specified by the **SETSYS** parameter, recycle includes enough partial tapes to reduce the
number of partial tapes to the specified number. Recycle selects those tapes to process in
the order of least amount of valid data to most amount of valid data.

**Tape encryption on the TS7740**
The importance of data protection is increasingly apparent with news reports of security
breaches, loss, theft of personal and financial information, and government regulation.
Encrypting the stacked cartridges minimizes the risk of unauthorized data access without
excessive security management burdens or subsystem performance issues. The encryption
solution for tape virtualization consists of several components:

- The encryption key manager
- The TS1120 and TS1130 encryption-enabled tape drives
- The TS7740 Virtualization Engine

You still see quite a few virtual tape environments that are not encrypted because of the
perception that tapes are not brought outside of the physical robotics or outside of the data
center. This scenario does not apply to certain tapes, such as Copy Export tapes. You can
encrypt select tapes only, such as in this exception. Always plan for a data center relocation
project with decrypted tapes. Is it necessary to encrypt tapes before a data center relocation?

For DFSMShsm, consider the following types of tapes that might be carried outside of the
data center (and consequently might require encryption):

- Dump tapes
- ABARS tapes
- Duplex copies
- Tape backups of CDSs

**Encryption key manager**
The TS7700 Virtualization Engine can use one of the following encryption key managers:

- Encryption Key Manager (EKM)
- IBM Security Key Lifecycle Manager (formerly Tivoli Key Lifecycle Manager (TKLM))

In this book, we use _key manager_ for EKM, TKLM, or Security Key Lifecycle Manager. The
key manager is the central point from which all encryption key information is managed and
served to the various subsystems. The key manager server communicates with the TS7740
Virtualization Engine and tape libraries, control units, and open systems device drivers.

**Prerequisites for tape encryption in the TS7740**
The IBM TS1120 and TS1130 tape drives provide hardware that performs the cryptography
function without reducing the data transfer rate.

The TS7740 provides the means to manage the use of encryption and the keys that are used
on a storage pool basis. It also acts as a proxy between the tape drives and the key manager
servers, by using redundant Ethernet to communicate with the key manager servers and
Fibre Channel connections to communicate with the drives. Encryption must be enabled in
each of the tape drives. Encryption on the TS7740 is controlled on a storage pool basis.


-----

The storage group DFSMS construct that is specified for a logical tape volume determines the
storage pool that is used for the primary and optional secondary copies in the TS7740
Virtualization Engine. The storage pools were originally created for management of physical
media. They are enhanced to include encryption characteristics. Storage pool encryption
parameters are configured through the TS7740 Virtualization Engine management interface
under physical volume pools.

For encryption support, all drives that attach to the TS7740 Virtualization Engine must be
encryption-capable and encryption must be enabled. If the TS7740 Virtualization Engine uses
TS1120 Tape Drives, they must also be enabled to run in their native E05 format. The
management of encryption is performed on a physical volume pool basis. Through the
management interface, one or more of the 32 pools can be enabled for encryption. Each pool
can be defined to use specific encryption keys or the default encryption keys that are defined
at the key manager server:

- Specific encryption keys

  Each pool that is defined in the TS7740 Virtualization Engine can have its own unique
encryption key. As part of enabling a pool for encryption, enter two key labels for the pool
and an associated key mode. The two keys might or might not be the same. Two keys are
required by the key manager servers during a key exchange with the drive. A key label can
be up to 64 characters. Key labels do not have to be unique per pool: The management
interface provides the capability to assign the same key label to multiple pools. For each
key, a key mode can be specified. The supported key modes are _Label_ and _Hash_ . As part
of the encryption configuration through the management interface, you provide IP
addresses for a primary and an optional secondary key manager.

- Default encryption keys

  The TS7740 Virtualization Engine encryption supports the use of a default key. This
support simplifies the management of the encryption infrastructure because no future
changes are required at the TS7740 Virtualization Engine. After a pool is defined to use
the default key, the management of encryption parameters is performed at the key
manager:

  -  Creation and management of encryption certificates
  -  Device authorization for key manager services
  -  Global default key definitions
  -  Drive-level default key definitions
  -  Default key changes as required by security policies

**Encryption processing**
For logical volumes that contain data to encrypt, host applications direct them to a specific
pool that is enabled for encryption by using the storage group construct name. All data that is
directed to a pool that is enabled for encryption will be encrypted when it is pre-migrated to
the physical stacked volumes or reclaimed to the stacked volume during the reclamation
process. The storage group construct name is bound to a logical volume when it is mounted
as a scratch volume. Through the management interface, the storage group name is
associated with a specific pool number.

When the data for a logical volume is copied from the tape volume cache to a physical volume
in an encryption-enabled pool, the TS7740 Virtualization Engine determines whether a new
physical volume needs to be mounted. If a new cartridge is required, the TS7740
Virtualization Engine directs the drive to use encryption during the mount process. The
TS7740 Virtualization Engine also provides the drive with the key labels that are specified for
that pool.


-----

When the first write data is received by the drive, a connection is made to a key manager and
the necessary key to encrypt is obtained. Physical scratch volumes are encrypted with the
keys in effect at the time of first write to beginning of tape (BOT). Any partially filled physical
volumes continue to use the encryption settings in effect at the time the tape was initially
written from BOT. The encryption settings are static until the volumes are reclaimed and
rewritten again from BOT.

**TS7740 communication with key manager**
Communicating with the key manager is through the same Ethernet interface that is used to
connect the TS7740 Virtualization Engine to your network for access to the management
interface. The request for an encryption key is directed to the IP address of the primary key
manager. Responses are passed through the TS7740 Virtualization Engine to the drive. If the
primary key manager did not respond to the key management request, the optional
secondary key manager IP address is used.

After the TS1120 or TS1130 drive completes the key management communication with the
key manager, it accepts data from the tape volume cache. When a logical volume needs to be
read from a physical volume in a pool that is enabled for encryption, either as part of a recall
or reclamation operation, the TS7740 Virtualization Engine uses the key manager to obtain
the necessary information to decrypt the data. The affinity of the logical volume to a specific
encryption key or the default key can also be used as part of the search criteria through the
TS7700 Virtualization Engine management interface.

**DFSMShsm considerations for the TS1130 tape drive**
DFSMShsm, a z/OS functional component, automatically manages low activity and inactive
data in both system-managed and non-system-managed environments. DFSMShsm also
provides automatic backup and recovery of active data in those environments. DFSMShsm
can use the TS1130 tape drive (3592 Model E06, 3592-3E) for all functions. DFSMShsm
normally uses non-WORM media (MEDIA5, MEDIA7, and MEDIA9) for non-ABARS
functions. DFSMShsm uses all media, including WORM (MEDIA6, MEDIA8, and MEDIA10)
for ABARS processing. DFSMShsm can use the WORM media for non-ABARs processing if
the WORM media is specifically allowed for non-ABARs processing by your installation.

**Modifying your dump classes**
The method for requesting encryption depends on whether you plan to use hardware
encryption or host-based encryption:

- To request hardware encryption for a dump class, specify it in the SMS data class for the
dump data.

- To request host-based encryption for a dump class, use the DFSMShsm **DEFINE**
**DUMPCLASS(ENCRYPT)** command. With **ENCRYPT** , include the **RSA** or **KEYPASSWORD**
subparameters to specify the type of host-based encryption. **ENCRYPT(NONE)** specifies
host-based encryption will not occur.

If your dump classes are currently defined to use host-based encryption (and possibly
host-based compression before encryption), it is recommended that you remove the
host-based encryption requests from any dump classes for which you plan to use tape
hardware encryption.

During the process of migrating your dump classes to use hardware encryption, certain dump
classes might still be defined to use host-based encryption, while their associated SMS data
classes are defined to use tape hardware encryption. Here, DFSMSdss ignores requests for
host-based encryption for these tape volumes and, instead, uses hardware encryption.


-----

With this processing, you can complete the migration to hardware encryption without needing
to modify your dump-requesting jobs. However, removing host-based encryption requests
from a dump class when tape hardware encryption is also requested can avoid confusion
about which process is active.

**Tips:** To determine whether hardware encryption or host-based encryption was used for a
particular tape volume, check the associated dump volume record (DVL).

If more than one dump class is specified (creating more than one dump copy), those dump
classes specify host-based encryption. If each dump class has a unique data class
assigned, and certain, but not all of the associated data classes request tape hardware
encryption, all dump copies will fail. Tape hardware encryption can override host-based
encryption for all dump classes that are associated with a source volume or none of the
dump classes, but it cannot override a subset of those dump classes.

**Reducing recall processing time**
In a multihost environment, DFSMShsm can experience delays during recall processing when
the **SETSYS TAPERECALLLIMITS** command is not specified. This optional parameter specifies
how long a single tape can remain mounted before DFSMShsm checks to see whether higher
priority recall requests are waiting to process from the same system or another host. The
syntax of the **TAPERECALLLIMITS** command is shown:

`SETSYS TAPERECALLLIMITS(TASK( _time_ ) TAPE( _time_ ))`

If this command is not specified, DFSMShsm uses the default values for this parameter, as
shown:

`SETSYS TAPERECALLLIMITS(TASK(15) TAPE(20))`

The optional parameters of the **TAPERECALLLIMITS** command are explained in detail:

- **TASK:** This parameter specifies the number of minutes that a recall task is
active on a single tape mount before DFSMShsm checks for any
higher priority recalls on that host. The maximum number of tape
recall tasks must be active. If a higher priority task is identified,
DFSMShsm completes its current recall task, dismounts the tape, and
processes the new higher priority task.

- **TAPE:** This parameter specifies the number of minutes that a recall task is
active on a single tape mount before additional multihost criteria are
considered. If a higher priority task is identified on another
DFSMShsm host, DFSMShsm completes the current recall task and
dismounts the tape for use on the other host.

After this action is performed, DFSMShsm excludes any recall request from that tape to allow
the host system time to try its delayed requests again and catch up.

So in the default setting of TASK(15), DFSMShsm processes recalls from a single tape mount
for 15 minutes before it checks for higher priority tasks on that system, which it will honor if it
identifies any.

In the default setting of TAPE(20), DFSMShsm processes recalls from a single tape mount for
20 minutes before it checks for higher priority tasks on another host, which it will honor if it
identifies any.

The following message indicates that TAPERECALLLIMITS is in effect:

ARC0312I RECALLS FROM TAPE volser TERMINATED FOR USAGE BY HOST h


-----

In this message, HOST _h_ identifies the host with the highest priority recall request.

When the recall processing stops for a specific recall task on a DFSMShsm host, all queued
recall requests on that host that require the dismounted tape volume are excluded from
consideration for 5 minutes. This period allows other hosts an opportunity to try their delayed
requests again for the dismounted tape volume.

When DFSMShsm processes multiple recall requests that use one tape mount, the recall task
processes the highest priority tape recall request that requires the currently mounted tape. If
DFSMShsm forces a dismount of the tape volume, the task and the tape are freed to process
other higher priority recalls.

The **TAPERECALLLIMITS** parameter means the same when it is applied to SMS-managed or
non-SMS-managed DASD volumes or data sets _._

**Notes:** TAPE( _time_ ) is longer than TASK( _time_ ) because TAPE( _time_ ) requires delaying
additional queued requests.

The storage administrator can disable the TAPERECALLLIMIT function by setting both the
TASK time and TAPE time to large values, such as 1000 or even 65535 minutes.

**Optimizing tape usage**
In an average installation, 50% of tape data sets contain inactive data (data that is written
once and never read again). Most inactive data sets are point-in-time backups, which are
application-initiated data set backups that are used only if an application or system failure
occurs. Although data set backup is critical, application-initiated backups to tape use the tape
media and subsystem poorly and require many costly operator tape mounts. The remaining
tape data sets, active data, cause the same inefficiencies in tape cartridge use. However, they
are usually not active; 60% - 90% of all tape data sets have all accesses on the same
calendar date.

You can use system-managed storage facilities to optimize management of both inactive and
active tape data sets. Tape mount management helps you understand your tape workload
and use advanced tape hardware and DFSMS facilities to provide the following benefits:

- Reduce tape mounts. Data sets that are good candidates to tape mount management are
written to a system-managed DASD buffer instead of a tape. They are then written to tape
by DFSMShsm with other data sets, and they are automatically retrieved by DFSMShsm if
they are accessed.

- Reduce tape library inventory and maximize media usage. Tape mount management
candidates are written by DFSMShsm to tape in single-file, compacted form. DFSMShsm
attempts to fill the entire tape volume before it uses another volume.

- Improve turnaround time for batch jobs, depending on tape data sets. Most tape
processing jobs are queued on tape drives, not mount time or tape I/O because, when the
drive is finally allocated, little data is written to the cartridge. Batch jobs that use data sets
that are written to the system-managed DASD buffer do not wait for tape mounts, and can
perform I/O at DASD or cache speed.

**Tapespansize statement**
In this statement, you specify the maximum number of megabytes of tape that DFSMShsm
can leave unused on tape while it tries to eliminate spanning data sets. When space is
insufficient to contain the entire next data set on the current tape without exceeding the
requested maximum usage, the next data set begins on an empty tape. The default value for
this parameter is 500 MB, but IBM recommends 4000 MB for 3490 and 3590.


-----

Using a TAPESPANSIZE other than the default size is a trade-off between the use of
DFSMShsm tapes and needing to mount tapes more frequently to read spanned data sets.

The following example shows how to set the TAPESPANSIZE on tapes to a recommended
value for 3490 and 3590:

`SETSYS TAPESPANSIZE(4000)`

**Note:** If the **SETSYS TAPEUTILIZATION** parameter is set to NOLIMIT , DFSMShsm makes no
attempt to reduce data set tape volume spanning. If the **SETSYS TAPEUTILIZATION**
parameter is set to a percent, DFSMShsm performs reduced data set spanning.

## 7.14 Tape mount management

Tape mount management provides bandwidth in the batch flow. Tape mount management
also absorbs the smaller suitable tape allocations on DASD instead of mounting physical
tapes and filling these tapes to a limited extent.

Today, virtual tape systems absorb the partially filled tapes. Virtual tape systems reduce the
need for tape mount management because the bandwidth is available through many virtual
drives (up to 512 in one tape environment).

You might still want to implement tape mount management because you still use physical
tape mounts for your customer and these tapes are only partially filled, or the number of
drives is limited. If you implement tape mount management, you might then want to offload
your tape environment for physical mounts with partially filled tapes (that are not needed for
disaster recovery).

Follow these steps to implement tape mount management:

1. Tape data sets are categorized according to size, pattern of usage, and other criteria, so
that appropriate SMS policies can be assigned to the tape mount management
candidates.

2. Data sets that are written to tape are intercepted at allocation and, if eligible for tape
mount management, redirected to a system-managed DASD pool. The pool serves as a
staging area for these data sets until they are migrated to tape by DFSMShsm. The
location of the data is transparent to the application program.

3. During interval migration that runs by the hour, DFSMShsm checks the occupancy of the
DASD buffer storage group to ensure that space is available when needed, migrating data
sets to a lower level of the storage hierarchy when they are no longer required on primary
DASD volumes.

4. DFSMShsm eventually moves the data to tape by using single-file format and data
compaction to create full tape cartridges.

5. If an application later requests a data set, DFSMShsm automatically recalls it from where
it resides in the storage hierarchy and allocates it on primary DASD for access. This
process can significantly reduce tape mounts and the number of cartridges that are
required to store the data.


-----

**Tape mount management implementation plan**
Implementing tape mount management requires careful planning. Include the following major
tasks in your implementation plan:

1. Analyze your current tape environment.
2. Simulate the proposed tape mount management environment.
3. Select DASD volumes to satisfy buffer requirements.
4. Consider and define SMS constructs for the candidate data sets.
5. Create the ACS routines, or update the current ACS routines.
6. Monitor the result and expect to need to tune DFSMShsm operations.

**Volume mount analyzer**
The _volume mount analyzer_ is a program that helps you analyze your current tape
environment. You use the volume mount analyzer to study tape mount activity, monitor tape
media use, and implement tape mount management at your installation. The volume mount
analyzer produces reports that you can use to tailor data classes, management classes, and
ACS routine filters, depending on your specific tape usage profile. By using tape mount
management, you can maximize the use of your tape media and reduce your tape mounts.
The volume mount analyzer consists of two programs: GFTAXTR and GFTAVMA.

By analyzing the SMF records of your installation, the volume mount analyzer can help you
determine the following information:

- The space that each tape data set uses on a tape volume
- The unused space on a tape volume
- The number of tape mounts you will save by implementing tape mount management
- The tape mount management DASD buffer space that you will need to hold the data sets until they are transferred to tape storage

**Using the volume mount analyzer for tape mount management**
The volume mount analyzer helps you analyze the costs and savings of automating your tape
data management by using DFSMS. It also helps you choose data sets for tape mount
management. Tape mount management is a method of reducing tape mounts by performing
the following functions:

- Allowing the system to manage the placement of data
- Taking advantage of hardware and software compaction
- Taking advantage of new tape technology
- Filling each tape volume to capacity automatically

Using DFSMS and tape mount management can help you reduce the number of both tape
mounts and tape volumes that your installation requires. The volume mount analyzer reviews
your tape mounts and creates reports that provide you with information you need to effectively
implement the tape mount management methodology that we suggest .

**Information that is obtained by volume mount analyzer**
The volume mount analyzer produces both summary and detailed reports of your tape activity
for a selected time period. You can use these reports with an estimate of the average cost of
a tape mount to determine the value of managing tape data with DFSMS or to write ACS
routines that implement tape mount management most effectively.

Related to DFSMShsm, DFSMS class analysis is used to see how data uses the different
DFSMShsm tiers. This understanding can be used to tune the environment.

The DFSMShsm usage of tape can be isolated and analyzed. The tape workload from
DFSMShsm can then be tuned and adjusted, if needed.


-----

## 7.15 DFSMShsm and 3592 hardware support

DFSMShsm and 3592 hardware support are described.

### 7.15.1 DFSMShsm modifications for 3592 Model J

Specific changes to DFSMShsm support for the new 3592 Model E05 tape drives are
described.

DFSMShsm can use 3592 Model E05 for all functions. The 3592 Model E05 uses existing
cartridge media (MEDIA5, MEDIA6, MEDIA7, and MEDIA8) and also supports two new
media types: MEDIA9 (700 GB R/W) and MEDIA10 (700 GB WORM). MEDIA9 is available
for all DFSMShsm functions. MEDIA10 is intended specifically for ABARS processing of
ABACKUP tapes.

DFSMShsm also adds support for enterprise format 2 (EFMT2), a new recording format that
is required for using the new MEDIA9 and MEDIA10 cartridge media. The changes are listed:

- Planning and installation
- Input tape utilization
- Output tape selection
- Output tape utilization
- Reuse capacity table
- EFMT2 formatted volume display
- WORM tape cartridge rejection at OPEN time
- ABARS with WORM tape cartridge

### 7.15.2 Planning and installation

In an HSMplex, you need to apply the z/OS support program temporary fixes (PTFs) without
enabling the PTFs to coexisting systems without full support for 3592-E05. This way, a
coexisting system can reject partial tapes that are written by the 3592-E05 with EFMT2
technology.

### 7.15.3 Input tape selection

You can use MEDIA5, MEDIA7, and MEDIA9 tapes as input for all DFSMShsm functions.
You can also use MEDIA6, MEDIA8, and MEDIA10 tapes for ABARS processing. MEDIA5,
MEDIA6, MEDIA7, and MEDIA8 tapes can be written in either of two recording formats:
EFMT1 or EFMT2. The MEDIA9 and MEDIA10 tapes must be written in EFMT2 format.


-----

### 7.15.4 Output tape selection

DFSMShsm selects 3592 Model J tape drives for output in SMS and non-SMS tape
environments. DFSMShsm performs all of its allocation requests by using these standard
dynamic allocation interfaces:

- Non-SMS-managed output tape selection: If multiple types of tape drives are installed that
emulate the 3590 device type, you must define an esoteric name for each model that
DFSMShsm uses. You must then define the esoteric names to DFSMShsm by using the
**SETSYS USERUNITTABLE(esoteric1:esoteric1,esoteric2:esoteric2,...)** command. You
must also specify the esoteric names as the unit names for the DFSMShsm functions that
you want. If a single type of tape drive is installed that emulates the 3590 device type, you
do not need to define an esoteric name; instead, you can specify the 3590-1 generic name
for the DFSMShsm functions that you want.

- SMS-managed output tape selection: DFSMShsm performs a non-specific allocation; it
then finds an acceptable output tape for the already allocated drive. If the 3590-1 generic
name is used with multiple types of tape drives that are emulating the 3590 device type,
see APAR OW57282 for further instructions. For output tape utilization, DFSMShsm writes
to 97% of the capacity of MEDIA5, MEDIA6, MEDIA7, and MEDIA8 tapes unless the
percentage is otherwise specified by the installation. You can specify other percentages by
using the **SETSYS TAPEUTILIZATION** command, depending on the needs of the installation.
DFSMShsm uses the reported cartridge type on the physical device to determine the
tape’s capacity.

### 7.15.5 Output tape utilization

DFSMShsm writes to 97% of the capacity of MEDIA5, MEDIA6, MEDIA7, MEDIA8, MEDIA9,
and MEDIA10 tapes unless the percentage is otherwise specified by the installation. Other
percentages can be specified through the **SETSYS TAPEUTILIZATION** command, depending on
the needs of the installation. DFSMShsm uses the reported cartridge type on the physical
device to determine the tape’s capacity.

For scratch tapes, the 3592-E05 can use empty MEDIA5, MEDIA7, or MEDIA9 tapes for all
DFSMShsm functions and also use empty MEDIA6, MEDIA8, and MEDIA10 tapes for
ABARS processing. Because the 3592-E05 can write in either of two recording formats
(EFMT1 or EFMT2) to MEDIA5, MEDIA6, MEDIA7, and MEDIA8 tapes, you must modify your
installation’s ACS routines to select the recording format to use on blank media (through the
data class that is assigned to the tape) if you want the 3592 Model E05 drives to use EFMT1.

For duplexed tapes, ensure that the data class selects the same media type and recording
technology for the original and the alternate copy. Otherwise, the process can fail when the
duplex tape is mounted for output, or when you use the alternate copy after a tape replace. If
different media or machine types are needed for the original and alternate tapes, see APARs
OW52309, OA04821, and OA11603 for more information.

### 7.15.6 Partial tapes

When the 3592-E05 uses a MEDIA5, MEDIA6, MEDIA7, or MEDIA8 partial tape for output,
the tape can be written in either EFMT1 or EFMT2 recording technology. In contrast, MEDIA9
and 10 _partial tapes_ are always recorded by using EFMT2 technology. DFSMShsm
automatically determines the format in which a tape is written and extends the tape in the
same format.


-----

### 7.15.7 Reuse capacity table

The reuse capacity table supports MEDIA5 and MEDIA7 tape cartridges for 3592 Model J
tape drives. DFSMShsm uses the reuse capacity table to determine the tapes that are eligible
for RECYCLE processing based on capacity for each media type. Each media type has
separate reuse capacities for backup and migration.

### 7.15.8 Worm tape cartridge rejection

DFSMShsm examines the OEVSWORM field that is passed by OPEN/EOV processing. The
bit is turned on by OPEN/EOV when a WORM tape cartridge is mounted. If a WORM tape
cartridge is mounted for any function other than ABACKUP or ARECOVER, DFSMShsm
returns to OPEN/EOV with RC08, resulting in ABEND913 RC34 and a failure of OPEN.
DFSMShsm fails the function and issues either message ARC0309I (for non-ABARs) or
ARC6410E (for an ARECOVER ML2 tape).

## 7.16 DFSMShsm use of performance scaling on 3592

If your installation requires fast access to data on a MEDIA5 or MEDIA9 tape, consider using
the 3592 performance scaling feature. In DFSMShsm, performance scaling applies in both
tape libraries and stand-alone environments. Performance scaling uses 20% of the physical
space on each tape and keeps data sets closer together and closer to the initial load point.
Performance scaling permits the same amount of data to exist on a larger number of tapes,
allowing more input tasks to run concurrently.

With performance scaling, you can effectively increase the “bandwidth” of operations that
read in data from tape. In contrast, performance segmentation allows the use of most of the
physical media, while enhancing performance for the first and last portions of the tape.

### 7.16.1 How to use performance scaling and segmentation for 3592

The 3592 Model E05 supports performance scaling and performance segmentation of media
tape cartridges. With these functions, you can optimize performance for MEDIA5 and
MEDIA9 cartridges. _A cartridge can be defined for performance scaling or performance_
_segmentation, but not both._

_Performance scaling_ , also known as _capacity scaling_ , is a function to contain data in a
specified fraction of the tape, yielding faster locate and read times. Performance scaling for
the 3592 Model E05 limits the data that is written to the first 20% (the optimally scaled
performance capacity) of the cartridge. To select performance scaling for a cartridge, follow
these steps:

1. Define a data class that requests performance scaling.

2. Modify or create ACS routines to associate the tape output functions to use performance
scaling with a data class that requests performance scaling (set to Y ).

The 3592 Model E05 tape drive also divides the tape into longitudinal segments. By using
this capability, you can segment the tape into two segments: one segment is a fast access
segment to be filled first, and the other segment is additional capacity to be filled after.
Performance scaling applies to MEDIA5 only.


-----

If you decide to use the performance segmentation attribute, follow these steps:

1. Define a data class that requests performance segmentation (set to Y ).

2. Modify or create ACS routines to associate the tape output functions to use performance
segmentation with a data class that requests performance segmentation.

3. Performance segmentation can also be used with the 3592 Model J1A with MEDIA5.

**Note:** Performance scaling and segmentation cannot occur on the TS7700 back-end
drives and media.

### 7.16.2 Activating performance segmentation or scaling

A data class must exist with the performance segmentation or performance scaling attribute
set to Y :

- The ACS routine must map a DFSMShsm single file tape data set name to the data class.

- Because the data class now determines whether a performance option is used, non-SMS
tape needs ACS routines if you want to use a 3592 performance option.

The IEC205I at the close of tape will indicate whether performance segmentation or scaling
was used. Performance segmentation and performance scaling apply only to MEDIA5.

**Note:** Performance segmentation and performance scaling are mutually exclusive.

### 7.16.3 List of SETSYS commands that are used for tape

The following **SETSYS** commands are used for tape:

- TAPEOUTPUTPROMPT
- TAPESPANSIZE
- PARTIALTAPE
- PERCENTFULL
- TAPEHARDWARECOMPACT
- SETSYS INPUTTAPEALLOCATION(NOWAIT)
- SETSYS TAPEMRIGRATION
- TAPEUTILIZATION
- USERUNITTABLE
- TAPERECALLLIMITS
- SETSYS TAPEMAXRECALLTASKS(tasks)
- SETSYS ARECOVERUNITNAME
- UNITNAME
- MIGUNITNAME
- TAPEMIGRATION
- BACKUP/DSBACKUP/BACKDS
- RECYCLEOUTPUT

### 7.16.4 Commands that address the tape management system

The following commands address the tape management system:

- SETSYS CDSVERSIONBACKUP
- BACKUPDEVICECATEGORY(TAPE)
- SETSYS SELECTVOLUME(SCRATCH)
- SETSYS TAPEDELETION(SCRATCHTAPE)
- SETSYS TAPESECURITY(RACF)
- SETSYS EXITON(ARCTVEXT)


-----

