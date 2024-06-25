# 9. Space management

DFSMShsm manages and maintains a multitiered environment to ensure that data availability
requirements are met at any time in a data set’s lifecycle. When data is created, it exists on a
primary disk. As specified in the rules that are set in the DFSMS management class, data
sets eventually move on to cheaper media, such as compressed disks or a compressed tape
media. These moves are based on the data that is no longer referenced.
This functionality is part of space management along with the cleanup, release of unused
space, reduction of extents, and recalls (retrieval of data sets from Level 1 or Level 2).
The space management function automatically maintains the environment as outlined, but it
can be customized to meet your individual requirements. This chapter explains in detail how
the function is logically split into subfunctions and how these subfunctions can be influenced
and adjusted by parameter settings in DFSMShsm.
The overall description assumes Data Facility Storage Management Subsystem (DFSMS)
management of data where storage groups and storage group settings together with
management class settings decide how data is managed.
The space management of non-storage management subsystem (SMS) data is covered and
the difference between SMS-managed data and non-SMS-managed data is explained.
Migration subtasking, which was introduced in z/OS 2.1, is also explained.


## 9.1 Functions in primary space management

Functions in primary space management are described.

### 9.1.1 Tiers used in DFSMShsm space management

The way DFSMShsm moves data between tiers can seem simple to the more experienced
user. For new users, Table 9-1 might be helpful. When DFSMShsm moves data off primary
volumes, the term _migrate_ (or _archive_ ) is used. When data is retrieved from a lower tier, the
operation is called _recall_ .

Table 9-1 describes the DFSMShsm tiers that are used in the space management function.

_Table 9-1  DFSMShsm tier levels_

|Tier levels in DFSMShsm|Description|
|---|---|
|Primary disk: Migration level 0 (also called ML0)|Disk that holds online data|
|Migration-level 1 (also called ML1)|Disk that holds DFSMShsm compressed data, which must be recalled to ML0 and unpacked to be readable|
|Migration-level 2 (also called ML2)|Tapes that hold DFSMShsm migrated, compressed data (must be recalled to ML0 and unpacked to be readable)|
|Small-data-set packing (SDSP)|A VSAM file on the ML1 disk that holds small data sets, which are under a specified size, to save DASD space on ML1|



Space management can be split into three main functions with individual subfunctions.

**Migration: Movement of data to a lower level**
Migration has the following subfunctions:

- Migration from level 0 to level 1
- Migration from level 0 to level 2
- Migration from level 1 to level 2
- Reconnect unchanged data sets that are recalled from level 2 (Fast Subsequent Migration
function)

**Cleanup-related tasks**
Cleanup has the following subfunctions:

- Extent reduction
- Release extents
- Release unused space
- Scratch obsolete and expired data sets (in control data sets (CDSs) and on volumes)

**Recall subfunctions**
Recall has the following subfunctions:

- Recall data sets from level 1 to level 0
- Recall data sets from level 2 to level 0


-----

## 9.2 Components in space management

The components in space management are described.

### 9.2.1 Automatic primary space management function

The automatic primary space management function focuses on migrating data off the primary
volumes in a storage group (that is managed by DFSMShsm) that is based on the
management class setting (Primary Days value). If this value is exceeded for the individual
data set, it is a candidate to migrate to a lower level in the hierarchy (ML1 or ML2). The
following tasks are also performed in the primary space management function:

- Deletion of temporary and expired data sets from the volumes (that are managed by
DFSMShsm) that are being processed. The management class that is assigned to all data
sets that are managed by DFSMS will control this deletion based on the expiration
attributes in the assigned management class. Or potentially, the management class that is
assigned to all data sets that are managed by DFSMS will control this deletion based on
an expiration date that can be assigned to the data set during the define process and
stored in the volume table of contents (VTOC).

- The release of unused space that is based on the management class value that was set
for this attribute occurs on physical sequential, partitioned, and extended format Virtual
Storage Access Method (VSAM) data sets.

- Reconnections of eligible data sets are based on the Fast Subsequent Migration (FSM)
function (must be active) and occur to the volumes where the recall occurred previously.

- The Extent Reduction function will be based on the SETSYS value for this function,
reducing the number of extents on physical sequential, partitioned, and direct access data
sets if they exceeded the specified number of extents. During the process of extent
reduction, they also release any unused space in the data sets and compress partitioned
data sets.

Primary space management continues until the SMS-managed volumes contain the specified
amount of free space, which is under the high threshold on the volume. If deletion of expired
data sets, Fast Subsequent Migration, and space reduction (phase one) of the remaining data
sets achieve the specified volume free space low threshold, no actual data sets are moved.
This cleanup is a two-phase approach, where data set move (phase two) occurs only if phase
one does not bring the volume free space below the high threshold.

If data sets are to be migrated during primary space management, they will be migrated in
compacted form, if possible. Data sets that are expected to be smaller than 110 KB (where 1
KB is 1,024 bytes) after they are compacted are candidates for small-data-set packing
(SDSP), if enabled.

**Controlling primary space management**
The window for primary space management is controlled through the DFSMShsm **SETSYS**
command. You need to consider the planned start and planned end of the function. Use
**CYCLESTARTDATE** to define a start date and the execution during the week by setting Y and N
values. The following example shows how to code the start cycle for primary space
management:

`DEFINE PRIMARYSPMGMTCYCLE (YYYYYYY CYCLESTARTDATE(1998/03/02))`


-----

Next, set the window for primary space management. Use the following example of starting
the primary space management at 17:00, and running it for 4 hours:

`SETSYS PRIMARYSPACEMANAGEMENTSTART(1700 2100)`

Bandwidth for MIGRATION at a DFSMShsm level can be influenced by setting the number of
parallel tasks for MIGRATION, which also can be for INTERVAL MIGRATION. Example 9-1
shows how to set this value through the **SETSYS** command.

_Example 9-1  Specify the number of migration tasks_

    SETSYS MAXMIGRATIONTASKS(nn)
    SETSYS MAXINTERVALTASKS(nn)

Other **SETSYS** commands relate to primary space management and migration. In the following
example, the extent reduction value is set:

SETSYS MAXEXTENTS(12)

This value must be exceeded before _extent reduction_ occurs.

To understand how data sets are handled through primary (and secondary) space
management, see the management class examples (Example 9-2 and Example 9-3). The first example shows attributes that decide expiration (based on the creation
date or the last used date), release, and how many days a data set will stay on primary DASD.

_Example 9-2  Example of management class setting as related to expiration and primary days_

    LINE    MGMTCLAS EXPIRE   EXPIRE    RET  PARTIAL   PRIMARY
    OPERATOR  NAME   NON-USAGE DATE/DAYS  LIMIT  RELEASE   DAYS
    ---(1)---- --(2)--- ---(3)--- ---(4)---- --(5)-- ----(6)---- ---(7)--
    FKSMF   NOLIMIT   NOLIMIT NOLIMIT YES_IMMED      7
    MCOBTAPE    30     30    30 NO        ----
    MC365     365   NOLIMIT NOLIMIT YES_IMMED     10
    MC54NMIG  NOLIMIT   NOLIMIT NOLIMIT CONDITIONAL     0
    MC54PRIM  NOLIMIT   NOLIMIT NOLIMIT YES         3
    MC54WORK     7   NOLIMIT NOLIMIT YES         3
    MHLTST     365   NOLIMIT NOLIMIT CONDITIONAL    30
    TOGRP1T3     1      1    10 NO         0
    
Data sets can expire based on creation date (EXPIRE DATE/DAYS) or usage (EXPIRE
NON-USAGE) and are captured in primary space management based on this factor. For the
Partial Release attribute, the following values can occur:

- **YES:** Release unused space automatically during the space management
cycle.

- **CONDITIONAL:** Unused space can be released automatically only if a secondary
allocation exists for the data set.

- **YES IMMED:** Release unused space when a data set is closed or during the space
management cycle, whichever comes first.

- **COND IMMED:** Unused space for data sets with secondary allocation is released
when a data set is closed, or during the space management cycle,
whichever comes first.

Based on these attributes, primary space management will release the data sets that are
candidates.


-----

The last value that is shown in Example 9-2 is “Primary Days”, which decides
how many days a data set stays on primary DASD. This value can be from _0 days_ to
_permanently_ remaining on the primary disk.

Example 9-3 is an example of “level 1 days” and “generation data group (GDG) action”. The
number of days on ML1 is based on the management class value that is determined when
space management moves these data sets on to ML2. A value of 0 causes a migration from
ML0 to ML2 directly.

GDG cleanup is also part of primary space management. In Example 9-3, when this
parameter is set, only two GDG versions will exist. When a new GDG is created, the oldest is
rolled off the GDG base and in this case EXPIRED.

_Example 9-3  Example of management class setting as related to lifetime on ML1 and GDG handling_

    MGMTCLAS LEVEL 1 GDG ON ROLLED-OFF
    NAME   DAYS   PRIMARY  GDS ACTION
    --(2)--- --(8)-- --(10)-- ---(11)--FKSMF     60    --- -------
    MCOBTAPE -------    --- -------
    MC365     30     2 EXPIRE
    MC54NMIG  9999    --- -------
    MC54PRIM  9999    --- -------
    MC54WORK  9999    --- -------
    MHLTST    60    --- -------
    TOGRP1T3    0    --- -------

**Definitions that are needed on a storage group level**
In addition to the settings in the DFSMShsm PARMLIB, and the management class attributes,
you will need settings at the storage group level to activate primary space management.

On the Interactive System Productivity Facility (ISPF) Primary Option menu, choose **ISMF** .
Choose option **6** and enter the storage group name in which you want to activate primary
space management. The Auto Migrate option must have a value that differs from N (none) to
activate primary space management.

The possible settings are Y , N , I , or P . These settings apply to the following functions:

- **Y:** Data sets are eligible for primary space management migration. If
SETSYS INTERVALMIGRATION was specified in DFSMShsm, the
data sets are also eligible for interval migration. If on demand
migration is activated, it will replace interval migration.

- **N:** Data sets are not eligible for automatic migration.

- **I:** Data sets are eligible for primary space management and interval
migration.

- **P:** Data sets are eligible for primary space management but not interval migration.


-----

Example 9-4 shows storage group settings for automatic migration.

_Example 9-4  ISMF Storage Group LIST view_

    LINE    STORGRP VIO  AUTO   MIGRATE SYSTEM
    OPERATOR  NAME   UNIT MIGRATE  OR SYS GROUP
    ---(1)---- --(2)--- (5)- --(6)--- -----(7)-----      DB8XL  ---- NO    --------
    DB9A   ---- NO    --------
    DB9ADET ---- NO    --------
    DB9AML1 ---- INTERVAL --------
    FKSMF  ---- NO    --------
    SGDB20  ---- YES    SC65TS

You either set a value for Migrate System or leave this field empty. For workload balancing,
you can leave this field blank and all systems in the DFSMShsm plex share the primary space
management workload. DFSMShsm is aware across the HSMplex of volumes that were
already processed by another system. This approach is advantageous because you can take
out systems from the HSMplex and still process the workload.

The manual workload balancing of automatic migration is also possible by specifying the
system name on the individual storage group. Example 9-5 shows output from DFSMShsm
primary space management.

_Example 9-5  DFSMShsm log from primary space management processing_

    ARC0520I PRIMARY SPACE MANAGEMENT STARTING
    ARC0522I SPACE MANAGEMENT STARTING ON VOLUME 279
    ARC0522I (CONT.) MHL001 (SMS) AT 16:30:11 ON 2012/08/26, SYSTEM SC64
    
    ....... multiple volumes being processed .......
    
    ARC0421I MIGRATION VOLUME X11111 IS NOW MARKED FULL
    ARC0521I PRIMARY SPACE MANAGEMENT ENDED SUCCESSFULLY

**Using DSSXMMODE for migration**
For better performance and to offload the DFSMShsm main task for various workloads,
migration and other DFSMShsm auto tasks can run in cross-memory mode, starting an
individual task outside of DFSMShsm. For DFSMShsm MIGRATION, the task name is in this
form: ARC _x_ MIGR where _x_ indicates the host ID that started the cross-memory task. The
following **SETSYS** command shows how to activate cross-memory mode for MIGRATION:

    SETSYS DSSXMMODE(MIGRATION(Y) )


**Using ML1 overflow volumes**
An optional keyword on the DFSMShsm **ADDVOL** command to add ML1 volumes is **OVERFLOW** .
Overflow volumes can be used for both backup and migration.

If you specify SETSYS ML1OVERFLOW(DATASETSIZE( _dssize_ )), the ML1 OVERFLOW
volumes are reserved for the following migration types:

- Migration of a data set that exceeds the _dssize_ value.
- Migration of data sets to ML1 that cannot find sufficient space elsewhere on the ML1
volumes that are added as NOOVERFLOW.


-----

Overflow volumes are also used for the following functions:

- Inline backup
- **HBACKDS** and **BACKDS** commands
- ARCHBACK macro for data sets that are larger than _dssize_ K bytes

The following example shows adding ML1 volumes to DFSMShsm:

ADDVOL ML1003 UNIT(3390) MIGRATION(MIGRATIONLEVEL1 OVERFLOW)

The default for the **ADDVOL** command is NOOVERFLOW .

Previously, DFSMShsm did not migrate data sets that were larger than 64 K. Now, you can
migrate data sets that are larger than 64 K (large format sequential data sets) and use the
overflow volumes for migration and backup copies that are based on the installation-specific
setting.

The advantage of this configuration is that these data sets no longer must go directly to ML2.

### 9.2.2 Secondary space management

The automatic secondary space management (SSM) function manages the migration and
cleanup from DFSMShsm ML1 volumes. The automatic SSM function performs these
functions:

- Migrates data

- Schedules TAPECOPY commands for migration tape copy-needed (TCN) records

- Deletes expired data sets from the _migration volumes_

- Deletes obsolete migration control data sets (MCDs), volume statistics records (VSRs),
and data set readys (DSRs) during migration cleanup

- Moves data sets (based on the management class attribute for ML1 days) from ML1 to
ML2 volumes

The automatic SSM function determines whether to perform Level 1 to Level 2 migration by
checking to see whether any ML1 volume is filled equal to or greater than its threshold.
DFSMShsm migrates all eligible data sets from all ML1 volumes to ML 2 volumes.

**Note:** SSM can have up to 15 concurrent cleanup tasks, which can be a combination of
disk and tape; however, still only one task migrates data from ML1 to ML2.

The SSM window and tasks are controlled by the **SETSYS** command, as shown in
Example 9-6.

_Example 9-6  Example of commands that control SSM_

    SETSYS SECONDARYSPACEMANAGEMENTCYCLE (YYYYYYY) CYCLESTARTDATE(yyyy/mm/dd))
    SETSYS SECONDARYSPMGMTSTART(1400)
    SETSYS MAXSSMTASKS (CLEANUP(2) TAPEMOVEMENT(1))

The number of cleanup tasks can be 0 - 15. The default is 2 . Tape movement tasks can be
0 - 15, as well. The default for this parameter is 1 . If the number of tasks is set to 0 , the
function is not running.

If tape duplexing is used, two tapes are used for each task in this function.


-----

**Small-data-set packing**
_Small-data-set packing_ (SDSP) is a DFSMShsm function to reduce the space requirement
when you migrate smaller data sets to ML1. Instead of migrating as individual data sets, these
data sets, which are based on a decided maximum size limit, are moved inside a VSAM file
on ML1 volumes. These ML1 volumes are called _SDSP_ , and packed efficiently. A number of
SDSPs can exist so that DFSMShsm can go to a new SDSP when and SDSP fills up. The
following example shows how to set this limit:

`SETSYS SMALLDATASETPACKING(KB(110))`

For a usable SDSP on an ML1 volume, you must also add this volume with the
**SMALLDATASETPACKING** keyword. The manual _z/OS DFSMShsm Implementation and_
_Customization Guide,_ SC35-0418 _,_ explains in detail how to create an SDSP .

**DFSMS V1.13 performance improvements for SDSP**

The performance improvements for SDSP that were introduced in DFSMS V1.13 are
described.

**_ARCMMEXT overhead removal_**
The ARCMMEXT exit looks at already migrated data sets to determine whether they need to
be moved on to another device or migration level. When the ARCMMEXT exit scans the
SDSP , it evaluates all of the data sets in the SDSP . The exit serializes all of the data sets in
the SDSP even though they might not be candidates for migration.

Now, secondary space management executes an initial MCD scan to identify candidates to
move on to a migration queue (SMQE), which is processed by the ARCMMEXT exit later.
SSM obtains improved performance through this process.

**_Balanced SDSP selection algorithm_**
Another new tuning option affects the way that DFSMShsm selects the SDSPs. Today, the
target SDSP is selected based on an ML1 volume ADDVOL sequence, which means that the
same SDSPs are always selected first. These SDSPs are always filled, so they must be
reorganized and the other SDSPs are often idle.

DFSMS V1R13 introduces a change in this approach. A new SDSP selection algorithm
ensures that the SDSPs are used evenly. This algorithm is based on the SDSP free-space
status. Free space on the SDSP is calculated in two ways:

- During ADDVOL processing
- Every time that an SDSP is closed (at the completion of data processing)

We suggest that you allocate several SDSPs to DFSMShsm so that the small data sets that
are candidates for SDSP processing are distributed across multiple SDSPs. Also, SDSPs
now are deallocated if they are not in use.

**Note:** If you are in a multiple DFSMShsm host environment and you use the secondary
host promotion function, a secondary host can take over the unique functions that were
performed by a primary host that failed. Secondary host promotion also allows another
host to take over the secondary space management functions from either a primary or
secondary host that failed. You must issue the **SETSYS PROMOTE** command in advance to
enable any host to support secondary host promotion.


-----

**9.2.3 Event-driven space management**

_Event-driven migration_ performs space management functions similarly to primary space
management functions, except based on events instead of a schedule.

Two types of event-driven migrations are possible:

- Interval migration
- On demand migration

**Interval migration and space management**
Interval migration occurs at an hourly interval, if it is enabled. Candidate volumes are volumes
over the volume threshold. If AUTOMIGRATE(I) is specified, DFSMShsm checks whether the
allocation exceeded the midpoint between Migration High and Migration Low. If
AUTOMIGRATE(Y) is specified, the Migration High threshold must be exceeded for the
volume to be a candidate for migration. In either case, migration does not occur until
allocation on the volume falls under the Migration High threshold or no more candidates exist
on the volume for migration.

Interval migration tasks differ from automatic space management tasks. For interval
migration, the following tasks are completed:

- The following information about the SMS configuration is collected:

- The volumes that belong to the DFSMS storage groups
- The management classes that are identified
- Allocated space on all volumes

- Unmatched data sets are notified through a message.

- Temporary data sets are deleted.

- Migration on candidate data sets occurs.

Unlike primary space management, these functions are not part of an interval migration:

- Partial release
- Extent reduction

Parameter settings at the volume level and data set level further affect the eligibility
requirements on both of these functions.

**_How to enable interval migration_**
You can activate Interval migration either by a PARMLIB setting in the ARCCMDxx member or
by using the **SETSYS** **INTERVALMIGRATION** parameter. Defining interval migration in PARMLIB
requires you either to restart DFSMShsm or to execute the command.

A **QUERY SETSYS** command informs you whether interval migration is active. In addition to the
**SETSYS** activation, you also must set the eligibility requirements.

**_How to disable interval migration_**
The interval migration function is easily disabled. To disable it, issue the **SETSYS**
**NOINTERVALMIGRATION** command. To disable the function permanently, also remove the
definition in PARMLIB or change it to NOINTERVALMIGRATION.

**_Eligibility requirements for interval migration_**
The eligibility requirements for interval migration must be met on both the volume and the
data set levels.


-----

At the volume level, the following requirements must be met:

- The volume must be either SMS or non-SMS managed.

- The volume is in a storage group with a setting of AM= I (Interval Migration) or AM= Y
(Automigration YES):

- If the storage group is defined with AM= I , the allocation on the volume must exceed
the midpoint between the low and high thresholds to be an eligible candidate.

- If the storage group is defined with AM= Y , the volume must exceed the high threshold
to be a candidate.

- If AM= Y is defined on the storage group, on demand migration must not be activated
because on demand migration will be performed instead of interval migration.

- The SETSYS INTERVAL MIGRATION setting must be in effect.

At the data set level, the following requirements must be met:

- The management class must allow both _command_ and _auto migration_ .

- The Primary Days Non Usage value in the management class must be exceeded.

- If the data set needs a backup, it fails to be a candidate.

Considerations for interval migration are listed:

- Neither interval migration or on demand migration is performed on volumes in a storage
group with a definition of AM= P (Automigrate Primary) or AM= N (Automigrate NO).

- Interval migration occurs for volumes in a storage group with the attribute AM= I ,
regardless of whether interval migration is active.

- With on demand migration active, on demand migration occurs instead of interval
migration if the AM setting on the storage group is Y .

- Interval migration can be set up to require an operator’s approval before it starts. Set this
requirement by issuing a **SETSYS REQUEST** command.

- The **SETSYS MONITOR(SPACE)** command can be issued to enable notifications to the
operator about the space allocation on volumes during interval migration.

- The order in which data sets are migrated is based on size. The largest data set is
migrated first, followed by the next largest data set.

### 9.2.4 DFSMShsm on demand migration

DFSMS V1.13 introduced an alternative for interval migration that is called _on demand_
_migration_ . The current interval migration runs at the top of every hour and spends many
resources to create an updated space check of every volume before it starts the space
management processing of all volumes over the high threshold.

The new on demand migration takes another approach by running continuously. Activating on
demand migration enables a new DFSMShsm listener that waits for a new DFSMS signal.
This signal is issued whenever an individual volume exceeds its high threshold due to new
allocations and extents. Whenever an individual volume exceeds its high threshold, on
demand migration is notified to process these volumes immediately, instead of waiting for the
top of the hour, as in interval migration.

**Note:** Interval migration is still available, and it is the default option for DFSMShsm.


-----

The ENF notification occurs through an ENF72 from DFSMS every time the volume high
threshold is exceeded. The notification is sent to all of the members in the SMSplex that
share the actual COMMDS, through which the communication occurs. Even though all
systems are notified, only one system will perform space management. Space management
on individual volumes continues until the low threshold is reached or no more data sets need
to be migrated, deleted, or expired.

To avoid too many notifications, DFSMS tracks the previous trigger value and issues a new
notification on the same volume only if it grows 25% of capacity above the threshold. This
notification occurs four times: at 25%, 50%, 75%, and 100%. If the growth is more than 60%
of the high threshold capacity, the first two notifications are skipped, but the next notification is
issued at 75%. If the capacity reaches a critical value (97% that is used), an ENF notification
will be issued on all volume checks. At 100% usage of the volume, another notification is
issued.

For extended address volumes (EAVs), both high thresholds (track-managed and
cylinder-managed) are monitored. If either high threshold is exceeded, an ENF notification is
issued to activate on demand migration for this volume.

**Activating on demand migration**
To activate on demand migration, you use a new **SETSYS** command: **ONDEMANDMIGRATION**
**(Y|N)** . When set to Y , on demand migration replaces interval migration for all
DFSMS-managed volumes. On demand migration performs the space management based
on ENF notifications from volumes that exceed thresholds. DFSMS limits this space
management to volumes in storage groups with an AM setting of Y . Interval migration no
longer runs on DFSMS-managed volumes in these storage groups hourly. Interval migration
runs only on non-SMS managed volumes and on volumes in storage groups with an AM
setting of I . Consider storage groups with an AM value of I for migration to on demand
migration.

By setting the **ONDEMANDMIGRATION** command to N , interval migration acts as it did before
DFSMS V1.13. ONDEMANDMIGRATION (N) is the default. The abbreviation for
ONDEMANDMIGRATION is ODM.

**Note:** On demand migration is valid for DFSMS-managed volumes only.

You can set a new notification limit to alert the number of volumes that is selected for on
demand migration. If this limit is exceeded, a message is displayed. This example shows the
notification that is based on ODMNOTIFICATIONLIMIT:

ARC1901E number of volumes eligible for on-demand migration has reached nnnn.

The limit is set by the new **SETSYS** **ODMNOTIFICATIONLIMIT(limit)** command. The default value
for this number is 100 . Issuing the **QUERY SETSYS** command also displays this value.

DFSMShsm performs the space management of a volume in two phases. The first phase
tries to bring the volume under the threshold without moving data. The second phase (if
needed) tries to get the volume under the threshold by moving around data.

When you migrate to ONDEMANDMIGRATION, consider your current setting of
MAXMIGRATIONTASKS to see whether ONDEMANDMIGRATION might a conflict with
potential parallel tasks that are initiated by the interval migration. The
SETSYS(REQUEST/NOREQUEST) setting is not valid for ONDEMANDMIGRATION
because the operator acceptance is not needed as it might be for interval migration.


-----

ONDEMANDMIGRATION is regarded as part of AUTOMIGRATION. Therefore, a HOLD
AUTOMIGRATION also puts a hold on ONDEMANDMIGRATION.

**Compatibility and coexistence**
When you migrate to DFSMS V1.13, consider the implementation of on demand migration. In
a mixed environment, the prior level systems continue their previous approach with interval
migration, and the on demand migration support performs space management on the
volumes that are outside of the hourly window that is used by interval migration.

The messages that relate to the new function are updated to reflect the status of on demand
migration.

### 9.2.5 Storage group processing priority for space management functions

The DFSMS storage group application in Interactive Storage Management Facility (ISMF)
offers a processing priority value option. The priority is used by DFSMShsm space
management functions and autobackup for selecting the order in which to process the
storage groups to meet individual client requirements. If the processing priority value is not
set, the default is a priority of 50 and DFSMShsm processes the storage groups in normal
processing order. See the ISMF storage group application with the priority setting in
Figure 9-1.

    POOL STORAGE GROUP DISPLAY         Page 2 of 2
    Command ===>
    
    CDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : BIGSTUFF
    
    Dump Class . . . . . . . . . . . . . . . :
    Dump Class . . . . . . . . . . . . . . . :
    Dump Class . . . . . . . . . . . . . . . :
    Allocation/Migration Threshold - High . : 99
    Low . . : 1
    Alloc/Migr Threshold Track-Managed - High: 99
    Low : 1
    Guaranteed Backup Frequency . . . . . . :
    BreakPointValue . . . . . . . . . . . . :
    **Processing Priority . . . . . . . . . . :** **50**

_Figure 9-1  ISMF Storage Application display for processing priority_

If individual priorities are set (values 1 -  100 ), DFSMShsm then processes the space
management functions based on priority from highest to lowest.

This approach can help ensure that more important storage groups are processed before less
important storage groups during interval migration, on demand migration, or primary space
management.

An example of how to use a prioritized processing order is in a class transition environment,
where you want to offload certain storage groups before they become populated from class
transition data movement.


-----

### 9.2.6 Automatic dynamic volume expansion space management

A control unit, such as the DS8000, supports dynamic volume expansion (DVE). With DVE,
you can schedule an automatic reformat of the volume. DFSMShsm senses this scheduled
automatic reformat and updates the volume information for future use.

**Note:** This automatic DVE occurs only within the same sysplex. If any DFSMShsm that is
outside of the sysplex manages the same volumes, this information is not updated until this
DFSMShsm is restarted.

### 9.2.7 Command space management

With automatic space management, you can also execute the individual functions through
commands.

**_DELETE command_**
Deletes can occur in multiple ways:

- DFSMShsm **DELETE** command
- Access method services (AMS) **DELETE** command
- TSO **DELETE** command
- IEHPROGM utility

Deleting a DFSMS-managed migrated data set cleans up the migration records in the CDS
without recalling the data set. Any discrete profile in RACF that relates to this data set is
cleaned up by RACF .

**Note:** The deletion of a DFSMS-managed data set does not include DFSMShsm backups.
The DFSMShsm backups are retained.

**_Deleting data sets that are not expired_**
If the data set has an expiration date and the date is not reached, DFSMShsm deletes the
data set only if the **PURGE** parameter of the **DELETE** command is specified. If a data set does
not have an expiration date, the **PURGE** parameter does not apply, and DFSMShsm deletes the
data set.

When users delete data sets with DFSMShsm, they can now bypass unnecessary recalls for
|PGM=IEFBR14 job steps by updating the ALLOCxx PARMLIB member to include a new
statement SYSTEM IEFBR14_DELMIGDS(NORECALL), or by using the new **SETALLOC**
command.

**_Migrating a specific data set_**
A data set can be command-migrated by using the **MIGRATE** or **HMIGRATE** commands. Migration
can occur from a primary volume or an ML1 volume. The values in the management class
setting for the number of days on a primary or a level 1 volume will be overwritten by a
command migration. To migrate a data set by command, the management class must have
migration set to either COMMAND or BOTH. See Example 9-7.

_Example 9-7  Command migration of individual data set_

    MIGRATE DATASETNAME(SAMPLE.DATASET)
    MIGRATE DATASETNAME(SAMPLE.DATASET1) MIGRATIONLEVEL2


-----

**_Migrating data sets by using wildcards_**
Command-migrating data to DFSMShsm level 1 or level 2 can also be invoked through a
wildcard specification instead of providing a fully qualified data set name.

If you specify **HMIGRATE **.DATA** , DFSMShsm migrates the USERID.HLQ.DATA and
USERID.DATA data sets but not USERID.HLQ.LIST.

For easier data set migrations, DFSMShsm bypasses already-migrated data sets when you
specify the **HMIGRATE** command with a wildcard filter ( ***** ). In releases before DFSMS V1.13,
HMIGRATE processing resulted in numerous error messages if your data set name
specification included migrated data sets.

**_Migrating a volume_**
The **MIGRATE(VOLUME)** command invokes migration at the volume level. Only the data sets with
a management class value of BOTH in COMMAND OR AUTO MIGRATE are included.

The following functions are executed:

- Migration occurs from ML0 to ML1 or ML2 only.
- The volume is considered managed by DFSMShsm.
- A list of management classes is obtained.

**Recalling DFSMS-managed data sets by command**
Recall can occur by specifying RECALL or HRECALL with the specific data set name as a
parameter. This process is the same as the automatic recall process. The volume selection
routine for this specific type of recall is the same as the volume selection routine for automatic
recalls. See 9.2.9, “Automatic recall” on page 226 .

The **FORCENONSMS** parameter can be used to direct a recall to a specific non-SMS-managed
volume.

Recalls of extended format data sets are implemented differently for VSAM and sequential
data sets. For VSAM extended format, data set recall can occur only to the same data set
type. For sequential extended format data sets, recall can occur to an extended format data
set or to a non-extended format data set.

If the **FORCENONSMS** parameter is used to RECOVER an extended format data set to a
_non_ -SMS-managed volume, it is recovered as a NON EXTENDED format data set. Extended
format data sets can exist only on DFSMS-managed volumes.

Recalling a large data set might cause a problem. If so, use RECALL
DFDSSOPTION(VOLCOUNT(N( _nn_ ) | ANY)) to raise the number of candidate volumes.

**_Recalling direct access data sets_**
When you recall basic direct access method (BDAM) data sets to non-SMS-managed
volumes, you can use the DAOPTION to specify the device type and how access occurs
(relative track or block). For example, the use of RELBLK allows the data set to be recalled to
a device type with a smaller track size than the previous primary volume. DAOPTION is used
for non-SMS-managed volumes only.


-----

### 9.2.8 Fast Subsequent Migration optional feature

Fast Subsequent Migration (FSM) is an optional feature that is available in DFSMShsm.
When FSM is enabled, it optimizes the migration from the primary disk to ML2. Without FSM
enabled, a recall from ML2 to the primary volume invalidates the CDS record that points to
the ML2 volume. If the data set is then later migrated back to ML2, migration occurs to a
different ML2 tape volume. With FSM enabled, the ML2 recalls and later migration of the
same data set to ML2 will change for those data sets that are not being updated during the
time they resided on the primary volume. At the time of the migration of these data sets, a
reconnection can be made to the previous CDS record that points to the copy on ML2 from
where the recall occurred earlier. This capability is not possible if this tape was recycled or if
the migration cleanup removed the CDS record.

The FSM function optimizes migration to ML2 processing for the following reasons:

- It reduces data movement and tape mounts that are caused by migrating a data set again
that is unchanged.

- Only CDS records are updated.

- It reduces the need for recycling ML2 tape.

**_Enabling Fast Subsequent Migration_**
FSM can be enabled in two ways:

- Add the **SETSYS TAPEMIGRATION** command with the **RECONNECT(ALL)** or
**RECONNECT(ML2DIRECTEDONLY)** parameter to the ARCCMDxx PARMLIB member of
SYS1.PARMLIB. Then, restart DFSMShsm.

- Issue the **SETSYS TAPEMIGRATION** command with the **RECONNECT(ALL)** or
**RECONNECT(ML2DIRECTEDONLY)** parameter after DFSMShsm is started.

**_How to disable FSM_**
You can disable the FSM function by issuing the **SETSYS TAPEMIGRATION RECONNECT(NONE)**
command or by adding the same command in the ARCCMDxx member in SYS1.PARMLIB.
DFSMShsm must be restarted.

Exceptions where FSM is valid:

- HFS files (see APAR - OA22497) and VSAM files with an alternate index or path is not
supported.

- Data set was updated since it was recalled.

- The data set occupies multiple ML2 tape volumes.

- The **MIGRATE** command was issued with the **CONVERT** parameter.

- The ARCHMIG macro is used with the **FORCML1=YES** parameter to direct the data set to an
ML1 volume.

- The **SETSYS TAPEMIGRATION** command with the RECONNECT(NONE) setting is in effect.

The following conditions exist for FSM at recall time:

- The data set is recalled from a single ML2 tape volume.

- The SETSYS USERDATASETSERIALIZATION setting must be in effect.

- The **SETSYS TAPEMIGRATION** command with the RECONNECT(ALL) or
RECONNECT(ML2DIRECTEDONLY) was issued.


-----

Migration cleanup is managed through the **SETSYS** **MIGRATIONCLEANUPDAYS** command. This
setting can affect the data sets that benefit from FSM. The FSM pointers are cleaned up on a
calculated remigration date plus the value in RECONNECTDAYS (either a specified value or
the default). For more details about the calculation, see the _z/OS V1R13.0 DFSMShsm_
_Storage Administration Guide_ , SC26-0421. Specify a RECONNECTDAYS value that provides
the optimum use of FSM in your environment. The following example shows the
**MIGRATIONCLEANUPDAYS** command:

    MIGRATIONCLEANUPDAYS( _recalldays_ _statdays_ _reconnectdays_ )

FSM no longer relies on the change bit in the format 1 data set control block (DSCB) to
determine the eligibility to reconnect. Now, FSM looks into two new bits in the format 1 DSCB:
one bit to determine that the data set was recalled and another bit to determine whether the
data set was opened for output since being recalled. The two bits are DS1IND08 and
DS1RECAL.

### 9.2.9 Automatic recall

Automatic recall is explained.

**Volume selection for recalls**
When automatic recall is invoked through a reference from a batch job, a TSO user, or a
program, DFSMS passes the following information to the automatic class selection (ACS)
routines:

- The name of the data set.

- The name of the storage class that is associated with the data set when the data set was
migrated. If the data set was not SMS-managed when it was migrated, no storage class is
passed.

- The name of the management class that is associated with the data set when the data set
was migrated. If the data set used the default management class or was not
SMS-managed when it was migrated, no management class is passed.

- The name of the data class that is associated with the data set when the data set was
migrated. If the data set did not use a data class or was not SMS-managed when it was
migrated, no data class is passed.

- The volume serial number and unit type of the volume where the data set originally
resided or of the volume that is specified by the request.

- An environment indicator of RECALL.

- The RACF user ID and group ID for non-DFSMShsm authorized users.

- The data set size in KB.

- The data set expiration date.

- A generation data group (GDG) indication.

- The data set organization.

- The record organization for VSAM data sets.

If DFSMS is active, Data Facility Product (DFP) will return a storage class and manage the
allocation. The volume will be assigned by DFSMSdfp selection and not the volume that
DFSMShsm bypassed. If DFSMS is not installed, or volume selection was unable to select a
volume, the allocation is treated as a non-SMS allocation.


-----

**Bandwidth in recall**
Set the requirements that relate to recall for your environment. The requirements can depend
on factors, such as the volume and the number of concurrently available tape drives. See
Example 9-8.

_Example 9-8  Example of setting the number of parallel recall tasks_

    SETSYS MAXRECALLTASKS(nn)
    SETSYS TAPEMAXRECALLTASKS(nn)

This value can be set on each DFSMShsm task in the sysplex. Specify a number 0 -  15 for this
value.

**Using the ARCCATGP group**
Issuing the **UNCATALOG** , **RECATALOG** , or **DELETE NOSCRATCH** command against a migrated data set
causes the data set to be recalled before the operation is performed. It is possible to
authorize certain users to issue these commands without recalling the migrated data sets by
connecting the user to the RACF group ARCCATGP . When a user is logged on under the
RACF group ARCCATGP , DFSMShsm bypasses the automatic recall for UNCATALOG,
RECATALOG, and DELETE NOSCRATCH requests for migrated data sets.

The following tasks are used to enable DFSMShsm to bypass automatic recall during catalog
operations:

- Define the RACF group ARCCATGP by using the RACF **ADDGROUP** command
(ARCCATGP).

- Connect users who need to perform catalog operations without automatic recall to
ARCCATGP by using the following RACF command: **CONNECT (** **_userid1_** **,. . .,** **_useridn_** **)**
**GROUP(ARCCATGP) AUTHORITY(USE)** .

- Each user who needs to perform catalog operations without automatic recall must log on
to TSO and specify the **GROUP(ARCCATGP)** parameter on the TSO logon window or the
GROUP=ARCCATGP parameter on the JOB statement of a batch job. See Example 9-9.

_Example 9-9  Jobcard pointing to ARCCATGP group_

    //HSMCAT JOB(accounting information),'ARCCATGP Example',
    // USER=ITSOHSM,GROUP=ARCCATGP,PASSWORD=password
    //STEP1 EXEC PGM=....

### 9.2.10 Common recall queue

The common recall queue is described.

**Advantages of using the common recall queue**
Activating the _common recall queue_ (CRQ) feature in DFSMShsm provides an optimized and
flexible way to queue and process recalls. The use of CRQ primarily results in putting recalls
on a shared queue across the sysplex, which means that all logical partitions (LPARs) can
share the recall workload and therefore increase the recall bandwidth.

Additionally, CRQ enables all LPARs to schedule recalls to devices that they cannot access
(both disk and tape). For example, if a data set is migrated to ML2, recall can occur from an
LPAR without access to a tape device by processing the actual recall by another LPAR in the
CRQplex.


-----

CRQ processing of the individual recalls is based on priority across the sysplex instead of by
taking the LPAR approach. Also, when CRQ is enabled, DFSMShsm recalls all active
requests that are scheduled against the same ML2 tape volume, which increases the
effectiveness of the recall process by mounting fewer tape volumes.

CRQ removes affinity to a single system in the sysplex in case the system is down for
maintenance or other reasons.

**Preparing for common recall queue**
CRQ needs a list structure in the coupling facility to be able to share information across the
sysplex. This structure must be defined to the coupling facility resource management (CFRM)
by the z/OS systems programmer. Before you create this structure, you must estimate the
number of recalls, which determines the size that is set in the list structure allocation. Use the
coupling facility (CF) sizer to calculate this size. The CF sizer is available at this link:

[http://www.ibm.com/systems/support/z](http://www.ibm.com/systems/support/z)

The size of the CRQ structure depends on the highest number of projected concurrent recall
requests that is expected to be on the queue at any point in time.

The CF sizer recommends that you configure the CRQ to hold no fewer than 3,900 recalls.
For the default, the CF sizer configures a size that will be able to hold 1,700 concurrent recalls
on the CRQ.

-----

DFSMShsm knows about filling the CRQ and stops adding recall requests to the CRQ when
the queue reaches 95% of its capacity. Rerouting to the CRQ starts again when the percent
full falls under 85%. In the meantime, all scheduled recall requests are routed to the local
queue.

If needed, the structure size can be altered dynamically with the standard nondisruptive CF
function through the SETXCF **START,ALTER** command.

**Enabling and using the common recall queue**
You enable the CRQ function in DFSMShsm by adding the **SETSYS**
**COMMONQUEUE(RECALL(CONNECT(** **_basename_** **)))** command to the ARCCMDxx member in
SYS1.PARMLIB. Next, you must restart DFSMShsm or issue the **SETSYS** command to the
DFSMShsm task. The _basename_ is a selection of your own. It must be five characters in
length. It must be referred to by all of the DHSM tasks that share this CRQ.

By executing the previous command, DFSMShsm connects to the structure and starts to use
CRQ. If DFSMShsm is already active and recalls are on the local queue, these recalls are
transferred to the CRQ shared queue.

The CRQ eligibility requirements must be met to pass recalls to the CRQ and process them
from the CRQ. The main eligibility requirements are that DFSMShsm functions are not held,
the CRQ is not over 95% used, and at least one host can process the CRQ. Read more about
eligibility requirements in the _DFSMShsm Storage Administration Guide_ , SC26-0421.

**Note:** A CRQ is limited to work in one HSMplex and it cannot be shared across multiple
HSMplexes. The CRQ is associated with the HSMplex that created it. DFSMShsm cannot
connect to a CRQ that is not associated.

Recall requests on the CRQ are processed in priority sequence. The priority is based on the
wait type request with the highest priority. Requests with the same priority are additionally
prioritized in first-in first-out sequence.

The host that starts the recall request invokes ARCRPEXT (if enabled). The processing host
calls ARCRDEXT (if enabled). ARCRPEXT can be used in the individual DFSMShsm tasks to
assign a priority other than the default ( 50 ) and process those tasks.

CRQ requests are processsed based on recall tasks available on individual DFSMShsm
hosts, and recall prioritization. If requests are on the local queue, local queue requests are
prioritized higher than the CRQ requests with the same priority.

When any host selects an ML2 recall request, any other ML2 requests to the same tape
volume are also processed. All of these recalls are prioritized. After a wait period, the ML2
tape that is used is unmounted if no more requests to this tape appear on the queue.

Use the **TAPERECALLLIMITS** command if you need to tune this process. This command sets the
limit for the amount of time that DFHSM holds the tape:

    TAPERECALLLIMITS(TASK( _time_ ) TAPE( _time_ ))

The time values for the subparameters **TASK** and **TAPE** specify in minutes how long a recall
request can continue ( **TASK** ) and how long a tape can be allocated ( **TAPE** ) before DFSMShsm
looks for higher priority recalls. If higher priority recalls exist, a tape takeaway occurs.


-----

When the recall processing stops for a specific recall task on a DFSMShsm host, all queued
recall requests on that host that require the unmounted tape volume are excluded from
consideration for five minutes. This period allows other hosts to try their delayed requests
again for the unmounted tape volume.

Read more about the **TAPERECALLLIMITS** command in _DFSMShsm Storage Administration_
_Reference,_ SC26-0422 _._

Two types of messages that relate to the individual recall are sent back to the originating host:
the notification to TSO and the message that posts the wait request. All other messages go to
the processing host.

Requests must be canceled before they become active on the queue. The cancellation must
occur on the originating host. In DFSMS V1.13, the **QUERY** command provides both the
originating host and the processing host in the query output.

The CRQ status can be monitored through the **QUERY** command to the CRQ function. See
Example 9-10.

_Example 9-10  Example of output from a query on CRQ_

    QUERY COMMONQUEUE(RECALL)
    ARC1545I COMMON QUEUE STRUCTURE FULLNESS: COMMON
    ARC1545I (CONT.) RECALL QUEUE:STRUCTURE ENTRIES=000% FULL, STRUCTURE
    ARC1545I (CONT.) ELEMENTS=000% FULL

**Disconnecting from the CRQ**
To disconnect from the CRQ feature, you must issue a **SETSYS** command or change the
DFSMShsm PARMLIB with the following **SETSYS** command:

    SETSYS COMMONQUEUE(RECALL(DISCONNECT))

Disconnecting from the CRQ function sends all upcoming recall requests in this DFSMShsm
task to the local queue. These recall requests are processed in the same manner that they
were processed before the CRQ was activated. DFSMShsm will process active CRQ
requests that originated from other hosts, but new requests will no longer be picked up from
the CRQ.

Disconnection from the CRQ structure occurs during DFSMShsm shutdown, so no action
must be taken that relates to CRQ then.

**Handling common recall queue failure**
If the failing host is processing a request from ML2 tape, all requests from that tape cannot be
selected until the failing host is restarted. The tape is marked as “in-use” by the failing host,
and that indication is not reset until the failing host restarts. If the failing host is not restarted
for a short time, enter the **LIST HOST(** **_hostid_** **) RESET** command, where **_hostid_** is the
DFSMShsm host ID of the host that failed.

Example 9-11 shows a list and reset by using this command.

_Example 9-11  Example of how to reset a serialization from a DFSMShsm host_

    LIST HOST( _hostid_ )
    HOST( _hostid_ ) RESET


-----

You can issue the **LIST** **HOST** command to receive information about SMS-managed volumes
with MCV records that are serialized with a host ID. For _hostid_ , substitute the identification
character of the DFSMShsm host whose serialization information you want listed or reset.

**Recall by FBID**
APAR OA24053 introduced the new **SETSYS TAPEDATASETORDER** command in DFSMShsm. The
new command provides additional flexibility to handle situations where ascending FBID order
offers a significant reduction in tape repositioning, such as when many data sets are recalled
or recovered from a single tape. You can specify the **SETSYS** command with the general
**TAPEDATASETORDER(PRIORITY | FBID)** parameter and the command will apply to both recall
and recover or you can specify a specific functional parameter.

The following examples demonstrate how to issue the command for a specific function:

- SETSYS TAPEDATASETORDER(RECALL(PRIORITY | FBID))
- SETSYS TAPEDATASETORDER(RECOVER(PRIORITY | FBID))

**Note:** If the **ALTERPRI** command is issued to reprioritize requests that reside on the Recall
or Recover queue, the **ALTERPRI** command takes priority over FBID. If you are enabling the
new function in a CRQ environment, it is recommended that you apply all PTFs to all
systems that connect to the CRQ.

**Extended TTOC**
When the tape table of contents (TTOC) reaches the limit of 330,000 data sets, future
attempts to migrate or back up a data set to that volume will result in spanning to a new
migration or backup tape even though the tape still has available capacity.

**EXTENDEDTTOC(Y|N)** is an optional parameter that specifies whether DFSMShsm uses
extended TTOCs to use tapes with higher capacity volumes in your installation more
efficiently. Extended TTOCs allow DFSMShsm to write 1,060,000 data sets (potentially) to a
migration tape or backup tape.

To use extended TTOCs in your installation, you must define your offline control data set
(OCDS) with a maximum record size of 6,144 bytes, as described in the _zDFSMShsm_
_Implementation and Customization Guide_ , SC35-0418. Then, enable the support by entering
the **SETSYS** command with the optional parameter **EXTENDEDTTOC(Y)** or its shortened form,
**EXTTC(Y)** .

**Note:** If you attempt to specify the **SETSYS EXTENDEDTTOC(Y)** command and the OCDS was
not defined with a record size of 6,144, DFSMShsm issues an error message and forces
the value of **EXTENDEDTTOC** to N .

To return to using non-extended TTOCs, enter the **SETSYS** command with **EXTENDEDTTOC(N)** or
its shortened form, **EXTTC(N)** .

Users must not specify the **SETSYS EXTENDEDTTOC(Y)** command on any host in an HSMplex
until the OCDS is redefined with a record length of 6,144 bytes and all hosts in an HSMplex
are prepared to issue the **SETSYS EXTENDEDTTOC(Y)** command.

The default value is **EXTENDEDTTOC(N)** .


-----
## 9.3 Space management of non-SMS-managed volumes

Space management for non-SMS-managed storage occurs at the volume level. These
volumes are known as _primary volumes_ . Management occurs at the same level as for
DFSMS-managed data, but you need to use command settings to specify the space
management rules for the individual volumes and associated data sets.

The following functions are performed in space management for non-SMS-managed
volumes:

- Automatic space management
- Interval migration
- Automatic recall
- Command space management

The differences between processing for SMS-managed and non-SMS-managed data sets are
listed:

- Two additional space management functions are available for non-SMS-managed
volumes:

  - Deletion (delete by age)
  - Retirement (delete if backed up)

- Deletion of list and utility data sets.

- Level 1 to level 2 migration is controlled on a processing-unit-wide basis rather than on a
data set management class basis.

- Target volumes for recall are selected by DFSMShsm rather than by DFSMSdfp.

- Volumes are identified individually to DFSMShsm by the use of the **ADDVOL** command.

### 9.3.1 Primary space management

During primary space management on non-SMS managed volumes, DFSMShsm deletes
LIST, UTILITY, and expired data sets on the volumes. Partial release is not performed. Only
extent reduction is performed.

Fast Subsequent Migration is also supported for non-SMS managed data.

For non-SMS managed data be part of space management, ensure that the following tasks
were completed:

- Specify the age for deleting LIST and UTILITY data sets.

- Specify the minimum age for data sets to migrate if no age is specified when the volume
was defined to DFSMShsm.

- Specify when data sets become eligible for level 1 to level 2 migration.

- Specify the recall characteristics for primary volumes.

- Define the primary volumes to DFSMShsm.

- Define the pools of volumes.


-----

**LIST data set settings**
LIST data sets are data sets with the following suffixes:

- LIST
- LINKLIST
- OUTLIST

To clean up these data sets, SETSYS SCRATCHFREQUENCY( _n_ ) must be defined in the
DFSMShsm PARMLIB. The value of _n_ in parentheses determines how many days to retain
these data sets. A SCRATCHFREQUENCY(999) setting ensures that these data sets are
kept for 999 days.

**Specifying a minimum age for migration**
To decide when to migrate data off the primary volumes, the **SETSYS DAYS(** **_n_** **)** setting must be
in effect. The value in the **DAYS(** **_n_** **)** parameter determines the minimum number of days on a
primary disk.

This value is an overall setting that is used if no setting was set on the **MIGRATE** command or
on the individual **ADDVOL** of a primary volume.

**Migrating data from ML1 to ML2**
Migration from ML1 to ML2 in a non-SMS-managed environment is the same for all data, and
it is determined by the SETSYS MIGRATIONLEVEL1DAYS( _n_ ) setting. This value sets the
minimum number of days that data sets resides on an ML1 volume.

**Note:** You must also specify the threshold on the ML1 **ADDVOL** command for DFSMShsm to
automatically migrate data from an ML1 volume to an ML2 volume.

**Recall in a non-SMS-managed environment**
In a non-SMS environment, volume selection is performed by DFSMShsm on recall. The
**SETSYS RECALL** command specifies the general volume destination to use for recalls. The
following command shows the DFSMShsm setting for the target volume for recalls:

    SETSYS RECALL(ANYSTORAGEVOLUME(LIKE))

Setting this parameter enables recalls to go to storage-mounted volumes. Another parameter,
**PRIVATEVOLUME** , enables recalls to PUBLIC, PRIVATE, or STORAGE-mounted volumes if
these volumes were added by an **ADDVOL** command with the **AUTORECALL** parameter. The use
of the **LIKE** parameter causes DFSMShsm to only use a volume with the same attributes as
the volume from which this data set was migrated. The **UNLIKE** parameter causes DFSMShsm
to choose a volume with different attributes.

**Defining primary volumes to DFSMShsm**
The automatic primary space management function occurs in the DFSMShsm host. First, this
volume must be in the list.

You can limit processing to only occur in one host by specifying **AUTOMIGRATION** on the **ADDVOL**
command in the selected host. You can also specify **NOAUTOMIGRATION** on the other
DFSMShsm hosts in the HSMplex.


-----

When you set the ADDVOL of a primary volume to DFSMShsm, consider the following
characteristics:

- Do you want automatic space management to run on this volume?
- Do you want this volume to be a candidate for receiving recalls?
- Which space management technique do you want to use?

The potential settings are shown for a sample ADDVOL setting for a primary volume:

ADDVOL vvvvvv UNIT(3390) PRIMARY(AUTOMIGRATION AUTORECALL MIGRATE(12)) THRESHOLD(95 80)

You must add the volumes that you want managed one by one. You must specify the **UNIT**
parameter and the volume type (Primary or Migration Level 1). AUTOMIGRATION specifies
that DFSMShsm is expected to perform space management on this volume. AUTORECALL
allows recalls to target this volume. DELETEBYAGE, DELETEIFBACKEDUP , or MIGRATE
specifies the management technique for this volume. In the previous example, data sets will
be migrated after a minimum of 12 days on this primary volume.

When you specify DELETEBYAGE( _nnn_ ) instead, DFSMShsm deletes the data sets without
the EXPIRATION DATE set when the data sets are not opened for a number of days that are
greater than the value that is specified in the **DELETEBYAGE** parameter. You can specify up to
999 for this value.

**DELETEIFBACKEDUP(** **_nnn_** **)** is another parameter. DFSMShsm deletes data sets with a valid
backup and that were not opened in the number of specified days. You can specify up to 999
for this value.

**Attributes on primary volumes**
In total, you can specify the following attributes to a primary volume:

- AUTOMIGRATION or NOAUTOMIGRATION
- AUTORECALL or NOAUTORECALL
- AUTOBACKUP or NOAUTOBACKUP
- BACKUPDEVICECATEGORY
- AUTODUMP or NOAUTODUMP

**Defining pools for non-SMS-managed data sets**
You can organize your data in non-SMS-managed pools. You can set up these pools to
manage data set names that were defined by you or to receive data sets that were migrated
away from these pools earlier.

To define a pool, use the **DEFINE** **POOL** command and specify one or more volumes to this pool
and at the same time give this pool an ID ( _name_ ). Add a statement that is similar to the
following statement to define a pool and provide an ID:

    DEFINE POOL(GROUPA VOLUMES(V11111 V22222 V33333))

A volume pool can be defined as shown:

    DEFINE VOLUMEPOOL (VPOOL1 VOLUMES(V44444 V55555 V66666))

Keeping data sets in a volume pool will ensure that data sets that are migrated off this volume
pool will be associated with all volumes in the pool so that these volumes later can be target
volumes for a recall of these data sets.


-----

You can specify non-resident volumes to the pool to ensure the recall of data sets that were
on these primary volumes. Through defining these volumes, the entire pool is associated with
the data sets during recall. You can define SMS-managed volumes to the volume pool, but
they will not be candidates for the non-SMS data sets that are recalled to this pool. To change
the volumes that belong to a volume pool, define the volumes that are already in the pool and
add new volumes to the list based on your requirements. A volume pool can hold 140
volumes in total.

One restriction is that all volumes in a pool must have the same geometry (3390, for
example). Volumes with different sizes (3390-3 and 3390-9) are allowed to go into the same
pool.

A primary volume can appear in multiple pools and it can be added with the **AUTORECALL**
parameter on the **ADDVOL** command.

In a multihost DFSMShsm environment, we recommend the following settings:

- A data set or volume pool must be the same in all DFSMShsm hosts.

- Volumes that are available for general recall must be accessible to all DFSMShsm hosts.

- A volume that is removed from the environment must be removed from DFSMShsm by
using the **DELVOL** command on _all_ DFSMShsm hosts, and from any **ADDVOL** commands that
might be in the startup member. This volume can remain in the pool definition. In this way,
DFSMShsm can translate a recall request to a currently active volume in the pool.

**Note:** To define pools in a JES3 environment, see the _z/OS V1R13 DFSMShsm Storage_
_Administration Guide_ , SC35-0421.

### 9.3.2 Automatic secondary space management

Migration cleanup and Level 1 to Level 2 migration for non-SMS-managed data sets work the
same way as for SMS-managed data sets.

During migration cleanup for non-SMS-managed data sets, DFSMShsm uses the **SETSYS**
parameter **EXPIREDDATASETS(SCRATCH)** to determine whether to delete expired data sets. The
**STATDAYS** and **RECALLDAYS** subparameters of the SETSYS MIGRATIONLEVEL1DAYS setting
influence when to delete statistics records and MCD records for recalled data sets. The
SETSYS MIGRATIONLEVEL1DAYS setting determines when to migrate from Ml1 to ML2.

### 9.3.3 Automatic primary space management

For automatic primary space management, DFSMShsm processes all primary volumes with
the AUTOMIGRATION attribute set. Whether to delete, retire, or migrate a primary volume is
determined by the specification on the **ADDVOL** command.

The space management function for non-SMS-managed data occurs in two passes.

**Pass one**
In pass one, DFSMShsm investigates all data sets on the volumes, independently of filling the
volume. A check of the integrity age ensures that the data set can be processed (for
environments with USERDATASETSERIALIZATION set, integrity age is 0 ). The inactive age
(number of days unreferenced) must be larger than the integrity age for the data set to be a
candidate for migration.


-----

The following actions are not performed for non-SMS-managed data sets:

- Issue a message for unmatched data sets
- Delete temporary data sets
- Release unused space in non-VSAM data sets
- Expire data sets by inactive age or age since creation

For non-SMS-managed data sets, the following actions are performed:

- Delete LIST and UTILITY data sets
- Delete data sets that passed their expiration dates
- Determine eligible data sets for migration, including Fast Subsequent Migration
- Determine eligible data sets to back up
- Determine eligible data sets for extent reduction

Remember, the SCRATCHFREQUENCY determines the cleanup of LIST and UTILITY data
sets. EXPIREDDATASETS determines the deletion of data sets that are past expiration, and
migration is determined by the DAYS value.

**Pass two**
In pass two, the following actions are taken:

- Sorts the candidates for migration into priority order
- Performs space management on the volume

Sorting the migration candidate into priority order is based on age and size. Space
management potentially performs the following functions, depending on the technique setting
on ADDVOL:

- Delete by age
- Delete if backed up
- Migrate

**Space management for VSAM data sets**
Migration of non-SMS-managed VSAM data sets occurs at the sphere level. The base cluster
and index do not need to be defined individually. PATHs are also included automatically.

When DFSMShsm migrates a VSAM sphere, it tracks up to eight components for one data
set.

For a non-SMS-managed component, only the components that reside in _one_ volume can be
migrated. DFSMShsm will not migrate VSAM data sets with a component that needs
verification. A VSAM data set with more than one alternate index or path will not be selected
for automigration. DFSMShsm will not migrate a data set that is marked as “forward
recovery”.

After the migration of the non-SMS-managed volumes, DFSMShsm performs extent
reduction in the same manner that it performs extent reduction for SMS-managed data sets.

**Automatic interval migration**
For non-SMS-managed volumes, interval migration from primary volumes occurs only when
the following requirements are met:

- You must define a threshold of occupancy.

- The high threshold of occupancy must be exceeded.

- The primary volume has an attribute of automatic migration.


-----

- The volume was added for automatic migration in a DFSMShsm host with interval
migration specified

- The space management technique of migration is set.

**Recall volume selection**
Recalls from non-SMS-managed volumes differ from recalls of SMS-managed data sets. For
non-SMS-managed data sets, DFSMShsm selects the target volume based on the following
pools:

- Data set pools
- Volume pools
- General pool (JES3) or a single pool that is configured by DFSMShsm

When a non-SMS-managed recall occurs, DFSMShsm builds a list of up to five candidate
volumes for the recall. The space information in DFSMShsm is based on the latest space
check that was performed on the primary volumes.

DFSMShsm selects a volume based on the following guidelines:

- If the data set that is recalled has a high-level qualifier (HLQ) that matches the HLQ of a
data set pool, the five volumes come from the data set pool.

- If the volume from which this data set was migrated is part of a volume pool, the candidate
volumes come from the volume pool.

- If the data set is not restricted to be recalled to a user-defined pool, DFSMShsm selects
volumes that were assigned to the default pool. Whether a volume is selected is based on
the following factors:

  - The subparameters that were selected for the **RECALL** parameter of the **SETSYS**
command.

  - The volumes with the **AUTORECALL** subparameter specified in the **ADDVOL** command and
with MIGRATE as the space management technique.

  - The volumes are online.

  - The use attribute of the volume.

  - An attribute of storage for the **RECALL(ANYSTORAGEVOLUME)** parameter.

  - An attribute of storage, public, or private for the **RECALL(PRIVATEVOLUME)** parameter.

When DFSMShsm builds the candidate list of volumes, DFSMShsm considers **LIKE** and
**UNLIKE** parameters. If LIKE is specified, DFSMShsm builds a list of volumes with the same
attributes as this volume, from where migration occurred. If the number of volumes with these
attributes is fewer than five, the list consists of fewer than five candidates.


-----

If UNLIKE was specified, DFSMShsm builds the candidate list based on all of the volumes in
the pool in the following order:

- If five or more volumes exist with like recall attributes, all of the volumes in the candidate
list have recall attributes that match those recall attributes of the volume from which the
data set was migrated.

- If not enough like volumes exist, DFSMShsm selects as many volumes with recall
attributes that match the recall attributes of the volume from which the data set was
migrated as possible, and selects unlike volumes in the following order:

  - Volumes with matching backup attributes (AUTOBACKUP|NOAUTOBACKUP and
BACKUPDEVICECATEGORY)

  - Volumes with matching migration attributes (AUTOMIGRATION|NOAUTOMIGRATION)

  - Volumes with no matching attributes

In the following cases, DFSMShsm does not require matching attributes:

- A data set that is being recalled to a user-defined pool

- A data set that was migrated by a command from a volume with the DELETEBYAGE
management technique

- A data set that was migrated by a command from a volume that was not managed by
DFSMShsm

- A data set that was migrated from a primary volume that was added to only one host in a
multiple DFSMShsm-host environment

When the candidate volume selection is complete, allocation can start. Allocation occurs by
selecting the first volume in the list. If this volume cannot meet the space requirements, the
next volume is selected. This process continues until allocation is successful or the volume
list is exhausted.

If DSS is the data mover, it might not be able to allocate a data set on the volume if the
volumes are too fragmented (requires the allocation to occur within five extents).

If the user tries to open a non-VSAM and non-SMS-managed data set by referring both to unit
and volser, automatic recall is initiated by DFSMShsm.

For a VSAM data set, DFSMShsm also captures the catalog locate and initiates a recall.

**Note:** A scratch request to a data set with a volser of MIGRAT is converted to a deletion of
a migrated data set and occurs without a recall. A delete PURGE of a data set with an
expiration date that was not met results in a recall and deletion.

**Command space management**
Command space management can be executed in the same way for non-SMS-managed
storage as for SMS-managed storage.

Migration from ML1 volumes differs compared to the same function on SMS-managed
volumes. For non-SMS-managed volumes, you can also perform the following tasks:

- Migrate from a single primary volume all data sets that are not used within the number of
days you specify

- Delete by age from a single primary volume all data sets that are not used within the
number of days you specify


-----

- Delete if backed up from a single primary volume all data sets that are not used within the
number of days you specify

- Migrate data sets from all non-SMS-managed primary volumes

- Control migration of individual data sets, groups of data sets, or volumes

- Recall a data set to a specific volume

- Recall an SMS-managed data set to a non-SMS-managed volume

**Volume space management by command**
Volume space management in a non-SMS-managed environment for primary volumes can be
performed in two ways. In the first method, space management is performed as it is in
automatic primary space management, which is based on the settings on the **ADDVOL**
command. The following example shows a MIGRATE VOLUME:

MIGRATE VOLUME(V11111)

In Example 9-12, the processing in automatic primary space management is still followed, but
the technique is taken from the setting on these commands.

_Example 9-12  MIGRATE VOLUME example of setting the technique_

    MMIGRATE VOLUME(V22222 DELETEBYAGE(days))
    MIGRATE VOLUME(V22222 DELETEIFBACKEDUP(days)
    MIGRATE VOLUME(V22222 MIGRATE(days)

You can use the **SETMIG** command to manage deviations to the general rules for migration.
The **SETMIG** command works at the data set name level, qualifier level, or volume level. The
following settings are valid:

- COMMANDMIGRATION
- MIGRATION
- NOMIGRATION

Example 9-13 shows examples of the **SETMIG** command.

_Example 9-13  SETMIG examples on exceptions_

    SETMIG VOLUME(V33333) NOMIGRATION
    SETMIG DATASETNAME(A.B.C) NOMIGRATION
    SETMIG LEVEL(HLG) MIGRATION

Recalling non-SMS-managed data sets can occur to a specific volume and can also be forced
to a non-SMS-managed volume, even if it was not previously managed by DFSMS.

This action is shown in Example 9-14. The first recall command recalls to a specific volume. If
the allocation is captured in the ACS routines, the data set is converted to DFSMS.

The second example recalls a DFSMS-managed data set and converts this data set into a
non-SMS-managed data set.

_Example 9-14  Recall commands_

    RECALL A.B.C UNIT(3390) VOLUME(V44444)
    RECALL A.B.C UNIT(3390) VOLUME(V44444) FORCENONSMS


-----

### 9.3.4 Migration subtasking

DFSMS V2.1 introduced a significant throughput improvement in auto migration by enabling
multiple migration subtasks that run in parallel under each migration task. This new function,
which is called _migration subtasking_ , increases bandwidth in auto migration and offers new
possibilities for managing migration requirements.

To understand how this new function changes the behavior of auto migration, it is important to
understand how DFSMShsm performed auto migration before DFSMS V2.1.

DFSMShsm auto migration processing was performed in three major serialized steps:

- A preprocessing step that checks eligibility, enqueues for serialization, and updates the
CDS and catalog

- A data movement step, which can be class transition (moving data between ML0 tiers) or a
migration hierarchy movement (Ml0 to Ml1 or ML2)
- A postprocessing step that performs the dequeue, the updates to the CDS and catalog,
and scratch cleanup

For small data sets, the overhead of premigration and postmigration processing is two/thirds
of the total processing time.

With the introduction of migration subtasking, parallel subtasks in each migration task enable
DFSMShsm to process more data sets in parallel in the same migration task. If the new
function is not enabled, migration processing continues as it did before.

**How migration subtasking changes auto migration**
The DFSMShsm performed auto migration before DFSMS V2.1 as a single-threaded process
for each migration task as outlined in the three steps. The parallelism that you can obtain with
DFSMShsm V2.1 adds more migration tasks and therefore more data sets at a time.

-----

In DFSMS V2.1, you can start multiple migration subtasks under each migration task to
enable parallel processing. The number of subtasks defaults to 5 . The processing for each
data set is still serialized in three major processing steps, but multiple data sets can be
processed in parallel due to the availability of multiple subtasks. The new feature increases
migration bandwidth, which provides relief in large environments where the migration window
is a bottleneck.

**Note:** If auto migration targets ML2 (tape), only one concurrent data movement can occur
at any time.

**Implementation of migration subtasking**
You can implement migration subtasking for all types of auto migration: primary space
management, on demand migration, and interval migration.

You activate migration subtasking through the **SETSYS** command only in the DFSMShsm
startup member. You cannot dynamically activate migration subtasking through a **SETSYS**
command. The following example shows how migration subtasking is activated:

`SETSYS MIGRATIONSUBTASKS(YES)`

The default for the **MIGRATIONSUBTASKS** parameter is NO . When the function is activated, the
default number of tasks is 5 (which can be patched). Testing of this function showed that five
tasks are ideal for CPU and memory consumption. The starting default value can be patched
up to a maximum value of 105 subtasks.


-----

You can also change the number of subtasks through the **SETSYS** command but you must
update the DFSMShsm PARMLIB and restart it to activate the update. Use the
**ADDITIONALSUBTASKS** parameter, which is a subparameter of the **MIGRATIONSUBTASKS**
parameter, to change the number of subtasks. The following example shows adding two
additional subtasks to the default number of subtasks:

SETSYS MIGRATIONSUBTASKS(YES ADDITIONALSUBTASKS(2))

The command adds the specified number to the current number of active subtasks, up to the
maximum limit of 105 subtasks. A **SETSYS QUERY** command displays the actual settings for
migration subtasking, as shown in Example 9-15.

_Example 9-15  Display of MIGRATIONSUBTASKS settings after adding additional tasks_

    ARC0267I MIGRATIONSUBTASKS=YES, 369
    ARC0267I (CONT.) ADDITIONALMIGSUBTASKS=02


-----

