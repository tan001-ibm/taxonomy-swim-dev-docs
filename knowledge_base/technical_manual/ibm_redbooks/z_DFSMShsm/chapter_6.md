
# 6. Data set format support, record-level sharing, and extended address volumes

In this chapter, we describe DFSMShsm and storage management subsystem (SMS)
considerations for _record-level sharing_ (RLS) and _extended address volumes_ (EAVs).


-----

## 6.1 Record-level sharing

Record-level sharing (RLS) offers an advantage because it allows sharing with a finer
granularity, which reduces contention as the number of systems sharing resources in
sysplexes increases.

### 6.1.1 VSAM record-level sharing

DFSMShsm supports Virtual Storage Access Method (VSAM) RLS for accessing the control
data sets (CDSs). RLS enables DFSMShsm to take advantage of the features of the coupling
facility for CDS access.

Accessing CDSs in RLS mode reduces contention when running primary space management
and automatic backup on two or more processors. DFSMShsm benefits from the serialization
and data cache features of VSAM RLS and does not have to perform CDS verify or buffer
invalidation.

**Requirements for CDS RLS serialization**
CDSs that are accessed in RLS mode enqueue certain resources differently from CDSs that
are accessed in non-RLS mode. Before you think about implementing RLS for your CDSs,
you must ensure that all of the following criteria are met:
- Global resource serialization (GRS) or an equivalent function is implemented.

  **Note:** This function is required only if you are sharing CDSs between multiple systems.
If you have one or more hosts on a single system, it is not required.

- Your CDSs must be SMS-managed.

- All processors in the installation must access the CDSs in RLS mode.

- All DFSMShsm hosts must specify CDSSHR=RLS in the DFSMShsm startup procedure.

- The CDS storage class must indicate which coupling facility to use.

- The CDSs must not be key range key-sequenced data sets (KSDSs).

- You must know how to implement recovery for RLS data sets.

- You must specify DFSMSdss as the data mover for CDSVERSIONBACKUP .

- If CDS backup is directed to tape, the **PARALLEL** parameter must be used.

**Making your CDSs RLS eligible**
Before CDSs can be accessed in RLS mode, you must define or alter them to be RLS eligible,
by using the LOG(NONE) attribute. You must define or alter all of your CDSs: migration
control data sets (MCDSs), backup control data sets (BCDSs), and offline control data sets
(OCDSs).

The following example shows how to use the **IDCAMS ALTER** command to make the CDSs that
we defined previously RLS eligible:

ALTER HSM.MCDS LOG(NONE)

Figure 6-1 is an example of how we might use the **DEFINE** command when we initially set up
our DFSMShsm environment. We show the definition for the MCDS only, but the same
method is used for the other CDSs.


-----

    CLUSTER (NAME(HSM.MCDS) VOLUMES(HG6622) -  
        CYLINDERS(100) FILE(HSMMCDS) -
        RECORDSIZE(435 2040) FREESPACE(0 0) -
        INDEXED KEYS(44 0) SHAREOPTIONS(3 3) -
        SPEED BUFFERSPACE(530432) -
        UNIQUE NOWRITECHECK **LOG(NONE)** ) -
        DATA(NAME(HSM.MCDS.DATA) -
        CONTROLINTERVALSIZE(12288)) -
        INDEX(NAME(HSM.MCDS.INDEX) -
    CONTROLINTERVALSIZE(4096))

_Figure 6-1  Sample DEFINE command for a DFSMShsm environment_

You must never use the **ALL** or **UNDO** parameters of the **LOG** keyword. If you ever need to
change the CDSs back to non-RLS-eligible, use the following command:

ALTER HSM.MCDS NULLIFY(LOG)

**Removing key range CDSs**
The easiest way to remove key range CDSs is to remove the **KEYRANGE((...))** parameter
from the IDCAMS DEFINE DATA statements that you used to define your CDSs as key range.
During startup, DFSMShsm dynamically calculates the key boundaries for each cluster. You
can then use the **QUERY CONTROLDATASETS** command to display both the low and high keys that
DFSMShsm calculates for each cluster.

**Determining the CDS serialization technique**
If you need to verify the CDS serialization technique that is used, use the **QUERY**
**CONTROLDATASETS** command.

If you use RLS, the following RLS messages are returned from the **QUERY CONTROLDATASETS**
command:

ARC0101I QUERY CONTROLDATASETS COMMAND STARTING
ARC0947I CDS SERIALIZATION TECHNIQUE IS RLS

## 6.2 RLS implementation

The steps that we took to implement VSAM RLS for our CDSs are shown. In addition to the
requirements that we detailed in “Requirements for CDS RLS serialization” on page 96, other
considerations must be met and assumptions are made about your system knowledge. You
need to be familiar with SMS for VSAM RLS, SMS constructs, SMS classes, SMS
configuration, and the coupling facility cache and lock structures. We do not recommend that
you undertake these steps until you consider how RLS implementation might affect your
system.

It is recommended that any pre-existing VSAM RLS structures be used for accessing the
DFSMShsm CDSs in RLS mode. Assigning the DFSMShsm CDSs to unique structures offers
no benefit.


-----

### 6.2.1 Defining the SHCDSs

Sharing Control Data Sets (SHCDSs) are linear data sets that contain information to allow
processing if a system failure might affect RLS. They also act as logs for sharing support. You
must consider SHCDS size and adhere to a naming convention for the SHCDSs. For
comprehensive information about defining these data sets, see _OS/390® DFSMSdfp Storage_
_Administration Reference,_ SC26-7331. We used the JCL in Figure 6-2 to allocate the
SHCDS.

    //DEFSHCDS JOB (999,POK),'MHLRES5',CLASS=A,MSGCLASS=T,
    // NOTIFY=MHLRES5,TIME=1440,REGION=4M
    //STEP1  EXEC PGM=IDCAMS
    //SYSPRINT DD SYSOUT=*
    //SYSIN DD *
    DEFINE CLUSTER (NAME(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS1)     LINEAR SHR(3 3) VOL(SHCDS1)     CYLINDERS(15 15) )
    DEFINE CLUSTER (NAME(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS2)     LINEAR SHR(3 3) VOL(SHCDS2)     CYLINDERS(15 15) )
    DEFINE CLUSTER (NAME(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS3)     LINEAR SHR(3 3) VOL(SHCDS3)     CYLINDERS(15 15) )

_Figure 6-2  Sample SHCDS allocation JCL_

### 6.2.2 Coupling facility cache and lock structures

If you need to allocate cache and lock structures specifically for DFSMShsm, use the
following recommendations:

- Cache structure: The size of the cache structure must be a minimum of 1 MB per
DFSMShsm host in the HSMplex. For example, if 10 hosts are in the HSMplex, the cache
structure needs to be a minimum of 10 MB.

- Lock structure: Use Table 6-1 to determine the minimum size for the lock structure.

_Table 6-1  Minimum size for lock structure_

|Number of DFSMShsm hosts|Minimum lock structure size|
|---|---|
|Fewer than 8|1 MB|
|At least 8, but not more than 23|2 MB|
|At least 24, but not more than 32|3 MB|
|More than 32|4 MB|



For example, if 10 DFSMShsm hosts are in the HSMplex, the lock structure must be a
minimum of 2 MB.

Certain changes to your “define coupling facility resource management (CFRM)” policies are
necessary. The DFSMShsm policy needs to be defined. We added the structures for cache
and locking to our current CFRM policy definitions, by using the administrative data utility
IXCMIAPU. See Figure 6-3 on page 99.


-----

    //DEFCFRM1 JOB (999,POK),'CFRM',CLASS=A,REGION=4096K,
    //       MSGCLASS=X,TIME=10,MSGLEVEL=(1,1),NOTIFY=MHLRES5
    //STEP1  EXEC PGM=IXCMIAPU
    //SYSPRINT DD  SYSOUT=*
    //SYSABEND DD  SYSOUT=*
    //SYSIN  DD  *
    DATA TYPE(CFRM) REPORT(YES)
    DEFINE POLICY NAME(CFRM19) REPLACE(YES)
    
    CF NAME(CF01)
    TYPE(009672)
    MFG(IBM)
    PLANT(02)
    SEQUENCE(000000040104)
    PARTITION(1)
    CPCID(00)
    DUMPSPACE(2048)
    
    CF NAME(CF02)
    TYPE(009672)
    MFG(IBM)
    PLANT(02)
    SEQUENCE(000000040104)
    PARTITION(1)
    CPCID(01)
    DUMPSPACE(2048)
    
    STRUCTURE NAME(IGWLOCK00)
    SIZE(28600)
    INITSIZE(14300)
    PREFLIST(CF02,CF01)
    REBUILDPERCENT(75)
    
    STRUCTURE NAME(HSMCACHE1)
    SIZE(64000)
    INITSIZE(32000)
    PREFLIST(CF01,CF02)
    REBUILDPERCENT(75)
    
    STRUCTURE NAME(HSMCACHE2)
    SIZE(64000)
    INITSIZE(32000)
    PREFLIST(CF02,CF01)
    REBUILDPERCENT(75)

_Figure 6-3  CFRM policy definition for RLS access of the DFSMShsm CDSs_

**Note:** The code in Figure 6-3 does not represent the entire policy data for the CFRM data
set. It represents the CFRM policy that specifies the requirements for the DFSMShsm RLS
structures.

The coupling facility cache structure names that we chose to use are HSMCACHE1 and
HSMCACHE2. The locking structure name is the required name of _IGWLOCK00_ .


-----

### 6.2.3 Altering the SMS configuration

You must update the SMS configuration with the coupling facility cache structure names that
you defined earlier. Use the Interactive Storage Management Facility (ISMF) panels. Enter
option 8 from the ISMF Primary Option menu for Storage Administrators to display the CDS
Application Selection panel (Figure 6-4).

    CDS APPLICATION SELECTION
    Command ===>
    
    To Perform Control Data Set Operations, Specify:
    CDS Name . . 'SYS1.SMS.SCDS'
    (1 to 44 Character Data Set Name or 'Active')
    
    Select one of the following Options:
    
    1. Display    - Display the Base Configuration
    2. Define    - Define the Base Configuration
    3. Alter     - Alter the Base Configuration
    4. Validate   - Validate the SCDS
    5. Activate   - Activate the CDS
    6. Cache Display - Display CF Cache Structure Names for all CF Cache Sets
    7. Cache Update - Define/Alter/Delete CF Cache Sets
    
    If CACHE Display is chosen, Enter CF Cache Set Name . . *
    (1 to 8 character CF cache set name or * for all)
    
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 6-4  CDS Application Selection panel for cache_

We entered option 7 to define the cache sets that relate to our coupling facility cache structure
names (Figure 6-5).


-----

    Panel Utilities Scroll Help
    -----------------------------------------------------------------------------                CF CACHE SET UPDATE
    Command ===>
    
    SCDS Name : SYS1.SMS.SCDS
    Define/Alter/Delete CF Cache Sets:    ( 002 Cache Sets Currently Defined )
    
    Cache Set           CF Cache Structure Names
    HSM1   HSMCACHE1
    
    HSM2   HSMCACHE2
    
    More CF Cache Sets to Add? . . N (Y/N)

_Figure 6-5  CF Cache Set Update panel_

The coupling facility cache structure names must be the same names that we previously
defined. (See 6.2.2, “Coupling facility cache and lock structures” on page 98.)

### 6.2.4 Storage class changes

We opted to alter the storage class where our CDSs were already defined. The storage class
is SC54GRT, and we altered it by selecting option **4** from the Storage Class Application
Selection panel (Figure 6-6 on page 102).


-----

    Panel Utilities Help
    -----------------------------------------------------------------------------           STORAGE CLASS APPLICATION SELECTION
    Command ===>
    
    To perform Storage Class Operations, Specify:
    CDS Name . . . . . . . 'SYS1.SMS.SCDS'
    (1 to 44 character data set name or 'Active' )
    Storage Class Name . . SC54GRT (For Storage Class List, fully or
    partially specified or * for all)
    Select one of the following options :
    1. List     - Generate a list of Storage Classes
    2. Display    - Display a Storage Class
    3. Define    - Define a Storage Class
    4. Alter     - Alter a Storage Class
    5. Cache Display - Display Storage Classes/Cache Sets
    If List Option is chosen,
    Enter "/" to select option   Respecify View Criteria
    Respecify Sort Criteria
    
    If Cache Display is Chosen, Specify Cache Structure Name . .

_Figure 6-6  Altering a storage class_

We added the coupling facility cache set information (Figure 6-7).

    Panel Utilities Scroll Help
    ------------------------------------------------------------------------------               STORAGE CLASS ALTER
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.SCDS
    Storage Class Name : SC54GRT
    
    To ALTER Storage Class, Specify:
    
    Guaranteed Space . . . . . . . . . Y      (Y or N)
    Guaranteed Synchronous Write . . . N      (Y or N)
    CF Cache Set Name . . . . . . . . HSM1    (up to 8 chars or blank)
    CF Direct Weight . . . . . . . . . 5      (1 to 11 or blank)
    CF Sequential Weight . . . . . . . 3      (1 to 11 or blank)

_Figure 6-7  Storage Class Alter panel_

We added the coupling facility cache set name that we associated previously with the
coupling facility cache structure name. The greater the weight value, the higher the
importance that it is cached. We chose our values randomly.

Because the CDSs were already in this storage class, we validated and then activated our
SMS Source Control Data Set (SCDS) of SYS1.SMS.SCDS1.


-----

If your data sets are not already allocated to a storage class that allows RLS, you must assign
them to a storage class that allows RLS.

### 6.2.5 Altering IGDSMSxx PARMLIB member

To specify to SMS that the RLS address space, SMSVSAM, starts at initial program load (IPL)
and to include other information, we added the following statements that are shown in
Figure 6-8 to our PARMLIB data set, member IGDSMS54.

    SMS ACDS(SYS1.SMS.ACDS) COMMDS(SYS1.SMS.COMMDS)
    DEADLOCK_DETECTION(15,4)
    SMF_TIME(YES) CF_TIME(1800) RLSINIT(YES)
    RLS_MAX_POOL_SIZE(100)

_Figure 6-8  Sample IGDSMSxx PARMLIB to implement RLS_

Each MVS system will have its own SMSVSAM address space after IPL if RLSINIT(YES) is
coded.

### 6.2.6 Activating the SMSVSAM address space

To activate the SMSVSAM address space, an IPL of the MVS system is necessary. To verify
that the SMSVSAM address space started on your system, issue the following command:

D SMS,SMSVSAM,ALL

The information that is returned states whether your system is active and the status of the
SMS complex.

### 6.2.7 Activating the SHCDS

If your SHCDS is already active, you do not need to activate it. However, if you are
implementing RLS for the first time, you must issue the commands that are shown in
Figure 6-9, which are modified to represent the names that we chose for our SHCDSs.

    VARY SMS,SHCDS(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS1),NEW
    VARY SMS,SHCDS(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS2),NEW
    VARY SMS,SHCDS(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS3),NEWSPARE

_Figure 6-9  Activating the SHCDSs_

The commands that are shown in Figure 6-9 perform the following functions:

- Activate the new primary SHCDS
- Activate the new secondary SHCDS
- Activate a spare SHCDS

### 6.2.8 DFSMShsm procedure changes

To your DFSMShsm started procedure member in SYS1.PROCLIB, you must add **CDSSHR=RLS**
after the PROC statement.

You must also add **'CDSSHR=&CDSSHR** to your EXEC statement.


-----

## 6.3 RLS implementation checklist

To review, follow these steps to implement RLS access for the DFSMShsm CDSs:

1. Ensure that all sharing systems are at the prerequisite levels of software.

2. If SHCDSs are not already defined, use IDCAMS to define SHCDSs and ensure that
SHAREOPTION(3,3) is specified.

3. Define your coupling facility cache and lock structures.

4. Alter your SMS configuration to include the cache set information.

5. Create or alter a storage class that contains the cache set information.

6. Assign the CDSs to the new or altered storage class.

7. Alter the SYS1.PARMLIB member IGDSMSxx with the RLS parameters.

8. Define the new RLS profiles in the RACF FACILITY class and authorize users.

9. Schedule an IPL of the MVS systems and ensure that the SMSVSAM address space
becomes active.

**6.3.1 DFSMShsm support for RLS-managed ICF catalogs**

z/OS V 2.1 introduced RLS support for integrated catalog facility (ICF) catalogs. ICF catalogs
are now basically enqueued at a record level, instead of enqueue happening at the entire
catalog (SYSIGGV2), which will serialize catalog access. RLS support gives better
accessibility and performance through record-level access.

When DFSMShsm invokes DFSMSdss for recovering an ICF catalog, a new
**BCSRECOVER(LOCK)** keyword is used on the **RESTORE** command. This command performs a
serialized LOCK on the ICF catalog in process, if it is not already locked or suspended. When
restore completes successfully, DFSMSdss issues an UNLOCK of the ICF catalog to enable
normal processing.

**Note:** ICF catalogs are still not supported by migration or recall.

For aggregate backup and recovery support (ABARS), you can specify RLS-managed ICF
catalog names in the ALLOCATE list and DFSMShsm will process them. For the ABARS
INCLUDE and ACCOMPANY list, ICF catalogs are not supported.

For non-RLS-managed ICF catalogs, DFSMShsm still invokes IDCAMS EXPORT/IMPORT
for processing ICF catalogs.

Toleration authorized program analysis report (APAR) OA36414 was created for z/OS V10,
z/OS V11, z/OS V12, and z/OS V13. The program temporary fix (PTF) and its prerequisites
pointed to in the APAR need to be applied to lower-level systems in the sysplex.

After this support is installed, DFSMShsm can restore RLS-managed ICF catalogs by
invoking DFSMSdss on pre-z/OS V2.1 hosts. The RLS subcell will be preserved and the RLS
state will be restored as RLS is quiesced.


-----

### 6.3.2 RLS considerations

Because DFSMShsm uses RLS for serialization of its CDSs, it is important to ensure that
SMSVSAM completely initializes before you start DFSMShsm. Likewise, DFSMShsm needs
to be brought down before SMSVSAM.

If the SMSVSAM server encounters an unusual error and terminates, DFSMShsm
terminates, also. Review the information in “z/OS DFSMS V2.1 VSAM record-level sharing
error recovery” on page 417 for a description of how DFSMShsm handles SMSVSAM server
failure.

## 6.4 EAV support, considerations, and coexistence

Extended address volumes (EAVs) were implemented in three releases: z/OS V1.10
supported EAV R1, z/OS V1.11 supported EAV R2, and z/OS V1.12 was enhanced to
support EAV R3. z/OS V.13 includes extra support to increase EAV capacity from 223 GB
(262,668 cylinders) to 1 TB (1,182,006 cylinders). This 1 TB support was rolled back to z/OS
V.12 with an SPE (OA28553).

The support that DFSMShsm currently provides for EAV is described. EAV concepts are
introduced, and then the particular support features of DFSMShsm for EAV are described.

### 6.4.1 DFSMShsm EAV support and considerations

EAVs require hardware and software. Due to the limit of 64 K devices per channel subsystem
(CSS), large installations still required more storage space. The introduction of logical
channel subsystems (LCSSs) helped to increase the capacity by double and quadruple, but
EAVs increase this capacity by potentially 4,000-fold, which is based on a maximum
theoretical capacity of 225 TB. The maximum EAV with DFSMS V1.13 is 1 TB.

**Extended address volumes**
EAVs require IBM DS8000® R4 direct access storage device (DASD) subsystems so that
EAVs can be defined, as shown in Figure 6-10. For EAVs of 1 TB, DS8800 or
DS8700 subsystems and z/OS V1.12 or later are required.


-----


An EAV is divided into two parts: track-managed space for the first 64 K cylinders and
cylinder-managed space beyond 64 K cylinders. Cylinder-managed space is in an area that is
called the _extended addressing space_ (EAS). Cylinder-managed space is allocated in
multicylinder units (MCUs), which are 21 cylinders each. Therefore, the smallest data set size
that uses cylinder-managed space is 21 cylinders. This size has implications for space usage.
It can affect the allocation of data sets, which also applies to DFSMShsm and how it uses
EAVs. See Figure 6-11 on page 107.

**Important:** Only the EAS of the EAV requires any special support. Even data sets that are
not EAS-eligible can still be allocated on an EAV in track-managed space that is within the
first 54 GB.


-----


**EAS-eligible data sets**
In EAV R1 (z/OS V1.10), only VSAM data sets were eligible for EAS. At EAV R2 (z/OSV.11),
extended format sequential data sets were added. Now at EAV R3 (z/OS V1.12+), almost all
data set types are eligible and DFSMShsm is enhanced to support almost all data set types.

DFSMShsm EAV support includes almost all data set types:

- Sequential, including extended, basic, and large formats
- Partitioned data set (PDS) and partitioned data set extended (PDSE)
- Basic direct access method (BDAM)
- Undefine data set organization (DSORG=U)
- Catalogs (basic catalog structure (BCS) and VSAM volume data set (VVDS)
- VSAM
- z/OS file system (zFS), which is VSAM

The DFSMShsm logic uses the EATTR attribute (as used in IDCAMS DEFINE and
ALLOCATE, JCL, dynamic allocation, and DATACLAS) when it performs volume selection.

The following data set types are not eligible for EAS:

- Page data sets
- Volume table of contents (VTOC) and VTOC indexes
- Hierarchical file system (HFS)
- VSAM with **IMBED** or **KEYRANGE** parameters that were inherited from earlier physical
migrations or copies

These data sets must remain allocated in track-managed space. HSM is not used to manage
page data sets, VTOCs, or VTOC indexes, in any case.


-----

**EATTR parameter**
To allow users to control the migration of non-VSAM data sets to EAS (during the earlier
releases of EAV), a new data set level attribute was defined. Each data set used a new
attribute, **EATTR** , to indicate whether the data set can support extended attributes (that is,
format-8 and format-9 data set control blocks (DSCBs)). By definition, a data set with
extended attributes can reside in EAS on an EAV. This attribute can be specified for
non-VSAM data sets and VSAM data sets.

The following values are valid for this attribute:

**EATTR = NO** No extended attributes. Allocation can select a volume with or without
extended addressing with no preference based on this attribute.
However, when the data set created, the data set has an F1 DSCB
and cannot have extents in the EAS. NO is the _default for non-VSAM_
data sets.

**EATTR = OPT** Extended attributes are optional. The data set can use extended
attributes and reside in EAS. This value is the _default for VSAM_ data
sets.

**EATTR handling**
EXIT28 is used for restore processing to pass DFSMSdss the attributes of pre-allocated data
sets (in particular the EATTR) that are deleted by DFSMShsm before its reallocation.

DFSMShsm uses an IDCAMS, which was modified to recognize the EATTR value on an
import of a catalog data set and also passes the EATTR value to an export.

**Support for DFSMShsm-owned data sets**
HSM data sets can now also be allocated as EAS-eligible in V1.12. By using the **SETSYS**
**USECYLINDERMANAGEDSPACE** command, migration and backup copies of data sets can be
EAS-eligible. Dynamic allocation of PDOx/y, LOGx/y, and JRNL as EAS-eligible is supported,
too.

**DFSMShsm journal data set**
Large and extended format sequential, PDSE, and VSAM data sets can occupy an entire EAV
with multiple extents. Since EAV R2 (z/OS V1.11), direct-access device space management
(DADSM) allows a single extent to span the track-managed and cylinder-managed space.
Therefore, a data set can occupy the entire volume with a single extent. Therefore, the
DFSMShsm journal can potentially occupy an entire volume.

**Command for EAS for migration and backup**
With DFSMShsm V1.12 migration data sets (migration level 1 (ML1) and migration level 2
(ML2)) and backup, data sets can be directed to cylinder-managed space on EAVs. All
members of an HSMplex need to be at V1.12 before you can use this function. You can use
the **SETSYS USECMS** command to allow EAS eligibility for migration and backup data sets
(Example 6-1).

_Example 6-1  MVS SETSYS USECMS command_

**SETSYS USECYLINDERMANAGEDSPACE(Y|N)**
**SETSYS USECMS(Y|N)**

The default is to not use cylinder-managed storage. The status of the
**USECYCLINDERMANAGEDSPACE** parameter can be displayed by using a **QUERY SETSYS** command.
The output is displayed in the ARC0153I message.


-----

**DFSMShsm data mover**
Support is added to provide the following functions:

- Migration and backup, and recall and recovery of a PDS
- Backup and recovery of uncataloged and user-labeled sequential data sets
- Backup and recovery of catalogs
- Migration and recall of user-labeled sequential data sets
- Recovery with a specified pre-allocated data set

**ABARS support**
Enhancements were made to ABARS to support the additional data set types for z/OS
DFSMS V1.12. See “EAS-eligible data sets” on page 107. The enhancements are listed:

- If the extended attributes for the EAS-eligible Instruction Data Set are lost during recovery
of this data set to non-EAV, message ARC0784I is issued.

- The algorithm of aggregate recovery of migrated data sets from the INCLUDE list is
changed. For storage requirements for an aggregate group, the calculation of the size of
the user catalog and user-labeled data set was corrected, and the message ARC6369I is
now issued with the correct space requirement value.

- A new algorithm is available to assign the JOBNAME and STEPNAME of the source data
set (or a pre-allocated data set) from the ALLOCATE list to the target data set in the same
way that DFSMSdss is created.

- The EATTR value of a source user catalog from the ALLOCATE list is passed to the target.

**Modified messages**
The following messages were changed or corrected, or the situations in which they are issued
were changed.

**_ARC0153I_**
ARC0153I now shows the USECMS status:

ARC0153I SCRATCHFREQ=days, SYSOUT(CLASS=class, COPIES=number, SPECIAL
FORMS={form | NONE}), SWAP={YES | NO},
PERMISSION={YES | NO}, EXITS={NONE | exits}, UNLOAD={YES | NO},
DATASETSERIALIZATION= {USER | DFHSM}, USECMS={YES | NO}

**_ARC0784I_**
This message is now issued when a recall, recover, or ARECOVER to a non-EAV-capable
volume causes the loss of format-9 DSCB data:

ARC0784I EXTENDED ATTRIBUTES FOR DATA SET dsname WERE NOT RETAINED DURING
THE RECALL | RECOVER | ARECOVER

**_ARC6369I_**
This message is displayed for ABARS support when space is recalculated and corrected for
EAS allocation for user catalogs and user-labeled data sets. The format though remains the
same:

ARC6369I STORAGE REQUIREMENTS FOR AGGREGATE GROUP agname, ARE: L0=number
{K|M|G|T}, ML1=number {K|M|G|T}, ML2=number{K|M|G|T}, TOTAL=number
{K|M|G|T}


-----

**SMS volume selection**
For SMS volume selection on recall, recovery, or ARECOVER, DFSMShsm will pass the
DATACLASS value to DFSMSdss and volume selection will be performed by SMS. The only
exception is for PDS. The DFSMShsm volume selection logic for SMS PDS data sets will be
consistent with the non-SMS volume selection logic.

**Non-SMS volume selection**
DFSMShsm checks the data set level attribute, EATTR, when it selects a non-SMS volume.
The EATTR data set level attribute specifies whether a data set can use extended attributes
(format-8 and format-9 DSCBs) and optionally reside in EAS on an EAV. The EATTR attribute
is optional. If you specify it, valid values are NO and OPT . For more information about the
EATTR attribute, see ALLOCATE and DEFINE CLUSTER in _z/OS DFSMS Access Method_
_Services for Catalogs,_ SC26-7394 _._

If an EATTR value is not specified for a data set, the following default data set EATTR values
are used:

- The default behavior for VSAM data sets is OPT .
- The default behavior for non-VSAM data sets is NO .

Preference for volume selection corresponds to the EATTR values for each data set:

- When EATTR = NO, the data set has no preference to any volume. For recalls, recovers,
and ARECOVERs, the volume with the most free space is selected, but only the
track-managed free space is used to determine the free space calculations of an eligible
EAV. The data set is allocated with a format-1 DSCB, which indicates that it is not eligible
to have extents in the EAS.

- If an EAV or non-EAV is chosen for allocation, and the value is EATTR= NO and the data set
used a format-8 or format-9 DSCB with vendor attributes, the data set is allocated with a
format-1 DSCB. The system issues message ARC0784I to indicate that extended
attributes were not retained for this recall, recovery, or ARECOVER. In this situation, the
format-9 information of the data set will be lost on the recall, recovery, or ARECOVER.

- When EATTR = OPT , the data set prefers EAV volumes if the size of the allocation is equal
to or greater than the breakpoint value (BPV) that is specified. If the requested size of the
data set allocation is smaller than the BPV, EAVs and non-EAVs have equal preference:

– If an EAV is chosen for allocation and the data set type is EAS-eligible, the data set is
allocated with format-8 or format-9 DSCBs in either the track-managed space or the
EAS.

– If a non-EAV is chosen for the allocation and the data set used format-8 or format-9
DSCBs with vendor attributes, the data set is allocated with a format 1. Then,
DFSMShsm issues the ARC0784I message, which signifies that extended attributes
were not retained for this recall, recovery, or ARECOVER. The EATTR value always
remains in the format-1 DSCB.

**EAV considerations for space management**
Because an EAV has two areas, track-managed space and cylinder-managed space (EAS),
new selection criteria were added with the **TRACKMANAGEDTHRESHOLD** parameter. It is used with
the **THRESHOLD** parameter. It was necessary because data set size is also considered when
data sets are migrated. Also, data sets in the EAS are larger on average than the data sets
that are below in track-managed space. Data sets in the EAS will be migrated before the data
sets that are in track-managed space, which might mean that the track-managed space is not
freed up.


-----

DFSMShsm examines both volume-level thresholds and track-managed thresholds for L0
EAVs. If either threshold is exceeded, migration eligibility is performed for data on the volume.
If track-managed threshold values are not specified, the default is to use the volume-level
threshold values. Data set eligibility is further qualified by the location of the first three extents
of the data set. If any of the first three extents are allocated in track-managed space, the data
set is eligible to be processed when the track-managed space or entire volume space is
managed. If none of the first three extents are allocated in track-managed space, the data set
is eligible to be processed when the entire volume space is managed. Table 6-2 shows how
selections are made.

_Table 6-2  Migration eligibility_

|Track-managed threshold exceeded|Volume threshold exceeded|Data set selection|
|---|---|---|
|Yes|No|Only data sets with one or more of the first three extents that are allocated in track-managed space|
|No|Yes|All data sets|
|Yes|Yes|All data sets|
|No|No|None selected|



For SMS volumes, DFSMShsm uses the track-managed threshold and the volume threshold
from the storage group. For non-SMS volumes, the track-managed threshold is specified on
the **ADDVOL** command when you add primary volumes to DFSMShsm:

ADDVOL PROD01 PRIMARY UNIT(3390) TRACKMANAGEDTHRESHOLD(80 40) THRESHOLD(85 60)

If no TRACKMANAGEDTHRESHOLD is specified, the volume threshold values are used, by
default.

**Support and considerations for 1 TB EAVs**
The support for DFSMShsm 1 TB resulted in no external updates. Only internal fields
required updating to handle the larger quantities that are dealt with. Although no externals
require definition or implementation for DFSMShsm, several considerations are new.

Testing shows a considerable increase in elapsed time for IBM HyperSwap® for larger
volume capacities due to the larger amounts of metadata. Spreading 1 TB EAVs over as
many LSSs as possible is best.

Additionally, although dynamic volume expansion automatically starts reformatting the VTOC
so the larger extra capacity that is added is recognized by the VTOC, you still need to
manually expand the size of the VTOC, VTOC index, and VVDS to handle the larger number
of expected data set entries. Instructions for dynamic volume expansion and expanding the
VTOC and VTOC index are covered in Chapter 10 of _z/OS V1.13 Technical Update,_
SG24-7961.

**Autodump considerations with EAVs**
With the introduction of large capacity volumes, such as EAV, users who use dump stacking
need to review DUMP CLASS and its dump stack figure.

A storage group of different capacity volumes (3390 Model 3, Model 9, Model 27, Model 54,
and EAV) might require the DUMP window to be extended for DFSMShsm to complete the full
volumes dump.


-----

A dump task with all high-capacity volumes to be dumped (such as Model 27, Model 54, or
EAV) requires more time to complete than another dump task, which only dumps all Model 3
volumes by using the same dump stacking figure.

We recommend that you group the same capacity volumes into the same dump class and
assign the appropriate dump stacking figure (the higher the volume capacity, the smaller the
dump stacking figure) to prevent large differences between the dump tasks’ elapsed times to
complete. This configuration helps minimize the need to extend the dump window.

**EAV considerations for recovery functions**
You can specify the EATTR value when you allocate a data set to define whether the data set
can use extended attributes (format-8 and format-9 DSCBs) and optionally reside in EAS. If a
data set is directed to a pre-allocated data set on a RECOVER or ARECOVER function, the
system uses the EATTR value of the pre-allocated data rather than the EATTR value that is
specified for the data set source recovery data.

New considerations exist for ARECOVER and EATTR. The following methods can be used to
allow the EATTR information from the ABACKUP site to be restored, with the data set:

- Use the **DSCONFLICT(RENAMETARGET)** parameter on the **ARECOVER** command, then manually
delete the existing data set. Because the ARECOVER resolved the conflict by renaming
the existing data set before it restores the ABARS copy, both copies will have their own
EATTR. You need to remove only the renamed existing data set.

- Use the **DSCONFLICT(BYPASS)** parameter on the **ARECOVER** command, then manually delete
the existing data set and existing conflict resolution data set, and then rerun the **ARECOVER**
command. Only the data sets that were bypassed or failed for another reason on the first
ARECOVER will be processed. Because no conflict exists, you will get the ABARS copy,
complete with its EATTR.

For more information, see the **EATTR** parameter of the **ALLOCATE** command in _z/OS DFSMS_
_Access Method Services for Catalogs_ , SC26-7394.

**Recommendations for using EAVs**
Because ML1, ML2, and backup volume allocations can reside in either the track-managed
space or the cylinder-managed space on an EAV beginning in V1R12, it is important to
understand that allocations in the cylinder-managed space are in multicylinder units (MCUs)
and can potentially result in an over-allocation.

You might want to evaluate the SMS and non-SMS BPV settings. If possible, enable the
system to avoid over-allocation for smaller disk space requests. For larger disk spaces, use
cylinder-managed space and allocate in MCUs for larger disk space requests.

For example, setting the BPV to a high value will result in smaller disk space requests made
by DFSMShsm more likely to reside in the track-managed area. Larger disk space requests
will prefer to use space that can end up in an over-allocation. However, this over-allocation will
be a much smaller percentage of the total disk space request and the over-allocation might be
acceptable. For more information about using EAVs, see _z/OS DFSMS Using the New_
_Functions_ , SC26-7473.

### 6.4.2 z/OS DFSMShsm earlier level coexistence

Because of the differences between the EAV releases, particularly the more limited function
EAS-eligible data set types, as shown in Table 6-3 on page 113, coexistence issues exist if
members in an HSMplex are at earlier levels.


-----

_Table 6-3  EAV data set type support_

|EAV release|z/OS release|Data set types supported|
|---|---|---|
|EAV R1|z/OS V1.10|VSAM|
|EAV R2|z/OS V1.11|VSAM and Extended Format Sequential Access Method (EFSAM)|
|EAV R3|z/OS V1.12 and later|VSAM, catalogs, sequential access method (SAM), basic direct access method (BDAM), PDS, PDSE, undefined, and zFS|



Members with earlier-level releases of DFSMShsm will be restricted in their function.

**One TB EAVs**
Space calculations for data sets that are migrated, backed up, or backed up with ABARs from
z/OS V1.10 to V1.11 will show incorrect values when recalled or recovered if they are greater
than 2 TB. See OW30632 for details. Attempts to recall or recover on z/OS V1.10 or V1.11 will
be failed.

Only z/OS V1.13 and z/OS V1.12 with OW28553 can bring 1 TB EAVs online.

**Toleration**
z/OS V1.10 and earlier DFSMShsm releases needed coexistence to handle the additional
EAS-eligible data sets and the new EATTR data set attribute.

No toleration maintenance is available for z/OS V1.11, except for OA30632, because the
basic toleration logic is carried forward from the two earlier APARS OA27146 and OA28804.
The same messages are issued in z/OS V1.11 as in z/OS V1.12 or z/OS V1.13, if for
example, a restore to a non-EAV volume results in the loss of DSCB9 data, it results in the
issue of an ARC0784I message. This basic toleration logic is from OA28804.

Although OA27146 and OA28804 are old, they are important in the context of coexistence
and toleration because they provided the logic in use today for coexistence.

_APAR OA27146_ provided DFSMShsm toleration for EAV R2 and applied to z/OS DFSMShsm
in an HSMplex environment with V1.11 DFSMShsm and V1.10 and V1.9 DFSMShsm levels
(z/OS V1.9 is out of support today):

- DFSMShsm toleration for EAV R2. z/OS V1.10 and V1.9 releases of DFSMShsm needed
coexistence to handle the additional EAS-eligible data sets and the new EATTR data set
attribute.

- Coexistence support for DFSMShsm in z/OS V1.10 to enable DFSMShsm to tolerate the
DFSMShsm support that is provided in z/OS V1.11. In z/OS V1.10, DFSMShsm can
RECALL, RECOVER, and ARECOVER both VSAM and non-VSAM data sets from a z/OS
V1.11 migration or backup copy with format-8 and format-9 DSCBs to non-EAVs.
Format-8 and format-9 DSCBs will be converted to format-1 DSCBs, and format-9 DSCB
attributes will be lost if such a data set is recalled, recovered, or recovered by using
ARECOVER to non-EAV. In this case, an ARC0784I message is issued if DFSMSdss
issues an ADR556W message. For z/OS V1.9, support is added for preserving the EATTR
value.


-----

- Non-VSAM data sets with format-8 and format-9 DSCBs will be visible to the system, but
not eligible for migration, backup, or ABACKUP processing. When one of these data sets
is encountered, it is failed by DFSMShsm. Messages ARC1220I for migration and
ARC1320I messages for backup with RSN21 or ARC0734I with RC20 and RSN21 are
issued.

  We get the following error message when we try to migrate the EAV data set
CVERNON.EAVTEST.EFSAM0 on a V1.10 level system:

  ARC1001I CVERNON.EAVTEST.EFSAM0 MIGRATE FAILED, RC=0020, REAS=0021
ARC1220I DATA SET NOT ELIGIBLE FOR MIGRATION
\***

  We get the following error message when we try to back up the EAV data set
CVERNON.EAVTEST.EFSAM0 on a V1.10 level system:

  ARC1001I CVERNON.EAVTEST.EFSAM0 BACKDS FAILED, RC=0020, REAS=0021
ARC1320I DATA SET NOT ELIGIBLE FOR BACKUP
COMMAND REQUEST 00000072 SENT TO DFSMSHSM
\***

- Recall/recover of non-VSAM data sets that have format-8 and format-9 DSCBs will be
completed to the track-managed space so that DFSMSdss will RESTORE them.
ADR556W is issued, and DFSMShsm issues the ARC0784I message. However, the
RECALL will fail if no track-managed space volume is available to satisfy the space
allocation request.

  Data set CVERNON.EAVTEST.EFSAM0 originally resided on EAV volume GKDD65 and it
is the only volume in storage group EAVGK. When we try to recall the data set on the
V1.10-level system, DFSMShsm tries to allocate a data set on a track-managed space
volume but none of them are available to storage group EAVGK. Therefore, DFSMShsm
failed the recall, as shown:

  ARC1001I CVERNON.EAVTEST.EFSAM0 RECALL FAILED, RC=0069, REAS=0709
ARC1169I RECALL/RECOVER FAILED DUE TO AN ERROR IN DFDSS
RECALL REQUEST 00000073 SENT TO DFSMSHSM
***

- Recover REPLACE of non-VSAM data sets is failed if a pre-allocated non-VSAM data set
is identified with a format-8 or format-9 DSCB. The ARC1158I message is issued.
Messages ARC1001I or ARC0734I provide the data set name and the reason code 51. If
a pre-allocated non-VSAM data set is identified with a format-1 DSCB, the EATTR value is
passed to DFSMSdss to preserve the value.

We get the error message ARC1158I when we try to recover data set
CVERNON.EAVTEST.EFSAM0 to replace the existing data set, which has format-8 and
format-9 DSCBs on a V1.10-level system:

ARC1001I CVERNON.EAVTEST.EFSAM0 RECOVER FAILED, RC=0058, REAS=0051
ARC1158I RECOVER DATA SET FAILED
COMMAND REQUEST 00000074 SENT TO DFSMSHSM
***

APAR OA22804 provided DFSMShsm V1.9 to V1.6 with toleration for EAV R1. The pre-V1.10
versions of z/OS DFSMShsm coexisted in a HSMplex with z/OS DFSMShsm V1.10. In this
situation, the EAV is inaccessible for the pre-V1.10 systems because they will be offline. Data
sets that were migrated or backed up from an EAV to ML1, ML2, and backup volumes
(non-EAV) under z/OS DFSMShsm V1.10 are accessible for the pre-V1.10 DFSMShsm
systems.


-----

This toleration APAR is required to support recall, recover, and recycle for these data sets in
the pre-V1.10 systems. If the toleration APAR is not installed, the recall, recover, and recycle
of these data sets fail during pre-V1.10 DFSMShsm processing. AUDIT and RECYCLE are
changed to process a data set common data set descriptor records (CDDs) with a format-8
DSCB.

APAR OA22804 provides coexistence support of the pre-V1.10 DFSMShsm release to
perform the following functions:

- Allow pre-V1.10 systems to recognize MCV records for EAVs.

- Restrict the selection of extent reduction request from the common recall queue (CRQ), if
the request is directed to an EAV.

- Restore EAS-eligible data sets from a backup or migrated copy on an online volume.

- Enable pre-V1.10 systems to process the new VCC keywords that might be specified in
the management class backup copy technique field.

- Support recall/recover on V1.9 or lower of an EAS data set with a format-8 DSCB that was
migrated/backed up on V1.10:

  - Converts to format-1 DSCBs

  - Issues the new ARC0784I message and update the functional statistics record (FSR)
record if format-9 DSCB vendor attributes are lost during the
RECALL/RECOVER/ARECOVER function:

    ARC0784I EXTENDED ATTRIBUTES FOR DATA SET dsname WERE NOT RETAINED DURING
THE RECALL | RECOVER | ARECOVER

    **Explanation:** The data set was recalled, recovered, or recovered by using
ARECOVER from a migration copy, backup copy, or an aggregate backup
successfully. However, JOBNAME, STEPNAME, creation time attributes, and
vendor attributes from the format-9 DSCB of the recalled or recovered data set were
not retained because the volume on which it was placed did not support format-8
and format-9 DSCBs. The recall, recovery, or aggregate recovery continues.

    **Restriction:** If a recall for extent reduction is directed to an EAV, the selection of this
request from the common recall queue (CRQ) will be restricted in the pre-V1.10
systems.

Use of EAV volumes in mixed environments where the volume cannot be brought online to all
systems can also result in recalling or recovering data sets that belong to a storage class that
has a guaranteed space attribute.

**Recovery considerations**
If you need to recover a data set backup that is made from an SMS EAV volume to a
lower-level system, perform the following tasks:

- If the EAV is still available on the higher-level system, delete or rename the data set on the
higher-level system.

- If the EAV is no longer available and a catalog entry exists, use IDCAMS DELETE
NOSCRATCH to remove the data set from the catalog.

**Note:** The backup copy must reside on a non-EAV and the volume must be online to the
lower-level system.


-----

