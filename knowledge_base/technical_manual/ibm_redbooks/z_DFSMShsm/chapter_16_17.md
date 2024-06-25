# 16. Common DFSMShsm functions

In this chapter, we describe functions that are common to multiple areas of DFSMShsm,
which allows us to document these functions in one place only. 

## 16.1 Changing priorities in DFSMShsm

In this section, we describe changing priorities in DFSMShsm.

### 16.1.1 Priority processing

You might see NOWAIT recalls and deletions queue up because many of these NOWAIT
recalls and deletions were submitted at the same time. These NOWAIT recalls and deletions
end up in the same DFSMShsm Management Work Elements (MWEs) queue. DFSMShsm
will give all of the WAIT requests higher priority than the NOWAIT requests. All users will start
one NOWAIT request before another user starts a second. The prioritization sequence will be
from the oldest to the newest request.

Recalls from tape are an exception. When a migration-level 2 (ML2) tape is mounted, all recall
requests that are going to this tape will be processed before this tape is unmounted. Recalls
from the tape are in prioritization order and continue on the condition that the
TAPERECALLLIMITS value is not exceeded.

When the TAPERECALLLIMITS value is reached and a high priority recall toward the same
tape is pending on another system, the recall process in this tape will end, even though not all
recalls were processed. If no higher priority recalls are in the queue, recalls will continue on
the tape that is mounted. However, a check is made after each recall to see whether a new,
higher-priority recall appeared.

**Using the ARCRPEXT**
The Return Priority installation exit can be installed to change priority when requests are
added to the queue. Prioritization can be performed for WAIT requests relative to other WAIT
requests and similarly for NOWAIT requests. You can also give recall requests a higher
priority than delete requests. In the same way, you can give interactive requests a higher
priority than batch-initiated requests.

DFSMShsm will prime the priority field with the value of 50 . With the ARCRPEXT, you can
change this value to a value 0 - 100 (100 is the highest priority). You can change this value for
both WAIT and NOWAIT requests. Two requests with the same priority will be processed in
first-in first-out (FIFO) order.

For RECALL and DELETE requests, “priority order” on the recall queue has the following
meaning:

- WAIT requests are queued before NOWAIT requests.

- Priorities of WAIT requests can be relative to other WAIT requests.

- Priorities of NOWAIT requests are relative to other NOWAIT requests (but after WAIT
requests):

  - NOWAIT requests (if any) with a relative priority of _50_ are interleaved by the user ID of
the submitter with other NOWAIT requests of priority 50.

  - NOWAIT requests with a relative priority _greater than 50_ are queued before those
NOWAIT requests with a priority of 50, without interleaving.

  - NOWAIT requests with a relative priority _less than 50_ are queued after those NOWAIT
requests with a priority of 50, without interleaving.

You can assign priority values other than 50 for those NOWAIT requests that you do not want
interleaved by user ID.


-----

For data set RECOVER requests, the following prioritization applies:

- WAIT requests are queued before NOWAIT requests.
- WAIT requests can be prioritized relative to other WAIT requests.
- NOWAIT requests can be prioritized after other NOWAIT requests, but after WAIT
requests.

For volume recover requests, the following priority sequence on the queue applies:

- WAIT requests are queued ahead of NOWAIT requests.
- WAIT requests can be prioritized relative to other WAIT requests.
- NOWAIT requests can be prioritized relative to other NOWAIT requests, but after WAIT
requests.

### 16.1.2 Tape Take Away in a single DFSMShsm host environment

The term _Tape Take Away_ in DFSMShsm is used to cover the situation where a long-running
function is preventing a request from being processed, for example, an ML2 recall from being
processed because the tape request for the recall is used by a long-running function.

Tape Take Away can be requested by the recall function, and Tape Take Away occurs fast and
smoothly. (The tape is released for the recall.)

For migration and recycle output, a tape might be needed that is in use by a long-running
function. In this case, migration or recycle will select a different tape for output.

If a recall needs a migration volume, Tape Take Away will start and the migration task will
close the requested volume and continue on a different output volume. This situation is valid
for any type of migration: primary, secondary, interval, and command-initiated migration.

Tape Take Away for recalls can also occur for long-running tasks, such as RECYCLE and
TAPECOPY input tapes.

Recycle occurs by releasing the connected set that is being processed and by moving on to
the next set. TAPECOPY is slightly different because a time gap exists before Take Away
occurs and TAPECOPY might finish successfully within this time.

Contention is also possible with recall itself. If many recalls are being processed, a higher
priority recall might suffer, due to missing available recall tasks. In this case, Tape Take Away
can also be scheduled.

For an ABACKUP contention with recall, two scenarios exist. If the recall is type NOWAIT,
Tape Take Away does not occur. If the recall request type is WAIT, the Tape Take Away starts
only after 10 minutes, which leaves the ABACKUP task a chance to finish before the tape is
requested by recall.

For ARECOVER, the recall request will not schedule a Tape Take Away. It will try to execute
the recall only within the specified limit for retries.

In general, if a tape is not freed up after 15 retries, DFSMShsm will send message
ARC0380A to the operator console with the options of responding: WAIT, CANCEL, or
MOUNT.

The WAIT option will reset the count and a new series of 15 retries can start with a
two-minute interval.


-----

ABACKUP can also invoke a Tape Take Away from RECYCLE or TAPECOPY input tapes. For
RECYCLE, the approach will be as for a recall Tape Take Away: Recycle will end on the
current connected set and will move on to the next one. For TAPECOPY, a time delay occurs
before Take Away occurs, leaving TAPECOPY the opportunity to finish before the tape is
taken away.

**Tape Take Away in a multiple DFSMShsm host environment**
The Tape Take Away process is similar in a single DFSMShsm host and a multi-DFSMShsm
environment. Therefore, this section will explain only the difference.

If a recall request to an ML2 tape is requested, and this ML2 tape is already in use by another
DFSMShsm host, TAPERECALLLIMITS sets the time value for when a Tape Take Away can
occur. Tape Take Away will only occur however if the upcoming recall has a higher priority
than the one already in process. The behavior differs in a common recall queue (CRQ)
environment, where all recalls in the CRQ queue to the same tape occur in the same tape
mount.

If a RECYCLE requests an input or output tape that is being used in a RECYCLE on a
different DFSMShsm host, the request will fail.

### 16.1.3 Altering priority by using the ALTERPRI command

There is a new DFSMShsm command, **ALTERPRI** , to alter the priority of queued requests.

You can alter the priority of the following request types:

- ABACKUP
- ARECOVER
- BACKDS
- BACKVOL
- DELETE
- FRBACKUP
- FREEVOL
- FRRECOV
- MIGRATE
- RECALL
- RECOVER

Use the **ALTERPRI** command to alter the priority of queued requests on an as-needed basis.
You must not use this command as the primary means of assigning priority values to new
requests.

Two options of setting priority are available with the **ALTERPRI** command:

- The **HIGH** parameter, which is the default, alters the specified request so that it has the
highest priority on its respective queue.

- Conversely, the **LOW** parameter alters the request so that it has the lowest priority on its
respective queue.

If you do not give any parameter, **HIGH** will be the default.

**Note:** You cannot alter the priority of **BACKVOL CDS** commands and requests that are
already selected for processing.


-----

The mutually exclusive **REQUEST** , **USER** , or **DATASETNAME** parameters indicate the requests that
DFSMShsm will reprioritize. Use the **QUERY REQUEST** command to determine the request
number to issue on the **ALTERPRI** command. DFSMShsm reprioritizes all queued requests
that match the REQUEST, USERID, or DATASETNAME criteria that are specified on the
**ALTERPRI** command.

To reprioritize a recall request on the CRQ, issue the **ALTERPRI** command on the same host
that originated the recall request. The syntax for the command is shown in Example 16-1.

Example 16-1 and Figure 16-1 provide examples of changing priority on DFSMShsm
requests.

_Example 16-1  Changing the DFSMShsm priority by using the ALTERPRI command_

     ALTERPRI DATASETNAME(USERA.TEST01) /* default used = HIGH */
     ALTERPRI REQUEST(nnnn) HIGH
     ALTERPRI USERID(USERA) LOW
     
The possible (optional) values of priority setting are HIGH or LOW ( HIGH is the default).

Setting these values will move a request to the highest priority on the queue (if HIGH is
requested) or to the lowest priority on the queue (if LOW is requested).

Certain commands will generate multiple requests with the same request number:

- BACKVOL STORAGEGROUP
- BACKVOL VOLUMES
- FRBACKUP DUMP
- FRBACKUP DUMPONLY
- FRRECOV DSNAME

If you change the priority on one of these requests by request number, all requests will get the
new priority.

**Example of changing priority by using ALTERPRI**
In this scenario, we show a DFSMShsm recall queue with a number of data sets that are
waiting. The lowest priority data set recall is at the bottom of the list in Example 16-2.

_Example 16-2  QUERY REQUEST on a DFSMShsm recall queue_

    -F DFHSM64,Q REQUEST
    STC20554 ARC0101I QUERY REQUEST COMMAND STARTING ON HOST=2
    STC20554 ARC1543I RECALL MWE FOR DATASET MHLRES4.DUMP.OUT.#1,
    ARC1543I (CONT.) FOR USER MHLRES4, REQUEST 00000211, WAITING TO BE
    ARC1543I (CONT.) PROCESSED ON A COMMON QUEUE,00000000 MWES AHEAD OF
    ARC1543I (CONT.) THIS ONE
    STC20554 ARC1543I RECALL MWE FOR DATASET MHLRES4.DUMP.OUT.#2,
    ARC1543I (CONT.) FOR USER MHLRES4, REQUEST 00000212, WAITING TO BE
    ARC1543I (CONT.) PROCESSED ON A COMMON QUEUE,00000001 MWES AHEAD OF
    ARC1543I (CONT.) THIS ONE
    STC20554 ARC1543I RECALL MWE FOR DATASET MHLRES4.DUMP.OUT.#3,


-----

    ARC1543I (CONT.) FOR USER MHLRES4, REQUEST 00000213, WAITING TO BE
    ARC1543I (CONT.) PROCESSED ON A COMMON QUEUE,00000002 MWES AHEAD OF
    ARC1543I (CONT.) THIS ONE
    STC20554 ARC1543I RECALL MWE FOR DATASET MHLRES4.TCPIP.PROFILE ,
    ARC1543I (CONT.) FOR USER MHLRES4, REQUEST 00000214, WAITING TO BE
    ARC1543I (CONT.) PROCESSED ON A COMMON QUEUE,00000003 MWES AHEAD OF
    ARC1543I (CONT.) THIS ONE

The lowest priority recall is marked in bold in Example 16-3. Now, ALTERPRI is executed with
the REQUEST number to change the priority to HIGH (equivalent to moving the recall request
to the top of the queue). See the command in Example 16-3.

_Example 16-3  Change priority by using ALTERPRI on the lowest priority recall_

    F DFHSM64,ALTERPRI REQUEST( **00000214** ) HIGH
    ARC0980I ALTERPRI REQUEST COMMAND STARTING
    ARC0982I RECALL MWE FOR DATA SET MHLRES4.TCPIP.PROFILE 546
    ARC0982I (CONT.) FOR USER MHLRES4, REQUEST 00000214, REPRIORITIZED TO
    ARC0982I (CONT.) HIGH
    ARC0981I ALTERPRI REQUEST COMMAND COMPLETED, RC=0000

The priority is successfully changed as we can see from the message in the command
output. Performing a new **QUERY REQUEST** command also confirms that the request number
that was changed moved to the top of the list. See the output from the query in Example 16-4.

_Example 16-4  QUERY REQUEST that shows the changed priority after the ALTERPRI command_

    -F DFHSM64,Q REQUEST
    STC20554 ARC0101I QUERY REQUEST COMMAND STARTING ON HOST=2
    STC20554 ARC1543I RECALL MWE FOR DATASET MHLRES4.TCPIP.PROFILE ?,
    ARC1543I (CONT.) FOR USER MHLRES4, REQUEST 00000214, WAITING TO BE
    ARC1543I (CONT.) PROCESSED ON A COMMON QUEUE,00000000 MWES AHEAD OF
    ARC1543I (CONT.) THIS ONE
    STC20554 ARC1543I RECALL MWE FOR DATASET MHLRES4.DUMP.OUT.#1,
    ARC1543I (CONT.) FOR USER MHLRES4, REQUEST 00000211, WAITING TO BE
    ARC1543I (CONT.) PROCESSED ON A COMMON QUEUE,00000001 MWES AHEAD OF
    ARC1543I (CONT.) THIS ONE
    STC20554 ARC1543I RECALL MWE FOR DATASET MHLRES4.DUMP.OUT.#2,
    ARC1543I (CONT.) FOR USER MHLRES4, REQUEST 00000212, WAITING TO BE
    ARC1543I (CONT.) PROCESSED ON A COMMON QUEUE,00000002 MWES AHEAD OF
    ARC1543I (CONT.) THIS ONE
    STC20554 ARC1543I RECALL MWE FOR DATASET MHLRES4.DUMP.OUT.#3,
    ARC1543I (CONT.) FOR USER MHLRES4, REQUEST 00000213, WAITING TO BE
    ARC1543I (CONT.) PROCESSED ON A COMMON QUEUE,00000003 MWES AHEAD OF
    ARC1543I (CONT.) THIS ONE
    STC20554 ARC0101I QUERY REQUEST COMMAND COMPLETED ON HOST=2

**Protecting an ALTERPRI command**
Each storage administrator command can be protected through the following fully qualified
discrete FACILITY class profile: STGADMIN.ARC. _command_ .

In this case, a security administrator can create the fully qualified, discrete profile to authorize
this command to storage administrators. You can obtain more details, such as the entire
command list and specific RACF profiles, in the _DFSMShsm Implementation and_
_Customization Guide_ , SC35-0418.


-----

# 17. Hints, tips, and suggested practices

In this chapter, we provide useful hints and tips, and recommend the preferred practices for
implementing DFSMShsm and the daily administration of DFSMShsm.


-----

## 17.1 Hints and tips

We provide hints and tips that were useful in our environment.

### 17.1.1 Use REGION=0M when you start DFSMShsm

Specify 0M for SIZE, which directs DFSMShsm to allocate the largest possible region size.
This specification reduces the chance that DFSMShsm will end abnormally because of an
out-of-space condition (ABEND878).

### 17.1.2 Large format journal

If you must back up the control data sets (CDSs) more than once a day due to the journal
filling up, consider allocating the journal as a large format sequential data set. Because the
journal can be allocated only on a single volume, allocating the journal as a large format data
set allows the journal to grow beyond 65535 tracks in size on that volume. Allocating the
journal as a large format data set allows more DFSMShsm activity to take place between
journal backups and helps to avoid journal full conditions.

### 17.1.3 CDS recovery

You do not want to be unprepared to recover one or more CDSs if one or more of the CDSs is
corrupted. The _DFSMShsm Storage Administration Guide,_ SC26-0421, contains step-by-step
instructions in the enhanced CDS recovery function section to guide you through the
complete recovery of a CDS from a backup copy. We recommend that you execute these
steps on a test system to become familiar with the enhanced CDS recovery function if you
need to use the procedure to recover from an actual corrupted CDS condition.

### 17.1.4 Delete migrated data sets without unnecessary recall

When users are deleting data sets with DFSMShsm, users can now bypass the unnecessary
recall for |PGM=IEFBR14 job steps by updating the ALLOCxx PARMLIB member to include a
new statement, SYSTEM IEFBR14_DELMIGDS(NORECALL) , or by using the new **SETALLOC**
command to enable no recall.

### 17.1.5 Using ARCCATGP

Issuing the **UNCATALOG** , **RECATALOG** , or **DELETE NOSCRATCH** command against a migrated data set
causes the data set to be recalled before the operation is performed. You can authorize
certain users however to issue these commands without recalling the migrated data sets by
connecting the user to the RACF group ARCCATGP . When a user is logged on under RACF
group ARCCATGP , DFSMShsm bypasses the automatic recall for UNCATALOG,
RECATALOG, and DELETE NOSCRATCH requests for migrated data sets.


-----

The following tasks are used to enable DFSMShsm to bypass automatic recall during catalog
operations:

- Define RACF group ARCCATGP by using the following RACF command:

  `ADDGROUP (ARCCATGP)`

- Connect users who need to perform catalog operations without automatic recall to
ARCCATGP by using the following RACF command:

  `CONNECT (userid1,. . .,userid _n_ ) GROUP(ARCCATGP) AUTHORITY(USE)`

- Each user who needs to perform catalog operations without automatic recall must log on
to TSO specifying the **GROUP(ARCCATGP)** parameter on the TSO logon panel or the
**GROUP=ARCCATGP** parameter on the JOB statement of a batch job. See Example 17-1.

  _Example 17-1  How to code a jobcard by using ARCCATGP_

      //HSMCAT JOB(accounting information),'ARCCATGP Example',
      // USER=ITSOHSM,GROUP=ARCCATGP,PASSWORD=password
      //STEP1 EXEC PGM=....

### 17.1.6 DFSMShsm using 3592 model E05

The following hints and tips are imported from the book: _DFSMS Software Support for IBM_
_System Storage TS1130 and TS1120 tape drives (3592)_ , SC26-7514:

- In a non-storage management subsystem (SMS) mixed tape hardware environment,
where multiple types of tape hardware are used to emulate 3590 devices, it is
recommended that you define unique esoterics for each type of tape hardware. This action
is necessary to avoid mixing incompatible recording technologies. You can define an
esoteric to DFSMShsm through the **SETSYS USERUNITTABLE** command, for example:
**SETSYS UUT(3592E05:3592E05 3590H:3590H)** . With esoterics defined, you can then direct
output to the set of drives that you want through the **SETSYS** command, for example: **SETSYS**
**BACKUP(TAPE(3592E05))**

- If your installation has an excessive number of spanning data sets, consider specifying a
larger value in the **SETSYS TAPESPANSIZE** command. A larger absolute value is needed to
represent the same amount of unused capacity on a percentage basis when the tape has
a larger total capacity. For example, if you allow 2% of unused tape to avoid tape spanning
for a 3590-H _xx_ device that uses enhanced media, specify a TAPESPANSIZE of 1200 MB.
To allow 2% unused tape for a MEDIA5 tape on a 3592 Model E05 device (no
performance scaling), specify a TAPESPANSIZE of 9999 MB. All size calculations for
scaled tapes are based on the scaled size and not the unscaled size.

- If the speed of data access on MEDIA5 or MEDIA9 tape is more important than the full
use of the capacity, consider the use of performance scaling. Performance scaling uses
20% of the physical capacity on each tape and keeps all data sets closer together and
closer to the initial tape load point. If you use performance scaling with the DFSMShsm
duplex function, be sure that the original tape and the alternate tape both use performance
scaling. Similarly, make sure that tapecopy input tapes and output tapes have the same
performance scaling attributes.


-----

**Note:** Performance scaling is not available on these tape cartridge media: MEDIA6,
MEDIA7, MEDIA8, and MEDIA10. If your installation is using MEDIA5 tapes with
performance scaling, consider the use of MEDIA7 tapes for high performance
functions. The available MEDIA5 tapes can then be used to their full capacity. Consider
performance segmentation as a compromise solution. Performance segmentation
increases the performance of data sets in the first 20% of the tape’s capacity, but also
uses the remaining capacity as a slower access segment. The average performance for
the tape is increased at the expense of losing a percentage of the MEDIA5 or MEDIA9
overall tape capacity. (You cannot determine which data sets reside in which segment.)

DFSMShsm recycle processing of 3592 Model E05 tapes can take significantly longer than
with smaller tapes because the amount of data that is moved at the same
RECYCLEPERCENT can be much larger. In addition to moving more data, the likelihood of a
tape takeaway for recall processing increases with the number of data sets that still remain on
the tape. One option for controlling overall recycle run time is the **LIMIT(** **_nnnn_** **)** parameter of
recycle. Recycle returns no more than the specified number of tapes to scratch during the
current recycle run. Because recycle sorts the tapes based on the amount of valid data still on
each volume, the tapes that are recycled require the least processing time.

Another option to consider is decreasing the **ML2RECYCLEPERCENT** parameter, the
**RECYCLEPERCENT** parameter, or both. Assume, for example, that your installation uses MEDIA7
tape for migration-level 2 (ML2) and MEDIA5 tape for backup. If the EFMT1 format is used
and you want no more than 6 GB of data to be moved when an ML2 tape is recycled, set
ML2RECYCLEPERCENT(10) because the MEDIA7 can hold 60 GB of data in EFMT1. If your
installation uses full capacity for backup tapes and you want no more than 6 GB of data to be
moved when a backup tape is recycled, set RECYCLEPERCENT(2) because a MEDIA5 tape
can hold 300 GB of data in EFMT1.

These examples assume that the ML2 and backup tapes in the installation are filled to
capacity because the calculations are based on the average fullness of marked full tapes on
your system (the reuse capacity). To determine how much data your current recycle threshold
implies, use the reuse capacity that is associated with the tapes. The current recycle
threshold percent multiplied by the reuse capacity gives the maximum amount of data on any
of the tapes when they are recycled. Although lowering the recycle threshold reduces recycle
processing time and decreases the number of times that each tape must be recycled, it might
also increase the overall number of tapes that are needed in your installation. Also, if you
have a mix of ML2 or backup tape capacities in need of recycle processing, you might want to
recycle tapes with the **RANGE** parameter and use the appropriate recycle threshold for the tape
capacities in the range.

In a storage management subsystem (SMS) tape environment, and optionally in a non-SMS
tape environment, the SMS data class construct can be used to select Write Once Read
Many (WORM) tapes for ABACKUP processing. The output data set prefix that is specified in
the aggregate group definition can be used by the automatic class selection (ACS) routines to
select a WORM data class. Set up the ACS routine and the output data set name to uniquely
identify the aggregate backup and recovery support (ABARS) output files that must go to
WORM tape. In a non-SMS tape environment, the default allows tape pooling to determine
whether ABARS data sets go to WORM or read/write media.


-----

Optionally, if the **DEVSUP** parameter **ENFORCE_DC_MEDIA=ALLMEDIATY** or
**ENFORCE_DC_MEDIA=MEDIA5PLUS** is used, the data class must request the appropriate media
type for it to be successfully mounted:

- Consider using the Fast Subsequent Migration function to reduce the need to RECYCLE
these high-capacity tapes.

- For a sysplex environment, consider the use of the common recall queue (CRQ) to
optimize mounts of migration tapes.

- AUDIT MEDIACONTROLS for a FAILEDCREATE situation usually needs to look at only
the last few files on a tape. If it is available for your system level, ensure that Audit APAR
OA04419 is applied.

- The 3592 Model E05 tape drive is used in 3590 emulation mode only, never 3490. The
3592 Model J1A can operate in 3490 emulation mode only when it uses MEDIA5 for
output.

**How to activate performance segmentation or scaling**
The following tasks must be complete to activate performance segmentation or scaling:

- A data class must exist with the _performance segmentation_ or _performance scaling_
attribute set to Y .

- The ACS routine must map a DFSMShsm single file tape data set name to the data class.

- Because data class now determines whether a performance option is used, non-SMS
tape needs ACS routines if you want a 3592 performance option.

IEC205I at close of tape will indicate whether performance segmentation or scaling was used.
Performance segmentation and performance scaling apply to MEDIA5 only.

**Note:** Performance segmentation and performance scaling are mutually exclusive.

**Reorganizing your CDSs**
Reorg with FREESPACE(0 0) and let DFSMShsm split midsection intervals:

- Performance is degraded for about 2 - 3 weeks during this process.
- Do not panic when you see the HURBA/HARBA ratio increase during first several days.

Execute the REORG job only when you are increasing size allocation. Ensure that _all_ hosts
are shut down before you attempt to reorganize any CDSs. Use DISP=OLD on a REORG job
to prevent accidentally bringing up DFSMShsm.

**Error handling in the CRQ environment**
If a z/OS image fails, the DFSMShsm host on that system also fails, but all of the recall
requests that originated on that host remain intact on the CRQ:

- The coupling facility notifies the remaining connected hosts of the failure.

- In-process requests on the failed host remain on the queue and are available for other
hosts to restart them.

If the failing host was processing a request from an ML2 tape, recall requests for data on
that tape cannot be selected.

- The tape is marked as “in-use” by the failing host. The “in-use” indicator can be reset by
restarting the failed host or by using the **LIST HOSTID(** **_hostid_** **) RESET** command.


-----

### 17.1.7 Experiencing a space problem during the recall process

If you are experiencing difficulty finding space to recall a large data set, you have two options.
You can force it to go multivolume:

HSEND RECALL datasetname DFDSSOPTION(VOLCOUNT(ANY))

Or, you can direct it to an empty, non-SMS disk:

HSEND RECALL datasetname FORCENONSMS UNIT(3390) VOLUME(nonSMSvol)

### 17.1.8 Replacing a lost or damaged ML2 tape

Hopefully, you duplex your ML2 tapes? First, identify the alternate volume with the following
**LIST TTOC** command:

HSEND LIST TTOC ( _volser_ ) DSI TERM

Then, from the TTOC listing, check that this tape is not part of a spanned set. If this tape is
part of a spanned set, find the spanned data set by running a **LIST TTOC** command on the
previous volume and recall the spanned data set.

Check that the “link” is broken by repeating the **LIST TTOC (** **_volser_** **)** command for the original
volume. It now shows no previous volume if the link is broken.

You can then replace the broken cartridge with an alternate cartridge, with the **TAPEREPL**
command:

HSEND TAPEREPL ORIGINALVOLUMES( _volser_ ) ALTERNATEUNITNAME( _esoteric_ )

The _esoteric_ is the correct tape unit for your alternate ML2 tapes.

### 17.1.9 ML1 pool full

If the ML1 pool is full, consider adding more volumes to the migration-level 1 (ML1) pool,
empty out older data, or move out the larger files.

The following command will move data that is one day old or older on ML1 to ML2:

`MIGRATE MIGRATIONLEVEL1 DAYS(1)`

You can also use the following JCL to generate **MIGRATE** command cards for any data set that
fits the selection criteria, as shown in Example 17-2.

_Example 17-2  Sample JCL to move data sets from ML1 to ML2_
     
     //MHLRESAR JOB (ACCOUNT),'NAME',CLASS=A,MSGCLASS=X
     //*
     //* LIST DATASETS ON ML1 AND MOVE THEM TO ML2
     //*
     //STEP01 EXEC PGM=IDCAMS
     //SYSPRINT DD SYSOUT=*
     //MCDS   DD DSN=HSM.MCDS,DISP=SHR
     //DCOLLECT DD DSN=&&DCOLDATA,UNIT=SYSALLDA,SPACE=(CYL,(150,100)),
     //       DISP=(NEW,PASS),DCB=(LRECL=700,RECFM=VB,DSORG=PS)
     //SYSIN  DD *
     DCOLLECT -
     OFILE(DCOLLECT) -
     MIGRATEDATA
     
     
     -----
     
     /*
     //DSNINFO EXEC PGM=ICETOOL,REGION=4M
     //TOOLMSG DD SYSOUT=*
     //DFSMSG  DD SYSOUT=*
     //DCOLLECT DD DISP=OLD,DSN=&DCOLDATA
     //TYPED  DD DISP=(,PASS),SPACE=(TRK,(10,5)),UNIT=SYSALLDA,
     //       DSN=&&DCOLTYPD
     //SORTED  DD DISP=(,CATLG),SPACE=(TRK,(2,1)),UNIT=SYSALLDA,
     //       DSN=MHLRESA.ML12ML2,
     //       DCB=(LRECL=80,RECFM=VB,BLKSIZE=0,DSORG=PS)
     //TOOLIN  DD *
     COPY FROM(DCOLLECT) TO(TYPED) USING(CPY1)
     SORT FROM(TYPED) TO(SORTED) USING(SRT1)
     /*
     //CPY1CNTL DD *
     
     -           ML1 DATASET
     
     -               |
     SORT FIELDS=COPY
     INCLUDE COND=(74,1,CH,NE,C'T',AND,
     
     -        ALLOCATE SPACE IS MORE THAN 1MB
     77,4,BI,GT,X'00001000',AND,
     
     -        MIG DATE IS MORE THAN 4 DAYS AGO
     85,4,PD,LT,DATE3P-4)
     RECORD TYPE=V
     OPTION VLSHRT
     /*
     //SRT1CNTL DD *
     
     -  SORT BY SPACE ALLOCATION
     SORT FIELDS=(93,4,BI,D)
     OUTREC FIELDS=(1,4,C' HSEND WAIT MIG DSN(''',29,44,C''') ML2')
     RECORD TYPE=V
     /*
     
In the sample JCL in Example 17-2, we use DCOLLECT on migration data and
select only any data set that is on an ML1 volume, the migration date is older than four days,
and also, its space allocation is larger than 1 MB.

**17.1.10 Identify ML1 volumes**

ML1 volumes are DFSMShsm-managed if those ML1 volumes are defined and exist in the
ARCCMDxx parmlib by using the **ADDVOL** command at the time that DFSMShsm starts up.

If you added more volumes to the ML1 pool dynamically and forgot to define them in
ARCCMDxx, they cannot be used until after the next DFSMShsm startup.

**Important:** The **LIST ML1** command is based on the migration control data set (MCDS) to
extract the information. This command gives you the false sense that those listed ML1
volumes are actually in use, but in fact they might not be DFSMShsm-managed at all.

You must use the **QUERY SPACE(** **_volser_** **)** command to actually discover whether a particular
ML1 volume is DFSMShsm-managed, as shown in Example 17-3.




_Example 17-3  QUERY SPACE command output_

    ARC0400I VOLUME SBXHS4 IS 97% FREE, 0000000042 FREE TRACK(S), 000009798 FREE
    ARC0400I (CONT.) CYLINDER(S), FRAG .014
    ARC0401I LARGEST EXTENTS FOR SBXHS4 ARE CYLINDERS   9507, TRACKS   142605
    ARC0402I VTOC FOR SBXHS4 IS 00090 TRACKS(0004500 DSCBS), 0004483 FREE
    ARC0402I (CONT.) DSCBS(99% OF TOTAL)
***

If the ML1 volume is not DFSMShsm-managed, you get a message that indicates that the
ML1 volume is not DFSMSHSM-managed, as shown in Example 17-4.

_Example 17-4  QUERY SPACE command message_

ARC0407I QUERY SPACE FAILED, VOLUME SBXHS5 NOT CURRENTLY MANAGED BY DFSMSHSM
***

### 17.1.11 RECALL failures on ML1 volumes

The following steps correct RECALL failures on ML1 volumes:

1. List the data set by using TSO 3.4 and issue the **HLIST DSN(/) MCDS TERM** command to
find the migration volume, migration date, and small-data-set packing (SDSP), if used.

2. Look at the migration volume through TSO 3.4 to find the HSM name, by using the
migration date to find I _nnnn_ (where _nnnn_ is the Julian date).

   **Note:** The HSM name will be 'prefix.HMIG.T _nnnnnn.xx.yyyyyy._ I _nnnn_ ' where T _nnnnnn_ is
the migration time backwards.

3. Issue the **HSEND FIXCDS D / DISPLAY ODS('HSM.HMIG.T** **_nnnnnn.xx.yyyyyy_** **.I** **_nnnn_** **')**
command.

4. Issue the **HSEND FIXCDS A / DISPLAY ODS('HSM.HMIG.T** **_nnnnnn_** **.** **_xx.yyyyyy_** **.I** **_nnnn_** **')**
command.

5. Run an **IEHLIST** command to print a dump of each record.

6. Run a DSS dump of HSM.HMIG.T _nnnnnn.xx.yyyyyy._ I _nnnn._

7. After all diagnostics are taken, list the data set by using TSO 3.4 and issue the **HSEND**
**FIXCDS D / DELETE** command. Run this command to delete the migration control record.
Run the **HLIST DSN(/) MCDS TERM** command to check that the entry was deleted.

8. Issue the **DELETE/NOSCRATCH** command to delete the catalog entry.

9. Try to recover data from a backup.

### 17.1.12 Dump stacking and a storage group

If you are planning to use the full volume dump with stacking feature, it is better to group all
volumes with the same volume capacity in the same storage group and assign DUMPCLASS
with the appropriate stack number.


-----

For example, if you set up a pool of 100 volumes to have full volume dump, which consists of
80 model 9 volumes and 20 model 27 volumes, dumpclass is defined with DUMPSTACK=20,
and also MAXDUMPTASKS is 5. DFSMShsm might possibly assign all or most of 20 volumes
of model 27 into one dump task, and divide the other volumes evenly to the remaining four
dump tasks. As a result, it will take longer to finish the dump for all 20 model 27 volumes.
Therefore, auto dump might not complete within its defined window.

Alternatively, if you put all of the 80 model 9 volumes into one storage group and assign
DUMPCLASS=FDVMOD9 with DUMPSTACK=20, and put all of the 20 model 27 volumes into
another storage group and assign DUMPCLASS=FDVMOD27 with DUMPSTACK=5, each of
the dump tasks will either get 20 model 9 volumes or 5 model 27 volumes. Therefore, the
dump window is shorter.

### 17.1.13 Schedule automatic backup in multiple DFSMShsm host environments

Automatic backup is performed in four phases:

- Backing up the CDSs
- Moving backup versions from ML1 to tape
- Backing up migrated data sets
- Backing up DFSMShsm-managed volumes from a storage group with the automatic
backup attribute

Only the PRIMARY HOST performs the first three phases. Therefore, it is best to schedule
auto backup for the primary host earlier than for the rest of the hosts, which allows the CDSs
to be backed up, and the other processors do not have to wait for the CDSs. Another tip is to
run automatic backup before space management so that backup versions that are created
with the **BACKDS** command are moved off the ML1 volumes where they temporarily reside.

### 17.1.14 Schedule primary space management and secondary space
**management**

Secondary space management (SSM) manages the migration and cleanup from ML1
volumes, including moving data sets from ML1 to ML2. Therefore, it is best to schedule SSM
before primary space management (PSM) so that SSM will free up more space on the ML1
volume for the PSM function.

### 17.1.15 Migration and storage group thresholds

Establishing unrealistically high and low thresholds in a storage group leads to excessive
cycles and missed space management windows.

Primary space management attempts to process down to a low threshold.

Interval migration attempts to process starting from the midway between the high and low
thresholds down to the low threshold.

If you define a low threshold of 1%, this threshold will be difficult to achieve and DFSMShsm
will work excessively to gain little benefit.


-----

### 17.1.16 Using INCREMENTALBACKUP(ORIGINAL)

Using INCREMENTALBACKUP(ORIGINAL) depends on the value of SETSYS
INCREMENTALBACKUP(ORIGINAL | CHANGEDONLY), which is defined in ARCCMDxx. If
you specify ORIGINAL, DFSMShsm will back up the data sets with no backup copies
(regardless of its change bit status) or a data set with the change bit set to on . Specifying
CHANGEDONLY means that only data sets with the data set changed indicator set to on are
backed up.

We recommend that you specify ORIGINAL from time to time (monthly or quarterly) to ensure
that all data sets that require backup are backed up.

### 17.1.17 Delete No Recall with IEFBR14**

Production jobs often use IEFBR14 with DISP=(x,DELETE) as first step. If the data set is
migrated, DFSMShsm will recall it in order to delete.

You can avoid this situation by coding a Delete No Recall parameter in
SYS1.PARMLIB(ALLOCxx), for example:

`SYSTEM IEFBR14_DELMIGDS(NORECALL)`

### 17.1.18 Tuning patches that are supported by DFSMShsm**

The **PATCH** command changes storage within the DFSMShsm address space. You can identify
the storage location to change with an absolute address or a qualified address.

For sites with unique requirements of DFSMShsm that are not supported by existing
DFSMShsm commands and parameters, these tuning patches might offer a solution. In
cases where the unique requirements remain the same from day to day, you might choose to
include the patches in the DFSMShsm startup member. These DFSMShsm-supported
patches remain supported from release to release without modification.

Before you apply any patches to your system, consider the following information:

- You must be familiar with the DFSMShsm **PATCH** command.

- To check for any errors when you use the **PATCH** command, you must specify the **VERIFY**
parameter.

- To see the output from a **PATCH** command, specify the **ODS** parameter.

Too many patches are available to be covered in this book. However, if you need to implement
any patches for your particular needs, all of the complete patches are listed and described in
Chapter 16 “Tuning DFSMShsm” of the _DFSMShsm Implementation and Customization_
_Guide,_ SC35-0418.


-----

