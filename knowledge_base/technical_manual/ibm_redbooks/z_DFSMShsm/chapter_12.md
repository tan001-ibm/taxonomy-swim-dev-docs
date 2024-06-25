# 12. Control data set considerations

In this chapter, we describe the care and maintenance of your DFSMShsm control data sets
(CDSs).

## 12.1 DFSMShsm journal and control data sets

Before starting DFSMShsm, several data sets that are central to DFSMShsm operation must
be allocated, and procedures must be created in the appropriate procedure library.
DFSMShsm requires a minimum of one CDS, a maximum of three CDSs, and one journal
data set.

We briefly explain how CDSs work, CDS reorganization, backup and recovery techniques and
considerations, multicluster CDSs, record-level sharing (RLS) for CDSs, and CDS
enhancements.

### 12.1.1 How control data sets work

Most DFSMShsm tasks that involve user data sets, such as backing up, migrating, and dump,
must be tracked so DFSMShsm can recover the data set if the user accidentally deletes it,
recall a migrated data set, or restore a dump in a disaster recovery.

DFSMShsm tracks its activities through the backup control data set (BCDS), migration control
data set (MCDS), offline control data set (OCDS), and journal data set.

Each CDS has its own function in the DFSMShsm structure. Although BCDS, MCDS, or
OCDS are not required by DFSMShsm, at least one of them must be allocated. Not having
one of these CDSs affects functions that are provided by DFSMShsm. Although the journal
data set is not required by DFSMShsm to run, we strongly recommend that you allocate the
journal for recovery.

The OCDS stores all information about offline media that is used by DFSMShsm. It works
with MCDS and BCDS to supply DFSMShsm with all of the information necessary to perform
all tape-related tasks. If you do not allocate the OCDS, or if the OCDS becomes unavailable,
you will still be able to perform all DFSMShsm function, but all requests must use DASD.

The BCDS is responsible for storing backup information for every data set that is managed
and backed up by DFSMShsm on either on DASD or tape. It also records information about
aggregate backup and recovery support (ABARS) data sets, dumps, and fast recovery
backups. Every time a recover is issued for any of these functions, DFSMShsm uses BCDS
information to process the request. If BCDS is not allocated, DFSMShsm is not able to
perform ABARS, backups, dumps, or fast recovery backup-related tasks. You will not be able
to recover the data from any existing backups that are controlled by DFSMShsm if the BCDS
is lost.

The MCDS, or migration control data set, records all information regarding DFSMShsm
migration to DASD and tape. DFSMShsm uses MCDS information to migrate and recall data
sets from DASD volumes and uses MCDS and OCDS information to migrate and recall data
sets from tape. If you decide not to use MCDS, or if it becomes unavailable, DFSMShsm will
not be able to migrate or recall data sets from DASD or tape.

The journal records information about all DFSMShsm activities. Every time DFSMShsm
performs an activity, such as the activities described here, it updates both the CDSs and the
journal data set, so, if any CDSs are lost, you can recover DFSMShsm to the current point by
recovering the last CDS backup, and applying the journal updates. If you lose your journal
data set, you will not be able to perform CDS recovery.


-----

DFSMShsm uses all of the described data sets to store and retrieve the information that it
needs to perform its activities.

All of the CDSs are VSAM key-sequenced data sets (KSDSs), and occasionally require
reorganization to remove invalid data, and to extend space allocation.

The journal data set is a sequential data set, and it is nulled every time a successful CDS
backup runs. You might want automation set up to automatically back up CDSs every time
that a journal reaches a determined percent utilization. For more information about backing
up CDSs, see the 12.3, “CDS backup procedures”.

### 12.1.2 CDS reorganization

The DFSMShsm CDSs, as any other VSAM files, require occasional reorganization.
Detection of a need to reorganize can be based on the information that is received
automatically through the ARC0909I message. This message is issued when the occupancy
threshold that is requested in the **SETSYS MONITOR** command is exceeded. You can obtain the
current occupancy status by using the **QUERY CONTROLDATASETS** command. Status is reported
in the ARC0148I message. See Example 12-1.

_Example 12-1  Output from HSEND Q CDS command_

    ARC0101I QUERY CONTROLDATASETS COMMAND STARTING ON HOST=2
    ARC0947I CDS SERIALIZATION TECHNIQUE IS RLS
    ARC0148I MCDS TOTAL SPACE=72000 K-BYTES, CURRENTLY ABOUT 27% FULL, WARNING
    ARC0148I (CONT.) THRESHOLD=80%, TOTAL FREESPACE=94%, EA=YES, CANDIDATE
    ARC0148I (CONT.) VOLUMES=4
    ARC0948I MCDS INDEX TOTAL SPACE=210 K-BYTES, CURRENTLY ABOUT 26% FULL, WARNING
    ARC0948I (CONT.) THRESHOLD=80%, CANDIDATE VOLUMES=4
    ARC0148I BCDS TOTAL SPACE=72000 K-BYTES, CURRENTLY ABOUT 48% FULL, WARNING
    ARC0148I (CONT.) THRESHOLD=80%, TOTAL FREESPACE=96%, EA=YES, CANDIDATE
    ARC0148I (CONT.) VOLUMES=4
    ARC0948I BCDS INDEX TOTAL SPACE=210 K-BYTES, CURRENTLY ABOUT 46% FULL, WARNING
    ARC0948I (CONT.) THRESHOLD=80%, CANDIDATE VOLUMES=4
    ARC0148I OCDS TOTAL SPACE=72000 K-BYTES, CURRENTLY ABOUT 34% FULL, WARNING
    ARC0148I (CONT.) THRESHOLD=80%, TOTAL FREESPACE=96%, EA=YES, CANDIDATE
    ARC0148I (CONT.) VOLUMES=4
    ARC0948I OCDS INDEX TOTAL SPACE=210 K-BYTES, CURRENTLY ABOUT 33% FULL, WARNING
    ARC0948I (CONT.) THRESHOLD=80%, CANDIDATE VOLUMES=4
    ARC0148I JOURNAL TOTAL SPACE=3875053 K-BYTES, CURRENTLY ABOUT 000% FULL,
    ARC0148I (CONT.) WARNING THRESHOLD=080%, TOTAL FREESPACE=100%, EA=NO, CANDIDATE
    ARC0148I (CONT.) VOLUMES=0
    ARC0101I QUERY CONTROLDATASETS COMMAND COMPLETED ON HOST=2

You do not have to reorganize the CDSs for performance reasons. The occurrence of control
interval (CI) or control area (CA) splits in the CDSs does not affect DFSMShsm performance.

For the MCDS, mid-section records
are generated for each migrated data set or VSAM component. For the BCDS, mid-section
records are generated for each backup copy.


-----


**Note:** Because DFSMShsm has its own logic when it defines keys, it is not possible to
guarantee that free space that is left in CIs and CAs will ever be used.

Consider the following information before you reorganize your CDSs. The following list is a
summary of considerations for reorganizing the CDSs:

- Only reorganize them when you have to increase their size, or when you are moving the
CDSs to another DASD device.

  Because DFSMShsm does not perform many sequential reads for data, its performance is
not directly affected by CI and CA splits, and you can increase DFSMShsm availability
time with fewer reorganizations.

- DFSMShsm must be shut down in all hosts that have access to the CDSs, and no other
job in any host must be using them during this operation.

  Shutting down all DFSMShsm hosts will ensure that no updates are performed on CDSs
or the journal during reorganization. An update during the reorganization process can lead
to a mismatch between CDSs and the loss of user data.

- Allocate the CDS that you are going to reorganize with DISP=OLD.

  Using DISP=OLD in your JCL will request exclusive access to CDSs, so no DFSMShsm or
jobs will be able to allocate these data sets.

- Reorganize the CDSs with FREESPACE(0) and let DFSMShsm split the mid-section
intervals. Performance will be degraded for about two or three weeks during the CI and CA
split process.

  This approach is recommended because allocating FREESPACE might result in wasted
non-used space in CDSs.


-----

When you decide to reorganize DFSMShsm CDSs, follow a few procedures to ensure that
your reorganization is successful, or ensure that you have the necessary information to
proceed with a backout if the reorganization fails.

First, bring down all DFSMShsm hosts that access the CDSs, and leave only one host up. It is
necessary to leave one host up so you can perform CDS backups before the reorganization.

In sequence, you can create a backup of all CDSs and the journal, so if you must back out
your reorganization, you have a current backup to recover from, reducing recovery time.

To create this backup, you can hold all DFSMShsm tasks to ensure that no new tasks come in
during CDS reorganization. After you put all functions on hold, and you confirm that no
DFSMShsm functions are running, you can issue the command to back up CDSs and clean
up the journal. If any tasks are running on any DFSMShsm hosts, wait for them to complete
before you back up the CDSs. Do not forget to also hold the common queue if it is
implemented in your environment. See Example 12-2.

_Example 12-2  Holding all DFSMShsm functions and performing CDS backup_

HSEND HOLD ALL
HSEND HOLD COMMONQUEUE
HSEND BACKVOL CDS

Example 12-3 shows the message that you will receive when your **BACKVOL CDS** command is
complete.

_Example 12-3  Output from HSEND BACKVOL CDS command_

    ARC0740I CDS BACKUP STARTING AT 12:23:50 ON 2012/08/23, SYSTEM SC64, TO DASD
    ARC0740I (CONT.) IN PARALLEL MODE, DATAMOVER=DSS
    ARC0742I BACKUP FOR MCDS STARTING AT 12:23:50 ON 2012/08/23, BACKUP COPY
    ARC0742I (CONT.) TECHNIQUE IS STANDARD
    ARC0742I BACKUP FOR BCDS STARTING AT 12:23:50 ON 2012/08/23, BACKUP COPY
    ARC0742I (CONT.) TECHNIQUE IS STANDARD
    ARC0742I BACKUP FOR OCDS STARTING AT 12:23:50 ON 2012/08/23, BACKUP COPY
    ARC0742I (CONT.) TECHNIQUE IS STANDARD
    ARC0750I BACKUP FOR JRNL STARTING AT 12:23:50, ON 2012/08/23, BACKUP TECHNIQUE
    ARC0750I (CONT.) IS QUIESCED(B)
    ARC0743I JRNL SUCCESSFULLY BACKED UP TO HSM.JRNL.BACKUP.D0000025, ON VOLUME(S)
    ARC0743I (CONT.) SBXHS5, TIME=12:23:50, DATE=2012/08/23
    ARC1007I COMMAND REQUEST 00000612 SENT TO DFSMSHSM
    ARC0743I MCDS SUCCESSFULLY BACKED UP TO HSM.MCDS.BACKUP.D0000025, ON VOLUME(S)
    ARC0743I (CONT.) SBXHS5, TIME=12:23:51, DATE=2012/08/23
    ARC0743I OCDS SUCCESSFULLY BACKED UP TO HSM.OCDS.BACKUP.D0000025, ON VOLUME(S)
    ARC0743I (CONT.) SBXHS5, TIME=12:23:51, DATE=2012/08/23
    ARC0743I BCDS SUCCESSFULLY BACKED UP TO HSM.BCDS.BACKUP.D0000025, ON VOLUME(S)
    ARC0743I (CONT.) SBXHS5, TIME=12:23:51, DATE=2012/08/23
    ARC0748I LAST SUCCESSFUL CDS BACKUP-SET QUALIFIER IS D0000029
    ARC0741I CDS BACKUP ENDING AT 12:23:51 ON 2012/08/23, STATUS=SUCCESSFUL

After backups finish, you can shut down the last DFSMShsm host.

With all DFSMShsm hosts down, you can run your job to examine CDSs, perform export,
allocate new CDSs, and import data. Each step is described in a different example, so you
can skip or replace the content that is provided, as required.


-----

The first job uses IEFBR14 to allocate the data sets to use during the export job. See
Example 12-4.

_Example 12-4  Creating export data sets_

    //MHLRES71 JOB (999,POK),MSGLEVEL=1,NOTIFY=&SYSUID
    //******************************************************************//
    //******************************************************************//
    //**                               **//
    //**  Before submitting this JCL, update the following fields  **//
    //**                               **//
    //** ?UID - The HLQ used during export processing         **//
    //** ?UNIT - The Unit name used for export data sets       **//
    //**                               **//
    //** You may also want to review space allocation, to make sure  **//
    //**       you have enough space to export         **//
    //**                               **//
    //******************************************************************//
    //******************************************************************//
    //ALLOCATE EXEC PGM=IEFBR14
    //EXPMCDS DD DSN=?UID.EXPORT.MCDS,DISP=(,CATLG),
    // UNIT=?UNIT,SPACE=(CYL,(20,2))
    //EXPBCDS DD DSN=?UID.EXPORT.BCDS,DISP=(,CATLG),
    // UNIT=?UNIT,SPACE=(CYL,(20,2))
    //EXPOCDS DD DSN=?UID.EXPORT.OCDS,DISP=(,CATLG),
    // UNIT=?UNIT,SPACE=(CYL,(20,2))

Next, the job will examine all CDSs to ensure that no discrepancies exist between INDEX and
DATA components. See Example 12-5.

_Example 12-5  Examining CDSs_

    //MHLRES71 JOB (999,POK),MSGLEVEL=1,NOTIFY=&SYSUID
    //******************************************************************//
    //******************************************************************//
    //**                               **//
    //**  Before submitting this JCL, update the following fields  **//
    //**                               **//
    //** ?UID.MCDS - Change to your MCDS name             **//
    //** ?UID.BCDS - Change to your BCDS name             **//
    //** ?UID.OCDS - Change to your OCDS name             **//
    //**                               **//
    //******************************************************************//
    //******************************************************************//
    //IDCAMS EXEC PGM=IDCAMS,REGION=512K
    //SYSPRINT DD SYSOUT=*
    //SYSIN DD *
    LISTCAT ENT(?UID.MCDS) ALL
    LISTCAT ENT(?UID.BCDS) ALL
    LISTCAT ENT(?UID.OCDS) ALL
    EXAMINE NAME(?UID.MCDS) INDEXTEST
    EXAMINE NAME(?UID.BCDS) INDEXTEST
    EXAMINE NAME(?UID.OCDS) INDEXTEST


-----

**Note:** If you receive a return code other than zero, you can check the job log for error
messages. For more information about IDCAMS errors messages, see _z/OS MVS System_
_Messages Vol 6 (GOS-IEA),_ SA22-7636.

If all examinations ran acceptably, you can proceed with the export job. The export process
will read your CDS and copy all of the valid data to a sequential data set. See Example 12-6.

_Example 12-6  Exporting CDSs_

    //EDGUXA0L JOB ,RMM,MSGLEVEL=1,MSGCLASS=H,REGION=0M,NOTIFY=&SYSUID
    //******************************************************************//
    //******************************************************************//
    //**                               **//
    //**  Before submitting this JCL, update the following fields  **//
    //**                               **//
    //** ?UID.MCDS - Change to your MCDS name             **//
    //** ?UID.BCDS - Change to your BCDS name             **//
    //** ?UID.OCDS - Change to your OCDS name             **//
    //** ?UID.EXPORT.MCDS - Change to your EXPORT MCDS data set    **//
    //** ?UID.EXPORT.bCDS - Change to your EXPORT bCDS data set    **//
    //** ?UID.EXPORT.oCDS - Change to your EXPORT OCDS data set    **//
    //**                               **//
    //******************************************************************//
    //******************************************************************//
    //IDCAMS EXEC PGM=IDCAMS,REGION=512K
    //MCDS DD DSN=?UID.MCDS,DISP=OLD
    //BCDS DD DSN=?UID.BCDS,DISP=OLD
    //OCDS DD DSN=?UID.OCDS,DISP=OLD
    //SYSPRINT DD SYSOUT=*
    //SYSIN DD *
    EXPORT ?UID.MCDS ODS(?UID.EXPORT.MCDS) TEMPORARY
    EXPORT ?UID.MCDS ODS(?UID.EXPORT.MCDS) TEMPORARY
    EXPORT ?UID.MCDS ODS(?UID.EXPORT.MCDS) TEMPORARY

After the export job runs, you can submit a new job to delete old CDSs and allocate new
CDSs. This example is a sample job, and you might need to update parameters to meet your
requirements. You can execute a **LISTC** against your current CDSs to check their allocation
specifications. Before you submit this job, confirm that all export jobs ran with RC=0 and all
export data sets are cataloged.

_Example 12-7  Deleting old CDSs and allocating new ones_

    //MHLRES71 JOB (999,POK),MSGLEVEL=1,NOTIFY=&SYSUID
    //*****************************************************************//
    //*****************************************************************//
    //**                               **//
    //** In order to submit this job, you have to update several   **//
    //** parameters, including CDSs names, volsers, units and space **//
    //**           among others.              **//
    //**                               **//
    //*****************************************************************//
    //*****************************************************************//
    //IDCAMS EXEC PGM=IDCAMS
    //SYSPRINT DD SYSOUT=*


-----

    //HSMMCDS DD UNIT=3390,VOL=SER=HG6622,DISP=SHR
    //HSMBCDS DD UNIT=3390,VOL=SER=HG6622,DISP=SHR
    //HSMOCDS DD UNIT=3390,VOL=SER=HG6622,DISP=SHR
    //SYSIN  DD *
    DEFINE CLUSTER (NAME(?UID.MCDS) VOLUMES(HG6622) -
    CYLINDERS(10) FILE(HSMMCDS) -
    RECORDSIZE(435 2040) FREESPACE(0 0) -
    INDEXED KEYS(44 0) SHAREOPTIONS(3 3) -
    SPEED BUFFERSPACE(530432) -
    UNIQUE NOWRITECHECK LOG(NONE)) -
    DATA(NAME(?UID.MCDS.DATA) -
    CONTROLINTERVALSIZE(12288)) -
    INDEX(NAME(?UID.MCDS.INDEX) -
    CONTROLINTERVALSIZE(4096))
    DEFINE CLUSTER (NAME(?UID.BCDS) VOLUMES(HG6622) -
    CYLINDERS(10) FILE(HSMBCDS) -
    RECORDSIZE(6544 6544) FREESPACE(0 0) -
    INDEXED KEYS(44 0) SHAREOPTIONS(3 3) -
    SPEED BUFFERSPACE(530432) -
    UNIQUE NOWRITECHECK) -
    DATA(NAME(?UID.BCDS.DATA) -
    CONTROLINTERVALSIZE(12288)) -
    INDEX(NAME(?UID.BCDS.INDEX) -
    CONTROLINTERVALSIZE(4096))
    DEFINE CLUSTER (NAME(?UID.OCDS.OCDS) VOLUMES(HG6622) -
    CYLINDERS(10) FILE(HSMOCDS) -
    RECORDSIZE(1800 2040) FREESPACE(0 0) -
    INDEXED KEYS(44 0) SHAREOPTIONS(3 3) -
    SPEED BUFFERSPACE(530432) -
    UNIQUE NOWRITECHECK) -
    DATA(NAME(?UID.OCDS.DATA) -
    CONTROLINTERVALSIZE(12288)) -
    INDEX(NAME(?UID.OCDS.INDEX) -
    CONTROLINTERVALSIZE(4096))

The last step of your reorganization is importing the data back to the new CDSs. Execute the
IDCAMS program with the **IMPORT** command. See Example 12-8.

_Example 12-8  Importing data back to new CDSs_

    //MHLRES77 JOB ,RMM,MSGLEVEL=1,MSGCLASS=H,REGION=0M,NOTIFY=&SYSUID
    //IDCAMS EXEC PGM=IDCAMS,REGION=512K
    //******************************************************************//
    //******************************************************************//
    //**                               **//
    //** Before submitting this job, please update the following   **//
    //**             parameters              **//
    //**                               **//
    //** ?UID.MCDS - Your MCDS data set                **//
    //** ?UID.BCDS - Your BCDS data set                **//
    //** ?UID.OCDS - Your OCDS data set                **//
    //** ?UID.EXPORT.MCDS - Your EXPORT MCDS data set         **//
    //** ?UID.EXPORT.BCDS - Your EXPORT BCDS data set         **//
    //** ?UID.EXPORT.OCDS - Your EXPORT OCDS data set         **//
    //**                               **//
    //******************************************************************//


-----

     //******************************************************************//
     //MCDS DD DSN=?UID.MCDS,DISP=OLD
     //**                               **//
     //******************************************************************//
     //******************************************************************//
     //MCDS DD DSN=?UID.MCDS,DISP=OLD
     //BCDS DD DSN=?UID.BCDS,DISP=OLD
     //OCDS DD DSN=?UID.OCDS,DISP=OLD
     //SYSPRINT DD SYSOUT=*
     //SYSIN DD *
     IMPORT IDS(?UID.EXPORT.MCDS) ODS(?UID.MCDS) IEMPTY
     IMPORT IDS(?UID.EXPORT.BCDS) ODS(?UID.BCDS) IEMPTY
     IMPORT IDS(?UID.EXPORT.OCDS) ODS(?UID.OCDS) IEMPTY

### 12.1.3 CDS and journal performance suggestions

When you allocate the journal and CDSs, it is important to consider your required availability
and performance levels. Particularly during automatic functions, intense activity occurs on
these data sets, and it is important to keep the I/O response time to a minimum.

You can take several simple actions to help you achieve the CDS performance goals.
Although most of them are not required, we recommend that you read all of the following
actions, and consider implementing them when possible.

First, when allocating the CDSs, think about the type of media and hardware to use. Selecting
a cached DASD for CDSs will help you to increase performance when reading and writing
data on a CDS. Using DASD fast write for the journal gives you an increased speed when
writing data to the journal, which is the likely operation you will perform to this data set.

You can select one cluster for each CDS, or up to four clusters for MCDS and BCDS, and only
one cluster for OCDS. Either way, you cannot define a single cluster to extend to more than
one volume. Using more than one cluster for a CDS is referred to as _multicluster CDS_ .

Do not specify a secondary allocation for a cluster, or you will not be able to use multicluster
for the specified CDS. You might receive more than one monitor message when the CDS
threshold is reached; deadlocks might occur. DFSMShsm will also display ARC0130I during
the start to inform you that secondary allocation was specified for this CDS.

Allocate each CDS and journal on a different volume where possible to minimize the risk of
contention between a CDS and other CDSs or data sets on the volume. If not possible, you
must not allocate CDSs on the same volume as JES or system data sets.

The data component of one CDS must not be on the same volume as the index component of
another CDS unless the index of the first CDS is also on the volume.

For performance reasons, index components of different CDSs must not be on the same
volume. You must not allocate the index of a different CDS with the data component of
another CDS if its index is not on the same volume.

If CDSs and the journal are storage management subsystem (SMS)-managed, they can be
assigned to a storage class with the GUARANTEED SPACE attribute. This attribute will make
it easier for you to manage where the data sets are placed, and avoid allocating two indexes
on the same volume.


-----

Allocate CDSs on DASD devices that offer the Concurrent Copy feature, and use the **SETSYS**
**CDSVERSIONBACKUP** command to back up the CDSs and journal to minimize the outage during
CDS backup.

Ensure that the CDSs, CDS backup copies, journal, and journal backup copies are not
allowed to migrate. Migrating CDSs or the journal can make your DFSMShsm stop working,
and migrating backup data sets will cause CDS backups to fail.

Use different head disk assemblies (HDAs) for CDSs and the journal; consider the use of
volumes that connect to different control units. This way, recovery times will be shorter in a
hardware failure because not all of the CDSs will need to be recovered.

Implement and test CDS backup and recovery procedures before you begin to manage user
or application data. After you test the CDS backup and recovery procedures, document them.

### 12.1.4 CDS and journal backup

The DFSMShsm primary host automatically backs up its CDSs and the journal, by using the
**SETSYS CDSVERSIONBACKUP** parameter as the first phase of automatic backup. According to
your SETSYS environment specifications, this backup can be scheduled to DASD or tape.
However, unlike the backup of user data sets, CDSs must be backed up to only one type of
I/O device. You cannot use a mixture of tape and DASD. If backup to DASD is implemented, a
set of sequential data sets must be pre-allocated to which the backup copies will be written.

We recommend that you use multiple backup versions that are written to pre-allocated DASD
data sets. DFSMShsm will change the suffix of the data set name dynamically and it is similar
to that of a generation data set (GDS). For tape backups, it is not necessary to preallocate
any data sets.

We recommend at least eight, and preferably 14 backups. Certain clients keep two to four
CDS backups on DASD, then they use the ARCCBEXT to trigger a process to copy the
backups to tape. They let their tape management system, such as DFSMSrmm, keep the
tapes for a designated period, such as 14 days, and then return them to scratch. For more
information about how to set up ARCCBEXT exits, see _DFSMS Installation Exits,_ SC26-7396.

Our recommendation is that you use DFSMSdss to back up the CDSs from within
DFSMShsm. The invocation of DFSMSdss checks the structure of the CDSs as it dumps
them and advises you of any errors that might make recovery of the CDSs impossible.

Because the area of CDS and journal backup can be complicated, we describe in the next
sections considerations you must consider, and how to set up your environment for
successfully backing up your CDSs and journal.

**Creating CDS and journal backups**
When DFSMShsm starts, it gets environmental and function information from PARMLIB. You
are not required to use SYS1.PARMLIB. The PARMLIB pointed to by the HSMPARM DD
statement in the startup procedure will use the ARCCMD00 member or an alternate member
that you indicate by the **CMD** keyword. The **SETSYS** , **DEFINE** , and **ADDVOL** commands that define
how DFSMShsm manages your data are in the PARMLIB member (see PARMLIB member
ARCCMD00 in the starter set).


-----

The **SETSYS CDSVERSIONBACKUP** command determines how DFSMShsm backs up your CDSs.
You can specify the following factors with the subparameters of the **CDSVERSIONBACKUP**
command:

- The data mover that backs up the CDSs (DFSMSdss is recommended.)
- The number of backup versions to keep for the CDSs
- The device type on which to store the backup versions of the CDSs
- The names of the backup version data sets

By using SMS storage groups and management classes, you can specify that Concurrent
Copy is used to back up the CDSs, assuming that they are allocated on volumes that connect
to a controller that provides Concurrent Copy.

If your CDSs, journal, or backup data sets are SMS-managed, ensure that they will not back
up or migrate. If they are non-SMS-managed, ensure that they are not under DFSMShsm
control, so they will not migrate or take backups.

If you use DASD backup data sets, you can use the following example job to preallocate the
data sets. See Example 12-9.

_Example 12-9  Allocating CDSs and journal backup data sets_

    //MHLRES71 JOB (999,POK),MSGLEVEL=1,NOTIFY=&SYSUID
    
    //ALLOCBK EXEC PGM=IEFBR14
    
    //********************************************************************//
    //********************************************************************//
    
    //*  THIS SAMPLE JOB ALLOCATES AND CATALOGS THE CONTROL DATA SET  *//
    
    //*  BACKUP VERSION DATA SETS ON DASD VOLUMES.           *//
    
    //*                                 *//
    //*  ENSURE THAT BACKUP VERSION DATA SETS ARE PLACED ON VOLUMES  *//
    
    //*  THAT ARE DIFFERENT FROM THE VOLUMES THAT THE CONTROL DATA   *//
    
    //*  SETS ARE ON.                         *//
    
    //*                                 *//
    //* PARAMETER PARAMETER DEFINITION                 *//
    
    //*                                 *//
    
    //* ?BKUNIT  - UNIT TYPE OF VOLUME TO CONTAIN THE CDS      *//
    
    //*        BACKUP VERSION.                  *//
    
    //* ?BKVOL1  - VOLUME SERIAL OF VOLUME TO CONTAIN THE FIRST CDS *//
    
    //*        BACKUP VERSION.                  *//
    
    //* ?BKVOL2  - VOLUME SERIAL OF VOLUME TO CONTAIN THE SECOND CDS *//
    
    //*        BACKUP VERSION.                  *//
    
    //* ?BKVOL3  - VOLUME SERIAL OF VOLUME TO CONTAIN THE THIRD CDS *//
    
    //*        BACKUP VERSION.                  *//
    
    //* ?BKVOL4  - VOLUME SERIAL OF VOLUME TO CONTAIN THE FOURTH CDS *//
    
    //*        BACKUP VERSION.                  *//
    
    //* ?SCBVOL1  - STORAGE CLASS NAME FOR BACKUP VERSIONS      *//
    
    //* ?MCDFHSM  - MANAGEMENT CLASS NAME OF THE HSM CONSTRUCT    *//
    
    //* ?CDSSIZE  - NUMBER OF CYLINDERS ALLOCATED TO CDS BACKUP    *//
    
    //*        VERSIONS.                     *//
    
    //* ?JNLSIZE  - NUMBER OF CYLINDERS ALLOCATED TO JOURNAL DATA   *//
    
    //*        SET.                       *//
    
    //* ?UID    - AUTHORIZED USER ID (1 TO 7 CHARS) FOR THE HSM-  *//
    
    //*        STARTED PROCEDURE. THIS WILL BE USED AS THE    *//
    
    //*        HIGH-LEVEL QUALIFIER OF HSM DATA SETS.      *//
    
    //*                                 *//
    
    //********************************************************************//
    
    //********************************************************************//
    
    
-----

    //MCDSV1 DD DSN=?UID.MCDS.BACKUP.V0000001,DISP=(,CATLG),
    
    //  VOL=SER=?BKVOL1,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //MCDSV2 DD DSN=?UID.MCDS.BACKUP.V0000002,DISP=(,CATLG),
    
    //  VOL=SER=?BKVOL2,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //MCDSV3 DD DSN=?UID.MCDS.BACKUP.V0000003,DISP=(,CATLG),
    
    //  VOL=SER=?BKVOL3,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //MCDSV4 DD DSN=?UID.MCDS.BACKUP.V0000004,DISP=(,CATLG),
    
    //  VOL=SER=?BKVOL4,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //BCDSV1 DD DSN=?UID.BCDS.BACKUP.V0000001,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL1,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //BCDSV2 DD DSN=?UID.BCDS.BACKUP.V0000002,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL2,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //BCDSV3 DD DSN=?UID.BCDS.BACKUP.V0000003,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL3,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //BCDSV4 DD DSN=?UID.BCDS.BACKUP.V0000004,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL4,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //OCDSV1 DD DSN=?UID.OCDS.BACKUP.V0000001,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL1,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //OCDSV2 DD DSN=?UID.OCDS.BACKUP.V0000002,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL2,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //OCDSV3 DD DSN=?UID.OCDS.BACKUP.V0000003,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL3,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //OCDSV4 DD DSN=?UID.OCDS.BACKUP.V0000004,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL4,SPACE=(CYL,(?CDSSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //JRNLV1 DD DSN=?UID.JRNL.BACKUP.V0000001,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL1,SPACE=(CYL,(?JNLSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //JRNLV2 DD DSN=?UID.JRNL.BACKUP.V0000002,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL2,SPACE=(CYL,(?JNLSIZE,5)),UNIT=?BKUNIT,
    
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
    //JRNLV3 DD DSN=?UID.JRNL.BACKUP.V0000003,DISP=(,CATLG)
    
    //  VOL=SER=?BKVOL3,SPACE=(CYL,(?JNLSIZE,5)),UNIT=?BKUNIT,
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    //JRNLV4 DD DSN=?UID.JRNL.BACKUP.V0000004,DISP=(,CATLG)
    //  VOL=SER=?BKVOL4,SPACE=(CYL,(?JNLSIZE,5)),UNIT=?BKUNIT,
    //  MGMTCLAS=?MCDFHSM,STORCLAS=?SCBVOL1
    
**Note:** Use the job that is shown in Example 12-9 only if you plan to use DASD
for backup. You might also want to change parameters, such as data set names. This
example shows single cluster CDSs. For a multicluster CDS, you need to create the
backup data set for every cluster.


-----

**Defining the CDS and journal backup environment**
If defining a CDS and journal backup environment is new to you, this step-by-step example
will guide you through the process so that you will be able to define an environment that suits
your installation needs:

1. Prevent the CDSs and the journal from being backed up as part of the user data set
backup. The CDSs and the journal are backed up separately as specified by the **SETSYS**
**CDSVERSIONBACKUP** command.

2. Modify the block size for CDS backup version data sets.

   Preallocate DASD backup data set copies with a block size that is equal to one-half the
track size of the DASD device. For example, the half-track capacity of a 3390 device is27998.
 
   If the **BLKSIZE** keyword is specified on the pre-allocated DASD backup data set copies, it
must be in the range of 7892 - 32760, inclusively.

3. If the CDSs and journal are SMS-managed, perform the following actions:

   - Place them on volumes that are defined in a storage group with this attribute value:

     AUTO BACKUP ===> NO

     or

   - Associate them with a management class with the following attribute:

     BACKUP ===> N

     If the CDSs and the journal are non-SMS-managed, issue the **ALTERDS** command to
prevent them from being backed up outside of the **CDSVERSIONBACKUP** .

4. Prevent the CDSs and journal from migrating. Allowing the CDSs and the journal to
migrate is inadvisable because you might not be able to recover if any of the CDSs are
damaged.

   If the CDSs and journal are SMS-managed, perform the following actions:

   - Place them on volumes that are defined in a storage group with this attribute value:

     AUTO MIGRATE ===> NO

     or

   - Associate them with a management class with the following attribute:

     COMMAND OR AUTO MIGRATE ===> NONE

   - If the CDSs and journal are non-SMS-managed, issue the **SETMIG** command to prevent
them from migrating.

5. Determine whether your CDSs are backed up by Concurrent Copy. If you want your CDSs
to be backed up by Concurrent Copy, you must review the following areas:

   - Ensure that they are associated with a management class Backup Copy Technique
attribute of CONCURRENT REQUIRED or CONCURRENT PREFERRED .

   - Ensure that they are on a DASD volume with a concurrent-copy-capable controller.

   - Ensure that you specify DATAMOVER(DSS) in step 6.


-----

6. Determine whether the data mover for the CDSs is DFSMShsm or DFSMSdss. We
recommend DFSMSdss as the data mover because DFSMSdss validates the CDSs
during backup and supports Concurrent Copy.

   - If you specify **SETSYS CDSVERSIONBACKUP(DATAMOVER(DSS))** , DFSMShsm invokes
DFSMSdss to perform a logical dump of the CDSs and uses sequential I/O to back up
the journal. DFSMSdss validates the CDSs while backing them up and uses
Concurrent Copy if it was specified in the management class.

   - If you specify **SETSYS CDSVERSIONBACKUP(DATAMOVER(HSM))** , DFSMShsm exports the
CDSs and backs up the journal with sequential I/O. The CDSs are not validated during
backup.

7. Choose the number of backup versions that you want to keep for the CDSs. The number
of backup versions that DFSMShsm keeps is determined by the number that you specify
on the **BACKUPCOPIES** subparameter of the **SETSYS CDSVERSIONBACKUP** command.

**Note:** Whenever DFSMShsm actively accesses the CDSs in RLS mode, DFSMSdss
must be specified as the data mover for the CDS backup. If data is directed to tape, the
**PARALLEL** parameter must also be specified. If either condition is not met during
automatic CDS version backup, these values override existing values, and message
ARC0793I is issued. If either of these conditions is not met when **BACKVOL CDS** is issued,
the command fails.

8. Choose the device category (DASD or tape) to which you want DFSMShsm to back up
your CDSs and journal. Parallel is faster than serial and required to use Concurrent Copy.

   - If you specify **SETSYS CDSVERSIONBACKUP(BACKUPDEVICECATEGORY(DASD))** , DFSMShsm
always backs up the CDSs in parallel to DASD devices. If you are backing up the CDSs
and the journal to DASD, you must preallocate the backup version data sets. You can
preallocate the DFSMShsm CDS and journal data set by running the starter set job
ALLOCBK1 before you start DFSMShsm.

   - If you specify **SETSYS CDSVERSIONBACKUP(BACKUPDEVICECATEGORY(TAPE))** , DFSMShsm
backs up the CDSs to tape.

     Whether tape CDS backups are in parallel is determined by the data mover that you
specify and the optional PARALLEL|NOPARALLEL option for DFSMShsm CDS
backup.
   - If you specify **SETSYS CDSVERSIONBACKUP(DATAMOVER(DSS))** , CDSs are backed up to
tape in parallel. Concurrent Copy can be used.
   - If you specify **SETSYS CDSVERSIONBACKUP(DATAMOVER(HSM))** , DFSMShsm backs up the
CDSs serially. Concurrent Copy is not available, and the CDSs are not validated during
backup.
   - If you specify **SETSYS CDSVERSIONBACKUP(DATAMOVER(HSM) PARALLEL)** , DFSMShsmbacks up the CDSs to tape in parallel. Concurrent Copy is not available, and the CDSs are not validated during backup.

     If you are backing up the CDSs and the journal to tape, DFSMShsm dynamically allocates scratch tape volumes so that you do not need to preallocate backup version data sets.


9. Determine the names for the backup data sets.

   You specify the names that are assigned to the backup version data sets by using the
**MCDSBACKUPDSN** , **BCDSBACKUPDSN** , **OCDSBACKUPDS** , and **JRNLBACKUPDSN** subparameters of the
**SETSYS CDSVERSIONBACKUP** command. Your backup version data set names can be up to 35
characters (including periods) but they cannot end in a period.

   Example 12-10 is an example of the **SETSYS CDSVERSIONBACKUP** command and its
subparameters, as it appears in PARMLIB member ARCCMDxx.

   _Example 12-10  Sample ARCCMD configuration for CDS backups_

       SETSYS CDSVERSIONBACKUP(DATAMOVER(DSS)  BACKUPCOPIES(4)  BACKUPDEVICECATEGORY(DASD)  MCDSBACKUPDSN(HSM.MCDS.BACKUP)  BCDSBACKUPDSN(HSM.BCDS.BACKUP)  OCDSBACKUPDSN(HSM.OCDS.BACKUP)  JRNLBACKUPDSN(HSM.JRNL.BACKUP))
    
After you complete all of these steps successfully, on your next start, DFSMShsm will use
these pre-allocated data sets to hold the backup copies.

**Backing up the CDS and journal manually**
Because automatic backup typically produces good copies of the CDSs and journal, it is not
usually necessary to back them up manually. However, if you are ever in a position where
automatic backup failed to make successful copies of the CDSs or journal, you can use
DFSMShsm commands to create successful copies.

Remember the following points before you issue the command to back up the CDSs and
journal manually:

- Do not issue the command during intense DFSMShsm activity because the CDSs cannot
change while they are being backed up.

- After you issue the command, the only way to prevent the backup is to stop DFSMShsm.

- The structural integrity of the CDSs is validated only if you specified DATAMOVER(DSS).

Use the DFSMShsm **BACKVOL** command to back up the CDSs and journal. With this
command, you can perform the following tasks:

- Identify the data mover as either DFSMShsm or DFSMSdss.
- Specify the backup device category.
- Specify that you want parallel backup to occur.

To back up the CDSs and journal, you can use the following command:

`HSEND BACKVOL CDS`

DFSMShsm will use the attributes that are specified in its PARMLIB to define the backup
method and media.

If you set DATAMOVER(DSS), a DFSMSdss logical dump is used, so that you can use
Concurrent Copy. This approach might reduce any serialization delays that are introduced by
the exclusive enqueue that is placed on the CDSs while the backup occurs. Additionally, the
CDSs will be validated.

If you code DATAMOVER(HSM) instead, access method services (AMS) are invoked to
export the CDSs. No structural validation is done.


-----

In all cases, the journal is backed up by using sequential I/O.

If you want to back up the CDSs and journal to tape, use the following command:

`HSEND BACKVOL CDS(BACKUPDEVICECATEGORY(TAPE(PARALLEL)))`

When **PARALLEL** is specified, the default data mover is DFSMSdss, so a DFSMSdss logical
dump will be used to back up the CDSs. One tape drive will be allocated for each CDS and
the journal.

If you want to back up the CDSs serially, you must specify **TAPE(NOPARALLEL)** .

### 12.1.5 VSAM record-level sharing

DFSMShsm supports VSAM RLS for accessing the CDSs. RLS enables DFSMShsm to take
advantage of the features of the coupling facility for CDS access.

Accessing CDSs in RLS mode reduces contention when running primary space management
and automatic backup on two or more processors. DFSMShsm benefits from the serialization
and data cache features of VSAM RLS and does not need to perform CDS verification or
buffer invalidation.

**Requirements for CDS RLS serialization**
CDSs that are accessed in RLS mode enqueue certain resources differently than CDSs that
are accessed in non-RLS mode. Before you consider implementing RLS for your CDSs, you
must ensure that all of the following criteria are met:

- Global resource serialization (GRS) or an equivalent function is implemented.

  **Note:** GRS is required only if you are sharing CDSs between multiple systems. If you
have one or more hosts on a single system, it is not required.

- All operating systems running DFSMShsm must be coupling facility-capable, and the
processors must share access to the coupling facility.

- Your CDSs must be SMS-managed.

- All processors in the installation must access the CDSs in RLS mode.

- All DFSMShsm hosts must specify CDSSHR=RLS in the DFSMShsm startup procedure.

- You must not convert the ARCGPA/ARCRJRN reserve to an enqueue for performance
reasons.

- The CDSs storage class must indicate the coupling facility to use.

- The CDSs must not be keyrange KSDSs.

- You must know how to implement recovery for RLS data sets.

- You must specify DFSMSdss as the data mover for **CDSVERSIONBACKUP** .

- If CDS backup is directed to tape, the **PARALLEL** parameter must be used.

**Making your CDSs RLS-eligible**
Before CDSs can be accessed in RLS mode, you must define or alter them to be
RLS-eligible, by using the LOG(NONE) attribute. You must define or alter them to be
RLS-eligible for all your CDSs (MCDSs, BCDSs, and OCDSs).


-----

The following example shows how to use the **IDCAMS ALTER** command to make the CDSs that
we previously defined RLS-eligible:

`ALTER HSM.MCDS LOG(NONE)`

Example 12-11 is an example of how we can use the **DEFINE** command when we initially set
up our DFSMShsm environment. We show only the definition for the MCDS, but the same
definition must be done for the other CDSs.

_Example 12-11  Sample IDCAMS DEFINE command for the DFSMShsm data set_

    DEFINE CLUSTER (NAME(HSM.MCDS) VOLUMES(HG6622) -
    CYLINDERS(100) FILE(HSMMCDS) -
    RECORDSIZE(435 2040) FREESPACE(0 0) -
    INDEXED KEYS(44 0) SHAREOPTIONS(3 3) -
    SPEED BUFFERSPACE(530432) -
    UNIQUE NOWRITECHECK **LOG(NONE)** ) -
    DATA(NAME(HSM.MCDS.DATA) -
    CONTROLINTERVALSIZE(12288)) -
    INDEX(NAME(HSM.MCDS.INDEX) -
    CONTROLINTERVALSIZE(4096))

You must never use the **ALL** or **UNDO** parameters of the **LOG** keyword. If it ever is necessary to
change the CDSs back to non-RLS-eligible, use the following command:

`ALTER HSM.MCDS NULLIFY(LOG)`

**Removing keyrange CDSs**
The easiest way to remove keyrange CDSs is to remove the **KEYRANGE** ((...)) parameter from
the IDCAMS DEFINE DATA statements that you used to define your CDSs as keyrange.
During startup, DFSMShsm dynamically calculates the key boundaries for each cluster. You
can then use the **QUERY CONTROLDATASETS** command to display both the low and high keys that
DFSMShsm calculates for each cluster.

If your CDSs are defined with keyranges, you can perform a CDS reorganization, and remove
the **KEYRANGE** parameter from the **DEFINE** command.

**Determining the CDS serialization technique**
If you need to verify the CDS serialization technique that is used, use the **QUERY**
**CONTROLDATASETS** command:

`HSEND Q CDS`

If you are using RLS, the following messages are the first messages that are returned as a
result of your **QUERY CDS** command:

ARC0101I QUERY CONTROLDATASETS COMMAND STARTING ON HOST=2
ARC0947I CDS SERIALIZATION TECHNIQUE IS RLS

**RLS implementation**
We show the steps that we took to implement VSAM RLS for our CDSs. In addition to the
requirements that we describe in “Requirements for CDS RLS serialization”,
other considerations must be met. We make assumptions about your system knowledge. You
must be familiar with SMS for VSAM RLS, SMS constructs, SMS classes, SMS configuration,
and the coupling facility cache and lock structures. We do not recommend that you undertake
these steps until you consider how RLS implementation can affect your system.


-----

We recommend that any pre-existing VSAM RLS structures are used for accessing the
DFSMShsm CDSs in RLS mode. No benefit exists to assigning the DFSMShsm CDSs to
unique structures.

**Define the SHCDSs**
Shared Control Data Sets (SHCDSs) are linear data sets that contain information to allow
processing if a system failure might affect RLS. They also act as logs for sharing support.

You must consider size for the SHCDS and you must adhere to a naming convention. For
comprehensive information about defining these data sets, see the _OS/390 V2R10.0_
_DFSMSdfp Storage Administration Reference,_ SC26-7331.

We used the JCL in Example 12-12 to allocate the SHCDS.

_Example 12-12  Sample SHCDS allocation job_

    //DEFSHCDS JOB (999,POK),'MHLRES5',CLASS=A,MSGCLASS=T,
    // NOTIFY=MHLRES5,TIME=1440,REGION=4M
    //STEP1  EXEC PGM=IDCAMS
    //SYSPRINT DD SYSOUT=*
    //SYSIN DD *
    DEFINE CLUSTER (NAME(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS1)     LINEAR SHR(3 3) VOL(SHCDS1)     CYLINDERS(15 15) )
    DEFINE CLUSTER (NAME(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS2)     LINEAR SHR(3 3) VOL(SHCDS2)     CYLINDERS(15 15) )
    DEFINE CLUSTER (NAME(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS3)     LINEAR SHR(3 3) VOL(SHCDS3)     CYLINDERS(15 15) )
    
**Coupling facility cache and lock structures**
If you need to allocate cache and lock structures specifically for DFSMShsm, use the
following recommendations:

- Cache structure: The size of the cache structure must be a minimum of 1 MB per
DFSMShsm host in the HSMplex. For example, if 10 hosts are in the HSMplex, the cache
structure must be a minimum of 10 MB.

- Lock structure: Use Table 12-1 to determine the minimum size for the lock structure.

_Table 12-1  Minimum size for the lock structure_

|Number of DFSMShsm hosts|Minimum lock structure size|
|---|---|
|Fewer than 8|1 MB|
|At least 8, but not more than 23|2 MB|
|At least 24, but not more than 32|3 MB|
|More than 32|4 MB|



For example, if 10 DFSMShsm hosts are in the HSMplex, the lock structure must be a
minimum of 2 MB.


-----

Certain changes to your CFRM policy definitions will be necessary. The DFSMShsm policy
needs to be defined. We added the structures for cache and locking to our current CFRM
policy definitions by using the administrative data utility IXCMIAPU, as shown in
Example 12-13.

_Example 12-13  Adding structures for cache and locking to the CFRM policy definitions_

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

**Note:** The code in Example 12-13 does not represent the entire policy data
for the CFRM data set. It represents the CFRM policy that specifies the requirements for
the DFSMShsm RLS structures.

The coupling facility cache structure names that we chose to use are HSMCACHE1 and
HSMCACHE2. The locking structure name is the required name of IGWLOCK00.


-----

**Alter the SMS configuration**
You must update the SMS configuration with the coupling facility cache structure names that
you defined. Use the ISMF panels. Enter 8 from the ISMF Primary Option menu for storage
administrators to display the CDS Application Selection panel (Figure 12-2).

    CDS APPLICATION SELECTION
    Command ===>
    
    To Perform Control Data Set Operations, Specify:
    CDS Name . . 'SYS1.SMS.MHLRES3.SCDS'
    (1 to 44 Character Data Set Name or 'Active')
    
    Select one of the following Options:
    7 1. Display    - Display the Base Configuration
    2. Define    - Define the Base Configuration
    3. Alter     - Alter the Base Configuration
    4. Validate   - Validate the SCDS
    5. Activate   - Activate the CDS
    6. Cache Display - Display CF Cache Structure Names for all CF Cache Sets
    7. Cache Update - Define/Alter/Delete CF Cache Sets
    8. Lock Display - Display CF Lock Structure Names for all CF Lock Sets
    9. Lock Update  - Define/Alter/Delete CF Lock Sets
    If CACHE Display is chosen, Enter CF Cache Set Name . . *
    If LOCK Display is chosen, Enter CF Lock Set Name . . . *
    (1 to 8 character CF cache set name or * for all)
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 12-2  CDS Application Selection panel for cache_


-----

Enter option 7 to define the cache sets that relate to your coupling facility cache structure
names (Figure 12-3).

    CF CACHE SET UPDATE         PAGE 1 OF 1
    Command ===>
    
    SCDS Name : SYS1.SMS.MHLRES3.SCDS
    Define/Alter/Delete CF Cache Sets:    ( 002 Cache Sets Currently Defined )
    
    Cache Set           CF Cache Structure Names
    HSM1   HSMCACHE1
    
    HSM2   HSMCACHE2
    
    More CF Cache Sets to Add? . . N (Y/N)
    Use ENTER to Perform Validation; Use UP/DOWN Command to View other Pages;
    Use HELP Command for Help; Use END Command to Save and Exit.

_Figure 12-3  CF Cache Set Update panel_

The coupling facility cache structure names must be the names that you previously defined

**Storage class changes**
We took the option of altering the storage class where our CDSs were defined. The storage
class is SC54GRT, and we altered it by entering option 4 from the Storage Class Application
Selection panel (Figure 12-4).


-----

    STORAGE CLASS APPLICATION SELECTION
    Command ===>
    
    To perform Storage Class Operations, Specify:
    CDS Name . . . . . . . 'SYS1.SMS.MHLRES3.SCDS'
    (1 to 44 character data set name or 'Active' )
    Storage Class Name . . CDSRLS  (For Storage Class List, fully or
    partially specified or * for all)
    Select one of the following options :
    4 1. List     - Generate a list of Storage Classes
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
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 12-4  Altering a storage class_

We added the coupling facility cache set information (Figure 12-5).

    STORAGE CLASS ALTER
    Command ===>
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Class Name : CDSRLS
    To ALTER Storage Class, Specify:
    Guaranteed Space . . . . . . . . . Y      (Y or N)
    Guaranteed Synchronous Write . . . N      (Y or N)
    Multi-Tiered SG . . . . . . . . . .       (Y, N, or blank)
    Parallel Access Volume Capability  N      (R, P, S, or N)
    CF Cache Set Name . . . . . . . . . HSM1    (up to 8 chars or blank)
    CF Direct Weight . . . . . . . . . 5      (1 to 11 or blank)
    CF Sequential Weight . . . . . . . 3      (1 to 11 or blank)
    CF Lock Set Name . . . . . . . . .       (up to 8 chars or blank)
    Disconnect Sphere at CLOSE . . . . N      (Y or N)

_Figure 12-5  Storage Class Alter panel page 2_

We added the coupling facility cache set name that we associated previously with the
coupling facility cache structure name. The greater the weight value, the higher the
importance that it is cached. We chose our values randomly.


-----

Because the CDSs were already in this storage class, we validated and then activated our
SMS SCDS of SYS1.SMS.SCDS1.

If your data sets are not allocated to a storage class that allows RLS, you must assign them to
one.

**Altering the IGDSMSxx PARMLIB member**
To specify to SMS that the RLS address space, SMSVSAM, starts at IPL, and to include other
information, we added the information in Example 12-14 to our PARMLIB data set, member
IGDSMS54.

_Example 12-14  Sample IGDSMSxx PARMLIB to implement RLS_

    SMS ACDS(SYS1.SMS.ACDS) COMMDS(SYS1.SMS.COMMDS)
    DEADLOCK_DETECTION(15,4)
    SMF_TIME(YES) CF_TIME(1800) RLSINIT(YES)
    RLS_MAX_POOL_SIZE(100)

Each MVS system has its own SMSVSAM address space after IPL if RLSINIT(YES) is coded.

### 12.1.6 RACF FACILITY classes

RACF 2.2 provides two new profiles that you can use to restrict access to certain RLS
functions. The following two new FACILITY class profiles are added:

- STGADMIN.IGWSHCDS.REPAIR to use the **AMS SHCDS** command
- STGADMIN.VSAMRLS.FALLBACK to use the V **SMS,SMSVSAM,FALLBACK** command

If you want to limit the access to these commands, you have to set up these profiles and
authorize users to them.

**Activating the SMSVSAM address space**
To activate the SMSVSAM address space, an IPL of the MVS system is necessary. To verify
that the SMSVSAM address space started on your system, issue the following command:

`D SMS,SMSVSAM,ALL`

The information that is returned informs you whether your system is active. The returned
information also shows the status of the SMS complex.

**Activating the SHCDS**
If your SHCDS is already active, you do not need to activate it. However, if you are
implementing RLS for the first time, issue the commands in Example 12-15, after you modify
the commands to represent the names you chose for your SHCDSs.

_Example 12-15  Activating SHCDSs_

    VARY SMS,SHCDS(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS1),NEW
    VARY SMS,SHCDS(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS2),NEW
    VARY SMS,SHCDS(SYS1.DFPSHCDS.WTSCPLX1.VSHCDS2),NEWSPARE

These commands perform the following functions:

- Activate the new primary SHCDS
- Activate the new secondary SHCDS
- Activate a spare SHCDS


-----

**DFSMShsm procedure changes**
Add the following information to the PROC statement for your DFSMShsm started procedure
member in SYS1.PROCLIB:

`CDSSHR=RLS`

Add the following information to your EXEC statement:

`'CDSSHR=&CDSSHR'`

**RLS implementation checklist**
To review, you must follow these steps to implement RLS access for the DFSMShsm CDSs:

1. Ensure that all sharing systems are at the prerequisite levels of software.

2. If the SHCDSs are not defined, use IDCAMS to define SHCDSs and ensure that
SHAREOPTION(3,3) is specified.

3. Define your coupling facility cache and lock structures.

4. Alter your SMS configuration to include the cache set information.

5. Create or alter a storage class that contains the cache set information.

6. Assign the CDSs to the new or altered storage class.

7. Alter SYS1.PARMLIB member IGDSMSxx with the RLS parameters.

8. Define the new RLS profiles in the RACF FACILITY class and authorize users.

9. Schedule an IPL of the MVS systems and ensure that the SMSVSAM address space
becomes active.

10. Activate the SHCDSs.

## 12.2 Multicluster control data sets

As the DFSMShsm workload increases in your installation, the activity that each of the CDSs
is required to record will also increase. As CDSs grow, they can require more space than the
physical DASD devices allow (for example, the capacity of a 3390-3 is 2.8 GB). If this situation
occurs, the CDSs can be split across multiple volumes to accommodate their larger size. Only
the MCDS and BCDS can be split, and they are limited to up to four clusters.

If, however, your site is not experiencing any problems with the space requirements of your
CDSs, we do not recommend that you implement a multicluster environment. If you are
experiencing space problems, you might want to first try increasing the size of your CDSs to
relieve the space problem. If the space that is required for your CDS requires more than one
volume, you need to split your CDS.

We recommend the use of non-keyrange multicluster CDSs. The following section applies
only to non-keyrange data sets.

For more information about defining CDSs without keyranges, see “Removing keyrange
CDSs”.


-----

### 12.2.1 Multicluster performance considerations

Multicluster CDSs are a related group of VSAM clusters with the following requirements:

- CDSs cannot be empty when you start DFSMShsm. The MCDS must be defined so that
the high key for the first cluster is greater than X'10' \ C'MHCR'.

- All clusters in a multicluster CDS must be cataloged in the same user catalog.

- All clusters must be on DASD of the same category.

- Only the primary allocation can be specified for the index and data; no secondary
allocation can be specified for either the index or data.

- The index and data are limited to one volume; they cannot be on different volumes.

- For a multicluster BCDS, the RECORDSIZE and data CONTROLINTERVALSIZE for each
cluster depend on the maximum number of backup versions that you want to be able to
keep for any data set.

- CDSs that are accessed by RLS cannot be defined as keyrange data sets. If previous
CDSs are defined as multicluster and use keyranges, they must be redefined without
keyranges. DFSMShsm will dynamically calculate the key boundaries for the redefined
CDSs.

We recommend that if possible you attempt to keep each cluster on separate DASD
subsystems to improve performance and to reduce recovery time if it is necessary.

### 12.2.2 Conversion steps

Many steps are required to implement a multicluster CDS. The following steps assume a
familiarity with DFSMShsm and a requirement at your site to adopt multicluster CDSs.

Follow these steps to implement a multicluster environment:

1. Stop DFSMShsm on all z/OS images.

2. Back up the CDS with the **AMS EXPORT** command.

3. Determine whether to split the CDS into two, three, or four clusters.

4. Modify the JCL that is shown in Example 12-16 to suit your installation’s requirements.

_Example 12-16  Sample JCL for non-keyrange multicluster MCDS_

    //HSMCDS  JOB ,MSGLEVEL=(1,1)
    //*******************************************************************/
    //* SAMPLE JCL THAT ALLOCATES NON-KEY-RANGE MULTICLUSTER      */
    //* RLS-ELIGIBLE MIGRATION CONTROL DATA SETS.            */
    //*******************************************************************/
    //*
    //STEP1  EXEC PGM=IDCAMS,REGION=512K
    //SYSPRINT DD SYSOUT=A
    //SYSUDUMP DD SYSOUT=A
    //SYSIN DD *
    DEFINE CLUSTER (NAME(HSM.RLS.MCDS1)      STORAGECLASS(SC54GRT)      CYLINDERS(2) NOIMBED NOREPLICATE      RECORDSIZE(200 2040) FREESPACE(0 0) 
    INDEXED KEYS(44 0) SHAREOPTIONS(3 3)      UNIQUE LOG(NONE))     DATA -

-----

    (NAME(DFHSM.RLS.MCDS1.DATA)       CONTROLINTERVALSIZE(4096))     INDEX      (NAME(DFHSM.RLS.MCDS1.INDEX)       CONTROLINTERVALSIZE(4096))
    
    DEFINE CLUSTER (NAME(DFHSM.RLS.MCDS2)       STORAGECLASS(SC54GRT)       CYLINDERS(2) NOIMBED NOREPLICATE       RECORDSIZE(200 2040) FREESPACE(0 0)       INDEXED KEYS(44 0) SHAREOPTIONS(3 3)       UNIQUE LOG(NONE))     DATA      (NAME(DFHSM.RLS.MCDS2.DATA)      CONTROLINTERVALSIZE(4096))     INDEX      (NAME(DFHSM.RLS.MCDS2.INDEX)      CONTROLINTERVALSIZE(4096))

5. Copy the old CDS to the new multicluster CDS with the **AMS REPRO** command, as shown in
Example 12-17.

   _Example 12-17  Sample JCL for non-keyrange multicluster REPRO command_

       //******************************************************************/
       
       //* COPY THE OLD CONTROL DATA SETS INTO THE NEWLY DEFINED     */
       
       //* NON-KEY-RANGE MULTICLUSTER CONTROL DATA SETS.         */
       
       //* NOTE: THE FROMKEY/TOKEY VALUES ARE ONLY SAMPLES. THE ACTUAL  */
       
       //* PARAMETERS USED FOR THESE KEYWORDS SHOULD BE DERIVED FROM   */
       
       //* ACTUAL CDSS BEING USED.                    */
       
       //******************************************************************/
       
       //*
       
       //STEP2  EXEC PGM=IDCAMS,REGION=512K
       
       //SYSPRINT DD SYSOUT=A
       
       //SYSUDUMP DD SYSOUT=A
       
       //SYSIN DD *
       
       REPRO INDATASET(HSM.MCDS) OUTDATASET(HSM.RLS.MCDS1)    FROMKEY(X'00') TOKEY(MIDDLE.KEY1)
       
       REPRO INDATASET(HSM.MCDS) OUTDATASET(HSM.RLS.MCDS2)    FROMKEY(MIDDLE.KEY2) TOKEY(X'FF')
       
6. Modify your DFSMShsm startup procedure in SYS1.PROCLIB and in other JCL, such as
DCOLLECT and ARCIMPRT, that references the multicluster CDS. A separate DD card
must exist for each cluster of a multicluster CDS. The ddnames need to be MIGCAT,
MIGCAT2, MIGCAT3, and MIGCAT4; or BAKCAT, BAKCAT2, BAKCAT3, and BAKCAT4.

7. If you back up your CDSs to DASD, preallocate new CDS backup data sets. You need
backup versions for each cluster in the CDS.

**Note:** Do not delete the current CDS. Instead, maintain it for a period until you
determine that the new CDS is valid.

8. Make backup copies of the CDSs.

9. Monitor the growth of the multicluster CDS.

After you complete all of these steps successfully, DFSMShsm will use the multicluster CDSs.


-----

## 12.3 CDS backup procedures

You can back up DFSMShsm CDSs in various ways, depending on your hardware and
system level. System performance directly relates to the time that it takes to back up the
CDSs and the journal because all DFSMShsm activity and all JES2/JES3 setup and
initialization activity are suspended during this process. By improving the backup
performance of CDSs and the journal, you can decrease the time that DFSMShsm is
unavailable to process requests. To decrease the required time to back up the CDSs and the
journal data sets, certain backup techniques are available. We describe the use of Concurrent
Copy, SnapShot, and virtual Concurrent Copy as your backup procedures. We also describe
the CDS backup improvements that were introduced in the last z/OS versions.

### 12.3.1 Concurrent Copy

_Concurrent Copy_ is a combined hardware, Licensed Internal Code (LIC), and software
systems management solution that creates data dumps or copies while user processing
continues.

When DFSMShsm uses Concurrent Copy to back up the CDSs, the period during which
system functions are suspended is dramatically reduced. DFSMShsm reserves the data sets
only for the time that it takes to initialize a Concurrent Copy session for each of them and to
back up and nullify the journal. (DFSMShsm always backs up the journal itself; it does not use
DFSMSdss and consequently Concurrent Copy.) After these operations complete,
DFSMShsm releases the CDSs for other processing but continues with the backup because
the data sets are now protected by Concurrent Copy. This action can reduce the period during
which DFSMShsm functions (and in JES3 systems, job initiation) are suspended to only a few
minutes. This capability makes it possible to reduce the time that is required to restart
DFSMShsm after a failure if a backup copy of one of the CDSs is required.

To use Concurrent Copy for CDS backup, you must perform the following tasks:

- Allocate the CDSs on system-managed volumes that are attached to DASD subsystems
that have Concurrent Copy available

- Ensure that the CDSs receive a management class as described next

- Specify that DFSMShsm will use DFSMSdss as the data mover

DFSMShsm uses Concurrent Copy for automatic backup when processing system-managed
data sets only. If your CDSs are not located on system-managed volumes, you will need to
reallocate them.

When you reallocate the DFSMShsm CDSs, specify (or ensure that your ACS routines select)
a storage class with GUARANTEED SPACE=YES and a management class that contains the
following specifications:

- Expiration limits (EXPIRE AFTER DAYS NON-USAGE and EXPIRE AFTER DATE/DAYS)
and RETENTION PERIOD are set to NOLIMIT .

- COMMAND or AUTO MIGRATE is set to NONE .


-----

- ADMIN or USER COMMAND BACKUP is set to NONE .

- BACKUP COPY TECHNIQUE is set to CONCURRENT PREFERRED . We recommend that you
specify _PREFERRED_ rather than REQUIRED because PREFERRED ensures that
DFSMShsm backs up the CDSs even if Concurrent Copy is not available. If you specify
_CONCURRENT REQUIRED_ and Concurrent Copy is not available (for example, because
of a cache failure), DFSMShsm does not back up the CDSs.

If you use tape as your backup device for the CDSs, also specify the PARALLEL option to
request DFSMShsm to back up the CDSs in parallel. If you do not specify _PARALLEL_ ,
DFSMShsm backs up the CDSs to tape one-by-one, reducing the benefit of Concurrent
Copy. DFSMShsm always backs up the CDSs to DASD in parallel. If contention for tape
drives is a problem in your installation, consider changing your CDS backup device to
DASD when you implement Concurrent Copy.

Concurrent Copy makes it possible to take backups more frequently, reducing the required
time to bring a backup copy “up-to-date” from the DFSMShsm journal. You can take a
backup of the CDSs at any time by using the DFSMShsm **BACKVOL CDS** command. In
addition to using this command to take more frequent CDS backups, you might also need
to use this command if a system problem causes CDS backup to fail after the Concurrent
Copy sessions are initialized.

### 12.3.2 Virtual Concurrent Copy

_Virtual Concurrent Copy_ provides a copy operation that is similar to Concurrent Copy by
using a combination of RAMAC Virtual Array (RVA), SnapShot, and DFSMSdss. If you are
using Concurrent Copy to back up the CDSs, and the device they reside on is an RVA, you do
not need to take any additional action. Virtual Concurrent Copy will be automatically invoked.

You invoke virtual Concurrent Copy by specifying the **CONCURRENT** keyword on a DFSMSdss
**COPY** or **DUMP** statement. When the **CONCURRENT** keyword is specified on a DFSMSdss **COPY**
command and all of the requirements for DFSMSdss SnapShot are met, DFSMSdss tries to
perform a DFSMSdss SnapShot. In this case, the logical completion and physical completion
of the copy occur at the same time.

The benefit of virtual Concurrent Copy is that it can be performed in the following
circumstances, when a DFSMSdss SnapShot cannot:

- The data needs to be manipulated ( **DUMP** command used, reblocked, or track packed to
unlike) and the source resides on an RVA.

- The source and target reside on different RVAs, and DFSMSdss is the data mover.

- The source resides on an RVA, the target is a non-RVA device, and DFSMSdss is the data
mover.

- A multivolume source data set spans multiple RVAs, and DFSMSdss is the data mover.

- A multivolume source data set resides on a combination of RVAs and devices that connect
to an IBM 3990-6 control unit, and DFSMSdss is the data mover.

To perform a virtual Concurrent Copy, space must be made available in the same RVA
subsystem as the source data set. During a virtual Concurrent Copy, data is “snapped” from
its source location to an intermediate location. This intermediate location is known as the
_working space data set_ (WSDS). Data is gradually copied from the WSDS to the target
location.

So for virtual Concurrent Copy to succeed, even though the Concurrent Copy criteria are met,
you must predefine at least one WSDS, but it is more likely that you will need to allocate
multiple WSDSs.


-----

The allocation requirements, catalog search, system data mover (SDM), recommendations,
performance, and restrictions for WSDSs are comprehensively covered in _Implementing_
_DFSMSdss SnapShot and Virtual Concurrent Copy_ , SG24-5268. We recommend that you
obtain a copy of this publication before you implement virtual Concurrent Copy in your
installation.

## 12.4 CDS extended address volume

As your systems and DFSMShsm-managed data grow, more CDS records will be created to
store information about migration, backup, dump, and other DFSMShsm functions that relate
to data sets. Depending on the amount of data that is managed by DFSMShsm, the
4 GB limit for VSAM data sets might not be enough to store all CDS records.

To avoid this issue, with DFSMShsm, you can use extended addressability VSAM data sets
and extended address volumes (EAVs) to store your CDSs.

With the extended addressability VSAM data set, you can allocate VSAM data sets beyond
the 4 GB limit. To create CDSs larger than 4 GB, you must define them as SMS-managed
data sets, and assign a data class with “extended addressability” set to Y .

The EAVs are volumes that are defined to z/OS with more than 65,520 cylinders, and all of
the data sets that are created beyond this line are cylinder-managed data sets. No special
requirements exist for using EAVs, but we recommend that you use SMS-managed CDSs, so
you can take the advantage of the large space that is available on the volume.

z/OS V1R13 supports volumes as large as 1 terabyte, which means that MCDS and BCDS
are limited to 4 TB each because each data set can have up to four clusters. The OCDS is
limited to 1 TB because it must be a single-cluster CDS.

## 12.5 CDS new backup technique

Starting in z/OS V1R13, DFSMShsm introduces a new backup technique, called _nonintrusive_
_journal backup_ . With this new feature, you can reduce the time that DFSMShsm activities are
unavailable during CDS backup.

Before V1R13, the CDSs and journal were quiesced during CDS backups. Quiescing the
CDSs and journal was necessary because the journal must be backed up in real time (that is,
no Concurrent Copy option) and therefore holds up DFSMShsm activity.

As the journal data set is written sequentially, the only changes to the journal are appends to
the end of the data set. Therefore, after the data is written to the journal, it will not be changed
or updated. Nonintrusive journal backup allows DFSMShsm to back up the data that is
already recorded in journal data set, without quiescing, and consequently prevents new
journal records from being recorded.

By using this new backup technique, DFSMShsm will no longer quiesce journal data sets
during the total journal backup. Instead, DFSMShsm will mark the last record that is written to
the journal data set before the backup starts. Then, DFSMShsm quiesces the journal while it
backs up the control record, and then releases it from quiesce, while asynchronous backup is
performed against all records before the logical end mark.


-----

After DFSMShsm finishes backing up all records before the logical end mark, it will quiesce
the journal and CDSs, and back up the remaining records and CDS data. Journal and CDS
updates will be prevented during this time.

To make it possible, the journal data set is now backed up before the CDSs backup. This
change is necessary so all of the journal and CDS updates that are performed during the
asynchronous copy of journal records are also backed up during CDS backups and journal
synchronous copy.

By allowing DFSMShsm activity while backing up the existing data in the journal data set and
by quiescing the journal only when backing up the latest recorded data, you can significantly
reduce the time that DFSMShsm activities are unavailable.

The elapsed time for nonintrusive journal backup is approximately the same as for the
quiesced backup. The advantage of using nonintrusive journal backup is the reduced
unavailability time for DFSMShsm processing.

To use nonintrusive journal backup, your system must be on z/OS V1R13. You must meet the
following requirements:

- SETSYS JOURNAL(RECOVERY) is set on your DFSMShsm PARMLIB

- SETSYS CDSVERSIONBACKUP(DATAMOVER(DSS)) is set on your DFSMShsm
PARMLIB.

- Your CDSs must be SMS-managed.

- The management class of the CDSs must specify Concurrent Copy technique.

- When running on multiple logical partitions (LPARs) with z/OS images before V1R13, you
must install toleration APAR OA32293.


-----

If any of these conditions are not met, DFSMShsm will automatically quiesce the journal and
the CDSs for backup.

**ARCCAT release**
ARCGPA/ARCCAT is obtained (shared) whenever a function wants to update the DFSMShsm
CDS records. In certain cases, the resource is held for a long time. Typically, when the CDS
backups appear to be stopped, the process is simply waiting to exclusively obtain
ARCGPA/ARCCAT while another function owns the resource as shared.

CDS backups can be delayed for an extended time if the **BACKVOL CDS** command was issued
during a migration or backup of a large data set, tape audits, or other DFSMShsm activities
that can run for several minutes or hours.

Starting on z/OS V1R13, DFSMShsm will handle ARCGPA/ARCCAT resources so that when
a CDS backup process starts in a host, all DFSMShsms in the HSMplex finish their currently
running CDS updates, and immediately release ARCGPA/ARCCAT resources.

This approach enables CDS backup to start immediately. After the CDS backup is complete,
all of the processes that were running before the backup start to reacquire ARCGPA/ARCCAT
resources and continue processing.


-----

When you are using the ARCCAT release in z/OS V1R13, the patches that were used on
older DFSMShsm releases to periodically release ARCCAT will not be effective.

To use the ARCCAT release, you must be running z/OS V1R13 or higher. If any of your hosts
are under this version, apply the coexistence APAR OA32293. The ARCCAT release is not
fully effective until all hosts are running z/OS V1R13 or higher because only those hosts can
perform the release.

The ARCCAT release depends on cross-system coupling facility (XCF) services to
communicate the start of CDS backup to other HSM hosts. In the absence of XCF services,
the ARCCAT release will be performed only on the host that performs the CDS backup.


-----

