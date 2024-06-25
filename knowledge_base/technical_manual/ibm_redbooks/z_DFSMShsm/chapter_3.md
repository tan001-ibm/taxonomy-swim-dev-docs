# 3. Getting Started

In this chapter, we describe how to set up your DFSMShsm environment by using the
samples that are provided in SYS1.SAMPLIB as a starting point. These samples are referred to
in the DFSMShsm manuals as the “DFSMShsm starter set”. Detailed information about
defining DFSMShsm to z/OS and defining and maintaining its control data sets (CDSs) and
accompanying data is in the DFSMShsm Implementation and Customization Guide,
SC35-0418.
The complete details of all of the possible parameters that can be used to tailor DFSMShsm
are in z/OS DFSMSdfp Storage Administration Reference, SC26-7402.
We describe several of the decisions that you need to make before you begin a DFSMShsm
configuration. In this chapter, we describe setting up the required procedures and PARMLIB
members for the started task. We then explain allocating the required data sets to enable
DFSMShsm to start.
We also describe managing interactions with your DFSMShsm address spaces. For example,
we describe starting and stopping the DFSMShsm address spaces and where DFSMShsm
fits in your automation hierarchy. We also describe ways of passing commands to the
DFSMShsm address space and mention the built-in DFSMShsm recovery ability.

## 3.1 DFSMShsm components

A DFSMShsm environment consists of one or more DFSMShsm address spaces.
DFSMShsm can be run only as a started task and only one DFSMShsm can exist.

Each address space shares access to a set of up to three CDSs and a journal:

- Migration control data set (MCDS)
- Backup control data set (BCDS)
- Offline control data set (OCDS)

In addition to the CDSs, each DFSMShsm address space can have two pairs of allocated log
data sets:

- ARCLOGX and ARCLOGY

  This pair is used to log activity (as it occurs) in the DFSMShsm address space, such as
the DFSMShsm actions. Each data set of this pair must be allocated on the same volume.
DFSMShsm uses these data sets as a circular log. DFSMShsm always writes to the
ARCLOGX data set. When the ARCLOGX data set fills, DFSMShsm automatically
renames the ARCLOGX to ARCLOGY and the ARCLOGY to ARCLOGX and continues
writing to the ARCLOGX data set.

- ARCPPOX and ARCPDOY

  This pair is used to log information that is required only for problem determination. These
data sets are used in the same way as the ARCLOG data sets as a circular log.

DFSMShsm can also manage several types of media:

- Migration level one (ML1): ML1 is always direct access storage device (DASD). The DASD
must not be managed by the storage management subsystem (SMS). The DASD must be
dedicated to DFSMShsm use.

- Migration level two (ML2): ML2 is almost always tape but can be DASD. ML2 tape volumes
can reside in SMS-managed tape libraries or as non-SMS volumes. If ML2 is DASD, the
DASD must not be SMS-managed.

- Backup volumes: DFSMShsm can use DASD or tape backup volumes or a combination.
As with migration volumes, DASD backup volumes must not be SMS-managed.

- Dump volumes: Dump volumes are always tape volumes.

## 3.2 DFSMShsm starter set

In the following chapters of this book, we describe in detail the parameters that are required to
implement functions and housekeeping tasks that are automated by DFSMShsm. This
section concentrates on the basic building blocks of a DFSMShsm system.

We describe the use of the DFSMShsm starter set, which is a provided set of sample job
control language (JCL) and code. If you are installing DFSMShsm for the first time, the starter
set provides useful samples. However, like all samples, the data that is supplied needs to be
tailored for your site requirements. If you are reviewing an existing configuration, the starter
set might contain information about new defaults and changed functions.

DFSMShsm provides two members with sample code and JCL. These members are
delivered in SAMPLIB.


-----

The first member, ARCSTRST. ARCSTRST, is JCL to create and populate a data set called
_HSM.SAMPLE.CNTL_ .

The second member, ARCTOOLS. ARCTOOLS, is JCL to create and populate a number of
data sets:

- The first data set is HSM.SAMPLE.TOOL. This data set contains samples that you can
use for both capacity planning and creating multicluster DFSMShsm CDSs.

- The second data set that is allocated by ARCTOOLS is HSM.ABARUTIL.JCL. This data
set contains samples that can be used when you implement aggregate backup and
recovery support (ABARs) processing.

- The next data set that is allocated by ARCTOOLS is HSM.ABARUTIL.DOCS.

- The last set is HSM.ABARUTIL.PROCLIB, which contains JCL for a sample ABARS
address space.


In this chapter, we concentrate on the contents that are created by the ARCSTRST member
in HSM.SAMPLE.CNTL and how the contents can be used to create a new DFSMShsm
environment or upgrade a current DFSMShsm environment.

### 3.2.1 Contents of HSM.SAMPLE.CNTL

When you run the ARCTOOLS member, you create the data set HSM.SAMPLE.CNTL and
populate it with several members:

- ALLOCBK1
- ALLOSDSP
- ARCCMD01
- ARCCMD90
- ARCCMD91
- HSMEDIT
- HSMHELP
- HSMLOG
- HSMPRESS
- STARTER

These members are reviewed in the following topics in a suggested running order. It is not
necessary to run all of the members before you start DFSMShsm. If you choose not to run
specific jobs, consider the impact of this selection. In certain cases, you might need to
implement equivalent functions.

**STARTER**
This member provides the basis of your DFSMShsm environment. STARTER contains steps
that will allocate the following objects:

- A user catalog and alias
- All DFSMShsm CDSs
- The journal data set
- Member DFSMShsm in SYS1.PROCLIB
- Member ARCCMD00 in SYS1.PARMLIB
- Member ARCSTR00 in SYS1.PARMLIB


-----

You must decide which DFSMShsm functions you plan to use before you run the STARTER
member. You must determine the following information before you run the STARTER
member:

- A naming convention for DFSMShsm CDSs.

- A naming convention for data that is owned by DFSMShsm.

- The DFSMShsm functions that are implemented.

- The CDSs that are needed.

- In which volumes the DFSMShsm CDSs exist.

- Estimate the initial size of the CDSs.

- Determine how access to the CDSs is serialized. The starter set assumes that Virtual
Storage Access Method (VSAM) record-level sharing (RLS) will not be used for
serialization.

- Decide how many backup copies of each control data set are required.

- Determine a user ID to use for the DFSMShsm task and define it in advance to your
security system. If you are using small-data-set packing (SDSP), the DFSMShsm user ID
is used as the high-level qualifier (HLQ) of the SDSP data sets.

- If you are not using the DFSMShsm user ID as the HLQ for DFSMShsm data sets, ensure
that you establish security rules in advance for the names that will be used.

**DFSMShsm CDSs**

The STARTER JCL allocates four single volume CDSs:

- MCDS contains information about migrated data sets, DFSMShsm migration volumes,
users, non-SMS DASD volumes, and statistics. MCDS is a key-sequenced data set
(KSDS) and a required data set. A multicluster MCDS can consist of more than one
physical data set.

- BCDS contains information about backup copies of data sets, backup, and dump volumes.
This data set is not required if DFSMShsm availability functions will not be used. The
BCDS is one or more KSDSs. A multicluster BCDS can consist of more than one physical
data set.

- OCDS contains information about the contents of tape volumes. The OCDS is required if
you intend to use tape volumes. The OCDS cannot be multicluster; it must be a single
data set. The OCDS is a KSDS.

- JRNL records changes to the MCDS, BCDS, and OCDS. JRNL is only used for recovery.
JRNL is a single volume sequential data set.

We include the DEFINE CLUSTER for the single cluster MCDS from the starter set as an
example (see Figure 3-1). Because the requirements of the individual data sets
change over time, always check with the correct level of the _z/OS DFSMShsm_
_Implementation and Customization Guide,_ SC23-6869, for the required characteristics of
each of the CDSs.


-----

    /****************************************************************/
    /* THIS PROCEDURE ASSUMES A SINGLE CLUSTER MCDS. IF MORE THAN */
    /* ONE VOLUME IS DESIRED, FOLLOW THE RULES FOR A MULTICLUSTER */
    /* CDS.                            */
    /****************************************************************/
    /*                               */
    IF MAXCC = 0 THEN DO
    DEFINE CLUSTER (NAME(?UID.MCDS) VOLUMES(?MCDSVOL) -
    CYLINDERS(?CDSSIZE) FILE(HSMMCDS) -
    STORCLAS(?SCCDSNM) -
    MGMTCLAS(?MCDFHSM) -
    RECORDSIZE(435 2040) FREESPACE(0 0) -
    INDEXED KEYS(44 0) SHAREOPTIONS(3 3) -
    SPEED BUFFERSPACE(530432) -
    UNIQUE NOWRITECHECK) -
    DATA(NAME(?UID.MCDS.DATA) -
    CONTROLINTERVALSIZE(12288)) -
    INDEX(NAME(?UID.MCDS.INDEX) -
    CONTROLINTERVALSIZE(2048))
    END
    /*                               */

_Figure 3-1  DEFINE for the MCDS from the starter set_

**DFSMShsm procedure**
We describe the parameters in the DFSMShsm procedure that is provided by the starter set.


-----

Figure 3-2 contains a copy of the sample DFSMShsm procedure that is provided by the
DFSMShsm starter set. Several comment lines were removed.

    //*******************************************************************/
    //*          DFSMSHSM START PROCEDURE           */
    //*******************************************************************/
    //* IF ALL OF THE DFSMSHSM STARTUP PROCEDURE KEYWORDS ARE NEEDED,  */
    //* TOTAL LENGTH WILL EXCEED THE 100-BYTE LIMIT, IN WHICH CASE   */
    //* YOU SHOULD USE THE KEYWORD STR=XX IN PARM= TO IDENTIFY THE   */
    //* PARMLIB MEMBER CONTAINING THE ADDITIONAL KEYWORDS AND PARMS.  */
    //*******************************************************************/
    //DFSMSHSM  PROC CMD=00,   USE PARMLIB MEMBER ARCCMD00 FOR CMDS
    //      STR=00,     PARMLIB MEMBER FOR STARTUP PARMS
    //      EMERG=NO,    SETS HSM INTO NON-EMERGENCY MODE
    //      CDSQ=YES,    CDSs SERIALIZED WITH ENQUEUES
    //      PDA=YES,     PROBLEM DETERMINATION AID
    //      SIZE=0M,     REGION SIZE FOR DFSMSHSM
    //      DDD=50,     MAX DYNAMICALLY ALLOCATED DATASETS
    //      HOST=?HOSTID,  PROC.UNIT ID AND LEVEL FUNCTIONS
    //      PRIMARY=?PRIMARY LEVEL FUNCTIONS
    //*******************************************************************/
    //DFSMSHSM EXEC PGM=ARCCTL,DYNAMNBR=&DDD,REGION=&SIZE,TIME=1440,
    //     PARM=('EMERG=&EMERG','CMD=&CMD','CDSQ=&CDSQ',
    //     'UID=?UID','PDA=&PDA','HOST=&HOST','STR=&STR',
    //     'PRIMARY=&PRIMARY')
    //HSMPARM DD DSN=SYS1.PARMLIB,DISP=SHR
    //MSYSOUT DD SYSOUT=A
    //MSYSIN  DD DUMMY
    //SYSPRINT DD SYSOUT=A,FREE=CLOSE
    //SYSUDUMP DD SYSOUT=A
    //MIGCAT  DD DSN=?UID.MCDS,DISP=SHR
    //JOURNAL DD DSN=?UID.JRNL,DISP=SHR
    //ARCLOGX DD DSN=?UID.HSMLOGX1,DISP=OLD
    //ARCLOGY DD DSN=?UID.HSMLOGY1,DISP=OLD
    //ARCPDOX DD DSN=?UID.HSMPDOX,DISP=OLD
    //ARCPDOY DD DSN=?UID.HSMPDOY,DISP=OLD
    //BAKCAT  DD DSN=?UID.BCDS,DISP=SHR
    //OFFCAT  DD DSN=?UID.OCDS,DISP=SHR
    
_Figure 3-2  DFSMShsm procedure from the DFSMShsm starter set_

Do not include unnecessary keywords.

The EXEC statement of the sample that is shown in Figure 3-2 contains the following
parameters:

- **SIZE:** Is the DFSMShsm region size. Allocate the largest region possible both
above and below the line. Use REGION= 0M , if possible.

- **DDD:** Is used to calculate the maximum number of data set allocations that
DFSMShsm can hold in anticipation of reuse. The supplied value is 50 .

- **PARM:** Is used to pass variable information to the processing program. The
length of the PARM subparameters on the EXEC statement must not
exceed 100 characters (including commas and excluding parentheses;
this rule is a JCL restriction).


-----

The **PARM** parameter of the sample that is shown in Figure 3-2 contains the
following subparameters:

- **CMD:** Specifies the two-character suffix of the ARCCMDxx member of the
library that is pointed to by the HSMPARM DD statement. It contains
the parameters that define the DFSMShsm working options. The
default value is 00 .

- **EMERG:** Overrides the SETSYS EMERGENCY definition and allows, in case of
errors, DFSMShsm to be started in a configuration that can be used for
recovery. In normal conditions, this parameter is set to the default value
of NO .

- **UID:** Is the user ID to be assigned to the DFSMShsm started task.

- **HOST:** Represents a unique host identifier, _HOSTID_ , for each host. The
_HOSTID_ that you assign to a DFSMShsm address space must be
unique to that address space within the sysplex. _HOSTID_ allows an
optional second character in the value. The function of that second
character is now specified by the PRIMARY= keyword. The second
character, if specified, is considered only if the PRIMARY= keyword is
not specified.

- **PRIMARY:** Specify whether this system is a primary system (use Y ) or a secondary
system (use N ). The backup and dump level functions are performed on
the primary processing unit. The automatic secondary space
management functions can be performed on any host. The default
value is Y .

- **HOSTMODE:** Specifies how this instance of DFSMShsm relates to various functions
of DFSMShsm. HOSTMODE= MAIN specifies that this DFSMShsm
processes implicit requests, such as recalls and deleting migrated data
sets, from user address spaces; processes explicit commands from
Time Sharing Option (TSO), such as **HSENDCMD** and **HBACKDS** ; manages
ABARS secondary address spaces; allows MODIFY commands from a
console; and can run an automatic backup, a dump, and space
management. HOSTMODE= AUX specifies that this DFSMShsm allows
**MODIFY** commands from a console and can run automatic backup,
dump, or space management. The default value is MAIN .

- **STR:** Specifies a PARMLIB member that contains DFSMShsm startup
parameters, which are logically concatenated with any remaining
parameters that are specified on the EXEC statement. The value for
the **STR** keyword must be two characters.

- **PDA:** Specifies that the problem determination aid (PDA) tracing begins
before the **SETSYS PDA** command is processed or the DFSMShsm
initialization is complete.

- **CDSQ:** Specifies that DFSMShsm serializes its CDSs with a global enqueue
product (global resource serialization (GRS), for example) instead of
serializing with volume reserves. When you specify YES for this
parameter, DFSMShsm serializes the use of the CDSs (between
multiple z/OS images) with a global (SYSTEMS) exclusive enqueue
and still allows multiple tasks within a single z/OS image to access the
CDSs concurrently.


-----

The DD statements in the procedure point to the following data sets:

- **HSMPARM:** Is the library that contains the DFSMShsm **SETSYS** parameters. It is
typically SYS1.PARMLIB, but can be any other source format library.
The member names must be ARCCMD followed by a two-character
suffix, as indicated in the **CMD** parameter of the procedure EXEC
statement.

- **MSYSOUT:** Is a system data set that is used by DFSMShsm to interact with IBM
MVS™ services and to receive messages from the TSO Terminal
Monitor Program (TMP) and messages that are issued when dynamic
memory allocation occurs.

- **MSYSIN:** Is a system data set that is used by DFSMShsm for support of TSO
processing. It must point to a DUMMY data set.

- **SYSPRINT:** Is the standard DFSMShsm message data set.

- **SYSUDUMP:** Is used to collect dumps that are generated when an error occurs in the
DFSMShsm primary or secondary address spaces. It is used only if the
SETSYS NOSYS1DUMP option was requested.

- **MIGCAT:** Points to the MCDS, which is a required data set. Even if you will not
use DFSMShsm for migration, you must define an MCDS. The starter
set assumes that only a single cluster MCDS will be used.

- **JOURNAL:** Points to the journal data set. This data set is not required, but we
strongly recommend that you use the JRNL data set.

- **ARCLOGX:** Is one of two data sets to which DFSMShsm sends log information. It is
used as an alternative to the data set that is pointed to by ARCLOGY
DD.

- **ARCLOGY:** Is one of two data sets to which DFSMShsm sends log information. It is
used as an alternative to the data set that is pointed to by ARCLOGX
DD.

- **ARCPDOX:** Is one of two trace data sets to which DFSMShsm sends useful
information for debugging. It is used only when the PDA trace is
activated with a **SETSYS** parameter. It is not necessary if **SETSYS**
**PDA(NONE)** is coded in the DFSMShsm set of parameters.

- **ARCPDOY:** Is used in the same way as the trace data set that is pointed to by the
ARCPDOX DD. It is used when the other data set is full.

- **BAKCAT:** Points to the BCDS. The BCDS is used only if you are using backup
processing. Again, the assumption is that only a single cluster BCDS
will be used.

- **OFFCAT:** Points to the OCDS. The OCDS is required only if you will be use
DFSMShsm with tapes.

The starter set provides an empty ARCSTR00 member that is placed into SYS1.PARMLIB.
This member is used to contain unique startup parameters. ARCSTR00 can also contain
additional parameters from the EXEC statement if you exceed the JCL limit of 100 characters
for an EXEC statement.

**ARCCMD00 member**

Tailor your DFSMShsm started tasks through **SETSYS** commands in the DFSMShsm PARMLIB
member ARCCMDxx to ensure that the settings are defined to DFSMShsm at startup
initialization. The general format for the **SETSYS** command is shown:

    SETSYS _parameter_ ( _option_ )


-----

These parameters are contained in the PARMLIB member that is pointed to by the
HSMPARM DD statement in the ARCCMDxx started procedure. The ARCCMD00 that is
created by the STARTER job is not intended to be sufficient for a production DFSMShsm
environment. It does not contain definitions for ML2 and does not create entries for DASD
volumes. It provides a starting place for you to continue tailoring your environment.

These parameters can be divided into four groups that define the following areas:

- Base options
- Space management characteristics
- Availability management characteristics
- ABARS support

This section provides details and suggestions for coding values for scenarios where the
STARTER set does not provide a value or where we are changing the value that is specified
by the starter set that is described in this topic. For more information, see _V1R10.0 DFSMS_
_Storage Administration Reference (for_ _DFSMSdfp_ , _DFSMSdss_ , _DFSMShsm_ _)_ , SC26-7402,
which contains a detailed explanation of each possible **SETSYS** keyword. Topics in this section
are not covered in the same order that the **SETSYS** commands appear in the starter set
ARCCMD00 member.

The base parameters set the basic working options of DFSMShsm. In general, these
parameters do not directly relate to a particular function. Instead, they establish several
defaults, such as tape management, that define the way DFSMShsm implements the
available options.

Define the following base parameters:

- What is the TSO user ID of the DFSMShsm administrator or system programmer?

  The **AUTH** command authorizes this user to issue DFSMShsm commands. The command
can be entered in the input stream as many times as needed to define more than one
authorized user. However, grant CONTROL authority to a few users only because they can
grant authorization to others. The format of this command is shown:

    AUTH _uid_ DATABASEAUTHORITY(USER)
    AUTH _uid_ DBA(CONTROL)

  Even if you will use RACF to control access to DFSMShsm commands, define at least one
user that can issue commands if a problem occurs with the security system.

- What is the job entry subsystem (JES)?

  DFSMShsm defaults to JES2. If you want to use JES3, you must specify the **JES3**
parameter before you specify the first **ADDVOL** command. Specify the JES3 parameter in
the following format:

    SETSYS JES2
    SETSYS JES3

- Do you want secondary host promotion for this DFSMShsm?

  If a primary DFSMShsm host fails, a host that is started as other than PRIMARY= Y is
allowed to take over selected primary host functions. This capability is host promotion. The
promoted host can assume responsibility for all backup functions that are limited to the
primary host and responsibilities for secondary space management (SSM).

If you want this hierarchical storage management (HSM) image to be promoted as primary
host only, use statement (a) in Example 3-1. If you want this HSM image to be
promoted as SSM only, use statement (b) in Example 3-1. Use statement (c)
for this HSM image to be promoted as both primary and SSM.


-----

_Example 3-1  Secondary host promotion options_

    (a) SETSYS PROMOTE(PRIMARYHOST(YES) SSM(NO))
    (b) SETSYS PROMOTE(PRIMARYHOST(NO) SSM(YES))
    (c) SETSYS PROMOTE(PRIMARYHOST(YES) SSM(YES))

DFSMShsm host promotion is described in more depth in 3.4.7, “Host promotion”.

Do you want only some hosts to perform certain functions?

Do you want DFSMShsm to reblock eligible data sets during data set recall or recovery?

All system reblockable data sets are reblocked during recall and recovery if they require
reblocking. If data sets are not system reblockable but are of selected categories, this
parameter permits DFSMShsm to reblock them. If you do not want these data sets to be
reblocked, use statement format (a) in Example 3-2. The format (b) statement in
Example 3-2 allows reblocking during recall or in recover to any device type that is
supported by DFSMShsm, including target volumes of the same type as the source
volume.

_Example 3-2  Reblocking options_

    (a) SETSYS NOCONVERSION
    (b) SETSYS CONVERSION(REBLOCKTOANY)

How do I specify data set serialization for data sets that are being backed up or migrated?

To prevent a data set from being changed during backup or migration, access to the data
set is controlled by using serialization. The DFSMShsm serialization mechanism is
determined by specifying one of the following **SETSYS** parameters.

Use statement (a) in Example 3-3 when you are sharing volumes and a serialization
facility, such as GRS, is not provided. In this case, a reserve is placed on the volume. An
example of sharing volumes without GRS is with a VM or VSE system.

Use statement (b) in Example 3-3 when you are in a single-system environment or are
sharing data between z/OS systems and are using GRS to serialize at the data set level.

We strongly recommend that you specify SETSYS USERDATASETSERIALIZATION
where you can.

_Example 3-3  Data set serialization specification_

    (a) SETSYS DFHSMDATASETSERIALIZATION
    (b) SETSYS USERDATASETSERIALIZATION

The following considerations apply to data set serialization:

- The Fast Subsequent Migration function supports data set reconnection in a
USERDATASETSERIALIZATION environment only.

- Certain data sets, such as multivolume physical sequential data sets, are processed
with SETSYS USERDATASETSERIALIZATION only.

- In a multiple-system environment, do not specify USERDATASETSERIALIZATION
unless you installed and enabled a data set serialization facility on your systems.
Otherwise, serious data integrity problems can occur.

If you use SETSYS DFHSMDATATSERIALIZATION, DFSMShsm will not select data sets
for processing where either the creation date or the date last altered is today.


-----

Do you want to activate a DFSMShsm exit?

You can use DFSMShsm installation exits to customize DFSMShsm processing according
to your installation requirements. These exits are described in _z/OS V2R10.0 DFSMS_
_Installation Exits,_ SC26-7396-14, where you can obtain additional information about the
DFSMShsm exit points.

If you are not planning to use a DFSMShsm exit, specify statement (a) in Example 3-4.

Statement (b) in Example 3-4 shows how to activate an exit. In this example, we activate
the DFSMShsm tape volume exit ARCTVEXT. This exit is called when a tape that is
owned by DFSMShsm no longer contains valid data and therefore becomes empty. The
ARTVEXT exit is used to tell the OEM tape management systems that DFSMShsm
released the ownership of a DFSMShsm tape. If you use DFSMSrmm, ARCTVEXT is not
required. However, for an independent software vendor (ISV) tape management system
(TMS) product, use statement (b).

_Example 3-4  DFSMShsm exit options_

    (a) SETSYS EXITOFF(TV)
    (b) SETSYS EXITON(TV)

**Note:** If you installed an ISV TMS, specify statement (b) in Example 3-4 and ask the
tape management system vendor for its version of ARCTVEXT.

Do you want to write updated CDS records in the journal data set?

Journaling is necessary because it is the only way to provide point-in-time forward
recovery if a CDS is damaged. The **JOURNAL** parameter specifies that DFSMShsm write
the BCDS, MCDS, and OCDS records in the journal data set when it updates them. The
following statement guarantees that control is not returned until each CDS change is
written to both the CDS and JRNL data set:

`SETSYS JOURNAL(RECOVERY)`

The JRNL data set is performance-critical and needs to be allocated on a
high-performance device with write caching enabled.

We strongly recommend both using the JRNL data set and specifying SETSYS
JOURNAL(RECOVERY). Without this setting, data that is required for DFSMShsm CDS
recovery will not be guaranteed to be recorded.

Do you want System Measurement Facility (SMF) to contain DFSMShsm statistics?

If yes, specify statement (a) in Example 3-5 with an SMF user code SMFID to assign it to
records that are generated by DFSMShsm. DFSMShsm uses two consecutive user codes
(SMFID and smfid+1). The fields are required by the **REPORT** command. If you do not want
DFSMShsm to write SMF records, use statement (b) in Example 3-5.

_Example 3-5  Options to set SMF to contain DFSMShsm statistics_

    (a) SETSYS SMF(smfid)
    (b) SETSYS NOSMF

Which SYSOUT class do you want to assign to the DFSMShsm output?

With this command, the class and number of copies are assigned to the DFSMShsm
procedure output. DFSMShsm defaults are class A and one copy (A 1). Use the following
statement to assign the class and number of copies to the DFSMShsm procedure output:

`SETSYS SYSOUT( _class_ _copies_ )`


-----

Do you want the DFSMShsm dump to be written in a system dump data set?

With statement (a) in Example 3-6, which we recommend, dump is written in a system
dump data set every time that an error occurs in the primary or secondary address space.
This format is required if you are using Interactive Problem Control System (IPCS). With
statement (b) in Example 3-6, DFSMShsm dumps are written where indicated with a
SYSABEND, SYSUDUMP , or SYSMDUMP DD statement in the DFSMShsm start
procedure.

_Example 3-6  Writing the DFSMShsm dump to a system dump data set options_

    (a) SETSYS SYS1DUMP
    (b) SETSYS NOSYS1DUMP

Do you want DFSMShsm to write all activity log messages?

ACTLOGMSGLVL determines which ARC0734I data set movement messages will be
written to the activity log. Statement (a) in Example 3-7, which we recommend, specifies
that messages will be generated and logged for all activities. Statement (b) in Example 3-7
specifies that messages will be generated only for activities with a nonzero return code.
Statement (c) specifies that only the original space management message is generated.

_Example 3-7  Options for writing activity log messages_

    (a) SETSYS ACTLOGMSGLVL(FULL)
    (b) SETSYS ACTLOGMSGLVL(EXCEPTIONLY)
    (c) SETSYS ACTLOGMSGLVL(REDUCED)

Do you want DFSMShsm to write the activity log messages on DASD or in a SYSOUT
queue?

With ACTLOGTYPE, you can choose whether to create the activity logs as DASD data
sets (statement (a) in Example 3-8) or SYSOUT data sets (statement (b) in Example 3-8).
For class, substitute an alphanumeric character for the class DFSMShsm is to use for
output. DASD specifies that DFSMShsm dynamically allocate DASD data sets with the
HLQ HSMACT.

If the activity logs are written to DASD, the activity logs can be managed by DFSMShsm
according to the SMS management class specifications. If the activity logs are written to
SYSOUT, you will need to arrange for this data to be archived, for example, by being sent
to an external writer.

_Example 3-8  Writing the activity log messages to DASD or SYSOUT_

    (a) SETSYS ACTLOGTYPE(DASD)
    (b) SETSYS ACTLOGTYPE(SYSOUT(class))

Do you want DFSMShsm to print information about the system console and in the activity
logs?

**MONITOR** is an optional parameter that specifies the informational messages that
DFSMShsm will print at the system console and the values that DFSMShsm will use for
monitoring space in the journal and CDSs. Do not code this parameter if you do not want
DFSMShsm monitor messages on the system console and SYSLOG or OPERLOG.


-----

Otherwise, you can select the information that you want by coding **STARTUP** for messages
at DFSMShsm initialization. The **SPACE** subparameter is used to print the volume
space-use messages at the system console, during space management. The **VOLUME**
subparameter specifies that data set processing messages (ARC0734I) are printed on the
system console. You can suppress messages from the console by using PARMLIB
MPFLSTxx or your automation product to keep the messages available in SYSLOG or
OPERLOG for later review. Specify where to direct DFSMShsm informational messages:

`SETSYS MONITOR(STARTUP SPACE VOLUME)`

Do you want warning messages on space utilization inside the CDSs and the journal?

Code this parameter only if you want DFSMShsm to change the default threshold value for
the CDSs and journal occupancy, that is, 80 %:

`SETSYS MONITOR(JOURNAL(70) BCDS(70) MCDS(70) OCDS(70))`

Do you want to limit the amount of space that DFSMShsm allocates in the common
service area (CSA)?

CSA storage is used to save WAIT Management Work Elements (MWEs). Reasonable
starting values, which are also the default values, are shown in statement (a) in
Example 3-9. If you do not want to limit the amount of space that is allocated in the CSA,
choose statement (b).

_Example 3-9  Options to limit the amount of space that DFSMShsm allocates in the CSA_

    (a) SETSYS CSALIMITS(MWE(4) MAXIMUM(100) ACTIVE(90) INACTIVE(30))
    (b) SETSYS NOCSALIMIT

How many tasks do you want to assign to the RECYCLE process?

Although a single DFSMShsm host can process only one RECYCLE request at a time,
RECYCLE can initiate up to 15 tape processing tasks. Each RECYCLE task allocates two
tape drives. Through the following statement, you can limit the number of tape drives that
RECYCLE will request:

    SETSYS MAXRECYCLETASKS( _nn_ )

Do you want to use the hardware compaction feature on 3480X tape devices?

The improved data recording capability (IDRC) feature on 3480s, if installed, can be used
on DFSMShsm 3480X output tapes if you code statement (a) in Example 3-10. If you do
not want to use this feature on 3480X devices, code statement (b). This parameter is
ignored for 3490 and 3590 devices.

_Example 3-10  Hardware compaction feature on the 3480X tape drives_

    (a) SETSYS TAPEHARDWARECOMPACT
    (b) SETSYS NOTAPEHARDWARECOMPACT

Do you want to reuse partially full tapes?

To enable each task, except for data set migration, to initially request a scratch tape, use
statement (a) in Example 3-11. Data set migration is excluded to avoid
marking a tape volume full after a command migration of a single data set to ML2. This
option is valuable when you use automatic cartridge loaders. However, this option is not
useful if the DFSMShsm tape output is directed to high-capacity tapes, such as the IBM
Magstar® media tape, because they will not be used well.

If you want DFSMShsm to append to tapes that were not marked full when they were last
written to, use statement (b) Example 3-11. With statement (b), DFSMShsm
marks tapes full only when the maximum block count that is specified in the SETSYS
**TAPEUTILIZATION** parameter is reached.


-----

DFSMShsm automatically processes TAPECOPY requests for tape volumes that are
marked as full, and partial tapes without alternates are not selected for copy.

If you specify the global PARTIALTAPE parameter, it applies to both migration and backup.
If you specify a specific function, the parameter applies only to that function. If you specify
a global value and a specific function in a single command, the command fails. If you want
to specify this parameter differently for migration and backup, do not code the sample that
is shown here. Use statement (c) or (d) instead.

_Example 3-11  Handling partial tapes_

    (a) SETSYS PARTIALTAPE(MARKFULL)
    (b) SETSYS PARTIALTAPE(REUSE)
    (c) SETSYS PARTIALTAPE(MIGRATION(MARKFULL | REUSE)
    (d) SETSYS PARTIALTAPE(BACKUP(MARKFULL | REUSE)

When you use a Virtual Tape Server (VTS) subsystem for migration or backup output, the
**MARKFULL** parameter can improve performance by reducing the need to remount
yesterday’s partial volume to extend it today. Additionally, this usage can reduce the
reclamation process of the VTS subsystem because the physical tape, which contains the
initial partial tape that was extended, was not invalidated on a true tape.

Do you want to reduce the occurrences of data sets that span tape volumes?

In the following statement, you specify the maximum number of megabytes of tape that
DFSMShsm can leave unused on tape while trying to eliminate spanning data sets:

    SETSYS TAPESPANSIZE( _nnn_ )

When space is insufficient to contain the entire next data set on the current tape without
exceeding the requested maximum utilization, the next data set begins on an empty tape.
The default value for this parameter is 500 MB. However, IBM recommends a value of
4000 MB for all device types from 3480 through 3592 tape cartridges.

Do you want DFSMShsm to wait when a tape allocation is requested?

If you select statement (a) in Example 3-12, DFSMShsm continues other processing when
no device is immediately available and reissues the allocation request every 10 seconds
for one minute. After seven attempts, DFSMShsm asks the operator whether to try again
or fail the allocation and therefore the function. If the operator prompt is issued,
DFSMShsm waits for the write to operator with reply (WTOR) to be answered even if the
drives that were requested are now available. We recommend that you use the **NOWAIT**
parameter.

If you want DFSMShsm to wait until a tape request is satisfied, use statement (b) in
Example 3-12. In statement (b), all DFSMShsm functions that request allocations, opens,
or closes are stopped until this request is satisfied. JES3 installations are forced to use
**WAIT** because JES3 schedules tape drives.

_Example 3-12  Setting DFSMShsm wait option when a tape allocation is requested_

    (a) SETSYS INPUTTAPEALLOCATION(NOWAIT)
    SETSTS OUTPUTTAPEALLOCATION(NOWAIT)
    SETSYS RECYCLETAPEALLOCATION(NOWAIT)
    
    (b) SETSYS INPUTTAPEALLOCATION(WAIT)
    SETSYS OUTPUTTAPEALLOCATION(WAIT)
    SETSYS RECYCLETAPEALLOCATION(WAIT)


-----

Do you want DFSMShsm to use its own tape pool?

With statement (a) in Example 3-13, DFSMShsm requests scratch tapes every time that a
new volume is needed for output and every time that a continuation volume is needed at
end of volume (EOV). The SELECTVOLUME specification is typically made with the
TAPEDELETION specification. So, according to the TAPEDELETION specification in
statement (a), when DFSMShsm no longer needs a tape volume, it tells the TMS that it
released the ownership of that tape volume. The TMS is then responsible for returning
tape volumes to the global scratch pool.

With statement (b) in Example 3-13, DFSMShsm selects the tape within a defined tape
pool if any are available. It is expected that the installation supplied a set of standard label
tapes and identified them to DFSMShsm, by using the **ADDVOL** command. When a tape
volume no longer contains valid data, the TAPEDELETION specification in statement (b)
tells DFSMShsm to keep it for reuse. If you specify the global parameter, it applies to all
functions (migration, backup, and dump). If you specify the parameter for a specific
function, it applies only to that type of function.

We recommend that you not use specific scratch pools. With global scratch pools, you can
reduce the overhead of needing to manage specific pools of tapes that are dedicated to
DFSMShsm. Global scratch pools are more easily integrated with tape automation
solutions.

_Example 3-13  Setting whether DFSMShsm uses its own tape pool_

    (a) SETSYS SELECTVOLUME(SCRATCH)
    SETSYS TAPEDELETION(SCRATCHTAPE)
    
    (b) SETSYS SELECTVOLUME(SPECIFIC)
    SETSYS TAPEDELETION(HSMTAPE)

Do you want to use installation-defined esoteric unit names for tape allocations?

DFSMShsm always requests a mount for a specific tape for input processing, so cartridge
loaders are of little value for input. To ensure that non-cartridge-loader devices are used
for input, you can direct DFSMShsm to use esoteric unit names in a special way that
directs a cartridge to be allocated on a different set of devices for input and for output.
Esoteric tape unit names that were defined to your z/OS system must be defined to
DFSMShsm before they can be recognized and used in DFSMShsm commands as valid
unit types. All esoterics that are identified to DFSMShsm with the **SETSYS USERUNITTABLE**
command must appear in a single command.

To identify esoteric tape unit names to DFSMShsm, use statement (a) in Example 3-14 on
page 44. If an esoteric name represents a set of units with automatic cartridge loaders,
and the esoteric name is used to allocate a device for output, the output esoteric is
translated to the generic tape unit equivalent for mounting the tape for input. If you do not
want that translation, you can specify an alternate unit name translation, such as the
example that is shown in statement (b) in Example 3-14 on page 44. Substitute eso with
the esoteric name that you want to use for output allocations, and esi with the esoteric
name that you want to use for input allocations. In statement (b), eso designates tape
drives with automatic cartridge loaders, and esi designates tape drives without automatic
cartridge loaders.

You can use statement (c) in Example 3-14 on page 44 if you do not want the translation to
occur for certain esoteric names. In this example, a tape that was written by DFSMShsm
to a tape that is associated with the esoteric name eso1 will be allocated for input in a tape
unit that is also associated with eso1 .


-----

If no esoteric tape unit names are identified to DFSMShsm, use statement (d) in
Example 3-14. Any previously defined esoteric names are no longer known to
DFSMShsm. Statement (d) is the default.

_Example 3-14  Using installation-defined esoteric unit names for tape allocations_

    (a) SETSYS USERUNITTABLE(eso1,eso2,eso3,..)
    (b) SETSYS USERUNITTABLE(eso1:esi1,eso2:esi2...)
    (c) SETSYS USERUNITTABLE(eso1:eso1,eso2:eso2...)
    (d) SETSYS NOUSERUNITTABLE

How long do you want DFSMShsm to wait for a tape mount?

Indicate the number of minutes (maximum 120 ) that DFSMShsm can wait for a tape mount
before it asks the operator about volume availability. The DFSMShsm default is 15
minutes. The following statement shows that DFSMShsm waits 10 minutes for a tape
mount:

`SETSYS MOUNTWAITTIME(10)`

Do you want DFSMShsm to ask the operator about the availability of tapes for input before
it uses them?

If you want DFSMShsm to ask the operator whether the requested input tapes are
available, use statement (a) in Example 3-15. If you do not want DFSMShsm to issue this
operator request, use statement (b) in Example 3-15. If all of the requested input tapes are
in an IBM automated tape library, these action messages are not issued, even if
(...TAPES(YES)) is specified.

_Example 3-15  Options for DFSMShsm to ask the operator about input tape availability_

    (a) SETSYS TAPEINPUTPROMPT(MIGRATIONTAPES(YES))
    SETSYS TAPEINPUTPROMPT(BACKUPTAPES(YES))
    SETSYS TAPEINPUTPROMPT(DUMPTAPES(YES))
    
    (b) SETSYS TAPEINPUTPROMPT(MIGRATIONTAPES(NO))
    SETSYS TAPEINPUTPROMPT(BACKUPTAPES(NO))
    SETSYS TAPEINPUTPROMPT(DUMPTAPES(NO))

Do you want to create a duplex duplicate of your migration and backup tapes?

The **DUPLEX** parameter allows DFSMShsm to create two tapes concurrently. The two
copies of data can be written to the same or different locations. By using statement (a) in
Example 3-16, you can duplex both BACKUP and MIGRATION tapes. If you do not want to
duplex your tapes, use statement (b) in Example 3-16, or do not specify the **DUPLEX**
parameter on the **SETSYS** command. Use statement (c) if you want to duplex migration
tapes only and statement (d) for duplexing backup tapes only.

_Example 3-16  Options for creating a DUPLEX duplicate of migration and backup tapes_

    (a) SETSYS DUPLEX(MIGRATION(Y) BACKUP(Y))
    (b) SETSYS DUPLEX(MIGRATIOM(N) BACKUP(N))
    (c) SETSYS DUPLEX(MIGRATIOM(Y) BACKUP(N))
    (d) SETSYS DUPLEX(MIGRATIOM(N) BACKUP(Y))


-----

Do you want to limit the maximum tape utilization?

DFSMShsm provides the **TAPECOPY** command and the **DUPLEX** tape option to create
alternate migration and backup copies of cartridge-type tapes. The usefulness of these
copies depends on a one-to-one cartridge copy. Because not all cartridges of a certain
type have the same capacity, DFSMShsm writes only 97% of the capacity of a cartridge,
by default. If you want to change the amount of data that is written to migration and backup
tapes, use statement (a) in Example 3-17. You might need to issue the **SETSYS**
**TAPEUTILIZATION** command for each tape unit type that you use if you want to change a
particular default.

The **LIBRARYMIGRATION** and **LIBRARYBACKUP** subparameters are the only vehicle by which
you can limit the amount of media that is used in a library migration cartridge because
esoteric unit names are ignored in an SMS-managed tape library.

In a duplex tape environment, if you specify NOLIMIT , DFSMShsm uses the default
PERCENTFULL value of 97 %. This value is not likely to be a problem unless you are
targeting a 3590-Bx or vendor drive in 3490 emulation mode. These drives need a percent
value of a few hundred to fully use the tape’s capacity. The value of 97 % is fine if you are
targeting a 3590-Ex drive, even if it is in 3490 emulation mode.

IBM recommends that you use 97 (default) in all cases. If you are using the IBM 3590-Bx
drives in emulation mode, the recommended value for PERCENTFULL is 2200 . (For drives
with a microcode level before 1.9.19.0, use a value of 1100 instead.)

IBM provides TAPEUTILIZATION recommendations for new drives and media as they
become available, usually with the program temporary fixes (PTFs) that provide new
function support.

For OEM drives and media, request this information from your vendor.

_Example 3-17  Limiting the maximum tape utilization_

     (a) SETSYS TAPEUTILIZATION(UNITTYPE(unit) PERCENTFULL(95))
     SETSYS TAPEUTILIZATION(LIBRARYMIGRATION PERCENTFULL(95))
     SETSYS TAPEUTILIZATION(LIBRARYBACKUP PERCENTFULL(95))
     
     (b) SETSYS TAPEUTILIZATION(UNITTYPE(unit) PERCENTFULL(100))
     SETSYS TAPEUTILIZATION(LIBRARYMIGRATION PERCENTFULL(100))
     SETSYS TAPEUTILIZATION(LIBRARYBACKUP PERCENTFULL(95))
     
     (c) SETSYS TAPEUTILIZATION(UNITTYPE(unit) PERCENTFULL(1100))

Do you want all migrated and backup copies on DASD to be RACF indicated?

If the RACF environment is set up with always-call support and generic profiles that are
defined for migration and backup qualifiers, we recommend that you use statement (a) in
Example 3-18. Migration copies and backup versions will not be RACF indicated.

Use statement (b) in Example 3-18 to indicate RACF for migration copies and backup
versions.

_Example 3-18  RACF indication options for migrated and backup copies_

    (a) SETSYS NORACFIND
    (b) SETSYS RACFIND


-----

How do you want to protect DFSMShsm tapes?

DFSMShsm requires that you select one of three ways to protect tapes. If you want to
protect them with RACF, use statement (a) in Example 3-19. 
This method is recommended if RACF is installed. If RACF is not installed,
you can use statement (b) in Example 3-19. This method protects tapes with the 99365
expiration date. If you choose statement (c) in Example 3-19, DFSMShsm keeps tapes as
password-indicated so that only security-authorized programs can access them. If you
choose statement (d), you are using RACF and also expiration dates for protecting
DFSMShsm tapes. These options are fully supported by other security software products,
such as CA-TOPSECRET or CA-ACF2. Security options are not mutually exclusive; they
can be used in combination.

_Example 3-19  Options to protect DFSMShsm tapes_

    (a) SETSYS TAPESECURITY(RACF | RACFINCLUDE)
    (b) SETSYS TAPESECURITY(EXPIRATION | EXPIRATIONINCLUDE)
    (c) SETSYS TAPESECURITY(PASSWORD)
    (d) SETSYS TAPESECURITY(RACF EXPIRATION)

Do you want the system to overwrite deleted migration and backup copies of data sets
according to the RACF erase-on-scratch flag?

Statement (a) in Example 3-20 applies only if RACF is installed. It controls the overwriting
after a migration or backup copy is deleted from a DASD volume that is owned by
DFSMShsm, only if the erase-on-scratch option is associated with the RACF profile of the
user’s data set. Specify statement (b) in Example 3-20 if you do not need this overwriting
to occur.

_Example 3-20  Options to overwrite deleted migration and backup copies of data sets_

    (a) SETSYS ERASEONSCRATCH
    (b) SETSYS NOERASEONSCRATCH

Specifying SETSYS ERASEONSCRATCH can cause a performance impact.

Do you want to control whether individual backup requests go to DASD or tape?

With the data set backup commands, a user can target a specific output device type.
However, if an output device type is not specified, the **SETSYS DSBACKUP** command controls
whether individual backup requests go to DASD or tape. With this command, you can also
define the number of tasks that are available, up to 64 tasks, to the data set backup
function. The **SETSYS DSBACKUP** command has three parameters: **DASDSELECTIONSIZE** , **DASD** ,
and **TAPE** .

The **DASDSELECTIONSIZE** **(** **_maximum_** **_standard_** **)** parameter on the **SETSYS DSBACKUP**
command allows DFSMShsm to balance the workload between DASD and tape tasks for
all WAIT requests that do not target a device type:

    SETSYS DSBACKUP DASDSELECTIONSIZE( _maximum_ _standard_ )

DFSMShsm uses a selection process that is based on data set sizes and the availability of
tasks. If the data set is greater than a specified threshold maximum, the data set is
directed to tape. Any data set that is less than or equal to a specified threshold standard is
directed to the first available task, either DASD or tape.

To direct the data set to back up to tape, use statement (b) in Example 3-21. Use
statement (a) in Example 3-21 for DASD only. This SETSYS directs all backups to one or
the other. You can also direct individual requests with the **TARGET** keyword.


-----

_Example 3-21  Directing the data set to back up to tape or DASD_

    (a) SETSYS DSBACKUP(DASD(TASK(0)))
    (b) SETSYS DSBACKUP(TAPE(TASK(0)))

Do you want RACF discrete profiles to be copied during the data set backup?

This parameter applies only if RACF is installed. If you use generic profiles with
always-call support, select statement (a) in Example 3-22. However, if you do not have an
always-call environment, use only discrete profiles and specify statement (b).

_Example 3-22  Options to copy RACF discrete profiles during the data set backup_

    (a) SETSYS NOPROFILEBACKUP
    (b) SETSYS PROFILEBACKUP

Do you want to issue authorized DFSMShsm commands in a TSO batch environment
without RACF?

To issue authorized DFSMShsm commands in a TSO batch environment without RACF,
the user ID that is associated with the job must match the user ID that was entered with
the **AUTH** command. If RACF is installed, RACF passes this information to DFSMShsm. In
this case, use statement (a) in Example 3-23. If RACF is not installed, use statement (b).
An installation without RACF has DFSMShsm retrieve the user ID for TSO batch requests
from the protected step control block (PSCB). An installation must ensure that the user ID
is placed in the PSCB.

_Example 3-23  Issuing authorized DFSMS commands in a TSO batch environment_

    (a) SETSYS NOACCEPTPSCBUSERID
    (b) SETSYS ACCEPTPSCBUSERID

Do you want DFSMShsm to compact data sets during migration or backup?

If you want DFSMShsm to compact data sets during backup and migration to DASD and
during backup and migration to tape, specify statement (a) in Example 3-24. Optionally,
you can choose the functions that use DFSMShsm compaction. For example, if you want
compaction to occur during migration to DASD only, specify statement (c). We recommend
that you specify statement (d). If you do not want compaction to occur, do not specify the
**SETSYS COMPACT** command.

If most of your migration and backup data goes to IDRC-capable tape devices or RAMAC
Virtual Array (RVA) DASD, our recommendation is to not specify the **SETSYS COMPACT**
command.

_Example 3-24  SETSYS COMPACT command options_

    (a) SETSYS COMPACT(ALL)
    (b) SETSYS COMPACT(DASDBACKUP)
    (c) SETSYS COMPACT(DASDMIGRATE)
    (d) SETSYS COMPACT(DASDBACKUP DASDMIGRATE)
    (e) SETSYS COMPACT(TAPEBACKUP)
    (f) SETSYS COMPACT(TAPEMIGRATE)

Do you want to specify the percentage of minimum space savings for software
compaction?


-----

If you request software compaction, DFSMShsm compares any history that it has of the
number of bytes that are written to the total bytes of the original data set and computes the
percentage of bytes that are saved by compaction. If the saved percentage is not greater
than or equal to the percentage you specified, DFSMShsm will not compact the data set
during subsequent migrations or backups. The DFSMShsm percentage saved default is
40 . Use the following statement to specify the percentage of minimum space savings for
software compaction:

`SETSYS COMPACTPERCENT( _pct_ )`

Do you want to optimize DFSMShsm compaction?

If you use the DFSMShsm data compaction feature, you can optimize the algorithm that
associates the source and object compaction tables with the data set types. The algorithm
passes the last-level qualifier of the data set names that belong to these two groups to
DFSMShsm with the following statements.

_Example 3-25  Optimizing DFSMShsm compaction_

    SETSYS SOURCENAMES(src1,src2,......)
    SETSYS OBJECTNAMES(obj1,obj2,.......)

Do you want DFSMShsm to use IBM z Systems™ Enterprise Data Compression (zEDC)
for z/OS hardware compression?

zEDC hardware compression is I/O adapter-based hardware compression that was
introduced on IBM zEnterprise® EC12 and IBM BC12 servers. The server requires the
zEDC Express feature for the PCI Express (PCIe) I/O drawer (licensed feature number
FC0420).

For the operating system, you need the PTFs for zEDC Express for z/OS feature and for
queued sequential access method (QSAM) and basic sequential access method (BSAM)
zEDC exploitation (see authorized program analysis report (APAR) OA42195) with
coexistence support on z/OS V1.12 and z/OS V1.13.

For DFSMShsm, you must activate compression in the IGDSMSxx member and set up the
zEDC compression in the DFSMShsm startup member. You can choose different levels to
implement the zEDC compression as shown in Example 3-26.

_Example 3-26  zEDC hardware compression options in DFSMShsm migration and backup_

    SETSYS ZCOMPRESS(ALL NONE)
    SETSYS ZCOMPRESS(DASDBACKUP(YES))
    ETSYS ZCOMPRESS(DASDMIGRATE(YES))
    SETSYS ZCOMPRESS(TAPEBACKUP(YES))
    SETSYS ZCOMPRESS(TAPEMIGRATE(YES))

For DFSMShsm DUMP processing, you can specify zCOMPRESS(YES) on the
DUMPCLASS if you want to activate zEDC compression for DFSMShsm dumps, also.

For more information about the server setup for zEDC compression, see the _IBM_
_zEnterprise EC12 Technical Guide,_ SG24-8049, and the _IBM zEnterprise BC12 Technical_
_Guide_ , SG24-8138. For more information about the specific DFSMShsm setup, see _zEDC_
_Compression: DFSMShsm Sample Implementation_ , REDP-5158.

Do you want DFSMShsm to load DFSMSdss in separate address spaces?

Enabling DSSXMMODE causes workload that requires DFSMSdss to be offloaded to
secondary address spaces. This offloading reduces contention for resources in the
primary DFSMShsm address space. However, you need to perform additional setup steps
beyond the **SETSYS** command.


-----

_Example 3-27  Options for loading DFSMSdss in separate address spaces_

    SETSYS DSSXMMODE(Y)
    SETSYS DSSXMMODE(N)

- Do you want DFSMShsm to automatically generate a recycle command when recall takes
a tape away from recycle?

When a tape takeaway occurs, a new recycle command for the original tape is generated
automatically by using the following command:

`SETSYS RECYCLETAKEAWAYRETRY(YES MAXRETRYATTEMPTS( _nn_ ) DELAY( _mmmm_ | no)`

This function is helpful because the storage administrator no longer needs to perform this
task manually.

**ARCSTR00 member**
This member contains additional startup parameters as specified in the **STR=00** parameter of
the DFSMShsm startup procedure. The sample member that is provided by the STARTER job
is an empty member, which is shown in Figure 3-3.

    /*****************************************************************/
    /*       DFSMSHSM ADDITIONAL STARTUP COMMAND MEMBER    */
    /*****************************************************************/
    EMERG=NO,STARTUP=YES,CDSSHR=YES,CDSQ=YES,CDSR=NO

_Figure 3-3  Additional startup parameters_

The ARCSTR00 member can contain the following additional parameters:

**STARTUP** Specifies whether startup messages are printed at the operator
console. The default value is NO .

**CDSSHR** Specifies that DFSMShsm runs in a particular multiple processor or
single processor environment instead of letting DFSMShsm determine
the environment. If you specify NO for this keyword, DFSMShsm
performs no multiple processor serialization: no other processor must
be concurrently processing the CDSs. Specifying YES for this keyword
causes DFSMShsm to perform multiple processor serialization of the
type that is requested by the **CDSQ** and **CDSR** keywords. If your system
supports VSAM RLS, you can specify CDSSHR= RLS to cause
DFSMShsm to perform multiprocessor serialization by using RLS
(serialization at a record level). The CDSs are accessed in RLS mode,
and any values that are specified for CDSQ and CDSR are ignored.
Other MVS images access the CDSs concurrently.

**CDSR** Specifies that DFSMShsm serializes its CDSs with volume reserves. If
you use a cross-system serialization product, you can specify
CDSQ= YES instead. CDSQ= YES serializes the CDSs (among multiple
MVS images) with a global (SYSTEMS) exclusive enqueue but allows
other tasks within a single MVS image to access the CDSs
concurrently. When a serialization technique is not specified, the
default serialization technique depends on the following specified
HOSTMODE. If HOSTMODE= MAIN , DFSMShsm assumes
CDSR= YES . If HOSTMODE= AUX , DFSMShsm indicates an error with
message ARC0006I.


-----

**RNAMEDSN** Specifies whether to use a new serialization method so that no
interference occurs between HSMplexes that are contained within a
single GRSplex. When you specify YES for this parameter, you are
invoking the new method of serialization, which uses the data set
name of the CDSs and the journal. The default value is NO .

**RESTART** Specifies that DFSMShsm must be restarted for all DFSMShsm
abnormal ends (ABENDS). The format of the keyword is
**RESTART** =( _a_ , _b_ ), where _a_ specifies the name of the procedure to be
started, and _b_ specifies any additional keywords or parameters to be
passed to the procedure. If you access the CDSs in RLS mode, use
the **RESTART** keyword so that DFSMShsm automatically restarts after
an SMS VSAM server error.

**LOGSW** Specifies whether to automatically swap the log data sets at startup.
The default value is NO .

**CELLS** DFSMShsm uses the cell-pool (CPOOL) function of MVS to obtain
and manage virtual storage in its address space for the dynamically
obtained storage for certain high-usage modules, and for data areas
DFSMShsm frequently gets and frees. The **CELLS** parameter provides
the cell sizes for five cell pools. The default value is
( 200 , 100 , 100 , 50 , 50 ).

### 3.2.2 ALLOCBK1

ALLOCBK1 allocates CDS backups: four copies of each of the MCDS, BCDS, OCDS, and
JRNL. This function assumes that you are writing backups of the CDS to DASD. DFSMShsm
backups can be written to either tape or DASD. We recommend writing CDS backups to
DASD where possible.

We also strongly recommend increasing the number of backups to provide a week’s worth of
data at a minimum. The number of backup copy data sets that DFSMShsm uses is
determined by the **SETSYS CDSVERSION BACKUPCOPIES** parameter. You must have at least as
many data set backup copies as are allocated because the specification of **BACKUPCOPIES** or
**CDS** backup fails at a certain point.

### 3.2.3 ALLOSDSP

You need to run this job only if you use SDSP data sets. This job allocates and initializes a
single SDSP data set. If you defined ML1 volumes with ADDVOL SDSP , you need to allocate
and initialize one SDSD for each ML1 volume.

### 3.2.4 ARCCMD01

DFSMShsm SETSYS specifies whether to use tape for ML2. This member needs to be
copied into the DFSMShsm startup member that is created by the STARTER job. Figure 3-4
shows the contents of the ARCCMD01 member.


-----
    
    /****************************************************************/
    /* DFSMSHSM STARTUP COMMAND MEMBER FOR LEVEL 2 TAPE MIGRATION */
    /*                               */
    /* APPEND THIS COMMAND STREAM TO ARCCMD00 TO PROVIDE LEVEL 2  */
    /* TAPE MIGRATION                       */
    /****************************************************************/
    /****************************************************************/
    /*     DFSMSHSM LEVEL 2 TAPE MIGRATION PARAMETERS     */
    /****************************************************************/
    /*                               */
    
    SETSYS -
    TAPEMIGRATION(ML2TAPE)  /* MIGRATE TO LEVEL 2 TAPE.     */
    
    SETSYS -
    MIGUNITNAME(3590-1)    /* START WITH 3590-1 ML2 TAPE    */
    /* UNIT.              */
    SETSYS -
    ML2RECYCLEPERCENT(20)   /* LOG MESSAGE WHEN VALID DATA   */
    /* ON AN ML2 TAPE FALLS BELOW    */
    /* 20%.               */
    /* 20%.               */
    SETSYS -
    TAPEMAXRECALLTASKS(1)   /* ONE TAPE RECALL TASK AT A TIME  */
    
    /*                               */
    /****************************************************************/
    /* SEE MEMBER ARCCMD91 IN HSM.SAMPLE.CNTL FOR AN EXAMPLE    */
    /* OF ADDVOL COMMANDS TO BE USED IN CONJUNCTION WITH LEVEL   */
    /* 2 TAPE MIGRATION.                      */
    /****************************************************************/
    
_Figure 3-4  ARCCMD01 member from the DFSMShsm starter set_

### 3.2.5 ARCCMD90

This member provides DFSMShsm **ADDVOL** sample commands. Place these commands at the
end of the ARCCMD00 member that is created by the STARTER job.

**SETSYS** commands are required for any non-SMS primary volumes and ML1 volumes.

### 3.2.6 ARCCMD91

The starter set provides a sample **ADDVOL** command for ML2 tape volumes. This command is
shown in the Figure 3-5.


-----

    /*                               */
    /****************************************************************/
    /* THE FOLLOWING EXAMPLE ADDVOL COMMANDS CAN BE USED WITH THE  */
    /* ARCCD00 MEMBER OF THE STARTER SET TO IDENTIFY LEVEL 2 TAPE  */
    /* MIGRATION VOLUMES. AFTER YOU HAVE ADDED A VOLUME SERIAL   */
    /* NUMBER AND A UNIT TYPE IN THE SPACE PROVIDED, APPEND THIS  */
    /* COMMAND STREAM TO YOUR ARCCMD00 MEMBER.           */
    /*                               */
    /****************************************************************/
    /*                               */
    ADDVOL ______        /* ADD A VOLUME (PROVIDE SERIAL)  */    MIGRATION      /* AS A MIGRATION LEVEL 2 TAPE   */     (MIGRATIONLEVEL2)  /* VOLUME.             */    UNIT(______)     /* PROVIDE PROPER UNIT TYPE.    */
    /*                               */
    
_Figure 3-5  Starter set ARCCMD91 member_

These commands do not need to be added to your ARCCMD00 member. ADDVOL for ML2
volumes needs to be issued only if you use a dedicated tape pool for DFSMShsm. If you plan
to use scratch volumes for DFSMShsm input, you do not need to pre-define volumes by using
**ADDVOL** commands.

### 3.2.7 HSMHELP

Add HSMHELP to a data set in your ISPF HELP concatenation, for example, SYS1.HELP .
HSMHELP provides **HELP** command support for authorized DFSMShsm commands.

### 3.2.8 HSMLOG

Use the ARCPRLOG program to print the contents of the ARCLOGX and ARCLOGY data
sets. This member is not required for the initial setup of DFSMShsm, but it is required if you
intend to use this data for later performance reporting. The sample JCL that is provided is
shown in Figure 3-6.


-----

     //****************************************************************/
     //*     THIS SAMPLE JOB PRINTS THE DFSMSHSM LOG       */
     //*                               */
     //* REPLACE THE ?UID VARIABLE IN THE FOLLOWING SAMPLE JOB WITH  */
     //* THE NAME OF THE DFSMSHSM -AUTHORIZED USERID (1 TO 7 CHARS). */
     //*                               */
     //* (NOTE: UID AUTHORIZATION IS VALID IN A NON-FACILITY CLASS  */
     //* ENVIRONMENT ONLY, OTHERWISE, FACILITY CLASS PROFILES     */
     //* WILL BE USED FOR AUTHORIZATION CHECKING.)          */
     //****************************************************************/
     //*
     //PRINTLOG EXEC PGM=ARCPRLOG
     //ARCPRINT DD SYSOUT=*
     //ARCLOG DD DSN=?UID.HSMLOGY1,DISP=OLD
     //ARCEDIT DD DSN=?UID.EDITLOG,DISP=OLD
     //*
     //EMPTYLOG EXEC PGM=IEBGENER
     //SYSPRINT DD SYSOUT=*
     //SYSIN DD DUMMY
     //SYSUT2 DD DSN=?UID.HSMLOGY1,DISP=OLD
     //SYSUT1 DD DUMMY,DCB=(?UID.HSMLOGY1)
     /*

_Figure 3-6  Sample JCL from HSMLOG member_

If your DFSMShsm ARCLOG data sets are filling rapidly, this sample needs to be updated,
perhaps to use a daily generation data group (GDG) for processing. We describe managing
ARCLOG data in more detail in 3.1, “DFSMShsm components”.

### 3.2.9 HSMEDIT

This sample job runs the ARCPEDIT program and formats copies of the ARCLOGX and
ARCLOGY data that is produced by ARCPRLOG. Figure 3-7 shows the sample JCL.

    //EDITLOG JOB ?JOBPARM
    //*
    //****************************************************************/
    //*      THIS JOB PRINTS THE EDIT-LOG DATA SET       */
    //*                               */
    //* REPLACE THE FOLLOWING ?UID VARIABLE WITH THE NAME OF THE   */
    //* DFSMSHSM-AUTHORIZED USER (1 TO 7 CHARS).           */
    //* (NOTE: UID AUTHORIZATION IS VALID IN A NON-FACILITY CLASS  */
    //* ENVIRONMENT ONLY, OTHERWISE, FACILITY CLASS PROFILES WILL BE */
    //* USED FOR AUTHORIZATION CHECKING.)              */
    //****************************************************************/
    //*
    //EDITLOG EXEC PGM=ARCPEDIT
    //ARCPRINT DD SYSOUT=*
    //ARCLOG DD DSN=?UID.EDITLOG,DISP=SHR
    /*

_Figure 3-7  Sample JCL from HSMEDIT member_


-----

### 3.2.10 HSMPRESS

This sample job reorganizes DFSMShsm CDSs. It is not required during the initial
DFSMShsm setup, but it is required later. Due to the pattern of updates in DFSMShsm CDSs,
we recommend regular reorganization of the DFSMShsm CDSs.

## 3.3 Media that is owned by DFSMShsm

Media that is owned by DFSMShsm consists of both tape volumes and DASD volumes. No
special preparation is required for tape volumes. They require a standard tape label.
DFSMShsm DASD volumes must be initialized as non-SMS managed. Generally, these
volumes contain large numbers of non-VSAM data sets. A VSAM volume data set (VVDS) is
required only when an ML1 volume contains an SDSP data set. However, consider increasing
volume table of contents (VTOC) and index VTOC sizes for volumes that are owned by
DFSMShsm.

## 3.4 DFSMShsm operations

Stopping and starting DFSMShsm in a single system environment and a multiple system
environment are described. We also describe automating DFSMShsm and where it fits with
automating started tasks.

### 3.4.1 Stopping DFSMShsm

DFSMShsm stops in response to either an **MVS** **STOP** command or a **MODIFY** **STOP** command
that is issued to the address space. It also shuts down cleanly in response to an authorized
user that issues the **HSEND** **STOP** command. Figure 3-8 shows SYSLOG information from a
DFSMShsm shutdown.

    P HSM
    ARC0016I DFSMSHSM SHUTDOWN HAS BEEN REQUESTED
    ARC1502I DISCONNECTION FROM STRUCTURE SYSARC_MVSLS_RCL 313
    ARC1502I (CONT.) WAS SUCCESSFUL, RC=00, REASON=00000000
    ERB425I III: UNABLE TO GATHER RESOURCE HSM
    ARC0002I DFSMSHSM SHUTDOWN HAS COMPLETED
    GSDMV20I -JOBNAME STEPNAME PROCSTEP CCODE ELAPSED-TIME CPU-TIME STEPNO
    GSDMV21I -HSM   HSMS   HSM     0   46:06:12   3.96S   1
    IEF404I HSM - ENDED - TIME=11.54.40
    GSDMV22I -HSM   ENDED. NAME-* NO NAME PROVIDED * TOTAL CPU TIME=
    .02 TOTAL ELAPSED TIME=2766.2
    $HASP395 HSM   ENDED

_Figure 3-8  Sample DFSMShsm shutdown_

If DFSMShsm does not respond to a stop command, you can use the **MVS CANCEL** command
to stop the address space.


-----

In certain circumstances, DFSMShsm takes longer to complete a stop command. In
particular, if you specified, or defaulted to, PARMLIB SMFPRMxx DDCONS( YES ) for started
tasks, DFSMShsm must perform potentially significant processing to consolidate EXCP
sections for SMF30 records. We recommend that you specify DDCONS( NO ) for PARMLIB
SMFPRMxx. For more information about factors that can affect DFSMShsm operation, see
_z/OS MVS System Management Facilities,_ SA22-7630.

### 3.4.2 Starting DFSMShsm

Start the DFSMShsm address space by using the **MVS START** command. In a single system
environment, start DFSMShsm as the primary address space. See Figure 3-9.

As DFSMShsm starts, it provides a series of status informational messages as the various
initialization phases complete. Full DFSMShsm functions are not available until DFSMShsm
issues message ARC0008I.
    
    S HSM.HSMS,HOST=SN
    $HASP100 HSM   ON STCINRDR
    IEF695I START HSM   WITH JOBNAME HSM   IS ASSIGNED TO
    , GROUP SYS1
    $HASP373 HSM   STARTED
    IEF403I HSM - STARTED - TIME=12.02.22
    ARC0037I DFSMSHSM PROBLEM DETERMINATION OUTPUT DATA 330
    ARC0037I (CONT.) SETS SWITCHED, ARCPDOX=CCTS.HSM.AAIS.PDOX,
    ARC0037I (CONT.) ARCPDOY=CCTS.HSM.AAIS.PDOY
    ARC0001I DFSMSHSM 1.13.0 STARTING HOST=S IN 331
    ARC0001I (CONT.) HOSTMODE=MAIN
    ARC1700I DFSMSHSM COMMANDS ARE RACF PROTECTED
    ARC0041I MEMBER ARCCMD00 USED IN SYS1.PARMLIB
    ARC0100I SETSYS COMMAND COMPLETED
    ARC0100I SETSYS COMMAND COMPLETED
    ...
    ARC0270I DUMP CYCLE DEFINITION SUCCESSFUL
    ARC0216I DUMPCLASS DEFINITION MODIFIED, CLASS=TEST1
    ARC0216I (CONT.) RC=0
    ARC0120I MIGRATION VOLUME LSHM01 ADDED, RC=0000,
    ARC0120I (CONT.) REAS=0000
    ...
    ARC0171I SETMIG LEVEL CATALOG PROCESSED
    ARC0120I MIGRATION VOLUME LSHM01 ADDED, RC=0000,
    ARC0120I (CONT.) REAS=0000
    ARC0171I SETMIG LEVEL MASTER PROCESSED
    ARC0171I SETMIG LEVEL ACDN PROCESSED
    ARC0171I SETMIG LEVEL CATALOG PROCESSED
    ...
    ARC0270I BACKUP CYCLE DEFINITION SUCCESSFUL
    ARC0180I USER MHLRESBAUTHORIZATION IS NOT CHANGED , 4
    ARC0180I (CONT.) RC= 4
    ARC0038I RESOURCE MANAGER SUCCESSFULLY ADDED. RETURN
    ARC0038I (CONT.) CODE=00
    ARC0008I DFSMSHSM INITIALIZATION SUCCESSFUL

_Figure 3-9  DFSMShsm startup_


-----

**Starting DFSMShsm in a multisystem environment**
In a multisystem environment, start only a single DFSMShsm address space identifier (ASID)
as a primary host. All DFSMShsm address spaces must share MCDS, BCDS, OCDS, and
JRNL data sets.

Multiple hosts can share HSMPARM data by using either host-specific ARCCMDxx members
or a single ARCCMDxx member with ONLYIF statements. More than one command can be
specified for a specific host and ONLYIF command by using the BEGIN and END that are
shown in Example 3-28.

_Example 3-28  ONLYIF BEGIN and END_

    ONLYIF HSMHOST(1,2,3)
    BEGIN
    SETSYS AUTODUMPSTART(0300 0400 0500)
    SETSYS BACKUP(TAPE)
    SETSYS MAXDUMPTASKS(3)
    DEFINE BACKUP(Y)
    DEFINE DUMPCYCLE(Y)
    END

You must ensure that other data sets that are allocated in the DFSMShsm startup are unique
to each address space. If you use separate procedure libraries, maintaining individual
DFSMShsm start members is simple; however, remember that you might need to propagate
changes to multiple locations.

If you use a single shared procedure library, you can use system symbols to ensure that
unique data sets are used. You can also create unique symbols for other startup variables.

You need to perform an initial program load (IPL) to create the symbols before they can be
used.

You need to ensure that each DFSMShsm host has a unique ID. Host IDs are specified by
using the HOST= x override on the DFSMShsm startup procedure. You also need to ensure
that only a single primary DFSMShsm is started within a sysplex. You establish a host as
primary host either by using the **PRIMARY=YES** startup parameter or the older form of startup
parameter, **HOST=xY** .

If you do not start any host as a primary host, the functions that can be performed only by a
primary host will not be run by any DFSMShsm address space:

- Backing up CDSs as the first phase of automatic backup
- Backing up data sets that were migrated before they were backed up
- Moving backup versions of data sets from ML1 volumes to backup volumes
- Deleting expired dump copies automatically
- Deleting excess dump VTOC copy data sets

It is possible to define more than one primary host. However, these functions might run more
frequently than expected.

### 3.4.3 MASH implications

The requirements for running multiple DFSMShsm address spaces on a single system are
the same as the requirements in the multisystem description in the topic “Starting
DFSMShsm in a multisystem environment”. Additionally, you must create a
unique name for each DFSMShsm address space within a system.


-----

It is possible to issue commands to other than the primary address space by using console
support. However, if you issue **HSEND** commands from TSO, they are always processed by the
primary DFSMShsm address space on the systems where they were issued, including
commands that are issued through batch unless an EMCS console environment is
established in which to direct your command to a specific DFSMShsm address space.

### 3.4.4 Sending commands to DFSMShsm

Commands can be issued by using either modify commands from a console or Extended
Multiple Console Support (EMCS) console, or through a TSO background job. If using a
background job, the user that issues the commands needs to be authorized to the command
that is being issued.

If you use modify commands to communicate to DFSMShsm address spaces, the capability
to issue a single command from one system that affects all DFSMShsm address spaces that
are running on this system and other systems in the sysplex is beneficial. The z/OS route
command and the z/OS modify command support the use of wildcards to identify address
spaces.

### 3.4.5 Secondary address spaces

Secondary address spaces that are created by either ABARs or as the result of SETSYS
DSSXMMODE do not respond directly to commands. Commands to influence these address
spaces need to be issued to the DFSMShsm address space that created the secondary
address space.

All commands must be issued to the primary DFSMShsm system not the secondary address
spaces.

### 3.4.6 Automation

For DFSMShsm startup, we recommend that you not start DFSMShsm until your JES and
TMSs are initialized. DFSMShsm has no dependencies on subsystems other than JES being
available; however, if your TMS is not initialized, tape requests are failed. DFSMShsm must
be started before batch work is permitted to start to prevent jobs from failing or waiting for
data set recalls.

Similarly, Data Facility Storage Management Subsystem (DFSMS) needs to be shut down
after batch and TSO users finish, but before your TMS and JES are terminated.

### 3.4.7 Host promotion

If a primary DFSMShsm host fails, another DFSMShsm host in the sysplex, on the same or
another z/OS image, can take over a subset of the functions of the primary host. Two levels of
promotion are available. The level of promotion is determined by the **SETSYS PROMOTE**
command.

In a sysplex that remains DFSMShsm, hosts are notified through a cross-system coupling
facility (XCF) group when either a primary host or an SSM host terminates.


-----

If a secondary host is promoted to primary host, it issues message ARC1522I to indicate that
the promotion is complete. Information about backup and dump windows from the failing
primary host is copied to the host that is being promoted with the status of the exit
ARCCBEXT.

If the promoted host does not normally perform autobackup, it performs unique sections of
autobackup, for example, control data set backup, but it will not start volume backup. Similarly,
if you limit DFSMShsm functions by system or system group at the storage group level, a
promoted host cannot overcome these restrictions.

When you implement any automation, consider that host promotion might occur as a result of
an unscheduled DFSMShsm host shutdown.


-----

