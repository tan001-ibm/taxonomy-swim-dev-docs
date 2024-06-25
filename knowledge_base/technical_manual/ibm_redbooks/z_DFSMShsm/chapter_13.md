# 13. Monitoring and reporting 

You can monitor DFSMShsm in real time or historically in a number of ways. DFSMShsm
commands provide a comprehensive view of what is happening, what happened, and what
needs to happen.
In this chapter, we explain how to use the DFSMShsm commands to collect information about
DFSMShsm processing and monitor DFSMShsm activity.
Each of these commands gathers information from different sources within DFSMShsm, such
as the migration control data set (MCDS), backup control data set (BCDS), offline control data
set (OCDS), functional statistics records (FSRs), or the DFSMShsm address space. 

## 13.1 List command

The DFSMShsm **LIST** command obtains its information from the control data sets (CDSs):
MCDS, BCDS, and OCDS. You can list the following categories of information:

- Aggregate backup and recovery support (ABARS) activity
- Backup volumes
- Data sets
- Dump classes
- Dump volumes
- Host information
- Migration information
- Primary volume information
- Tape volume information
- User authorization

Many options are available for each of the **LIST** commands; we do not show every variation.
The commands that we show cover each area of DFSMShsm for which the **LIST** command
gathers information. Several of the commands use only the MCDS or BCDS to gather
information. Other commands use both the MCDS and BCDS. You might want to investigate
the commands further to determine whether additional parameters suit your installation
requirements.

To use the commands that are listed in this chapter, you must have the necessary RACF or
DFSMShsm AUTH. If you do not have this authority, or do not know how to get it.

Certain **LIST** commands might result in a large amount of data to be displayed, interfering
with your terminal displays, or increasing DFSMShsm SYSOUT. You can select the output
that you want for the specified command by issuing one of the commands that are shown in
Example 13-1.

_Example 13-1  Select output to terminal, a data set, or SYSOUT class_

    HSEND _request_ TERM
    HSEND _request_ ODS(' _dsname_ ')
    HSEND _request_ SYSOUT( _class_ )

The **LIST** command can create a large amount of output, so be sure to direct any output that
might be large, for example, a list of all migration volumes, to an output data set.

When you direct the output to DASD, you must specify the fully qualified data set name. If the
output data set does not exist, DFSMShsm dynamically allocates it. If the specified output
data set exists, DFSMShsm appends the output to the end of the data set.

**Aggregate backup and recovery support**
If you are running aggregate backup and recovery support (ABARS) in your environment, in
certain situations, it will be necessary to list and report ABARS information for accounting or
disaster recovery. Use the following command to list information about an ABARS aggregate
group and ABACKUP and ARECOVER information:

HSEND LIST AGGREGATE(PAY1) VERSION(0001) ODS(MHLRES7.ABARS.LIST)

Example 13-2 is a sample output from the previous command.


-----

    _Example 13-2  Sample LIST AGGREGATE output_
    
    -- DFSMSHSM CONTROL DATASET AGGREGATE BACKUP AND RECOVERY VERSION LISTING -----
    AT 19:39:10 ON 2012/08/31 FOR SYSTEM=SC64

    ABR RECORD KEY = MHLRESA.2012240000101
    AGGREGATE GROUP NAME = MHLRESA              DFSMSHSM VERSION = VD.1
    AGGREGATE ACCOUNT CODE =
    ABACKUP DATE = 2012/08/27   ABACKUP TIME = 12:16:39
    VERSION = 0001        COPY NUMBER = 01
    UNIT NAME = 3590-1      NUMBER OF COPIES = 01
    SOURCE SYSTEM = SC64
    MANAGEMENT CLASS = MC54NMIG......................
    REMOTE DESTINATIONS =     DESTINATION ID        TRANS. OK
    NONE
    
    CONTROL FILE NAME = HSM.DMP.C.C01V0001
    DFSMSDSS DATA FILE NAME = HSM.DMP.D.C01V0001
    INTERNAL I/O DATA FILE NAME = NONE
    ACTIVITY LOG/INSTRUCTION FILE NAME = HSM.DMP.I.C01V0001
    
    ABACKUP ACTIVITY LOG DATA SET NAME = HSMACT.H2.ABACKUP.MHLRESA.D12240.T121639
    INSTRUCTION DATA SET NAME = MHLRESA.ABARS.INSTRUCT
    FILTER OUTPUT DATA SET NAME = MHLRESA.ABARS.OUTPUT
    
    CONTROL FILE VOLSERS =
    VOLS= THS014
    LIBS= LIB2
    DFSMSDSS DATA FILE VOLSERS =
    VOLS= THS006
    LIBS= LIB2
    
**Backup and migration**
You can also use the **LIST** command to gather information about backups, migration, and
recalls, to help you to identify the high-level qualifiers (HLQs) and applications that perform
more backups, or request more recalls. You can use this information to help you to manage
retention periods on primary volumes, and to identify data sets that are being backed up
incorrectly.

To list the backup volumes that are assigned to DFSMShsm, you can issue the following
command:

`HSEND LIST BACKUPVOLUME ODS(MHLRES5.LBV)`

To retrieve information about a specific application or HLQ, you can add the **LEVEL** parameter
to restrict the listing to a determined mask of data sets:

HSEND LIST LEVEL(MHLRES7) BCDS ODS(MHLRES7.DSN3)

**Note:** This command will display all BCDS information about HLQ MHLRES7 and send
output to MHLRES7.DSN3. This command can return many entries, and displaying the
output directly to your terminal is not recommended.

Example 13-3 shows the output from the previous command.


-----

_Example 13-3  Sample LIST LEVEL BCDS output_

     -  DFSMSHSM CONTROL DATASET - BACKUP DATASET-- LISTING ----- AT 13:49:27 ON
     12/09/04 FOR SYSTEM=SC64
     
     DSNAME = MHLRES7.ACSTEST.LISTDS            BACKUP FREQ = ***, MAX
     ACTIVE BACKUP VERSIONS = ***
     
     BACKUP VERSION DATA SET NAME         BACKUP FROM  BACKUP  BACKUP  SYS
     GEN VER UNS/ RET BACKUP NEW
     VOLUME VOLUME DATE   TIME   CAT
     NMBR NMBR RET DAYS PROF  NAME
     
     HSM.BACK.T011418.MHLRES7.ACSTEST.A0313    VT0032 MLD10C 10/11/09 18:14:01 YES
     000 001  NO ***** NO   N**
     
     TOTAL BACKUP VERSIONS = 0000000001
     
The **LIST** command also supports the use of the **SELECT** parameter. With this parameter, you
can select only the information that you need from DFSMShsm CDSs. The **SELECT** parameter
can include specific volume names, age, tape status, and other information.

In the following command, we show how to use the **LIST LEVEL MCDS** command with the
**SELECT** parameter with **VOLUME** to list migration information about a specific volume. We show
you how to use **AGE** to determine the last referred to date range that you want to list. This
command lists MCDS information for MHLRES5.PAY1 data sets:

`HSEND LIST LEVEL(MHLRES5.PAY1) MCDS SELECT(VOLUME(HG661A) AGE(10 50))`

This command will list all migration information about data sets that were originally allocated
on HG661A, and not referenced in at least 10 days and no more than 50 days. Example 13-4
shows the example output.

_Example 13-4  Sample LIST MCDS LEVEL output_

    -  DFSMSHSM CONTROL DATASET - MIGRATED DATASET-- LISTING ----- AT 14:47:52 ON
    12/09/04 FOR SYSTEM=SC64
    
    DATASET NAME                 MIGRATED LAST REF MIGRATED TRKS
    QTY  TIMES DS SDSP  QTY  LAST MIG
    ON VOLUME  DATE   DATE  ALLOC 2K
    BLKS MIG ORG DS 16K BLKS VOLUME
    
    MHLRES5.PAY1.SALARY1              HSM14D 12/08/19 12/08/20 000001
    0000001 001  PS NO  ******  ******
    
    -  DFSMSHSM CONTROL DATASET - SUMMARY-- LISTING ----- AT 14:47:52 ON 12/08/30 FOR
    SYSTEM=SC64
    
    MIGRATED    TRACKS   K-BYTES
    DATA SETS   MIGRATED   MIGRATED
    0000000001   000000001   00000001
    
    ----- END OF - MIGRATED DATASET - LISTING -----


-----

To list migration information, including data sets that were recalled to a primary volume, you
can use the following command:

`HSEND LIST DATASETNAME MCDS INCLUDEPRIMARY ODS(MHLRES7.DSN4)`

Example 13-5 shows the output that is generated by the previous command.

_Example 13-5  Sample output from LIST DATASETNAME MCDS command_

    -  DFSMSHSM CONTROL DATASET - MIGRATED DATASET-- LISTING ----- AT 14:58:42 ON 12/
    
    DATASET NAME                 MIGRATED LAST REF MIGRATED TRKS
    QTY TIMES DS SDSP  QTY  LAST MIG
    ON VOLUME  DATE   DATE  ALLOC 2K
    BLKS MIG ORG DS 16K BLKS VOLUME
    
    CVERNON.EAVTEST.EFSAM0             ONLINE 09/08/14 09/08/14 0000015
    0000000 00002 PS NO  ******  ******
    DB9CD.DSNDBC.DB2R7.DB2R7.I0001.A001      ONLINE 09/10/07 09/10/07 0000005
    0000000 00003 VS NO  ******  ******
    HAIMO.HSMMIG.TEST1               ONLINE 10/05/20 10/05/20 0000001
    0000000 00001 PO NO  ******  ******
    HERING.ALLQUOTA.UNLOAD             ONLINE 08/03/18 08/03/19 0000001
    0000000 00002 PS NO  ******  ******
    HERING.CP.TESTFILE               ONLINE 08/03/19 08/03/19 0000001
    0000000 00001 PS NO  ******  ******

All recalled data sets show MIGRATED ON VOLUME as ONLINE. This list includes how
many times the data set was migrated, migrated date, migrated volume, and other attributes.

The **LIST** command can also help you to identify the data sets that are stored in a specific
tape, so you can take the necessary actions if a tape is damaged, or lost. To list data set
information from a backup tape, you can use the following command, which lists the tape
table of contents (TTOC):

HSEND LIST TTOC( _volser_ ) DATASETINFORMATION ODS('MHLRES7.DSN8')

This command will show all of the data sets that are stored on the specific tape. The data sets
that you see in Example 13-6 are backups that are taken by DFSMShsm from production data
sets.

_Example 13-6  Output from LIST TTOC command_

    -  DFSMSHSM CONTROL DATASET - TAPE VOLUME TTOC - LISTING - AT 17:10:23 ON 12/09/0
    
    VOLSER  UNIT  VOL   REUSE   VALID  PCT  VOL  RACF PREV  SUCC
    NAME  TYPE  CAPACITY  BLKS  VALID STATUS    VOL   VOL
    THS000  3590-1 D(01) 0000028900 0000268759 100  PART  NO  *NONE* *NONE*
    
    DATA SET NAME             NUM BLOCKS RELATIVE FBID VS
    HSM.BACK.T342400.MHLRES7.NONSMSX.A1001     0000002631   0000008   NO
    HSM.BACK.T552900.MHLRES7.NONSMSZ.A2234     0000002631   0000010   NO
    HSM.BACK.T273100.MHLRES7.NONSMSA.A2235     0000005261   0000011   NO
    HSM.BACK.T184600.MHLRES7.NONSMSA.A2234     0000002631   0000012   NO
    HSM.BACK.T585600.MHLRES7.NONSMSB.A2234     0000002631   0000013   NO
    HSM.BACK.T025700.MHLRES7.NONSMSV.A2234     0000002631   0000014   NO


-----

To identify which data sets the backup or migration version refers to, you can use the following
command:

`HSEND FIXCDS C _hsmdataset_ DISPLAY`

For migration data sets, use D instead of C. Then, look at offset +0000 to check the data set
name. See Example 13-7.

_Example 13-7  Output from FIXCDS C command_

    MCH=  017C2400 CA0E367A BF26466B CA0E367A BF26466B
    -  @               *
    + **0000** D4C8D3D9 C5E2F74B D5D6D5E2 D4E2E74B C2D2D740 40404040 40404040 40404040
    -  **MHLRES7.NONSMSX.BKP       ***
    +0020 40404040 40404040 40404040 E3C8E2F0 F0F00000 78048083 00243404 0111001F
    -       THS000       *
    +0040 40006D2F 00940000 000003E8 02916609 00005233 00012220 0000D4D3 F9F4F0C4
    -       Y       ML940D*
    +0060 00000000 3030200F 00000000 0111001F 00000000 00000000 00000A47 40404040
    
**Note:** For migration data sets, the offset is +0004.

In other situations, you might need to know the data sets that were allocated in a specific
primary volume at the time DFSMShsm started the last incremental backup on that volume.
With the following command, you can retrieve the data set names on that volume, data set
organization, creation date, referenced date, and information about whether the data set was
changed. The following command lists the data sets on HG6600 when DFSMShsm
performed the last incremental backup:

`HSEND LIST PVOL(HG6600) BCDS BACKUPCONTENTS ODS(MHLRES5.DSN6)`

The output from this command is similar to the output that is shown in Example 13-8.

_Example 13-8  Output from the LIST PVOL command_

    -  DFSMSHSM CONTROL DATASET -PRIMARY VOLUME-BCDS-- BCONTENTS  --- AT 16:42:54 ON
    98/07/30 FOR SYSTEM=SC54
    CONTENTS OF BACKUP VTOC COPY # 00 FOR PRIMARY VOLUME HSM14A
    DATASET NAME                 ORG MULTI CREATED  REFERENCED
    EXP DATE RACF PSWD  CHANGED
    HSM.BCDS                   VS  ***  98/07/23 98/07/29
    00/00/00 ***  ***  YES
    HSM.BCDS.DATA                 VS  ***  98/07/23 98/07/29
    00/00/00 ***  ***  YES
    HSM.BCDS.INDEX                VS  ***  98/07/23 00/00/00
    00/00/00 ***  ***  NO
    HSM.MCDS                   VS  ***  98/07/23 98/07/29
    00/00/00 ***  ***  YES
    HSM.MCDS.DATA                 VS  ***  98/07/23 98/07/29
    00/00/00 ***  ***  YES
    HSM.MCDS.INDEX                VS  ***  98/07/23 00/00/00
    00/00/00 ***  ***  NO
    HSM.OCDS                   VS  ***  98/07/23 98/07/29
    00/00/00 ***  ***  YES
    HSM.OCDS.DATA                 VS  ***  98/07/23 98/07/29
    00/00/00 ***  ***  YES
    
    
    -----
    
    HSM.OCDS.INDEX                VS  ***  98/07/23 00/00/00
    00/00/00 ***  ***  NO
    HSMACT.H1.BAKLOG.D98208.T182753        PS  NO  98/07/27 98/07/27
    00/00/00 NO  NO   YES
    HSMACT.H1.BAKLOG.D98209.T105715        PS  NO  98/07/28 98/07/28
    00/00/00 NO  NO   YES
    HSMACT.H1.BAKLOG.D98210.T154052        PS  NO  98/07/29 98/07/29
    00/00/00 NO  NO   YES
    HSMACT.H1.BAKLOG.D98210.T165212        PS  NO  98/07/29 98/07/29
    00/00/00 NO  NO   YES
    HSMACT.H1.CMDLOG.D98204.T183225        PS  NO  98/07/23 98/07/23
    00/00/00 NO  NO   YES
    HSMACT.H1.CMDLOG.D98208.T182753        PS  NO  98/07/27 98/07/27
    00/00/00 NO  NO   YES
    HSMACT.H1.CMDLOG.D98208.T200850        PS  NO  98/07/27 98/07/27  0

Example 13-8 shows the data set names, DSORG, multivolume information,
creation date, referenced date, and changed attributes among others.

**Dumps**
The **LIST** command has multiple options to list dump classes and backups. We show several
common listings that are used on a daily basis.

The first step when you work with dumps is to generate a list of all dump classes that are
defined to DFSMShsm, and its configuration. A simple way to list all of the dump classes that
are defined to DFSMShsm is by issuing a **LIST DUMPCLASS** command. If you already know the
dump class name, you can put the dump class name in parentheses, and DFSMShsm will
display information about the specific dump class only. The following command lists all dump
classes that are defined to DFSMShsm:

HSEND LIST DUMPCLASS ODS(MHLRES7.DSN1)

This command will produce an output similar to the output that is shown in Example 13-9.

_Example 13-9  Sample output from LIST DUMPCLASS command_

    --- DFSMSHSM CONTROL DATASET -DUMP CLASS-BCDS--- LISTING --- AT 18:22:36 ON 12/0
    
    DUMP   UNIT   AUTO  DATASET RESET CLASS  CP FRR           TAPE
    VTOC
    CLASS   TYPE   REUSE RESTORE CHANGE DISABLE REQ AVA DAY FREQ  RETPD
    EXPDT COPIES STACK DISPOSITION
    CRYPCOPY 3590-1  YES  NO    NO   NO  NO NO **  000  000356
    ******* 000   030 ADDITIONAL TAPE COPY
    
    HWCOMP ENCRYPT ENCTYPE  ICOUNT RSAKEY/KPWD
    NO   KEYPW  CLRAES128 00100  PASSW0RD
    
    DUMP   UNIT   AUTO  DATASET RESET CLASS  CP FRR           TAPE
    VTOC
    CLASS   TYPE   REUSE RESTORE CHANGE DISABLE REQ AVA DAY FREQ  RETPD
    EXPDT COPIES STACK DISPOSITION
    DB0ATST1 VT3590  YES  YES   NO   NO  ** YES **  000  000001
    ******* 002   060 ********************


-----

    HWCOMP ENCRYPT ENCTYPE  ICOUNT RSAKEY/KPWD
    NO   NONE   ********* *****
    ****************************************************************

The output shows the dump classes, unit type that is used, backup frequency, retention
period, and stack among other attributes.

You can also create a list of all of the dump volumes that were created in your environment.
Issue the following command to list all dump volumes that are recorded on BCDS:

`HSEND LIST DVOL BCDS ODS('MHLRES7.DNS5')`

The command output in Example 13-10 shows you extra information about the dumps,
including the tape status (AVAIL, EXPIRED, or UNEXPIRED), source volsers, dump date and
time, and target tape volser.

_Example 13-10  HSEND LIST DVOL BCDS command output_

    -- DFSMSHSM CONTROL DATASET -DUMP VOLUME-BCDS-- LISTING   --- AT 19:30:38 ON
    
    DUMP  VOL  UNIT   FILE SOURCE       DUMPED   DUMPED
    PCT HW ENC C SET OF DUMP
    VOLSER STATUS TYPE   SEQ VOLSER SMS CLASS  DATE    TIME   EXP DATE IDRC
    LIBRARY FULL    P VOLSERS
    COB001 AVAIL 3590-1  *** ****** *  DB9CTST1 ********** **:**:** *NOLIMIT  **
    *NO LIB* **** ** **** *
    THS012 EXPIR 3590-1          SUNDAY            2008/04/21 Y
    LIB2   *** * *** N
    001 MLDE65 Y 2008/03/25 15:48:02
    THS012
    THS015 UNEXP 3590-1          SUNDAY            2012/09/23 Y
    LIB2   *** N *** N
    001 MLDB35 Y 2012/08/27 18:10:29
    THS015
    THS019 EXPIR 3590-1          DB0BTST1           2012/08/30 Y
    LIB2   *** N *** Y
    001 SBOXJ0 Y 2012/08/29 13:25:44
    THS019
    002 SBOXJ1 Y 2012/08/29 13:25:44
    THS019
    003 SBOXJ8 Y 2012/08/29 13:25:44
    THS019
    004 SBOXJ9 Y 2012/08/29 13:25:44
    THS019

**Note:** To list information about a specific volume, use the **HSEND LIST PRIMARY(DACMC1)**
**BCDS ALLDUMPS** command instead.

If you need a list of all of the data sets that are stored in a specific volume when it was
dumped, you can use the following command. You need to provide the dump volume name,
and the source volume that you want to list. If you do not enter a source volume, DFSMShsm
will list the volumes that are recorded on that tape. The following command lists the content of
SB0XJ0 on the THS019 dump tape:

`HSEND LIST DUMPVOLUME(THS019) DCONTENTS(SBOXJO) ODS('MHLRES7.DSN1')`


-----

The previous **LIST DUMPVOLUME** command gives you information about the dump tape and the
data sets on SB0XJ0. See Example 13-11.

_Example 13-11  LIST DUMPVOLUME output_

    DUMP  VOL  UNIT   FILE SOURCE       DUMPED   DUMPED
    PCT HW ENC C SET OF DUMP
    VOLSER STATUS TYPE   SEQ VOLSER SMS CLASS  DATE    TIME   EXP DATE IDRC
    LIBRARY FULL    P VOLSERS
    THS019 EXPIR 3590-1          DB0BTST1           2012/08/30 Y
    LIB2   *** N *** Y
    001 SBOXJ0 Y 2012/08/29 13:25:44
    THS019
    
    ENCTYPE  RSAKEY/KPWD
    ********* ****************************************************************
    
    DUMP COPY DATA SET NAME = HSM.DMP.DB0BTST1.VSBOXJ0.D12242.T442513
    CONTENTS OF VTOC COPY FOR SOURCE VOLUME SBOXJ0
    DATASET NAME                 ORG MULTI CREATED  REFERENCED
    EXP DATE RACF PSWD  CHANGED
    DB0BA.ARCHLOG1.A0000001            PS  NO  11/04/27 11/04/27
    38/09/11 NO  NO   YES
    DB0BA.ARCHLOG1.A0000002            PS  NO  11/04/28 11/04/28
    38/09/12 NO  NO   YES

**Users**
In addition to the other options, you can also obtain information about DFSMShsm users that
are authorized by the DFSMShsm **AUTH** command.

Use the following command to quickly identify users that have or must not have DFSMShsm
authorization. This command lists user access that is authorized by the **AUTH** command:

`HSEND LIST USER ODS('MHLRES7.LISTUSER')`

The output that is written to the data set looks like Example 13-12.

_Example 13-12  Sample LIST USER command output_

    -  DFSMSHSM CONTROL DATASET - USER-- LISTING ----- AT 20:12:31 ON 12/09/04 FOR SY
    
    ARC1700I DFSMSHSM COMMANDS ARE RACF PROTECTED
    
    USERID  AUTH
    
    BYRNEF  CNTL
    HAIMO   CNTL
    HGPARK  CNTL
    JME001  CNTL
    KOHJI   CNTL

In the command output, a number of users have an authority of “CNTL”. We recommend that
you assign this authority only as required and that most users are assigned “USER” authority.


-----

## 13.2 Query command

As DFSMShsm processes its units of work, it maintains information in its address space. As
the **LIST** command obtains its information from the CDSs, another command will interrogate
the address space. This command is the **QUERY** command, and it can be used to display
real-time information about your DFSMShsm system. The **QUERY** command returns
information that includes the following factors:

- The current SETSYS parameters
- The current ABARS parameters
- The status of outstanding DFSMShsm requests
- Volume space usage
- The status of each volume and data set subtask and long-running commands
- The progress of automatic functions
- The parameters that are used at DFSMShsm startup

Unlike the **LIST** command, the **QUERY** command asks the DFSMShsm address space for the
requested information. Therefore, you cannot use the **QUERY** command to request information
about CDS records and you cannot use the **LIST** command for address space inquires.

Before you issue any **QUERY** command, you must ensure that you have the necessary RACF
or DFSMShsm AUTH authorization to perform the queries. If you do not have the authority, or
do not know how to check it.

You can issue **QUERY** commands from the console or TSO session, code REXX programs, or
add the commands to the DFSMShsm startup procedure. Part of the displayed information
might be incorrect because DFSMShsm is not yet fully initialized.

**DFSMShsm**
When you work with **QUERY** commands, if you issue the **HSEND** command, the output is directed
back to your TSO terminal and to the DFSMShsm log. If you issue the **MODIFY** command
through the console, the information is returned to the system log.

The following example shows the **QUERY** command to display DFSMShsm active tasks:

`HSEND QUERY ACTIVE`

This command displays DFSMShsm active tasks for the specified host. Other hosts might
show different results because they might be performing other functions or have different
settings.

Example 13-13 shows an output from the **HSEND QUERY ACTIVE** command. You can use this
command to check what DFSMShsm is processing, and whether any tasks are on hold.

_Example 13-13  Sample output from HSEND QUERY ACTIVE command_

     ARC0101I QUERY ACTIVE COMMAND STARTING ON HOST=2
     ARC0144I AUDIT=NOT HELD AND INACTIVE, LIST=NOT HELD AND INACTIVE, RECYCLE=NOT
     ARC0144I (CONT.) HELD AND INACTIVE, REPORT=NOT HELD AND INACTIVE
     ARC0160I MIGRATION=NOT HELD, AUTOMIGRATION=NOT HELD, RECALL=NOT HELD,
     ARC0160I (CONT.) TAPERECALL=NOT HELD, DATA SET MIGRATION=INACTIVE, VOLUME
     ARC0160I (CONT.) MIGRATION=INACTIVE, DATA SET RECALL=INACTIVE
     ARC0163I BACKUP=NOT HELD, AUTOBACKUP=NOT HELD, RECOVERY=NOT HELD,
     ARC0163I (CONT.) TAPEDATASETRECOVERY=NOT HELD, DATA SET BACKUP=NOT HELD, VOLUME
     ARC0163I (CONT.) BACKUP=INACTIVE, DATA SET RECOVERY=INACTIVE, VOLUME
     ARC0163I (CONT.) RECOVERY=INACTIVE
     ARC0276I DATA SET BACKUP=INACTIVE, DATA SET BACKUP ACTUAL IDLETASKS=(ALLOC=00,
     
     
-----

    ARC0276I (CONT.) MAX=00)
    ARC1826I FRBACKUP=NOT HELD AND INACTIVE,FRRECOV=NOT HELD AND INACTIVE,FRBACKUP
    ARC1826I (CONT.) DUMP=NOT HELD AND INACTIVE,FRRECOV(TAPE)=NOT HELD AND INACTIVE,
    ARC1826I (CONT.) FRRECOV(DATASET)=NOT HELD AND INACTIVE
    ARC0642I DUMP=NOT HELD, AUTODUMP=NOT HELD, VOLUME DUMP=INACTIVE, VOLUME
    ARC0642I (CONT.) RESTORE=INACTIVE, DATA SET RESTORE=INACTIVE
    ARC0437I - TAPECOPY NOT HELD AND INACTIVE
    ARC0437I - TAPEREPL NOT HELD AND INACTIVE
    ARC0415I EXPIREBV=NOT HELD AND INACTIVE, LAST STORED BACKUP VERSION KEY=, LAST
    ARC0415I (CONT.) STORED ABARS VERSION KEY=, LAST PLANNED END KEY=
    ARC0460I PRIVATE AREA LIMIT=8168K, UNALLOCATED=5428K, LARGEST FREE AREAS=5412K,
    ARC0460I (CONT.) 16K
    ARC0460I EXTENDED PRIVATE AREA LIMIT=1460M, UNALLOCATED=1429M, LARGEST FREE
    ARC0460I (CONT.) AREAS=1428M, 236K
    ARC6018I AGGREGATE BACKUP/RECOVERY = INACTIVE
    ARC6019I AGGREGATE BACKUP = NOT HELD, AGGREGATE RECOVERY = NOT HELD
    ARC1540I COMMON RECALL QUEUE PLACEMENT FACTORS: CONNECTION STATUS=CONNECTED,
    ARC1540I (CONT.) CRQPLEX HOLD STATUS=NONE,HOST COMMONQUEUE HOLD STATUS=NONE,
    ARC1540I (CONT.) STRUCTURE ENTRIES=000% FULL,STRUCTURE ELEMENTS=000% FULL
    ARC1541I COMMON RECALL QUEUE SELECTION FACTORS: CONNECTION STATUS=CONNECTED,
    ARC1541I (CONT.) HOST RECALL HOLD STATUS=NONE,HOST COMMONQUEUE HOLD STATUS=NONE
    ARC0101I QUERY ACTIVE COMMAND COMPLETED ON HOST=2
    
We recommend that you use the **MVS** **MODIFY** command for queries that might be long, so you
can scroll up and down, and perform searches to find the information that you want in the log.
The **MVS MODIFY** command syntax is **/F,** **_hsmtask_** **,QUERY ACTIVE** . The **HSEND** command is not
necessary when you execute **MVS MODIFY** commands.

If multiple hosts are in your z/OS image, you can issue the command to display information
that is necessary to identify each host in the system for future commands. Use this command
to query DFSMShsm host images:

`HSEND Q IMAGE`

The information returns with the message that is shown in Example 13-14.

_Example 13-14  Result from HSEND Q IMAGE command_

    ARC0101I QUERY IMAGE COMMAND STARTING ON HOST=1
    ARC0250I HOST PROCNAME JOBID ASID MODE
    ARC0250I  1   HSM1 02980 0049 MAIN
    ARC0250I  2   HSM2 02981 004A AUX
    ARC0101I QUERY IMAGE COMMAND COMPLETED ON HOST=1

You can also list all DFSMShsm control parameters that are set by using a **LIST** command. It
will give you a list of ABARS, backup, migration, dump, and other settings, so you can verify
that all parameters are set correctly. Use the following command to display all DFSMShsm
control settings:

`HSEND Q SETSYS`

The output from the previous command is large. We recommend that you issue this command
by using **MVS MODIFY** , so you can page up and down, or search on the command output.

For a list of the parameters that were specified on the PROC statement in the DFSMShsm
startup procedure, issue the following command:

`HSEND Q STAR`


-----

The information is returned in the messages that are shown in Example 13-15.

_Example 13-15  Output from the HSEND Q STAR command_

    ARC0101I QUERY STARTUP COMMAND STARTING ON HOST=2
    ARC0143I PARMLIB MEMBER=ARCCMD64, DFSMSHSM AUTHORIZED USERID=HSM, HOSTID=2,
    ARC0143I (CONT.) PRIMARY HOST=NO, LOGSW=NO, STARTUP=NO, EMERGENCY=NO, CDSQ=NO,
    ARC0143I (CONT.) CDSR=NO, PDA=YES, RESTART=NOT SPECIFIED, CDSSHR=RLS,
    ARC0143I (CONT.) RNAMEDSN=NO, STARTUP PARMLIB MEMBER=ARCSTR00
    ARC0249I CELLS=(200,100,100,50,20),HOSTMODE=MAIN
    ARC0101I QUERY STARTUP COMMAND COMPLETED ON HOST=2

In certain cases, you will need information about daily statistics for DFSMShsm processing.
DFSMShsm tracks activity on a daily basis in the daily statistics record (DSR).

To display the DFSMShsm daily statistics information at any time throughout the day, issue
the following command:

`HSEND Q STAT`

DFSMShsm returns a summary of all its processing for the current day. The output includes
CPU time, and data sets that were migrated. See Example 13-16.

_Example 13-16  Statistics from the QUERY STAT command_

    ARC0101I QUERY STATISTICS COMMAND STARTING ON HOST=2
    ARC0155I DFSMSHSM STATISTICS FOR 12/09/05
    ARC0156I STARTUPS=01, SHUTDOWNS=00, ABENDS=00, MWES=0038, CPU TIME=00001.14
    ARC0156I (CONT.) SECONDS
    ARC0157I DS MIGRATED L1=00000001, DS MIGRATED L2=00000000, DS EXTENT
    ARC0157I (CONT.) REDUCTIONS=000000, DS MIGRATED FAIL=001, TRKS
    ARC0157I (CONT.) MIGRATED=00000380,  BYTES MIGRATED=000080596
    ARC0158I DS RECALLED L1=00000000, DS RECALLED L2=00000000, DS RECALL FAIL=000,
    ARC0158I (CONT.)  BYTES RECALLED=000000000, RECALL MOUNTS AVOIDED=00000, EXTRA
    ARC0158I (CONT.) ABACKUP MOUNTS=00000
    ARC0159I DS BACKUP=00000000, DS BACKUP FAIL=000, DS RECOVER=00000000, DS
    ARC0159I (CONT.) RECOVER FAIL=000, RECOVER MOUNTS AVOIDED=00000
    ARC0641I VOL DUMP=0, VOL DUMP FAIL=0, VOL RESTORE=0, VOL RESTORE FAIL=0, DS
    ARC0641I (CONT.) RESTORE=0, DS RESTORE FAIL=0
    ARC0145I DS DELETED=00000000, DS DELETE FAILED=000
    ARC0146I RECYCLED BACKUP VOLUMES=0000, DS=00000000, BLOCKS=000000
    ARC0146I RECYCLED MIGRATION VOLUMES=0000, DS=00000000, BLOCKS=000000
    ARC1825I FAST REPLICATION VOLUME BACKUPS=0 REQUESTED, 0 FAILED; VOLUME
    ARC1825I (CONT.) RECOVERIES=0 REQUESTED, 0 FAILED
    ARC0101I QUERY STATISTICS COMMAND COMPLETED ON HOST=2
    
DFSMShsm automatically records certain errors that it receives and records other errors by
using the diagnosis command, **TRAP** . To see which TRAP activity occurred, issue the following
command:

`HSEND Q T`


-----

The information that is returned in the messages is shown in Example 13-17.

_Example 13-17  Output from the HSEND Q TRAP command_

    ARC0101I QUERY TRAPS COMMAND STARTING ON HOST=2
    ARC0205I TRAP IN MODULE ARCLBUC FOR CODE 00404, TIMES=0001, TYPE=BY OCCURRENCE
    ARC0205I TRAP IN MODULE ARCALVOL FOR CODE 00004, TIMES=0002, TYPE=BY OCCURRENCE
    ARC0101I QUERY TRAPS COMMAND COMPLETED ON HOST=2
    
**Aggregate backup and recovery support**
If you want to query the current DFSMShsm control parameters that apply to aggregate
backup and recovery (ABARS), issue the following command:

`HSEND QUERY ABARS`

The information is returned in the messages that are shown in Example 13-18.

_Example 13-18  Sample return from the QUERY ABARS command_

    ARC0101I QUERY ABARS COMMAND STARTING ON HOST=2
    ARC6008I AGGREGATE BACKUP/RECOVERY PROCNAME = DFHSMABR
    ARC6009I AGGREGATE BACKUP/RECOVERY MAXADDRESSSPACE = 01
    ARC6366I AGGREGATE BACKUP/RECOVERY UNIT NAME = 3490
    ARC6367I AGGREGATE BACKUP/RECOVERY EXITS = NONE
    ARC6368I AGGREGATE BACKUP/RECOVERY ACTIVITY LOG MESSAGE LEVEL IS FULL
    ARC6371I AGGREGATE RECOVERY ML2 TAPE UNIT NAME = 3490
    ARC6372I NUMBER OF ABARS I/O BUFFERS = 02
    ARC6373I ABARS ACTIVITY LOG OUTPUT TYPE = DASD
    ARC6033I AGGREGATE RECOVERY UNIT NAME = 3490
    ARC6036I AGGREGATE BACKUP OPTIMIZE = 3
    ARC6036I AGGREGATE RECOVERY TGTGDS = SOURCE
    ARC6036I AGGREGATE RECOVERY ABARSVOLCOUNT = ANY
    ARC6036I AGGREGATE RECOVERY PERCENTUTILIZED = 090
    ARC6036I AGGREGATE BACKUP/RECOVERY ABARSDELETEACTIVITY = NO
    ARC6036I AGGREGATE BACKUP/RECOVERY ABARSTAPES = STACK
    ARC6036I AGGREGATE BACKUP ABARSKIP = NOPPRC, NOXRC
    ARC0101I QUERY ABARS COMMAND COMPLETED ON HOST=2

You can also query automatic functions that are being performed by DFSMShsm to retrieve
information about the processed volumes, and volumes to be processed by the automatic
function. This information might be helpful for you to determine how long it will take to finish
the current processing.

The following command shows you how to query the DFSMShsm automatic functions:

`HSEND QUERY AUTOP`

The information is returned in the messages that are shown in Example 13-19.

_Example 13-19  Output from previous example_

    ARC0101I QUERY AUTOPROGRESS COMMAND STARTING ON HOST=1
    ARC0247I PRIMARY SPACE MANAGEMENT IS CURRENTLY
    ARC0247I (CONT.) PROCESSING DFSMSHSM MANAGED VOLUMES
    ARC0246I SMS VOLUMES RESTRICTED TO PROCESSING BY THIS
    ARC0246I (CONT.) PROCESSING UNIT: NOT PROCESSED=0, TOTAL=0, SMS
    ARC0246I (CONT.) VOLUMES NOT RESTRICTED TO PROCESSING BY ANY
    ARC0246I (CONT.) PROCESSING UNIT: NOT PROCESSED=0, TOTAL=4, NON-SMS


-----

    ARC0246I (CONT.) VOLUMES: NOT PROCESSED=0, TOTAL=0
    ARC0101I QUERY AUTOPROGRESS COMMAND COMPLETED ON HOST=1

The same command gives the result that is shown in Example 13-20 when automatic
functions are not in progress.

_Example 13-20  Output from previous command if no automatic function is running_

    ARC0101I QUERY AUTOPROGRESS COMMAND STARTING ON HOST=3
    ARC0247I NO AUTOMATIC FUNCTION IS CURRENTLY PROCESSING DFSMSHSM MANAGED VOLUMES
    ARC0246I ODM VOLUMES RESTRICTED TO PROCESSING BY THIS PROCESSING UNIT: NOT
    ARC0246I (CONT.) PROCESSED=0, TOTAL=0, ODM VOLUMES NOT RESTRICTED TO PROCESSING
    ARC0246I (CONT.) BY ANY PROCESSING UNIT: NOT PROCESSED=0, TOTAL=0, NON-SMS
    ARC0246I (CONT.) VOLUMES: NOT PROCESSED=0, TOTAL=0
    ARC0101I QUERY AUTOPROGRESS COMMAND COMPLETED ON HOST=3

**Backup and migration**
Querying backup is necessary to identify possible tasks on hold, or to check and decide how
many tasks need to run concurrently for each function to avoid over-allocating tape drives and
avoid wasting tapes during backup processing.

To list information about your backup configuration, use the **BACK** parameter, which will give
you all of the DFSMShsm backup configurations that are active. Use the following command
to display the backup configuration:

`HSEND Q BACK`

The previous command will display output that is similar to Example 13-21.

_Example 13-21  Sample output from the HSEND Q BACK command_

    ARC0101I QUERY BACKUP COMMAND STARTING ON HOST=3
    ARC0638I MAXDUMPTASKS=03, ADSTART=(0000 0000 0000), DUMPIO=(3,2),
    ARC0638I (CONT.) VOLUMEDUMP=(STANDARD), MAXDUMPRECOVERTASKS=01
    ARC0273I DUMP CYCLE LENGTH=7 DAY(S), CYCLE=NNNNNNY, TODAY IS DAY=3, CYCLE
    ARC0273I (CONT.) START DATE=98/03/02, AUTODUMP/LEVEL FUNCTIONS NOT ELIGIBLE TO
    ARC0273I (CONT.) BE STARTED, CYCLE START TIME NOT SPECIFIED
    ARC0274I BACKUP=YES(TAPE(VT3590G2)), SPILL=YES(ANY), MAXDSRECOVERTASKS=03,
    ARC0274I (CONT.) MAXDSTAPERECOVERTASKS=03
    ARC0154I MAXBACKUPTASKS=03, ABSTART= (0000 0000 0000), VERSIONS=001,
    ARC0154I (CONT.) FREQUENCY=000, SKIPABPRIMARY=NO, BACKUP PREFIX=HSM,
    ARC0154I (CONT.) INCREMENTALBACKUP=CHANGEDONLY, PROFILEBACKUP=YES,
    ARC0154I (CONT.) INUSE=(RETRY=NO, DELAY=015, SERIALIZATION=REQUIRED)
    ARC0269I DS DASD BACKUP TASKS=02, DS TAPE BACKUP TASKS=02, DEMOUNTDELAY=0060,
    ARC0269I (CONT.) MAXIDLETASKS=00, DS BACKUP MAX DASD SIZE=000003000, DS BACKUP
    ARC0269I (CONT.) STD DASD SIZE=000003000, SWITCHTAPES TIME=0000,
    ARC0269I (CONT.) PARTIALTAPE=MARKFULL, GENVSAMCOMPNAMES=YES
    ARC0271I BACKUP CYCLE LENGTH=01 DAY(S), CYCLE=Y, TODAY IS DAY=01, VOLUME
    ARC0271I (CONT.) LIMIT/DAY=0001, AVAILABLE BACKUP VOLUMES=00008, CYCLE START
    ARC0271I (CONT.) DATE=98/03/02
    ARC0101I QUERY BACKUP COMMAND COMPLETED ON HOST=3
    

-----

You can also use the words ALL, DAILY, SPILL, and UNASSIGNED between parenthesis to
display information that relates to specific backup processing. The following command
displays information about spill backup processing:

`HSEND Q BACK(SPILL)`

Part of the information that is displayed in this command output is also available from the
general **HSEND Q BACK** command. Also, ARC0164I is added to indicate the backup volumes.
See Example 13-22.

_Example 13-22  Output from the QUERY BACK(SPILL) command_

    ARC0101I QUERY BACKUP COMMAND STARTING ON HOST=3
    ARC0638I MAXDUMPTASKS=03, ADSTART=(0000 0000 0000), DUMPIO=(3,2),
    ARC0638I (CONT.) VOLUMEDUMP=(STANDARD), MAXDUMPRECOVERTASKS=01
    ARC0273I DUMP CYCLE LENGTH=7 DAY(S), CYCLE=NNNNNNY, TODAY IS DAY=3, CYCLE
    ARC0273I (CONT.) START DATE=98/03/02, AUTODUMP/LEVEL FUNCTIONS NOT ELIGIBLE TO
    ARC0273I (CONT.) BE STARTED, CYCLE START TIME NOT SPECIFIED
    ARC0274I BACKUP=YES(TAPE(VT3590G2)), SPILL=YES(ANY), MAXDSRECOVERTASKS=03,
    ARC0274I (CONT.) MAXDSTAPERECOVERTASKS=03
    ARC0154I MAXBACKUPTASKS=03, ABSTART= (0000 0000 0000), VERSIONS=001,
    ARC0154I (CONT.) FREQUENCY=000, SKIPABPRIMARY=NO, BACKUP PREFIX=HSM,
    ARC0154I (CONT.) INCREMENTALBACKUP=CHANGEDONLY, PROFILEBACKUP=YES,
    ARC0154I (CONT.) INUSE=(RETRY=NO, DELAY=015, SERIALIZATION=REQUIRED)
    ARC0269I DS DASD BACKUP TASKS=02, DS TAPE BACKUP TASKS=02, DEMOUNTDELAY=0060,
    ARC0269I (CONT.) MAXIDLETASKS=00, DS BACKUP MAX DASD SIZE=000003000, DS BACKUP
    ARC0269I (CONT.) STD DASD SIZE=000003000, SWITCHTAPES TIME=0000,
    ARC0269I (CONT.) PARTIALTAPE=MARKFULL, GENVSAMCOMPNAMES=YES
    ARC0271I BACKUP CYCLE LENGTH=01 DAY(S), CYCLE=Y, TODAY IS DAY=01, VOLUME
    ARC0271I (CONT.) LIMIT/DAY=0001, AVAILABLE BACKUP VOLUMES=00008, CYCLE START
    ARC0271I (CONT.) DATE=98/03/02
    ARC0164I SPILL   VOLS = VT0006-A VT0005-A THS016-A
    ARC0101I QUERY BACKUP COMMAND COMPLETED ON HOST=3

If you discover that certain data sets are not migrated correctly, you can issue the following
command. To display all data sets that are being prevented from migrating because a **SETMIG**
**LEVEL(** **_hlq_** **)** command was issued, use the following command to query the migration
restrictions on DFSMShsm:

`HSEND Q RET`

DFSMShsm returns all data sets that are prevented from migrating by the **SETMIG LEVEL(** **_hlq_** **)**
command. See Example 13-23.

_Example 13-23  Sample output from the HSEND Q RET command_

    ARC0101I QUERY RETAIN COMMAND STARTING ON HOST=2
    ARC0175I LEVEL QUALIFIER AND MIGRATION RESTRICTION TYPE
    ARC0176I QUALIFIER=SYS1. RESTRICTION TYPE=NOMIGRATION
    ARC0176I QUALIFIER=SYSCTLG. RESTRICTION TYPE=NOMIGRATION
    ARC0176I QUALIFIER=HSM. RESTRICTION TYPE=NOMIGRATION
    ARC0101I QUERY RETAIN COMMAND COMPLETED ON HOST=2

The command to list the space usage of volumes can be applied to all your volumes or a
specific volser that is coded. Use the following command to list the space information for the
volume SBXH56:

`HSEND Q SPACE(SBXHS6)`


-----

DFSMShsm includes information about the free space, fragmentation, largest extents, and
data set control block (DSCB) usage. See Example 13-24.

_Example 13-24  Sample output from the HSEND Q SPACE command_

    ARC0400I VOLUME SBXHS6 IS 99% FREE, 0000000110 FREE TRACK(S), 000009972 FREE
    ARC0400I (CONT.) CYLINDER(S), FRAG .001
    ARC0401I LARGEST EXTENTS FOR SBXHS6 ARE CYLINDERS   9952, TRACKS   149280
    ARC0402I VTOC FOR SBXHS6 IS 00090 TRACKS(0004500 DSCBS), 0004464 FREE
    ARC0402I (CONT.) DSCBS(99% OF TOTAL)

**Note:** If we omitted the volser, DFSMShsm returns this information for all non-storage
management subsystem (SMS) primary and migration-level 1 (ML1) volumes. Additionally,
no ARC0401I message is issued.

**Control data sets**
Also, use the **QUERY** command to check CDS and journal usage, so you can schedule
changes to reorganize and increase CDS journals ahead of time, issue **BACKVOL CDS**
commands to the null journal data set, and avoid journal-full issues. Use this command to
display CDS information:

HSEND Q CDS

The output in Example 13-25 shows information about all CDS usage, space allocation, and
the threshold that is used for warning messages.

_Example 13-25  Sample output from the HSEND Q CDS command_

    ARC0101I QUERY CONTROLDATASETS COMMAND STARTING ON HOST=2
    ARC0947I CDS SERIALIZATION TECHNIQUE IS RLS
    ARC0148I MCDS TOTAL SPACE=72000 K-BYTES, CURRENTLY ABOUT 28% FULL, WARNING
    ARC0148I (CONT.) THRESHOLD=80%, TOTAL FREESPACE=94%, EA=YES, CANDIDATE
    ARC0148I (CONT.) VOLUMES=4
    ARC0948I MCDS INDEX TOTAL SPACE=210 K-BYTES, CURRENTLY ABOUT 27% FULL, WARNING
    ARC0948I (CONT.) THRESHOLD=80%, CANDIDATE VOLUMES=4
    ARC0148I BCDS TOTAL SPACE=72000 K-BYTES, CURRENTLY ABOUT 48% FULL, WARNING
    ARC0148I (CONT.) THRESHOLD=80%, TOTAL FREESPACE=95%, EA=YES, CANDIDATE
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
    
You can also display information about CDS backup data sets. The following command shows
how to use the **CDSV** command to query CDS backup data sets:

HSEND Q CDSV


-----

The **HSEND Q CDSV** command output includes backup data set names, the latest valid backup
version (LATESTFINALQUALIFIER), and the data mover that was used for backup. See
Example 13-26.

_Example 13-26  Sample output from the QUERY CDSV command_

    ARC0101I QUERY CDSVERSIONBACKUP COMMAND STARTING ON HOST=2
    ARC0375I CDSVERSIONBACKUP, MCDSBACKUPDSN=HSM.MCDS.BACKUP,
    ARC0375I (CONT.) BCDSBACKUPDSN=HSM.BCDS.BACKUP, OCDSBACKUPDSN=HSM.OCDS.BACKUP,
    ARC0375I (CONT.) JRNLBACKUPDSN=HSM.JRNL.BACKUP
    ARC0376I BACKUPCOPIES=0004, BACKUPDEVICECATEGORY=DASD,
    ARC0376I (CONT.) LATESTFINALQUALIFIER=D0000040, DATAMOVER=DSS
    ARC0101I QUERY CDSVERSIONBACKUP COMMAND COMPLETED ON HOST=2

**Requests**
Displaying requests is the easiest way to identify the type of requests that are waiting on the
queue. You can increase the number of tasks that are running concurrently if enough
resources are available, identify users that are sending multiple requests, and cancel or give a
request a higher priority, depending on the user’s need and the available resources.

You can use many options with **QUERY** commands. The first command displays a summary of
all requests that are being processed, or are waiting to be processed by DFSMShsm:

HSEND Q WAIT

DFSMShsm will sum all migration, backup, recall, and other requests, and display the results
in an easy-to-use format. See Example 13-27.

_Example 13-27  Sample output from the HSEND Q WAIT command_

    ARC0101I QUERY WAITING COMMAND STARTING ON HOST=2
    ARC1542I WAITING MWES ON COMMON QUEUES: COMMON RECALL QUEUE=00000000,
    ARC1542I (CONT.) TOTAL=00000000
    ARC0168I WAITING MWES: MIGRATE=00000000, RECALL=00000000, DELETE=00000000,
    ARC0168I (CONT.) BACKUP=00000001, RECOVER=00000000, COMMAND=00000000,
    ARC0168I (CONT.) ABACKUP=00000000, ARECOVER=00000000, FRBACKUP=00000000,
    ARC0168I (CONT.) FRRECOV=00000000, TOTAL=00000001
    ARC0101I QUERY WAITING COMMAND COMPLETED ON HOST=2

To produce a detailed list of all requests on the DFSMShsm queue, you can issue the
following command:

`HSEND Q REQ`

The output from the previous command is a list of all requests that re being processed, or
waiting to be processed by DFSMShsm. See Example 13-28.

_Example 13-28  Output from the previous QUERY command_

    ARC0101I QUERY REQUEST COMMAND STARTING ON HOST=2
    ARC0167I BACKUP MWE FOR DATA SET MHLRES7.LISTUSER FOR USER MHLRES7, REQUEST
    ARC0167I (CONT.) 00000145, WAITING TO BE PROCESSED, 00000 MWE(S) AHEAD OF THIS
    ARC0167I (CONT.) ONE
    ARC0101I QUERY REQUEST COMMAND COMPLETED ON HOST=2

**Note:** The previous command can result in large output if many requests exist for
DFSMShsm.


-----

You can also query for specific users to determine the requests that were sent for processing
by DFSMShsm by a single user. The following command is used to display requests from a
specific user:

`HSEND Q USER( _user_ )`

In addition, you can query for a data set name to retrieve information about this specific
request. The following command queries for information about requests that involve a specific
data set:

`HSEND Q DATASETNAME(' _dsname_ ')`

The output includes the user who requested the action, and the number of requests that are
ahead for processing. See Example 13-29.

_Example 13-29  Sample output from the Q DATASETNAME command_

    ARC0101I QUERY DATASETNAME COMMAND STARTING ON HOST=2
    ARC0167I BACKUP MWE FOR DATA SET MHLRES7.LISTUSER FOR USER MHLRES7, REQUEST
    ARC0167I (CONT.) 00000155, WAITING TO BE PROCESSED, 00000 MWE(S) AHEAD OF THIS
    ARC0167I (CONT.) ONE
    ARC0101I QUERY DATASETNAME COMMAND COMPLETED ON HOST=2

## 13.3 Report command

The **REPORT** command is used to gather and consolidate statistics about DFSMShsm
operations and functions. DFSMShsm during its processing stores daily function and
volume-related information in the MCDS. Use the **REPORT** command to gather information that
is generated at various levels:

- Gather the following information at a function level:

  - Backup
  - Migration
  - Delete
  - Recall
  - Recover
  - Recycle
  - Spill

- At a daily level

- At a volume level

- For statistics before or after a certain date

- For statistics between certain dates

- For a summary of all statistics reports

We show several **REPORT** commands that you can use to give you an idea of the type of
information that is available. We also introduce utilities in SYS1.SAMPLIB(ARCTOOLS).

Before you issue any **REPORT** command, you must ensure that you have the necessary RACF
or DFSMShsm AUTH authorization to perform the queries. If you do not have the authority, or
do not know how to check it.


-----

You also must specify DFSMShsm for how long you want to keep statistics information. The
following command sets this retention on DFSMShsm and can be added to DFSMShsm
PARMLIB:

`HSEND SETSYS MIGRATIONCLEANUPDAYS( _recall stats reconnect)_`

The preceding command has three parameters that you can use. The **recall** parameter tells
DFSMShsm how long to keep MCDS information about a recalled data set. The **stats**
parameter defines the number of days that DFSMShsm will keep statistics information. The
**reconnect** parameter sets the number of days that DFSMShsm holds MCDS information for
recalled data sets that are candidates for reconnection. The defaults are 10 (for _recall_ ), 30
(for _stats_ ), and 3 (for _reconnect_ ).

As in the **LIST** command, you can direct the **REPORT** command output to different places, such
as a terminal, SYSOUT, or data set, by using the **ODS** parameter. If the data set exists, the
REPORT output will be appended at the end of the data set. Otherwise, DFSMShsm will
create the data set.

To obtain a summary of all function-related activity that occurred today, issue the following
command:

`HSEND REPORT DAILY FUNCTION ODS(MHLRES5.DAILY.STATS)`

The output that is written to the data set looks similar to the output that is shown in
Example 13-30.

_Example 13-30  Output from the REPORT DAILY FUNCTION command_

    --DFSMSHSM STATISTICS REPORT ------- AT 16:55:21 ON 2012/09/05 FOR SYSTEM=SC64
    
    DAILY STATISTICS REPORT FOR 12/09/05
    
    STARTUPS=001, SHUTDOWNS=000, ABENDS=000, WORK ELEMENTS PROCESSED=000055, BKUP VOL
    RECYCLED=00000, MIG VOL RECYCLED=00000
    DATA SET MIGRATIONS BY VOLUME REQUEST= 0000001, DATA SET MIGRATIONS BY DATA SET
    REQUEST= 00000, BACKUP REQUESTS= 0000000
    EXTENT REDUCTIONS= 0000000 RECALL MOUNTS AVOIDED= 00000 RECOVER MOUNTS AVOIDED=
    00000
    FULL VOLUME DUMPS= 000000 REQUESTED, 00000 FAILED; DUMP COPIES= 000000 REQUESTED,
    00000 FAILED
    FULL VOLUME RESTORES= 000000 REQUESTED, 00000 FAILED; DATASET RESTORES= 000000
    REQUESTED, 00000 FAILED
    ABACKUPS= 00000 REQUESTED,00000 FAILED; EXTRA ABACKUP MOUNTS=00000
    DATA SET MIGRATIONS BY RECONNECTION = 000000, NUMBER OF TRACKS RECONNECTED TO
    TAPE = 00000000
    FAST REPLICATION VOLUME BACKUP = 00000000 REQUESTED, 00000000 FAILED
    FAST REPLICATION VOLUME RECOVER = 00000000 REQUESTED, 00000000 FAILED
    
    NUMBER ------READ-------- -----WRITTEN------
    ------REQUESTS---- AVERAGE ------AVERAGE TIME------  HSM FUNCTION  DATASETS TRK/BLK  BYTES  TRK/BLK  BYTES  SYSTEM USER
    FAILED  AGE  QUEUED WAIT PROCESS TOTAL
    
    MIGRATION
    PRIMARY - LEVEL 1 0000001 00000380 000080596 00000001 000032768 000001 00000
    00000 00001 0000 00000 00000 00000
    SUBSEQUENT MIGS  0000000 00000000 000000000 00000000 000000000 000000 00000
    00000 00000 0000 00000 00000 00000
    
    
    -----
    
    PRIMARY - LEVEL 2 0000000 00000000 000000000 00000000 000000000 000000 00001
    00001 00000 0000 00000 00000 00000
    RECALL
    LEVEL 1 - PRIMARY 0000000 00000000 000000000 00000000 000000000 000000 00000
    00000 00000 0000 00000 00000 00000
    LEVEL 2 - PRIMARY 0000000 00000000 000000000 00000000 000000000 000000 00000
    00000 00000 0000 00000 00000 00000
    DELETE
    MIGRATE DATA SETS 0000000 00000000 000000000 00000000 000000000 000000 00000
    00000 00000 0000 00000 00000 00000
    PRIMARY DATA SETS 0000000 00000000 000000000 00000000 000000000 000000 00000
    00000 00000 0000 00000 00000 00000
    BACKUP
    DAILY BACKUP   0000000 00000000 000000000 00000000 000000000 000000 00000
    00000 00000 0000 00000 00000 00000
    SUBSEQUENT BACKUP 0000000 00000000 000000000 00000000 000000000 000000 00000
    00000 00000 0000 00000 00000 00000
    DELETE BACKUPS  0000000 00000000 000000000 00000000 000000000 000000 00000
    00000 00000 0000 00000 00000 00000
    RECOVER
    BACKUP - PRIMARY 0000000 00000000 000000000 00000000 000000000 000000 00000
    00000 00000 0000 00000 00000 00000
    RECYCLE
    BACKUP - SPILL  0000000 00000000      00000000      000000 00000
    00000 00000 0000 00000 00000 00000
    MIG L2 - MIG L2  0000000 00000000      00000000      000000 00000
    00000 00000 0000 00000 00000 00000
    
The header of the REPORT output contains the number of mounts that were saved by
RECALL and RECOVER. This information is useful for addressing the changes in mounts
when you use larger capacity tapes.

It is also possible to define specific date periods that you want the reports, on the condition
that they are stored by DFSMShsm. To specify a period for the report, you can issue the
following command:

HSEND REPORT DAILY FROMDATE(12/08/12) FUNCTION TODATE(12/09/05)

The output is similar to the DSR. The difference is that you now get information at a function
level:

- A function summary for the period 24 July to 31 July inclusive
- A function summary report that totals all activity for the requested period

You can use the **FUNCTION** parameters to select only the necessary information you need, for
example:

- BACKUP
- DELETE
- MIGRATION
- RECALL
- RECOVER
- RECYCLE
- SPILL


-----

If you issue the following command, you obtain a report that details the data sets that are
backed up to daily volumes:

HSEND REPORT DAILY FUNCTION(BACKUP) ODS(MHLRES5.BACKUP.REPORT)

The output is similar to the output that was produced previously but restricted to the backup
function. Certain statistics are always reported, and these statistics will appear before the
requested function report.

It is also possible to create reports for all DFSMShsm-managed volumes, or specific volumes.
DFSMShsm will retrieve information about backup, migration, recover, and recycle. The
following command lists statistics for DFSMShsm-managed volumes:

HSEND REPORT VOLUMES FUNCTION ODS('MHLRES7.VOLUME.STATS')

Example 13-31 shows us the output from the previous command. If you intend to display data
about a single volume, put the _volser_ between parentheses after the **VOLUMES** parameter.

_Example 13-31  Output from the REPORT VOLUMES command_

    UNIT TYPE = 3390  , HSM VOLUME TYPE = LEVEL 1
    MIGRATED DATA SETS BY VOLUME REQUEST=000001, DATA SET MIGRATIONS BY DATA SET REQ
    MINIMUM AGE = 000, TOTAL TRACKS = 00003243, FREE TRACKS = 00147012, FRA
    VOLUME DUMP= NOT DONE; DUMP COPIES=000000 REQUESTED, FAILED=000000
    VOLUME RESTORE= NOT DONE; DATASET RESTORES=000000 REQUESTED, FAILED=000000
    DATA SET MIGRATIONS BY RECONNECTION = 000000 , NUMBER OF TRACKS RECONNECTED TO T
    FAST REPLICATION BACKUP = 00000 REQUESTED, 00000 FAILED
    FAST REPLICATION RECOVER = 00000 REQUESTED, 00000 FAILED
    
    NUMBER ------READ-------- -----WRITTEN------ ------REQUES
    HSM FUNCTION  DATASETS TRK/BLK  K-BYTES TRK/BLK  K-BYTES SYSTEM USER
    
    MIGRATION
    PRIMARY - LEVEL 1 0000001 00000000 000000000 00000001 000000032 000001 00000
    SUBSEQUENT MIGS  0000000 00000000 000000000 00000000 000000000 000000 00000
    PRIMARY - LEVEL 2 0000000 00000000 000000000 00000000 000000000 000000 00000
    RECALL
    LEVEL 1 - PRIMARY 0000000 00000000 000000000 00000000 000000000 000000 00000
    LEVEL 2 - PRIMARY 0000000 00000000 000000000 00000000 000000000 000000 00000
    DELETE
    MIGRATE DATA SETS 0000000 00000000 000000000 00000000 000000000 000000 00000
    PRIMARY DATA SETS 0000000 00000000 000000000 00000000 000000000 000000 00000
    BACKUP
    DAILY BACKUP   0000000 00000000 000000000 00000000 000000000 000000 00000
    SUBSEQUENT BACKUP 0000000 00000000 000000000 00000000 000000000 000000 00000
    DELETE BACKUPS  0000000 00000000 000000000 00000000 000000000 000000 00000
    RECOVER
    BACKUP - PRIMARY 0000000 00000000 000000000 00000000 000000000 000000 00000
    RECYCLE
    BACKUP - SPILL  0000000 00000000      00000000      000000 00000
    MIG L2 - MIG L2  0000000 00000000      00000000      000000 00000


-----

We do not show all of the **REPORT** command parameters. The information that is available
within the DSRs volume statistics records (VSRs) is enough to give you a good idea of the
activity. You might be able to spot patterns for particular days and functions, or identify
unusually high activity against specific volumes and take action to spread the work more
evenly.

## 13.4 Using the DFSMSrmm reporting tool

In certain cases, the standard reports and lists from DFSMShsm do not provide the
necessary information for the storage manager to determine DFSMShsm usage or to create
reports with specific information about data sets, such as the creation date or the number of
data sets that were migrated by using a specific management class.

To create these reports, you can use the DFSMSrmm report generator.

Although DFSMSrmm might not be installed on your system, you can still use report
generator to create your reports. They are available from ISMF panel option G.

We show how to get used to the report generator, run several predefined reports, and alter
the displayed fields.

When you get into the DFSMSrmm report generator, you will see options to work with reports,
report types, and the reporting tool, as shown in Figure 13-1.

    DFSMSrmm Report Generator
    Option ===>

    0 OPTIONS     - Specify dialog options and defaults
    1 REPORT     - Work with reports
    2 REPORT TYPE   - Work with report types
    3 REPORTING TOOL - Work with reporting tools
    4 MIGRATION    - Migration tasks for reporting
    X  EXIT      - Exit DFSMSrmm dialog
    Enter selected option or END command. For more info., enter HELP or PF1.

_Figure 13-1  DFSMSrmm Report Generator panel_

By entering option 1 , REPORT, you can search for predefined user, installation, or product
reports. You can use this window to display the existing reports, or to create a new report. If
you search for a report name that does not exist, report generator will return an empty list, so
that you can create a new report that you want.

We selected all predefined DFSMShsm reports, starting with ARC* from user and product
reports. Figure 13-2 shows all of the DFSMShsm-related reports that are
defined to the report generator.


-----
    
    DFSMSrmm Report Definitions     Row 1 to 15 of 16
    Command ===>                         Scroll ===> PAGE
    
    The following line commands are valid: A,D,G,H,J,L,M,N,S, and T
    
    S Name   Report title          Report type         User id
    
    -  -------- ------------------------------ ---------------------------- ------ ARCGAB01 ABARS ABACKUP Statistics    DFSMShsm ABARS Report    HSM
    ARCGAR01 ABARS ARECOVER Statistics   DFSMShsm ABARS Report    MHLRES7
    ARCGDB01 DCOLLECT BACKUP DATA      DFSMShsm DCOLLECT BACKUP   MHLRES7
    ARCGDD01 DCOLLECT DASD CAPACITY PLANNIN DFSMShsm DCOLLECT DASD CAP  MHLRES7
    ARCGDM01 DCOLLECT MIGRATION DATA    DFSMShsm DCOLLECT MIGRATION HSM
    ARCGDT01 DCOLLECT TAPE CAPACITY PLANNIN DFSMShsm DCOLLECT TAPE CAP  HSM
    ARCGS001 Statistics for DFSMShsm    DFSMShsm FSR-SMF Records   MHLRES7
    ARCGS002 Statistics for Backup     DFSMShsm FSR-SMF Records   HSM
    ARCGS003 Statistics for Migration    DFSMShsm FSR-SMF Records   HSM
    ARCGS004 Statistics for Recall     DFSMShsm FSR-SMF Records   HSM
    ARCGS005 Statistics for Recovery    DFSMShsm FSR-SMF Records   HSM
    ARCGS006 Statistics for Volume Dump   DFSMShsm FSR-SMF Records   HSM
    ARCGS007 Statistics for Restore from Du DFSMShsm FSR-SMF Records   MHLRES7
    ARCGS008 Statistics for FRBACKUP    DFSMShsm FSR-SMF Records   HSM
    ARCGS009 Statistics for FRRecover    DFSMShsm FSR-SMF Records   HSM

_Figure 13-2  Searching for all DFSMShsm-related predefined reports_

Several DFSMShsm reports are predefined to the report generator. Each report uses a
different input, and produces a different output. From this panel, you can add, copy, update, or
submit reports for processing.

To display information about the data that will be displayed on the report, enter S to select the
report. If you plan to add or remove fields on the report, you can create a new report or
document the changes in a place that other users of this report can access.

We selected report ARCGMD01 to collect information about data set migration. The report
collects information about migration date and time, data set name, original volume, number of
times migrated, and other information.

By entering G to generate the report, a new configuration window opens. In this window, we
need to provide the necessary information so that the report generator can create the JCL to
extract the report.

The input data set field is the data set that is used by the report generator as a source to
extract the information. Use a data set that your job has access to READ if the data set exists,
or ALTER if the data set will be created by the report generator.

If you do not have the DCOLLECT created for this run, or records that were extracted from
System Management Facilities (SMF) for other reports, you can set “Create report data” to Y
(on Figure 13-3). The report generator will add a step to create the input file and
gather the DCOLLECT or SMF data. Use the skeleton variables to control your extract and
report processing, such as the values that are used in SORT include conditions, and data set
names.

Figure 13-3 shows a sample configuration to create the report JCL.


-----

    DFSMSrmm Report Generation - ARCGDB01
    Command ===>
    
    Enter or change the skeleton variables for the generated JCL:
    
    Input data set . . . . 'MHLRES7.DCOLLECT.INPUT'
    
    Date format . . . . . . YYYYDDD
    (American, European, Iso, Julian, or free form)
    Required if you use variable dates (&TODAY) in your selection criteria.
    
    Create report data . . Y (Y/N)
    Choose Y if you want an extract step included into your generated JCL.
    
    Additional skeleton variables, for example if an extract step is included:
    Skeleton Variable_1 . .
    Skeleton Variable_2 . .
    Skeleton Variable_3 . .
    The skeleton selection depends on the reporting macro . . . : ARCUTILP
    and macro keyword . . : TYPE=B
    Enter END command to start the report generation or CANCEL

_Figure 13-3  Setting up the report variables_

When all of the definitions are set, you can use PF3 to exit this window. A message that is
similar to “ Report JCL ARCGDB01 stored on 'MHLRES7.REPORT.JCL '” displays at your terminal.

Before you submit your job, consider checking the DFSMShsm CDS names or SMF data sets
that report generator used to create the extract data to confirm that they are accurate.

Example 13-32 shows a sample output from the report that was created in the previous steps.

_Example 13-32  Sample output from the report generator tool_

    DCOLLECT MIGRATION DATA    - 1 -    2012/09/07    14:23:51
    
    MIG DATE  MIG TIME  DSN                      1ST SRC VO
    ---------  --------  --------------------------------------------  --------- 2012236  12353803  MHLRES7.HSM.BCDS                SBOX75
    2012237  22200769  MHLRES7.BV                   MLD10C
    2012242  16573407  MHLRES7.NONSMSE                MLD20C
    2012249  00050086  MHLRES7.NONSMSA.DSN1              ML940D
    2012250  10105729  MHLRES7.ACSTEST.LISTDS             MLD83A

If you determine that none of the existing reports provide the exact information that you want,
you can copy a similar report, and update it to include the information that you need.

To copy an existing report to a new report, you can search for it on the report option, and then
enter option N to copy the report definition. You must define a new report name, which will be
stored in your user report library.

In Figure 13-4, we show the process to create a new report that is based on an
existing report.


-----

    DFSMSrmm Report Definitions     Row 1 to 14 of 16
    Command ===>                         Scroll ===> PAGE
    
    The following line commands are valid: A,D,G,H,J,L,M,N,S, and T
    
    S Name   Report title          Report type         User id
    
    -  -------- EsssssssssssssssssssssssssssssssssssssssssssssssssssN ----- -------
    N ARCGAB01 e                          e    MHLRES7
    ARCGAR01 e                          e    MHLRES7
    ARCGDB01 e                          e P   MHLRES7
    ARCGDD01 e Enter the report name . . . . ARCABNEW      e CAP  MHLRES7
    ARCGDM01 e                          e TION MHLRES7
    ARCGDT01 e                          e CAP  HSM
    ARCGS001 DsssssssssssssssssssssssssssssssssssssssssssssssssssM s   MHLRES7
    ARCGS002 Statistics for Backup     DFSMShsm FSR-SMF Records   HSM
    ARCGS003 Statistics for Migration    DFSMShsm FSR-SMF Records   HSM
    ARCGS004 Statistics for Recall     DFSMShsm FSR-SMF Records   HSM
    ARCGS005 Statistics for Recovery    DFSMShsm FSR-SMF Records   HSM
    ARCGS006 Statistics for Volume Dump   DFSMShsm FSR-SMF Records   HSM
    ARCGS007 Statistics for Restore from Du DFSMShsm FSR-SMF Records   MHLRES7
    ARCGS008 Statistics for FRBACKUP    DFSMShsm FSR-SMF Records   HSM

_Figure 13-4  Copying an existing report to a new report_

After you copy the report, you can perform a new search by using its report name, and use S
to select the report to update. This option shows you all of the available fields on the report. If
the field that you need is not listed in this report, you can try to use another predefined report
as source, or code your own program to read the DCOLLECT or SMF data, and extract the
information that you need.

In Figure 13-5, we updated the ARCABNEW report to _not_ include a return code and reason
code for ABARS ABACKUP statistics.

    DFSMSrmm Report Definition - ARCABNEW  Row 15 to 21 of 48
    Command ===>                         Scroll ===> PAGE
    
    Report title . . . ABARS ABACKUP Statistics                 +
    Report footer . .
    Reporting tool . : ICETOOL                 Report width: 267
    Enter "/" to select option
    Edit the help information for this report
    
    Use END to save changes or CANCEL
    The following line commands are valid: S, and R
    
    S CO SO Field name      Column header text          CW Len Typ
    
    -  -- --- -------------------- ------------------------------------ --- --- ---
    14   WFSR2ABACKUP_TOTALSP TOTAL KB                8  8 N
    15   WFSR2CPUTIME     CPU TIME                8  4 N
    16   WFSR2ACCOUNT_CODE  ACC CODE               32 32 C
    WFSR2RC       RC (hex)                8  4 B
    WFSR2REAS      RSN (hex)               9  4 B
    
    -     WFSR2TYPE      TYPE - Use browse macro (M)      9  1 B

_Figure 13-5  Updating the ARCABNEW report_


-----

Exit the edit window by pressing PF3 to save changes.

## 13.5 Utilities in ARCTOOLS for DFSMShsm reporting

DFSMShsm ships utilities for reporting and other uses. All of the utilities are contained in the
ARCTOOLS member in SYS1.SAMPLIB.

To obtain these utilities, use the following steps:

1. Edit the ARCTOOLS member of SYS1.SAMPLIB and change the fields as instructed.

2. Run ARCTOOLS to allocate the HSM.SAMPLE.TOOL data set with several members that
can be used for reporting.

We introduce only the SCANBLOG, SCANMLOG, SCANFSR, FSRSTAT, and QUERYSET
utilities:

- **SCANBLOG utility:** This sample REXX exec scans several days’ worth of backup activity
logs and summarizes the results.

- **SCANMLOG utility:** This sample REXX exec scans several days’ worth of migration activity
logs and summarizes the results.

- **SCANFSR utility:** This sample REXX exec scans the FSR data that was extracted from
SMF (by using the IFASMFDP program) and summarized by FSR type
and management class.

- **FSRSTAT utility:** This sample REXX program reads FSR data and presents statistical
results. In certain cases, the data is presented as a histogram to group
data in certain categories for further analysis.

- **QUERYSET utility:** You cannot use the **QUERY** command to send its output to a data set. By
using TSO/E extended consoles, this sample REXX allows programs
to be submitted that can issue a console command, such as **QUERY** ,
and to get the results back for interpretation.

## 13.6 Using DCOLLECT

Another method of gathering data that relates to space utilization and capacity planning is to
use _DCOLLECT_ , an access method services (AMS) function that is run in a batch
environment. With DCOLLECT, you can obtain information about the following features:

- Active data sets
- Volumes
- Migrated data sets
- Backup data sets
- DASD capacity planning
- Tape capacity planning

For a detailed description of DCOLLECT, see _DFSMS Access Method Services for Catalogs,_
SC26-7394.

We show how you can produce an output data set that can be used later as input to a report
creation package.


-----

You can produce the output data set by invoking IDCAMS DCOLLECT or by using the
ARCUTIL utility, which is a DFSMShsm-related program that is used to capture DFSMShsm
information.

If DFSMShsm information is requested from within a DCOLLECT batch job, the ARCUTIL
load module is called, which is almost transparent to the user. The parameters, although
specified in a different place, are the same parameters that you use on the DCOLLECT
SYSIN DD card.

The sample DCOLLECT JCL that is shown in Example 13-33 calls ARCUTIL transparently.

_Example 13-33  Sample DCOLLECT JCL to call ARCUTIL_

    //DCOLLECT  JOB  ,'DCOLLECT RUN',CLASS=Z,MSGCLASS=H,REGION=4M
    //*
    //STEP1  EXEC  PGM=IDCAMS
    //*
    //SYSPRINT DD SYSOUT=*
    //ARCSNAP  DD SYSOUT=*
    //MCDS   DD DSN=HSM.MCDS,DISP=SHR
    //BCDS   DD DSN=HSM.BCDS,DISP=SHR
    //DCOUT   DD DSN=MHLRES5.DCOLLECT.OUTPUT,
    //       DISP=(NEW,CATLG,DELETE),
    //       SPACE=(1,(859,429)),AVGREC=K,
    //       DSORG=PS,RECFM=VB,LRECL=264
    //SYSIN   DD *
    DCOLLECT      OUTFILE(DCOUT)      NODATAINFO      NOVOLUMEINFO      MIGRATEDATA      BACKUPDATA      CAPPLANDATA      MIGRSNAPERR
    
If you want to invoke ARCUTIL directly by using the same DCOLLECT specified parameters
to capture DFSMShsm information, use the JCL that is shown in Example 13-34.

_Example 13-34  Sample ARCUTIL JCL_

    //JOB2  JOB 'ARCUTIL RUN',REGION=4M
    //STEP2  EXEC PGM=ARCUTIL,PARM='DCOLLECT MIGD CAPD BACD MSERR'
    //ARCSNAP SYSOUT=*
    //ARCTEST SYSOUT=*
    //ARCDATA DD DSN=MY.COLLECT.DATA,DISP=(,CATLG),
    //      SPACE=(CYL,(5,10)),UNIT=SYSDA
    //MCDS  DD DSN=HSM.MCDS,DISP=SHR
    //BCDS  DD DSN=HSM.BCDS,DISP=SHR

This JCL gathers the same information as the IDCAMS DCOLLECT execution, but this time
you directly invoke ARCUTIL.


-----

The DCOLLECT parameters are specified on the EXEC parm statement. DCOLLECT must
be the first command that is specified. It can be followed by any of the other **DCOLLECT**
keywords.

The output that is generated by IDCAMS DCOLLECT, or ARCUTIL, is not a report-like format.
Use another program to read this data and generate the reports that you require. You can use
DFSORT or ICETOOL to create the necessary reports.


-----

