
# 14. Problem determination

A correctly configured installation of DFSMShsm will execute without any issues most of the
time. However, if an issue occurs, be prepared to collect the correct documentation on the first
occurrence of the issue. This way, diagnosing the root cause of the issue begins immediately,
preventing the need to re-create or to wait for a recurrence of the issue to collect the
appropriate documentation.

In this chapter, we describe the most important sources of problem determination
documentation for DFSMShsm. We go through sample **AUDIT** commands and provide
scenarios about when to use them.


-----

## 14.1 Problem determination documentation

The most important sources of problem determination documentation for DFSMShsm are
described:

- Problem Determination Aid (PDA)
- DFSMShsm dumps
- SYSLOG
- JOBLOG
- Activity logs

A few common problem scenarios and the actions to address them are described. An
overview of the DFSMShsm **AUDIT** command and how it can be used during both problem
determination and recovery are provided.

### 14.1.1 Problem Determination Aid

PDA is the single most important diagnostic aid that is used to determine the root cause of
most of the issues that occur in DFSMShsm.

PDA is a trace facility that is built into DFSMShsm that traces the module flow in DFSMShsm.
This tracing is accomplished by incorporating PDA trace points at important locations in the
DFSMShsm modules (for example, when a module is entered or exited). When a task in
DFSMShsm encounters one of these trace points, the information that is contained in the
trace point is recorded in an internal trace buffer in the DFSMShsm address space. When the
internal trace buffer is full, the data in the trace buffer is written (optionally) to the PDA data
sets that the user defined on DASD.

The PDA facility is automatically enabled at DFSMShsm startup. You enable or disable PDA
processing with the **SETSYS PDA(NONE|ON|OFF)** command. We highly recommend that you
leave PDA tracing enabled when DFSMShsm is active. The PDA trace options are described:

- **SETSYS PDA(NONE)** , when specified at DFSMShsm startup, causes no data to be gathered;
the trace log data sets are not opened.

- **SETSYS PDA(ON)** causes DFSMShsm to request storage for data accumulation and opens
the DASD trace log data sets if they are allocated. If the trace log data sets are not
allocated, data is accumulated in internal storage only.

- **SETSYS PDA(OFF)** stops data accumulation. The trace log data sets remain open.

Two PDA output data sets need to be created on DASD (if you choose to retain the PDA data
on DASD). For DFSMShsm to use these data sets after they are created, they must be
defined to the ARCPDOX and ARCPDOY DD names in the DFSMShsm startup procedure.

**Writing to the PDA data sets**
DFSMShsm always writes to the data set name that is defined to the ARCPDOX DD name.
For example, if data set HSM.PDOX is defined to the ARCPDOX DD name, DFSMShsm will
always write to HSM.PDOX. When this data set becomes full, either an ABENDB37 or an
ABENDD37 is generated against the data set, which is normal. DFSMShsm uses these
ABENDs to determine that it needs to swap the PDA data set that it is writing to.


-----

This swap is why a user must define two PDA data sets in the DFSMShsm startup procedure:
one to the ARCPDOX DD name and one to the ARCPDOY DD name. When the ARCPDOX
data set becomes full, DFSMShsm swaps the data set names that are associated with the
ARCPDOX and ARCPDOY DD names. That is, DFSMShsm renames each data set by using
the other data set’s name. For example, if data set HSM.PDOX is defined to the ARCPDOX
DD name and HSM.PDOY is defined to the ARCPDOY DD name, HSM.PDOX is renamed to
HSM.PDOY and HSM.PDOY is renamed to HSM.PDOX.

After the swap occurs, DFSMShsm starts writing to the data set name that is associated with
the ARCPDOX DD name. By using the preceding example, DFSMShsm starts writing to the
HSM.PDOX data set (which was previously named HSM.PDOY). When this swap occurs,
HSM identifies an ARC0037I message.

**Allocating the PDA data sets**
Your PDA data sets are usually allocated as part of the starter set. Example 14-1 contains
sample JCL that can be used to allocate PDA data sets if the initial allocations are too small.
These data sets must be created on the same volume.

_Example 14-1  Sample JCL to allocate PDA data sets_

    //MHLRES4J JOB MSGLEVEL=1,CLASS=A
    //STEP1 EXEC PGM=IEFBR14
    //ARCPDOX DD DSN=HSM.HSMPDOX,DISP=(,CATLG),VOL=SER=HSM14C,
    // UNIT=3390,SPACE=(CYL,(30,2))
    //ARCPDOY DD DSN=HSM.HSMPDOY,DISP=(,CATLG),VOL=SER=HSM14C,
    // UNIT=3390,SPACE=(CYL,(30,2))
    /*

The larger you define the PDA data sets, the longer the time period of trace data the data sets
are able to contain. When the ARCPDOX data set becomes full and the data set names are
swapped, any previously recorded data in the data sets now being written to is overlaid.
Therefore, unless you archive the ARCPDOY data set after the ARC0037I is issued, you will
begin to lose this data the next time the ARCPDOX data set fills and another swap occurs.
You need to copy the ARCPDOY data set to tape or DASD before the ARCPDOX data set
becomes full to archive it.

**Enabling and switching the PDA logs**
If the PDA trace is not enabled at DFSMShsm startup, or you disabled it, issue the following
command at a system console to enable PDA tracing:

f _hsmprocname_ ,SETSYS PDA(ON)

The PDA data sets are automatically swapped when DFSMShsm is started. To manually
switch the data sets, issue the following command at a system console:

f _hsmprocname_ ,SWAPLOG PDA

**Archiving PDA data**
A generation data group (GDG) can be used to archive the PDA trace data. You can then
copy the generation data set (GDS) to tape or direct it to a DASD volume where it will be
migrated to tape.

Example 14-2 shows sample JCL to create a GDG to archive the PDA trace
data.


-----

_Example 14-2  Sample JCL to define a GDG base for archiving PDA_

     //MHLRES4G JOB MSGLEVEL=1,MSGCLASS=A
     //STEP1  EXEC PGM=IDCAMS
     //SYSPRINT DD  SYSOUT=A
     //SYSIN  DD  *
     DEFINE GDG (NAME('HSM.HSMTRACE') LIMIT(30) (SCRATCH)/*

**Sample JCL to define a GDG base for archiving PDA**
Example 14-3 shows sample JCL for copying the PDA data set to tape as a GDS for archival.

_Example 14-3  Sample JCL to copy PDA to tape_

    //MHLRES4C JOB MSGLEVEL=1,MSGCLASS=A
    //STEP1  EXEC PGM=IEBGENER
    //SYSPRINT DD  SYSOUT=A
    //SYSIN  DD  DUMMY
    //SYSUT1  DD  DSN=HSM.HSMPDOY,DISP=SHR
    //SYSUT2  DD  DSN=HSM.HSMTRACE(+1),
    //     UNIT=TAPE,
    //     DISP=(NEW,CATLG,CATLG),VOL=(,,1),
    //     DCB=(HSM.HSMPDOY)

When the PDA trace log switch occurs, DFSMShsm issues message ARC0037I. You can use
an automation product to trap this message and automatically submit a copy job to archive
the PDA trace data. This practice provides a sequential history of trace data over time so that
the data is available when needed for diagnosing problems.

Although PDA provides an excellent overview of the module flow in DFSMShsm over the time
frame that an issue occurs, it is not able to provide a snapshot of the state of DFSMShsm
exactly when an abend occurs. This functionality is provided by supervisor call (SVC) dumps.

### 14.1.2 DFSMShsm dumps

One of the most important diagnostic features that MVS provides is the ability to take a
snapshot of an address space when an error is detected or when a user believes that an error
is present. The following facilities, which exist in MVS, allow this data capture:

- Automatic dumps
- Console dumps
- Serviceability level indication processing (SLIP) traps

How these facilities can be used to capture errors in DFSMShsm is described.

**Automatic dumps**
DFSMShsm provides two **SETSYS** parameters that control how and what type of dumps are
taken when an abend occurs in the DFSMShsm address space:

- SYS1DUMP
- NOSYS1DUMP


-----

**_SETSYS SYS1DUMP_**
When SETSYS SYS1DUMP is specified and an abnormal end of task (abend) occurs in the
DFSMShsm address space, MVS requests a dump to be written to either a pre-allocated
SYS1.DUMPxx data set or to a dynamically allocated dump data set that is named according
to your installation standards. The dump is unformatted and requires the use of Interactive
Problem Control System (IPCS) to be analyzed. This dump type will be requested by the IBM
Support Center to diagnose the root cause of an abend in the DFSMShsm address space.
This setting is recommended.

**_SETSYS NOSYS1DUMP_**
When SETSYS NOSYS1DUMP is specified and an abend occurs in the DFSMShsm address
space, the dump that is produced is directed to the dump-related DD statements that are
specified in the DFSMShsm startup procedure. The following DD statements are applicable:

- SYSABEND
- SYSUDUMP
- SYSMDUMP

SYSABEND and SYSUDUMP dumps are both _formatted_ . They are user-readable and do not
require IPCS for analysis. SYSMDUMP dumps are unformatted and require IPCS for
analysis.

We recommend that you specify the **SETSYS SYS1DUMP** command in the ARCCMDxx member
of SYS1.PARMLIB. However, if you prefer to use SETSYS NOSYS1DUMP , ensure that you
specify that a SYSMDUMP is taken. The IBM Support Center prefers that clients gather
unformatted dumps as opposed to formatted dumps. Formatted dumps do not contain the
same amount of information as unformatted dumps and decrease the likelihood that the root
cause of an issue can be determined from them.

**Console dumps**
In addition to automatically generating dumps when abend is detected, MVS provides the
ability for an operator to issue a manual dump command to capture a snapshot of an address
space at any time. The main reason that we recommend taking a console dump is if you
suspect that DFSMShsm processing is stopped. A console dump is the only way to determine
exactly what is causing a problem in the DFSMShsm address space to occur.

Although the dump command and its associated parameters can be issued at the same time,
we recommend that you create IEADMCxx members in SYS1.PARMLIB in advance that
contain the dump parameters to use to capture dumps that are associated with DFSMShsm
processing. After these IEADMCxx members in SYS1.PARMLIB are created, you only need
to execute the following command at the system console to create a dump:

DUMP COMM=( _text_ ),PARMLIB= _xx_

_Text_ is the title that you want for the dump. The _xx_ is the PARMLIB member that contains the
address spaces and parameters that you want dumped.


-----

You need to create IEADMCxx PARMLIB members for three sets of console dump
parameters if a problem is experienced in DFSMShsm:

1. A dump of only the DFSMShsm address space, as shown in Example 14-4.

   _Example 14-4  Sample IEADMCxx parameters to dump only the DFSMShsm address space_

       JOBNAME=(DFHSM),
       SDATA=(ALLNUC,CSA,LPA,LSQA,PSA,RGN,SQA,SWA,TRT,SUM,GRSQ),END

2. A dump of DFSMShsm and other address spaces, as shown in Example 14-5.

   _Example 14-5  Sample IEADMCxx parameters to dump DFSMShsm and other address spaces_

        JOBNAME=(DFHSM,CATALOG,GRS),
        SDATA=(ALLNUC,CSA,GRSQ,LPA,LSQA,PSA,RGN,SQA,SWA,TRT,SUM),END

3. A dump of DFSMShsm and the common recall queue (CRQ) structure (if CRQ is
implemented in your environment), as shown in Example 14-6.

   _Example 14-6  Sample IEADMCxx parameters to dump the DFSMShsm address space and CRQ_

       JOBNAME=(DFHSM),
       STRLIST=(STRNAME=SYSARC_xxxxx_RCL,(LNUM=ALL,ADJ=CAP,EDATA=UNSER),
       LOCKE,(EMC=ALL),ACC=NOLIM),
       SDATA=(ALLNUC,CSA,GRSQ,LPA,LSQA,PSA,RGN,SQA,SWA,TRT,SUM,XESDATA,COUPLE),
       END

When you dump the CRQ, you need to replace _xxxxx_ with the name of the CRQ structure as
it was specified in the following command:

SETSYS COMMONQUEUE(RECALL(CONNECT( _xxxxx_ )))

For example, if the CRQ name that is used in this command is PLEX1, the **STRNAME** in the
parameter in the dump is **STRNAME=SYSARC_PLEX1_RCL** .

**SLIP traps**
Like console dumps, SLIP traps allow an unformatted dump of DFSMShsm to be taken when
an error condition is detected. Unlike console dumps however, with SLIP traps, you can
specify the system conditions that must be met for an automatic dump to be taken.

Just as console dumps allow the specification of the dump parameters in advance in an
IEADMCxx PARMLIB member, with SLIP traps, you can specify the SLIP trap parameters in
advance in PARMLIB member IEASLPxx.

For example, you might want a dump of DFSMShsm to be automatically generated if one of
the control data sets (CDSs) or the journal fails to be backed up during CDS backup. This
event generates an ARC0744E message. You can write the following SLIP in its own
IEASLPxx member in SYS1.PARMLIB to automatically capture a dump of DFSMShsm if this
message is issued. See Example 14-7.

_Example 14-7  Sample IEASLPxx parameters to dump the DFSMShsm address space on ARC0744E_

    SLIP SET,A=SVCD,MSGID=ARC0744E,
    JOBLIST=(DFHSM),
    SDATA=(ALLNUC,ALLPSA,CSA,LPA,LSQA,NUC,PSA,RGN,SUM,SQA,SWA,TRT,GRSQ),END


-----

After this SLIP is created, execute the following command at the system console to set the
SLIP trap:

SET SLIP= _xx_

Where _xx_ is the value of the newly created IEASLP _xx_ member. For more information about
how to write SLIP trap commands, see z/OS _MVS System Commands,_ SA22-7627.

**Dump analysis and elimination**
Dump analysis and elimination (DAE) is an MVS function that eliminates duplicate dumps in
MVS systems or across MVS systems.

DFSMShsm generates dumps to gather first-failure diagnostic information. Problems that
occur multiple times, perhaps on different hosts, typically generate a storage dump. The initial
dump is helpful for diagnosing the problem; additional dumps for the same problem usually
are not needed.

Controlling DFSMShsm storage dumps is a major issue with users. The ability to correctly
suppress duplicate dumps is an important step toward successfully managing dump data
sets.

**DAE implementation**
To implement DAE for DFSMShsm, the following tasks must be completed:

- SETSYS SYS1DUMP must be specified. SYS1DUMP is the DFSMShsm default.

- PARMLIB member ADYSETxx must be coded with the **SUPPRESSALL** keyword, for example:

DAE=START,RECORDS(400),SVCDUMP(MATCH,UPDATES,SUPPRESSALL)

- A single DAE data set can be shared across systems in a sysplex. The coupling services
of the cross-system coupling facility (XCF) and GRS must be enabled for the DAE data set
to be shared in a sysplex environment and for dumps to be suppressed across MVS
systems.

DFSMShsm uses DAE for functions that run in both the primary address space and the
aggregate backup and recovery support (ABARS) secondary address space.

DAE does not suppress SYSABEND, SYSUDUMP , SYSMDUMP , or SNAP dumps or dumps
that originate from **SLIP** or **DUMP** operator commands. Because these dumps are taken only on
demand, suppression is not desirable.

This support does not apply to dumps that are produced by DFSMShsm as a result of the
**TRAP** command. For more information about setting up the DAE data set, see _z/OS MVS_
_Diagnosis: Tools and Service Aids_ , GA22-7589.

### 14.1.3 SYSLOG

The SYSLOG reports events that are routed to the system operator to ensure the continued
successful functioning of the z/OS operating system and the programs that execute in it. It is
important to monitor the SYSLOG for any DFSMShsm messages that indicate an error
condition, and for any messages that are displayed outside of DFSMShsm that affect
DFSMShsm processing.


-----

For example, if an I/O error occurred on a drive that was in use by DFSMShsm, an IOS000I
message that indicates this error is only written to the SYSLOG; it is not sent to any
DFSMShsm-specific log. For this reason, if an issue occurs in DFSMShsm processing, it is
important to retain SYSLOGs that lead up to and include the time frame in which the issue
occurred.

### 14.1.4 JOBLOG

The DFSMShsm job log (JOBLOG) contains messages that span from the time that an
instance of DFSMShsm starts through the time that it is shut down. The messages that it
contains are also output to the SYSLOG, but unlike the SYSLOG, the job log contains
messages that are specific to only a single instance of DFSMShsm.

The job log is useful for reviewing the messages that are issued during the startup process of
an instance of DFSMShsm to ensure that all of the setup commands that are contained in the
ARCCMDxx PARMLIB member were processed successfully. For this reason, it is important
to retain the job log for review of an instance of DFSMShsm that experiences an issue.

### 14.1.5 Activity logs

The activity logs report on the backup, dump, migration, ABARS, and command processing of
DFSMShsm in your system. DFSMShsm has five activity logs:

- The migration activity log provides information about space management activity, including
**MIGRATE** commands for volumes and levels, interval migration, and automatic primary and
automatic secondary space management.

- The backup activity log provides information about automatic backup, volume command
backup, FRBACKUP and FRRECOV activities, and volume command recovery activities.

- The dump activity log provides information about automatic dump, command volume
dump, and command volume restore activities.

- The ABARS activity log provides information about aggregate backup and recovery
activities.

- The command activity log provides information about TAPECOPY and TAPEREPL activity
and records error or informational messages that occur during low-level internal service
processing.

With the **SETSYS ACTLOGTYPE** command, you can direct DFSMShsm activity logs to DASD or to
a SYSOUT class. For more information, see Chapter 3, “Getting started” on page 29. If you
specify **ACTLOGTYPE(DASD)** on the **SETSYS** command, DFSMShsm dynamically allocates DASD
data sets for the activity logs with names in the following format:

HSMACT. _Hmcvthost.function.agname.Dyyddd.Thhmmss_

The following definitions refer to the previous naming format:

**Hmcvthost** DFSMShsm host ID from the DFSMShsm startup PROC statement,
which is preceded by H

**function** Either ABACKUP , ARECOVER, BAKLOG, CMDLOG, DMPLOG, or
MIGLOG

**agname** Aggregate group name (only present if function is ABACKUP or
ARECOVER)

**Dyyddd** Year and day of allocation, which are preceded by D

**Thhmmss** Hour, minute, and second of allocation, which are preceded by T


-----

Although the activity logs are lower in importance to problem diagnosis than the
documentation that was previously described, they still provide value because the messages
that are sent to them can indicate potential issues in DFSMShsm. We recommend that you
monitor the activity logs for any indications of a potential issue.

## 14.2 Problem recognition, documentation collection, and recovery

After a problem occurs in DFSMShsm and you collect the appropriate documentation to
enable the diagnosis of the root cause of the issue, begin the recovery process to bring your
DFSMShsm back to its fully functional state. Although various issues can arise during
DFSMShsm processing, those issues fall into one of two categories: Hung situations
(stoppages) in the DFSMShsm address space and the DFSMShsm ARC _xxxx_ messages that
identify an undesirable condition.

### 14.2.1 Hung situations in the DFSMShsm address space

If DFSMShsm processing hangs or stops, the processing of any job in the z/OS environment
that requires DFSMShsm functionality to succeed might be held.

For example, if a job tries to access a data set but that data set was migrated by DFSMShsm,
the data set must be recalled before the job can access the data set. If DFSMShsm hangs
before the data set finished recalling, the job cannot continue processing until the hung
situation is recovered from.

For this reason, it is important for a DFSMShsm administrator to determine when DFSMShsm
processing stopped, to collect the appropriate documentation to diagnose the root cause of
the stoppage, and to recover from this condition.

**Recognizing a hung condition**
Stoppages or hung conditions in DFSMShsm processing are caused by DFSMShsm tasks
that cannot obtain the required resources to continue processing. DFSMShsm has two
variations of the **QUERY** command that can be used to recognize that DFSMShsm processing
might be hung: **QUERY WAITING** and **QUERY ACTIVE(TCBADDRESS)** .

Example 14-8 shows the output of the **QUERY WAITING** command in an instance of DFSMShsm
that does not have any work that is waiting to be executed.

_Example 14-8  Output from the QUERY WAITING command_

    09.24.40 f dfhsm,query waiting
    09.24.40 STC00018 ARC0101I QUERY WAITING COMMAND STARTING ON HOST=1
    09.24.40 STC00018 ARC0168I WAITING MWES: MIGRATE=00000000,
    ARC0168I (CONT.) RECALL=00000000, DELETE=00000000, BACKUP=00000000,
    ARC0168I (CONT.) RECOVER=00000000, COMMAND=00000000, ABACKUP=00000000,
    ARC0168I (CONT.) ARECOVER=00000000, FRBACKUP=00000000,
    ARC0168I (CONT.) FRRECOV=00000000, TOTAL=00000000
    09.24.40 STC00018 ARC0101I QUERY WAITING COMMAND COMPLETED ON HOST=1


-----

We recommend that you issue the **QUERY WAITING** command periodically to verify that
DFSMShsm is continuing to process the Management Work Elements (MWEs) that are listed
in the ARC0168I message. If this command is issued every 15 minutes, and over a few hour
period, all of the MWEs either stay at the same number or increase, this situation might
indicate that DFSMShsm processing is held up for an unknown reason and further analysis is
required.

If you suspect that DFSMShsm processing is hung, issue the **QUERY ACTIVE(TCBADDRESS)**
command. Example 14-9 shows the output of this command in an instance of DFSMShsm
that has five active recall tasks and CDS backup executing.

_Example 14-9  Output from the QUERY ACTIVE(TCBADDRESS) command_

    09.24.44      f dfhsm,query active(tcbaddress)
    09.24.44 STC00016 ARC0101I QUERY ACTIVE COMMAND STARTING ON HOST=1
    09.24.44 STC00016 ARC0142I CDS BACKUP, CURRENTLY IN PROCESS,
    ARC0142I (CONT.) TCB=x'009B4C68'
    09.24.44 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST9 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000058 ON HOST 1 , TCB=x'009A7E88'
    09.24.44 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST1 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000060 ON HOST 1 , TCB=x'009A6238'
    09.24.44 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST2 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000061 ON HOST 1 , TCB=x'009B25D0'
    09.24.44 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST3 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000062 ON HOST 1 , TCB=x'009B2438'
    09.24.44 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST4 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000063 ON HOST 1 , TCB=x'009B22A0'
    09.24.44 STC00016 ARC0101I QUERY ACTIVE COMMAND COMPLETED ON HOST=1

When you review the output from the **QUERY ACTIVE(TCBADDRESS)** command on your own
system, look for any messages that indicate a unit of work, for example the recall of a data
set, that you think is finished is still active. This situation indicates that the unit of work itself
might cause the hung status of the rest of DFSMShsm, or the unit of work might be a victim of
another unit of work that caused the hung situation.

To further narrow down the unit of work that might cause the hung situation that DFSMShsm
is experiencing, you can use the following Global Resource Serialization ( **GRS** ) command to
display any resource contention that is detected in the system:

D GRS,CONTENTION

Example 14-10 shows an example of a system that is experiencing resource contention on
the ARCGPA/ARCCAT resource.

_Example 14-10  Output from the D GRS,CONTENTION command_

    09.25.45      D GRS,CONTENTION
    09.25.45      ISG343I 09.25.45 GRS STATUS 022
    S=SYSTEM ARCGPA  ARCCAT
    SYSNAME    JOBNAME     ASID   TCBADDR  EXC/SHR  STATUS
    SYSTEM1  DFHSM       0030    009A7E88  SHARE   OWN
    SYSTEM1  DFHSM       0030    009B4C68 EXCLUSIVE  WAIT
    SYSTEM1  DFHSM       0030    009A6238  SHARE   WAIT
    SYSTEM1  DFHSM       0030    009B25D0  SHARE   WAIT
    SYSTEM1  DFHSM       0030    009B2438  SHARE   WAIT
    SYSTEM1  DFHSM       0030    009B22A0  SHARE   WAIT
    NO LATCH CONTENTION EXISTS


-----

If the output from this command indicates contention for a DFSMShsm resource, you might
be able to match the TCBs that are identified in this output as holding the DFSMShsm
resource that is under contention with the TCBs that are identified as a result of the **QUERY**
**ACTIVE(TCBADDRESS)** command. If you can, you might then use the functionality of the **CANCEL**
**TCBADDRESS** command to attempt to free the resources that are hanging up the rest of the
DFSMShsm processing. Example 14-10 on page 410 shows that TCB 009A7E88 holds
ARCGPA/ARCCAT, which matches recall request number 58 in Example 14-9 on page 410.

**Note:** The **CANCEL TCBADDRESS** command is only designed to cancel TCBs that are
identified by way of the **QUERY ACTIVE(TCBADDRESS)** command. If it is used to cancel a TCB
that is _not_ identified in the **QUERY ACTIVE(TCBADDRESS)** command output, the task will be
canceled, but the results are unpredictable.

In the example that is shown in Example 14-10 on page 410, the DFSMShsm resource
ARCGPA/ARCCAT is under contention by six TCBs. TCB 009A7E88 holds a SHARED
enqueue on the resource, preventing TCB 009B4C68 from obtaining the EXCLUSIVE
enqueue that it needs to continue processing. Any TCB that requests an enqueue on
ARCGPA/ARCCAT after TCB 009B4C68 must wait until TCB 009B4C68 enqueues and
dequeues the resource for the enqueue to succeed and that task to continue processing.

**Note:** Do not issue the **CANCEL TCBADDRESS** command without first taking a console dump of
DFSMShsm by using the instructions in Example 14-4 on page 406 and without following
the correct procedure for using the **CANCEL TCBADDRESS** command that we describe next.

**Collecting the documentation of a hung situation**
Before you act to resolve a hung condition, you must take a console dump of DFSMShsm by
using the parameters that are described in Example 14-4 on page 406. If you suspect or are
aware that the hung condition involves other address spaces, use the parameters that are
described in Example 14-5 on page 406. If the hung condition involves data set recalls that
are not processing and you are using CRQ, use the parameters that are described in
Example 14-6 on page 406 to take a console dump that includes a dump of the CRQ
structure.

After console dumps of the involved address spaces are collected, use the **CANCEL**
**TCBADDRESS** command to attempt to free the resources that are causing the hung situation in
DFSMShsm processing. In addition to the console dumps, we recommend that you retain the
PDA and SYSLOG for at least the 2-hour period that leads up to and includes the time that
the dump was taken.

**Recovering from a hung situation**
DFSMShsm provides the **CANCEL** **TCBADDRESS** command to allow the administrator to cancel
tasks that are hung and that prevent other tasks from processing normally. **CANCEL**
**TCBADDRESS** is a powerful command that you use only to cancel a task that you suspect is
hung and that prevents other tasks from processing normally. This command must _not_ be
used as a regular maintenance command to cancel tasks that are actively processing to
completion (for example, canceling the migration or recall of a large data set because it is
taking a long time to move the data). If you use this command to cancel a task that is actively
nearing completion, the results are unpredictable.

The TCBADDRESS that you choose to cancel with the **CANCEL** command must be obtained
from the **QUERY ACTIVE(TCBADDRESS)** command. Before you issue the **CANCEL** command, you
must first issue the **HOLD** command to hold the function that contains the task that you are
canceling.


-----

Example 14-11 shows an example of using the **HOLD** command to hold the recall function.

_Example 14-11  Output from the HOLD RECALL command_

    09.26.16      f dfhsm,hold recall
    09.26.16 STC00017 ARC0100I HOLD COMMAND COMPLETED

After the function that you are holding receives the ARC0100I message, issue the **QUERY**
**ACTIVE(TCBADDRESS)** command one more time to verify that the task that you want to cancel is
still identified. Example 14-12 shows the output from the **QUERY ACTIVE(TCBADDRESS)**
command.

_Example 14-12  Output from the QUERY ACTIVE(TCBADDRESS) command_

    09.27.32      f dfhsm,query active(tcbaddress)
    09.27.32 STC00016 ARC0101I QUERY ACTIVE COMMAND STARTING ON HOST=1
    09.27.32 STC00016 ARC0142I CDS BACKUP, CURRENTLY IN PROCESS,
    ARC0142I (CONT.) TCB=x'009B4C68'
    09.27.32 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST9 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000058 ON HOST 1 , TCB=x'009A7E88'
    09.27.32 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST1 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000060 ON HOST 1 , TCB=x'009A6238'
    09.27.32 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST2 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000061 ON HOST 1 , TCB=x'009B25D0'
    09.27.32 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST3 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000062 ON HOST 1 , TCB=x'009B2438'
    09.27.32 STC00016 ARC0162I RECALLING DATA SET DERDMANN.MIGTST4 FOR USER
    ARC0162I (CONT.) HSMATH0, REQUEST 00000063 ON HOST 1 , TCB=x'009B22A0'
    09.27.32 STC00016 ARC0101I QUERY ACTIVE COMMAND COMPLETED ON HOST=1

One enhancement in z/OS Data Facility Storage Management Subsystem (DFSMS) V2.1
was to include the host IDs, tape volsers, and device addresses for those tasks that are
processing tape in the ARC0162I message, as shown in Example 14-13.

_Example 14-13  ARC0162I message text_

    ARC0162I {MIGRATING | BACKING UP | RECALLING | RECOVERING
    | DELETING | RESTORING | FRRECOV OF} DATA SET dsname
    FOR USER userid, REQUEST request ON HOST hostid <<-- host ID
    
    [,TCB=X'tcbaddress’']:
    VOL = {volser1 | NONE}, ADDR = {address1 | NONE}; <-- volser
    VOL = {volser2 | NONE}, ADDR = {address2 | NONE}; <-- devaddr
    VOL = {volser3 | NONE}, ADDR = {address3 | NONE}]

Where volser1, volser2 or volser3 are the tape volume serial numbers
Where address1, address2 or address3 are the device addresses of the tape volumes.

If the TCB address is identified, you can issue the **CANCEL TCBADDRESS(x'aaaaaaaa')**
command to cancel the task, as shown in Example 14-14.


-----

_Example 14-14  Output from the CANCEL TCBADDRESS(x'aaaaaaaa') command_

    09.28.50 f dfhsm,CANCEL TCBADDRESS(X'009A7E88')
    09.28.50 STC00018 ARC0931I (H)CANCEL COMMAND COMPLETED, NUMBER OF TASKS
    ARC0931I (CONT.) CANCELLED=1
    09.28.50 STC00018 ARC0003I ARCRSTR1 TASK ABENDED, CODE 047D0000 IN
    ARC0003I (CONT.) MODULE UNKNOWN AT OFFSET 0000, STORAGE LOCATION
    ARC0003I (CONT.) 80E27174

When the **CANCEL TCBADDRESS** command is executed, DFSMShsm issues an ABEND7D0 for
the task that is being canceled. When this ABEND7D0 occurs, DFSMShsm error recovery
processing is invoked to clean up the task to the best of its ability.

The error recovery can take time to process to completion, so it is not unusual for a
subsequent **QUERY ACTIVE(TCBADDRESS)** command to still identify the TCB that the **CANCEL**
command was issued against. If the TCB is still identified in the **QUERY ACTIVE(TCBADDRESS)**
command after 10 minutes, issue a second **CANCEL** command of this task. If this subsequent
**CANCEL** command does not cancel the task successfully and the DFSMShsm processing still
appears to be hung, contact the IBM Support Center for assistance.

After the task is canceled, be sure to issue a **RELEASE** command of that function to allow
DFSMShsm to resume processing that function. Example 14-15 shows the output of
releasing the recall function.

_Example 14-15  Output from the RELEASE RECALL command_

    09.29.50 f dfhsm,release recall
    09.29.50 STC00026 ARC0100I RELEASE COMMAND COMPLETED

You can also release just the DASD recall function with the new **RECALL(DASD)** parameter on
the **RELEASE** command, as shown in Example 14-16.

_Example 14-16  Output from the RELEASE RECALL(DASD) command_

    09.45.50 f dfhsm,release recall(dasd)
    09.45.50 STC00026 ARC0100I RELEASE COMMAND COMPLETED

**Addressing issues with the common recall queue**
If an issue is experienced in DFSMShsm that involves recalls that are not processing
normally and the CRQ is in use, the first step that you must take to address the issue is to
dump DFSMShsm and the CRQ by using the parameters in Example 14-6 on page 406. After
a dump is taken of the CRQ, disconnect each instance of DFSMShsm from the CRQ by
issuing the following command on each DFSMShsm host in an HSMplex that connects to the
CRQ:

SETSYS COMMONQUEUE(RECALL(DISCONNECT))

When this command is specified in an DFSMShsm host, the host performs the following
functions:

- Directs all new recall requests to the local recall queue
- Moves all recall requests that originated on that host from the CRQ to the local queue
- Completes any remote requests that were previously selected from the CRQ
- Stops selecting requests from the CRQ


-----

The host disconnects from the CRQ when the following conditions are met:

- The host finishes processing all of the remote requests that were previously selected from
the CRQ.

- All requests that originated on that host moved from the CRQ to the local queue.

- Processing is complete for all requests that originated on that host that were selected for
remote processing.

After all of the hosts are disconnected from the CRQ, check to see whether the recall issues
remain:

- If the recall issues remain, capture the PDA and SYSLOG from the system or systems that
are still experiencing the issue and contact the IBM Support Center for assistance.

- If the recall issues are gone, execute the **AUDIT COMMONQUEUE(RECALL) FIX** command to
instruct DFSMShsm to fix any issues that it detects in the CRQ:

  - If the resulting ARC1544I message indicates that errors were found and fixed, you can
reconnect each instance of DFSMShsm to the CRQ and verify that recalls are
processing successfully by using the CRQ. If the recall issues return, the CRQ
structure might be corrupted beyond the capability of the **AUDIT** command to detect and
fix problems. In this case, you need to follow the procedure that is entitled “Case 10:
Correcting errors in the common recall queue” in the _DFSMShsm Storage_
_Administration Guide,_ SC26-0421, to delete and rebuild the CRQ structure.

  - If the ARC1544I message indicates that no errors were identified and fixed, you need
to follow the procedure that is entitled “Case 10: Correcting errors in the common recall
queue” in the _DFSMShsm Storage Administration Guide,_ SC26-0421, to delete and
rebuild the CRQ structure.

**Note:** The CRQ structure must be deleted and rebuilt only after you take a dump of the
structure, as described in Example 14-6 on page 406.

### 14.2.2 DFSMShsm messaging

Throughout DFSMShsm processing, DFSMShsm highlights messages that indicate the
success or failure of its many functions. These messages are sent to the logs that are
described in 14.1, “Problem determination documentation” on page 402.

DFSMShsm messages are identified to these logs by using the prefix “ARC”. DFSMShsm
identifies three types of messages:

- Action messages
- Error messages
- Informational messages

**Action messages**
_Action messages_ indicate that an action must be taken by an operator to address the condition
that is identified in the message. Example 14-17 shows an example of an action message
that might be issued to the operator.

_Example 14-17  Output from a DFSMShsm action message_

    11.15.40 STC00016 *0010 ARC0366A REPLY Y ONLY WHEN ALL 003 TAPE VOLUME(S)
    IS/ARE COLLECTED, N IF ANY NOT AVAILABLE


-----

This message indicates that a reply from the operator is required before this DFSMShsm
function can continue.

**Error messages**
_Error messages_ indicate that an error occurred in DFSMShsm processing that must be
addressed. Example 14-18 shows an example of an error message that might be issued to
the operator.

_Example 14-18  Output from a DFSMShsm error message_

    *11.25.42 STC00016 *ARC0744E MCDS COULD NOT BE BACKED UP, RC=0036,
    *ARC0744E (CONT.) REAS=0000. MIGRATION, BACKUP, FRBACKUP, DUMP, AND
    *ARC0744E (CONT.) RECYCLE HELD.

Error messages need to be reviewed to determine why the error occurred and what action
needs to be performed to correct the error.

**Informational messages**
_Informational messages_ show the user the functions that are processing in DFSMShsm.
Example 14-19 shows an example of an informational message that might be issued to the
operator.

_Example 14-19  Output from a DFSMShsm informational message_

    11.43.01 STC00016 ARC0101I QUERY SETSYS COMMAND COMPLETED ON HOST=1

Informational messages need to be monitored to ensure that DFSMShsm is functioning
correctly.

**Reviewing DFSMShsm messages**
We recommend that you review the logs for these messages regularly and become familiar
with how to look up the meanings of the identified DFSMShsm messages. The DFSMShsm
messages are documented in _z/OS MVS System Messages Vol 2 (ARC-ASA),_ SA22-7632.
This manual provides a description of each message and the actions, if any, that are
recommended to address the condition that is described in the message. Example 14-20
shows an example of an ARC1001I informational message.

_Example 14-20  Output of an ARC1001I message_

    ARC1001I DERDMANN.MIGTST MIGRATE FAILED, RC=0019, REAS=0001
    ARC1219I DATA SET IN USE BY ANOTHER USER OR JOB, MIGRATION REJECTED

The Explanation section for this message, which is shown in Figure 14-1 
indicates that the attempted migration of data set DERDMANN.MIGTST failed with an RC19,
REAS=0001 and that the ARC1219I message must be referenced to determine the meaning
of the RC19.


-----


the ARC1219I message indicates that the migration failed because
the data set was in use by another user or job. As the explanation indicates, the REAS=0001
from the ARC1001I must be used to match to the corresponding reason code meaning or
“Reascode Meaning” that is described in the ARC1219I message.

In this example, the REAS=0001 indicates that the migration failed because an error occurred
in allocating the non-VSAM data set to be migrated.


-----

The System Action and Application Programmer Response show at the end of each
message. The System Action indicates the action that DFSMShsm takes when the condition
that is described by this message occurs. The Application Programmer Response indicates
the action that the user must take when the condition that is described by this message
occurs.
space management operation of the data set and that if that data set requires migration, the
migration must be tried again when the data set is not in use.

If any DFSMShsm messages are identified in these logs that you do not think your
DFSMShsm environment needs to be receiving, contact the IBM Support Center for
assistance.

**z/OS DFSMS V2.1 VSAM record-level sharing error recovery**
If VSAM record-level sharing (RLS) is managing the DFSMShsm CDSs, DFSMShsm needs
the SMSVSAM server to be up and running. If the SMSVSAM server is unavailable, updates
to the CDSs will not occur. In previous releases, z/OS DFSMShsm was shut down and the
ARC0061I message was issued:

ARC0061I DFSMSHSM SHUTTING DOWN DUE TO SMSVSAM SERVER ERROR

Figure 14-4 explains why the message occurred.

    **Explanation** : An error with the SMSVSAM server has caused DFSMShsm to lose
    access to its control data sets. Either the SMSVSAM server did not initialize
    in the allotted amount of time, the SMSVSAM server was repeatedly terminating,
    or DFSMShsm was unable to open one or more of the CDSs after a server error
    occurred. All attempts to read, write, delete or update control data set
    records will fail. Most functions currently being processed will fail. Only
    those functions that are allowed to continue while DFSMShsm is in emergency
    mode will continue to be processed. To regain access to the control data sets,
    DFSMShsm must shut down and be restarted.
    **System action** : DFSMShsm is placed into emergency and shutdown modes. An abend
    is issued.

_Figure 14-4  Explanation of ARC0061I_

DFSMShsm will quiesce all CDS I/O activity. ARC0064E is issued, as seen in Figure 14-5 . Automation can be used to monitor for this message so that storage administrators
can be notified that this situation occurred and determine what happened. Both SMSVSAM
and DFSMShsm can be restarted.


-----

    ARC0064E SMSVSAM SERVER ERROR OCCURRED
    **Explanation:** An SMSVSAM server error caused DFSMShsm to lose access to its
    control data sets, and the CDS's had to be closed and reopened. One or more
    DFSMShsm requests may have failed due to the server error.
    **System action:** DFSMShsm processing continues.
    **System programmer response:** Search for DFSMShsm failure messages that were
    issued soon after the SMSVSAM server initialized. If errors occurred, we
    recommend you run AUDIT for the function(s) that failed. For example, for data
    set migration, backup or recycle errors, run AUDIT DATASETCONTROLS, for Fast
    Replication errors, run AUDIT COPYPOOLCONTROLS, etc. After the AUDIT is
    complete, failed requests can be reissued if necessary.
    Programmer response: None.
    Source: DFSMShsm

_Figure 14-5  New message ARC0064E_

## 14.3 Data recovery scenarios

Because data loss is always a possibility in computing, whether it is due to a user error or a
software defect, ensure that data recovery procedures are in place in case data loss occurs.

The _DFSMShsm Storage Administration Guide,_ SC26-0421, provides the following scenarios
to assist you with your data recovery procedures if data loss occurs in DFSMShsm:

- Damaged CDS and full journal

- Damaged journal and undamaged CDS

- Full journal and undamaged CDS

- Structurally damaged CDS and missing journal records

- Time sequence gaps in CDS records and journal records also missing

- Overwritten migration tape

- Overwritten backup tape

- Damaged ML1 volumes

- Reestablish access to previously deleted migrated data sets (no backup exists,
migration-level 2 (ML2) only)

- Correcting errors in the common recall queue

- Recovering a deleted ML1 data set without a backup

## 14.4 Auditing DFSMShsm

You can use the **AUDIT** command not only to detect and report discrepancies between CDSs,
catalogs, and DFSMShsm-owned volumes, but also to diagnose and often provide repairs for
these discrepancies. The **AUDIT** command is an effective tool for problem determination and
recovery.


-----

### 14.4.1 Introduction to the AUDIT command

To ensure data integrity, DFSMShsm uses numerous data set records to track individual data
sets. These records are contained in the following places:

- Master catalog, which is a list of data sets for the entire system

- User catalog, which is a list of data sets that are accessible from that catalog

- Journal, which keeps a running record of backup and migration transactions

- Small-data-set packing (SDSP) data sets on migration volumes

- Migration control data set (MCDS), which is an inventory of migrated data sets and
migration volumes

- Backup control data set (BCDS), which is an inventory of backed-up data sets and
volumes, dumped volumes, and backed-up aggregates

- Offline control data set (OCDS), which contains a tape table of contents (TTOC) inventory
of migration and backup tape volumes

In normal operation, these records stay in synchronization. However, because of data errors,
hardware failures, or human errors, these records can become unsynchronized. The **AUDIT**
command allows the system to cross-check the various records that relate to data sets and
DFSMShsm resources. The **AUDIT** command can list errors and propose diagnostic actions
or, at your option, complete certain repairs itself.

Consider the use of the **AUDIT** command for the following reasons:

- After any CDS restore (highly recommended)

- After an ARC184I message (error when reading or writing DFSMShsm CDS records)

- Errors on the RECALL or DELETE of migrated data sets

- Errors on BDELETE or RECOVER of backup data sets

- DFSMShsm tape-selection problems

- RACF authorization failures

- Power or hardware failure

- Consistency checking of fast replication CDS record relationships

- Periodic checks

- You can use the **AUDIT** command to cross-check the following sources of control
information:

  - MCDS or individual migration data set records

  - BCDS or individual backup data set records, ABARS records, or fast replicationrecords

  - OCDS or individual DFSMShsm-owned tapes

  - DFSMShsm-owned DASD volumes

  - Migration-volume records

  - Backup-volume records

  - Recoverable-volume records (from dump or incremental backup)

  - Contents of SDSP data sets

  - Fast replication dump, volume pair, version, and volume copy pool association records


-----

Use the **AUDIT** command at times of low system activity because certain executions of the
**AUDIT** command run a long time.

**Original and enhanced audit commands**
Two categories of audit commands exist: enhanced audit commands and original audit
commands. The _enhanced audit commands_ are designed to provide more comprehensive
auditing than the original audit commands. However, the _original audit commands_ are still
available to provide compatibility with CLISTs or job streams that were developed for use with
older versions of DFSMShsm. Because we recommend the use of the enhanced audit
commands over the original audit commands, the remainder of this chapter focuses on the
use of the enhanced audit commands.

**Required parameters and optional parameters**
When you issue the **AUDIT** command, you must specify one of the required parameters to
indicate a primary category of control information. This control information will direct or “drive”
the invocation of the audit. The optional parameters qualify what to audit, the output
destination, and other qualifying information.

For an enhanced audit, you must choose one of the following required parameters to drive the
audit:

- ABARSCONTROLS
- COMMONQUEUE
- COPYPOOLCONTROLS
- DATASETCONTROLS
- DIRECTORYCONTROLS VOLUMES
- MEDIACONTROLS
- VOLUMECONTROLS

Many optional parameters can be used when you execute an **AUDIT** command. For the list of
optional parameters that apply to each required **AUDIT** parameter, see the _DFSMShsm_
_Storage Administration Guide,_ SC26-0421.

**Important:** The two most important optional parameters are the **NOFIX** and **FIX**
parameters.

Use **NOFIX** if you want the **AUDIT** command to report any error that it detects and the actions
it can take to fix them but to _not_ take any of the fix actions.

To fix the errors that the **AUDIT** command can fix, you must specify the **FIX** parameter on
the **AUDIT** command.

For planned use of the **AUDIT** command, we recommend that you first execute the **AUDIT**
command with the **NOFIX** parameter. Then, after you review the errors that the **AUDIT**
command detects and the actions that the **AUDIT** command can take to fix them, execute
the **AUDIT** command with the **FIX** parameter.

Another important optional parameter is the **RESUME** parameter. Use the **RESUME** parameter
only with the **AUDIT MEDIACONTROLS** and **DATASETCONTROLS** parameters. The **RESUME** parameter
instructs DFSMShsm to start where it ended on a previous **AUDIT** command if that **AUDIT**
command ended before completion for a non-abnormal reason. For example, if an **AUDIT**
**DATASETCONTROLS(MIGRATION)** command is executing but causing contention that prevents
other DFSMShsm tasks from executing, you might want to issue the **HOLD AUDIT** command.
This command will stop the audit as soon as it finishes processing the current data set and
allow the other DFSMShsm functions to proceed.


-----

When the other DFSMShsm functions finish processing, you might want the audit to continue
where the previous audit ended rather than start the audit from the beginning. To continue the
previous audit, specify the **RESUME** parameter on the next issuance of the **AUDIT**
**DATASETCONTROLS(MIGRATION)** command.

**Note:** The **AUDIT MEDIACONTROLS RESUME** command can be executed only against a tape
and is only valid if the **FIX** parameter is specified. That is, both the **AUDIT** command that
was ended prematurely and the **AUDIT** command that is issued to resume processing must
specify the **FIX** parameter.

### 14.4.2 Auditing copy pools

DFSMShsm provides two methods of auditing the copy pools that it manages to ensure that
the fast replication CDS record relationships are correct:

- COPYPOOLCONTROLS

  Audits all copy pool records

- COPYPOOLCONTROLS( _cpname_ )

   the copy pool records for a specific copy pool

If any issues are experienced during fast replication and you decide to execute an audit of the
copy pool records to ensure that the fast replication CDS records exist and are accurate, be
sure to execute both forms of this **AUDIT** command. The **AUDIT** command of all of the copy
pools and the individual copy pool. These **AUDIT** commands execute different code paths and
detect and fix different discrepancies in the fast replication CDS records.

### 14.4.3 Auditing common queues

Common queues have interrelated entries that can become corrupted due to abends or
unexpected losses of connectivity. A corrupted common queue can cause certain requests to
not be processed. DFSMShsm automatically corrects certain inconsistencies, but for other
inconsistencies, it is necessary to issue the **AUDIT COMMONQUEUE** command. The **AUDIT**
command enables DFSMShsm to dynamically correct inconsistencies with minimal impact on
processing.

Consider the use of the **AUDIT COMMONQUEUE** command in the following situations:

- After receiving an ARC1506E message
- After receiving an ARC1187E message
- When recall requests are unexpectedly not being selected for processing

The internal structure of common queues is not externalized. An ARC1544I message is the
only output that the **AUDIT COMMONQUEUE** command returns. It does not return a specific
message for each error because individual error messages are of no value. The **OUTDATASET** ,
**SYSOUT** , and **TERMINAL** parameters are not used with **AUDIT COMMONQUEUE** .

Unlike other **AUDIT** command functions, auditing a common queue is not time-intensive and
can be performed at any time.


-----

### 14.4.4 AUDIT command examples

The following examples show how to instruct DFSMShsm to execute various **AUDIT**
commands and the output that is produced when different errors are detected. For a full list of
the errors that are detected by each **AUDIT** command, the fix actions that DFSMShsm can
take to address those errors, and troubleshooting hints for those errors that DFSMShsm
cannot fix on its own, see the _DFSMShsm Storage Administration Guide,_ SC26-0421.

**AUDIT DATASETCONTROLS(MIGRATION)**
When you specify the **AUDIT DATASETCONTROLS(MIGRATION)** command, DFSMShsm uses each
MCD record of a validly migrated data set in the MCDS to drive the audit. Example 14-21
shows that DFSMShsm detected two errors during this execution of the **AUDIT**
**DATASETCONTROLS(MIGRATION)** command: ERR 03 and ERR 22.

_Example 14-21  Output from the AUDIT DATASETCONTROLS(MIGRATION) command_

    -DFSMSHSM AUDIT-    ENHANCED AUDIT -- LISTING - AT 11:32:01 ON 12/08/21 FOR
    SYSTEM=3090
    COMMAND ENTERED:
    AUDIT DATASETCONTROLS(MIGRATION)ODS(DERDMANN.DSC.MIGOUT) NOFIX
    
    /* ERR 03 DERDMANN.MIGTST1 NOT CATALOGED, HAS MIGRATION COPY + */
    /* DFHSM.HMIG.T041811.DERDMANN.MIGTST1.A2234 */
    /* ERR 22 DERDMANN.MIGTST2 HAS NO ALIAS RECORD */
    /* FIXCDS A DFHSM.HMIG.T121811.DERDMANN.MIGTST2.A2234 CREATE(X'00000004' + */
    /*
    X'40404040404040404040404040404040404040404040404040404040404040404040404040404040
    40404040') */
    /* FIXCDS A DFHSM.HMIG.T121811.DERDMANN.MIGTST2.A2234 PATCH(X'00000004'
    DERDMANN.MIGTST2) */
    /* FIXCDS A DFHSM.HMIG.T121811.DERDMANN.MIGTST2.A2234 PATCH(X'00000000' M) */
    
    -  END OF -   ENHANCED AUDIT - LISTING -

When DFSMShsm can fix the errors that it detects, one or more **FIXCDS** commands will be
generated after the ERR number that is detected and before the next ERR number that is
listed. In Example 14-21, only the ERR 22 can be repaired by DFSMShsm; ERR 03 cannot.
Specifying the **FIX** parameter will apply the correct CDS updates to repair the ERR 22. To
address the ERR 03, review the troubleshooting hints that are listed in the _DFSMShsm_
_Storage Administration Guide,_ SC26-0421.

**AUDIT MEDIACONTROLS VOLUMES(volser)**
When you specify the **AUDIT MEDIACONTROLS VOLUMES(** **_volser_** **)** command, DFSMShsm uses
the information in the common data set descriptor (CDD), which is required at the start of
every DFSMShsm-generated copy, in each DFSMShsm-generated copy of a user data set on
the specified volume to drive the audit.


-----

Example 14-22 shows that DFSMShsm detected one error during this execution of the **AUDIT**
**MEDIACONTROLS VOLUMES(** **_volser_** **)** command: ERR 140.

_Example 14-22  Output from the AUDIT MEDIACONTROLS VOLUMES(volser) command_

    -DFSMSHSM AUDIT-    ENHANCED AUDIT -- LISTING - AT 14:45:18 ON 12/08/21 FOR
    SYSTEM=3090
    COMMAND ENTERED:
    AUDIT MEDIACONTROLS VOLUMES(MIG102) ODS(DERDMANN.MC.OUT) NOFIX
    
    /* ERR 140 MIG102 - DERDMANN.MIGTST2 HAS NO ALIAS RECORD */
    /* FIXCDS A DFHSM.HMIG.T121811.DERDMANN.MIGTST2.A2234 CREATE */
    
    -  END OF -   ENHANCED AUDIT - LISTING -

The **FIXCDS** command that is identified after the ERR 140 indicates that the ERR 140 can be
repaired by DFSMShsm. Specifying the **FIX** parameter will apply the correct CDS update to
repair the ERR 140.

**AUDIT VOLUMECONTROLS(MIGRATION)**
When you specify the **AUDIT VOLUMECONTROLS(MIGRATION)** command, DFSMShsm uses the
MCV records of all of the migration volumes in the MCDS to drive the audit. Example 14-23
shows that DFSMShsm detected one error during this execution of the **AUDIT**
**VOLUMECONTROLS(MIGRATION)** command: ERR 56.

_Example 14-23  Output from the AUDIT VOLUMECONTROLS(MIGRATION) command_

    -DFSMSHSM AUDIT-    ENHANCED AUDIT -- LISTING - AT 15:05:24 ON 12/08/22 FOR
    SYSTEM=3090
    COMMAND ENTERED:
    AUDIT VOLUMECONTROLS(MIGRATION) ODS(DERDMANN.VC.OUT2) NOFIX
    
    /* ERR 56 MC1 EXTENSION 01 MISSING */
    /* FIXCDS 1 L1VOL-00 VERIFY(X'00000001' BITS(1.......)) PATCH(X'00000001'
    BITS(0.......)) */
    
    -  END OF -   ENHANCED AUDIT - LISTING -

The **FIXCDS** command that is identified after the ERR 140 indicates that the ERR 56 can be
repaired by DFSMShsm. Specifying the **FIX** parameter will apply the correct CDS update to
repair the ERR 56.

**AUDIT COMMONQUEUE(RECALL)**
When you specify the **AUDIT COMMONQUEUE(RECALL)** command, DFSMShsm scans the entries
in the CRQ for logical inconsistencies in the structure.

Figure 14-6 shows the ARC1544I message that is issued only to the console. The ARC1544I
indicates that DFSMShsm did not detect any errors in the CRQ.

    13.52.42 SYSTEM1      f dfhsm,AUDIT COMMONQUEUE(RECALL) FIX
    13.52.42 SYSTEM1 STC00036 ARC0300I **OPER** ISSUED===>AUDIT
    COMMONQUEUE(RECALL) FIX
    13.52.42 SYSTEM1 STC00036 ARC1544I AUDIT COMMONQUEUE HAS COMPLETED, 0000
    ERRORS
    ARC1544I (CONT.) WERE DETECTED FOR STRUCTURE SYSARC_PLEX1_RCL, RC=00

_Figure 14-6  Output from the AUDIT COMMONQUEUE(RECALL) command_


-----

**AUDIT COPYPOOLCONTROLS**
When you specify the **AUDIT COPYPOOLCONTROLS** command, DFSMShsm reads sequentially
through all existing fast replication records and confirms the existence and accuracy of all
DFSMShsm records that are associated with copy pools. This command can detect orphaned
records. Example 14-24 shows that DFSMShsm detected four errors during this execution of
the **AUDIT COPYPOOLCONTROLS** command: ERR 180, ERR 178, and ERR 202.

_Example 14-24  Output from the AUDIT COPYPOOLSCONTROLS command_

    -DFSMSHSM AUDIT-    ENHANCED AUDIT -- LISTING - AT 11:59:56 ON 12/09/04 FOR
    SYSTEM=3090
    COMMAND ENTERED:
    AUDIT COPYPOOLCONTROLS ODS(DERDMANN.AUDIT.CPC.AFTER2)
    
    /* ERR 180 J RECORD NOT FOUND FOR VOLUME SRC003, IT BELONGS TO COPY POOL CP1 ,
    VER=001*/
    /* ERR 178 FRTV (I) RECORD FOR TARGET VOLUME TAR001, EXPECTS FRSV (J) RECORD FOR
    SOURCE VOLUME SRC003, WHICH WAS NOT FOUND */
    /* ERR 202 ORPHANED I (FRTV) RECORD FOUND FOR TARGET VOLUME ABC123, SOURCE VOLUME
    ...... */
    /* FIXCDS I ABC123 DELETE */
    /* ERR 178 FRTV (I) RECORD FOR TARGET VOLUME ABC123, EXPECTS FRSV (J) RECORD FOR
    SOURCE VOLUME ......, WHICH WAS NOT FOUND */
    /* ERR 178 FRTV (I) RECORD FOR TARGET VOLUME TAR001, EXPECTS FRSV (J) RECORD FOR
    SOURCE VOLUME SRC003, WHICH WAS NOT FOUND */
    
    -  END OF -   ENHANCED AUDIT - LISTING -

The **FIXCDS** command that is identified after the ERR 202 indicates that the ERR 202 can be
repaired by DFSMShsm. Specifying the **FIX** parameter will apply the correct CDS update to
repair the ERR 202. To address the ERR 180 and ERR 178, you must review the
troubleshooting hints that are listed in the _DFSMShsm Storage Administration Guide,_
SC26-0421.

**AUDIT COPYPOOLCONTROLS(copypoolname)**
When you specify the **AUDIT COPYPOOLCONTROLS(** **_copypoolname_** **)** command, the **AUDIT**
command reads the fast replication control data set (CDS) records to confirm the existence
and accuracy of all DFSMShsm records that are associated with the copy pool.

**Note:** This form of the **AUDIT COPYPOOLCONTROLS** command cannot detect orphaned
records.


-----

Example 14-25 shows that DFSMShsm detected two errors during this execution of the **AUDIT**
**COPYPOOLCONTROLS(** **_cpname_** **)** command: ERR 180 and ERR 178.

_Example 14-25  Output from the AUDIT COPYPOOLCONTROLS(copypoolname) command_

    -DFSMSHSM AUDIT-    ENHANCED AUDIT -- LISTING - AT 12:01:28 ON 12/09/04 FOR
    SYSTEM=3090
    COMMAND ENTERED:
    AUDIT COPYPOOLCONTROLS(CP1) ODS(DERDMANN.AUDIT.CP1.OUTPUT)
    
    /* ERR 180 J RECORD NOT FOUND FOR VOLUME SRC003, IT BELONGS TO COPY POOL CP1 ,
    VER=001                */
    /* ERR 178 FRTV (I) RECORD FOR TARGET VOLUME TAR001, EXPECTS FRSV (J) RECORD FOR
    SOURCE VOLUME SRC003, WHICH WAS NOT FOUND */
    
    -  END OF -   ENHANCED AUDIT - LISTING -

To address the ERR 180 and ERR 178, you must review the troubleshooting hints that are
listed in the _DFSMShsm Storage Administration Guide,_ SC26-0421.

**AUDIT ABARSCONTROLS(agname)**
When you specify the **AUDIT ABARSCONTROLS(** **_agname_** **)** command, DFSMShsm uses the ABR
record for the aggregate that is specified to drive the audit. If you do not specify an aggregate,
all of the aggregates in the BCDS are audited. Example 14-26 shows that DFSMShsm
detected one error during this execution of the **AUDIT** command: ERR 168.

_Example 14-26  Output from the AUDIT ABARSCONTROLS(agname) command_

    -DFSMSHSM AUDIT-    ENHANCED AUDIT -- LISTING - AT 14:26:14 ON 12/09/04 FOR
    SYSTEM=3090
    COMMAND ENTERED:
    AUDIT ABARSCONTROLS(DEREK) ODS(DERDMANN.AUDIT.ABARS.OUTPUT)
    
    /* ERR 168 - CONTROL FILE NAME = DEREK.C.C01V0001 , + */
    INDICATED IN THE ABR RECORD, BUT IS NOT CATALOGED. ABR RECORD KEY =
    DEREK.2012248000101 */

To address the ERR 168, you must review the troubleshooting hints that are listed in the
_DFSMShsm Storage Administration Guide,_ SC26-0421.

**AUDIT DIRECTORYCONTROLS VOLUMES(volser)**
When you specify the **AUDIT DIRECTORYCONTROLS VOLUMES(** **_volser_** **)** command, DFSMShsm
uses the data set control blocks (DSCBs) on the specified volume with the attributes that
DFSMShsm assigns to its data sets to drive the audit. Other DSCBs are ignored.

Example 14-27 on page 426 shows that DFSMShsm detected one error during this execution
of the **AUDIT DIRECTORYCONTROLS VOLUMES(** **_volser_** **)** command: ERR 140.


-----

_Example 14-27  Output from the AUDIT DIRECTORYCONTROLS VOLUMES(volser) command_

    -DFSMSHSM AUDIT-    ENHANCED AUDIT -- LISTING - AT 15:40:58 ON 12/09/04 FOR
    SYSTEM=3090
    COMMAND ENTERED:
    AUDIT DIRECTORYCONTROLS VOLUMES(MIG101) ODS(DERDMANN.AUDIT.DC.OUTPUT)
    
    /* ERR 140 MIG101 -  DFHSM.HMIG.T120115.DERDMANN.AUDIT.A2248 HAS NO ALIAS RECORD
    /* AUDIT DATASETCONTROLS(MIGRATION) LEVELS(DERDMANN.AUDIT)
    
    -  END OF -   ENHANCED AUDIT - LISTING -

The output after the ERR 140 indicates that you must execute the **AUDIT**
**DATASETCONTROLS(MIGRATION) LEVELS(DERDMANN.AUDIT)** command to further diagnose the
ERR 140.


-----

# 15. IBM Health Checker update

This chapter addresses a Data Facility Storage Management Subsystem (DFSMS) update
that relates to health checks and also summarizes the available products for DFSMS and
storage-related health checks.


-----

## 15.1 Health Checker function

The IBM Health Checker examines the health of selected system components at regular
intervals. It monitors settings, looks for single points of failure in the setup, and issues
warnings that are based on its findings. The settings for the Health Checker are in the
SYS1.PARMLIB member HZSPRMxx. To change the settings in the SYS1.PARMLIB member
HZSPRMxx, use System Display and Search Facility (SDSF) statements or the **F hzsproc**
command for dynamic updates. You can activate, deactivate, or update health checks by
using the **F hzsproc** command.

Example 15-1 lists examples of operating the Health Checker.

_Example 15-1  Examples of operating the Health Checker by command_

    S hzsproc /* Start health checker */
    F hzsproc,RUN,CHECK=(ibmhsm,*) /* Run a group of health checks */
    F hzsproc,RUN,CHECK=(ibmsms,sms_cds_reuse_option) /* Run individ. health check */
    F hzsproc,UPDATE,CHECK=(ibmsms,sms_cds_reuse_option),severity=(med) /*Update */

The interval between checks is set by the **INTERVAL** parameter or by a policy. All health checks
have a severity (LOW, MEDIUM, or HIGH) that is defined on the check. You can set this
severity to suit your installation’s needs. Currently, 500 health checks are available, but this
section focuses only on the checks that relate to DFSMS.

### 15.1.1 New DFSMS health check

The device support health check on tape libraries was added to the z/OS Health Checker
reporting on tape library errors that occur during IPL. This health check routine,
DMO_TAPE_LIBRARY_INIT_ERRORS, was designed to include more device support health
checks in the future. It is the only new health check that relates to DFSMS that was added in
DFSMS V1.13. The report shows explanations and suggested remedies. For more
information about this check, see the _IBM Health Checker for z/OS User’s Guide_ _,_ SA22-7994.
The following messages might be displayed in the report:

DMOH0101I CHECK(DMO_TAPE_LIBRARY_INIT_ERRORS) ran successfully and found no
exceptions.
DMOH0102I text.
DMOH0104E CHECK(DMO_TAPE_LIBRARY_INIT_ERRORS) determined that library device
initialization errors occurred during IPL.
DMOH0105I The check is not applicable in the current environment because there are
no ape libraries defined.

For more information, see Appendix A of _MVS System Messages and Codes, Volume 4,_
SA22-7634.

**Overview of current health checks that relate to DFSMS**
The list of current DFSMS and storage-related health checks spans multiple areas:

- Catalog
- Tape libraries
- Data Facility Hierarchical Storage Management System (DFHSM)
- Data Facility Removable Media Manager (DFRMM)
- z/OS file system (zFS)
- Systems-managed buffering
- DFSMS




- VSAM
- VSAM record-level sharing (RLS)

The following list shows the currently available health checks:

- CATALOG_IMBED_REPLICATE
- DMO_TAPE_LIBRARY_INIT_ERRORS.
- HSM_CDSB_BACKUP_COPIES
- HSM_CDSB_DASD_BACKUPS
- HSM_CDSB_VALID_BACKUPS
- PDSE_SMSPDSE1
- ZOSMIGV1R10_RMM_REJECTS_DEFINED
- ZOSMIGV1R10_RMM_VOL_REPLACE_LIM
- ZOSMIGV1R10_RMM_VRS_DELETED
- ZOSMIGV1R11_RMM_DUPLICATE_GDG
- ZOSMIGV1R11_RMM_REXX_STEM
- ZOSMIGV1R11_RMM_VRSEL_OLD
- SMB_NO_ZFS_SYSPLEX_AWARE
- ZOSMIGREC_SMB_RPC.
- SMS_CDS_REUSE_OPTION
- SMS_CDS_SEPARATE_VOLUMES.
- VSAM_INDEX_TRAP
- VSAMRLS_CFCACHE_MINIMUM_SIZE
- VSAMRLS_CFLS_FALSE_CONTENTION
- VSAMRLS_DIAG_CONTENTION
- VSAMRLS_QUIESCE_STATUS
- VSAMRLS_SHCDS_CONSISTENCY
- VSAMRLS_SHCDS_MINIMUM_SIZE
- VSAMRLS_SINGLE_POINT
- ZOSMIGV1R11_ZFS_INTERFACELEVEL
- ZOSMIGREC_ZFS_RM_MULTIFS
- ZOSMIGV1R11_ZFS_RM_MULTIFS
- ZOSMIGV1R13_ZFS_FILESYS

For more information about health checks, see _IBM Health Checker for z/OS: User’s Guide,_
SA22-7994.

**SMS health check function from SDSF**
To access the IBM Health Checker from the TSO SDSF primary menu, enter CK from the
command line or select **CK** from the displayed list (Figure 15-1 on page 430). SDSF is a
separately priced program.


-----

    HQX7780 ----------------- SDSF PRIMARY OPTION MENU -------------------------COMMAND INPUT ===> CK SCROLL ===> CSR
    
    DA  Active users           INIT Initiators
    I   Input queue            PR  Printers
    O   Output queue           PUN  Punches
    H   Held output queue         RDR  Readers
    ST  Status of jobs          LINE Lines
    NODE Nodes
    LOG  System log            SO  Spool offload
    SR  System requests          SP  Spool volumes
    MAS  Members in the MAS        NS  Network servers
    JC  Job classes            NC  Network connections
    SE  Scheduling environments
    RES  WLM resources           RM  Resource monitor
    ENC  Enclaves             CK  Health checker
    PS  Processes
    ULOG User session log
    END  Exit SDSF

_Figure 15-1  Accessing Health Checker with CK_

If you want to filter by CheckOwner, enter FILTER CHECKOWNER EQ IBMDMO . Figure 15-2 shows
the new health check on tape libraries on the startup after the IPL.

    SDSF HEALTH CHECKER DISPLAY (ALL)           LINE 44-61 (489)
    COMMAND INPUT ===>                      SCROLL ===> CSR
    PREFIX=ANDRE* DEST=(ALL) OWNER=* SYSNAME=*
    NP  NAME               CheckOwner    State Status
    S DMO_TAPE_LIBRARY_INIT_ERRORS   IBMDMO      ACTIVE(ENABLED)  SUCCE
    GRS_AUTHQLVL_SETTING       IBMGRS      ACTIVE(ENABLED) **EXCEP**
    GRS_CONVERT_RESERVES       IBMGRS      ACTIVE(DISABLED)  GLOBA
    
    .......... more health checks ...

_Figure 15-2  New health check on tape libraries on the startup after the IPL_

In Figure 15-2, note the headings: Name, CheckOwner, State, and Status. You can also scroll
to the right for more headings. The health checks are grouped as shown in the column
CheckOwner. You can activate or run a health check with a command against the individual
health check or group. You can verify at the far right side that this health check ran
successfully. Select the check DMO_TAPE_LIBRARY_INIT_ERRORS by entering S under
the heading NP . Figure 15-3 on page 431 shows the verification results.


-----

    SDSF OUTPUT DISPLAY DMO_TAPE_LIBRARY_INIT_ERRORS LINE 0    COLUMNS 02- 81
    COMMAND INPUT ===>                      SCROLL ===> CSR
    ********************************* TOP OF DATA**********************************
    CHECK(IBMDMO,DMO_TAPE_LIBRARY_INIT_ERRORS)
    START TIME: 09/21/2011 12:42:11.448711
    CHECK DATE: 20100128 CHECK SEVERITY: LOW
    
    DMOH0101I CHECK(DMO_TAPE_LIBRARY_INIT_ERRORS) ran successfully and found
    no exceptions
    
    END TIME: 09/21/2011 12:42:11.449351 STATUS: SUCCESSFUL

_Figure 15-3  Results of health check verification_

To demonstrate a health check with exceptions, select **SMS_CDS_SEPARATE_VOLUMES** ,
as shown in Figure 15-4. In this case, an exception was identified and explained, and a
recommendation to improve the setup was offered.

CHECK(IBMSMS,SMS_CDS_SEPARATE_VOLUMES)
START TIME: 10/06/2011 13:50:46.469308
CHECK DATE: 20090303 CHECK SEVERITY: MEDIUM

-  Medium Severity Exception *

IGDH1001E CHECK(IBMSMS,SMS_CDS_SEPARATE_VOLUMES) detected the
ACDS (SYS1.SMS.ACDS)
and COMMDS (SYS1.SMS.COMMDS)
allocated on the same volume.

Explanation: As a best practice, an ACDS/COMMDS must reside on a
volume, accessible from all systems in the SMS complex. To ease
recovery in case of failure, the ACDS should reside on a different
volume than the COMMDS. Also, you should allocate a spare ACDS on a
different volume. The control data set (ACDS or COMMDS) must reside
on a volume that is not reserved by other systems for a long period
of time because the control data set (ACDS or COMMDS) must be
available to access for SMS processing to continue.

_Figure 15-4  Health check with exceptions_

### 15.1.2 Upgrade and coexistence considerations

No particular upgrade action is required. The IBM Health Checker is an established, standard
product that adds new health checks automatically. The follow-up can be automated through
your automation package, which is based on the severity settings that you choose for your
environment. For more information about receiving email alerts, see _Exploiting the IBM_
_Health Checker for z/OS Infrastructure_ , REDP-4590.



-----

