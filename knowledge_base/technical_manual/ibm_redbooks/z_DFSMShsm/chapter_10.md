
# 10. Availability management means backup

DFSMShsm _availability management_ is a function to ensure that a recent copy of your direct
access storage device (DASD) data set exists. Any lost or damaged data sets can be
retrieved at the most recent level. Availability management consists of the following functions:

- Backup
- Dump
- Aggregate backup and recovery support (ABARS)
- Recovery
- Fast replication backup and recovery

The tasks for controlling the availability management of storage management subsystem
(SMS)-managed storage are accomplished by adding DFSMShsm commands to the
ARCCMDxx member and by specifying attributes in the SMS storage groups, storage
classes, and management classes. We assume that you are familiar enough with SMS to
access the Interactive Storage Management Facility (ISMF) panels to change the various
attributes and that you have the update authority to them.

In this chapter, we describe the preceding functions in detail.


-----

## 10.1 Backup availability management

One of the ways in which DFSMShsm ensures data availability is by automatically copying
new and changed user data sets to a backup volume. The copy of your data set is called a
_backup version_ . The backup version ensures that your data is still available if your original
data set is damaged or accidentally deleted.

Another way in which DFSMShsm ensures data availability is by automatically dumping
DASD volumes to tape.

DFSMShsm uses the following functions to ensure that your data is available:

- Backup:

  - Automatic backup
  - Command backup
  - Inline backup

- Recovery:

  - Data set recovery
  - Volume recovery

### 10.1.1 Automatic backup

The automatic backup function, which is also referred to as _incremental backup,_ ensures that
current copies of new and changed data sets exist in case the original data sets are damaged
or accidentally deleted. The automatic incremental backup function is a data-set-level
function when DFSMShsm is processing level 0 volumes or a level 1 migration volume to a
backup volume during the automatic backup window.

DFSMShsm relies on the data set’s management class attributes to determine how
DFSMShsm manages the data set. For DFSMShsm to perform automatic backup, you must
set up automatic backup requirements for both the DFSMShsm environment and the data set
level.

**Setting up automatic backup in DFSMShsm environment**
You must define a backup window and backup cycle for DFSMShsm to perform the automatic
backup process by specifying the **SETSYS** command in ARCCMDxx PARMLIB:

SETSYS AUTOBACKUPSTART( _StartTime_ _LatestStartTime_ _QuiesceTime_ )

The variables are described:

- _StartTime_ : Automatic backup start time
- _LatestStartTime_ : Start automatic backup no later than the specified time
- _QuiesceTime_ : Additional volumes for backup are not allowed after this time

Depending on the number of data sets that are modified in a certain day and the number of
available tape drives for the DFSMShsm automatic backup function, you can map out the
automatic backup window. Assign the number of MAXBACKUPTASKS to be equal to the
number of available tape drives that DFSMShsm can use.

If you omit the quiesce time, automatic backup processes for all eligible volumes, which might
lengthen the backup window and affect batch or online processes.


-----

Automatic backup is performed in four phases:

1. Backing up the control data sets (CDSs): This process is called _CDS version backup_ , and
it is controlled by the **CDSVERSIONBACKUP** parameter of the **SETSYS** command. Before it
backs up the CDSs and the journal data set, DFSMShsm issues an exclusive enqueue
against the CDSs. In a multiple DFSMShsm-host environment, if the CDSs are _not_
accessed with Virtual Storage Access Method (VSAM) record-level sharing (RLS),
DFSMShsm reserves the volumes that contain the journal and the CDSs. If the CDSs _are_
accessed by using RLS, DFSMShsm performs an exclusive enqueue. If cross-system
coupling facility (XCF) is available in the multihost environment, the CDS backup host
notifies other hosts through XCF that the enqueue is needed.

   Optional: DFSMShsm invokes the CDS backup installation exit (ARCCBEXT) so that you
can access or validate the backup versions after DFSMShsm creates them.

   When the CDS backup is complete, DFSMShsm releases the volumes that contain the
journal data set and each CDS, and removes the exclusive enqueue against the CDSs,
which allows other DFSMShsm functions to proceed.

2. Moving backup versions from ML1 to tape: Backup versions that are created by data set
backup commands can be stored temporarily on migration-level 1 (ML1) volumes. After
DFSMShsm backs up the CDSs, the primary host moves any backup versions that are
created by data set backup commands that temporarily reside on ML1 volumes to daily
backup volumes.

   The primary host is the only host that moves backup versions. Backup versions are moved
only once a day and normally during automatic backup. However, if automatic backup is
not scheduled to run on a particular day, you can use the **ML1BACKUPVERSIONS** parameter of
the **FREEVOL** command to move backup versions.

3. Backing up migrated data sets: A migrated data set might not be backed up because it
was migrated after it was created or changed but before it was backed up. This condition
can occur if a **MIGRATE** command is issued for the data set shortly after the data set is
created or changed. The migration can occur to either an ML1 volume or a DASD
migration-level 2 (ML2) volume.

   A data set needs to be backed up when all of the following conditions apply:

   - The data-set-changed indicator is on.

   - The value of the Auto Backup attribute in the management class is Y .

   - The value of the ADMIN OR USER COMMAND BACKUP attribute in the management
class is not NONE .

   - The value of the Auto Backup attribute in the storage group is Y .

   If DFSMShsm recalls the data set before it is backed up, DFSMShsm does not back up the
data set from the migration volume. Instead, DFSMShsm backs up the data set when it
backs up the volume that is managed by DFSMShsm that now contains the data set.

   If you specify the **BACKDS** command to back up a migrated data set, DFSMShsm does not
back up the data set again during this phase.

   The primary host is the only host that backs up migrated data sets. Migrated data sets are
backed up once a day and normally during automatic backup. However, if automatic
backup is not scheduled to run on a particular day, you can use the **BACKUP** parameter of
the **RELEASE** command to back up migrated data sets.


-----

4. Backing up volumes that are managed by DFSMShsm from a storage group with the auto
backup attribute: Before backup processing begins for SMS-managed volumes,
DFSMShsm requests from SMS a list of management classes that are defined to the
configuration in which DFSMShsm is running. The successful return of the list of
management classes indicates that SMS is installed and active. If DFSMShsm does not
receive the list of management classes, it issues a message to indicate that backup
cannot be performed for SMS-managed volumes.

   If SMS is active, DFSMShsm retrieves the list of volumes that are associated with storage
groups that have the AUTO BACKUP attribute value of Y . After it retrieves the list of
volumes, DFSMShsm starts the backup tasks to the maximum that is specified by the
**MAXBACKUPTASKS** parameter of the **SETSYS** command in the particular host. If the maximum
number of backup tasks is reached and still more volumes must be processed,
DFSMShsm waits for any already-active volume backup task to complete backing up its
current volume. When a volume backup task completes backing up its current volume, it
begins backing up the next SMS-managed volume with the automatic backup attribute.
This process continues until all SMS-managed volumes are backed up or the quiesce time
passes.

   Backup processing examines each data set on the volume individually. Backup processing
for each data set performs the following functions:

   - Checks for unmatched data sets (missing catalog or volume table of contents (VTOC)
entry or both) and issues a message

   - Deletes temporary data sets

   - Creates a VTOC copy data set entry for the volume

   - Backs up the eligible data sets

   - Resets the data-set-changed indicator and the last backup date

Always start automatic backup on the primary host processor about 10 - 30 minutes before
you start backup on any other hosts, depending on the expected time to accomplish the first
three phases of automatic backup. This delay allows the CDSs to be backed up, and the other
processors do not have to wait for the CDSs. Another tip is to run automatic backup before
space management so that backup versions that are created with the **BACKDS** command are
moved off the ML1 volumes where they temporarily reside.

You also need to define the backup cycle and the date that it will start the backup process.
This command is also in ARCCMDxx and the syntax is shown:

DEFINE BACKUP(YYYYYYN CYCLESTARTDATE(2012/08/06))

The preceding command establishes a 7-day backup cycle that starts on Monday, 6 August
2012. Sunday is a day off for automatic backup. Specifying CYCLESTARTDATE means that
the cycle will stay the same through each initialization of DFSMShsm.

We recommend that you code your DEFINE similar to the example. You might find that it is
easier to work with a 7-day cycle than a long string of Ys and Ns.

Your string of alphabetic Ys and Ns can represent up to 31 days in the cycle.

If you are running automatic backup every day, we recommend that you use a 1-day backup
cycle to reduce the number of partial tapes in use.

DFSMShsm can use either DASD or tape as target volumes. To direct your backup versions
either to tape or DASD, the syntax is shown:

SETSYS BACKUP(DASD | TAPE)


-----

If you are backing up to DASD and the volume becomes full, DFSMShsm moves older backup
versions of data sets to other volumes that are known as _spill volumes_ . These volumes are
usually tape. We recommend that you back up to tape, so spill processing is available. To
prevent spill processing, use the **SETSYS** **SPILL** command. The SETSYS SPILL command
syntax is shown:

SETSYS SPILL | NOSPILL

The following command shows how to use DFSMShsm to back up a data set with a different
prefix (change the DFSMShsm backup prefix):

SETSYS BACKUPPREFIX( _prefix_ )

If you want to create two copies concurrently, you can use the DUPLEX feature. The DUPLEX
tape function is an alternative to the TAPECOPY function. The intent of the DUPLEX function
is to keep one copy onsite and to store the other copy remotely. The command syntax for
creating the duplex for the BACKUP function is shown:

SETSYS DUPLEX(BACKUP(Y | N))

You also need to consider the number of DFSMShsm concurrent backup tasks to define to
DFSMShsm. Normally, the number of backup tasks is equivalent to the number of tape drives
that are available to DFSMShsm for backup. The maximum number of backup tasks is 64 .
The command syntax is shown:

SETSYS MAXBACKUPTASKS( _n_ )

The data set management class attributes define how the data set is treated for the creation
and retention of backup versions. DFSMShsm makes copies ( _versions_ ) of changed data sets
that are on level 0 volumes on backup volumes. DFSMShsm uses the management class
attributes and the guaranteed backup frequency attribute of the storage group for each data
set to determine whether to copy the data set. After the data sets are backed up, DFSMShsm
determines from the management class attributes for each data set how many backup
versions to keep and how long to keep them.

You specify backup versions to either DASD or tape. If backup is performed to DASD, your
DASD volumes can become filled with the current backup versions and the earlier backup
versions that were not discarded. When DASD backup volumes are full, DFSMShsm transfers
the old (non-current but valid) backup versions to spill backup volumes. The transfer of valid
versions from DASD backup volumes can be either to tape or to DASD.

If the backup or the spill processes are performed to tape, the tapes eventually contain many
invalid versions that were superseded by the more current versions that were made. When
tape volumes contain many invalid versions, they are selected by the recycle process, which
moves valid versions to spill backup volumes.

**Setting up the automatic backup process at the data set level**
For DFSMShsm to automatically back up the SMS-managed data set, the data set
management class of AUTO BACKUP must be Y . The data set must be on a volume that
belongs to a storage group that is eligible for AUTO BACKUP , as shown in Figure 10-1 and Figure 10-2.


-----

                        -  MA:NAGEMENT CLASS ALTER        
    Command ===>
    
    SCDS Name . . . . . . : SYS1.SMS.MHLRES3.SCDS
    Management Class Name : MC54PRIM
    
    To ALTER Management Class, Specify:
    Backup Attributes
    Backup Frequency . . . . . . . . 1    (0 to 9999 or blank)
    Number of Backup Vers . . . . . . 2    (1 to 100 or blank)
    (Data Set Exists)
    Number of Backup Vers . . . . . . 1    (0 to 100 or blank)
    (Data Set Deleted)
    Retain days only Backup Ver . . . 30    (1 to 9999, NOLIMIT or blank)
    (Data Set Deleted)
    Retain days extra Backup Vers . . 5    (1 to 9999, NOLIMIT or blank)
    Admin or User command Backup . . BOTH   (BOTH, ADMIN or NONE)
    **Auto Backup . . . . . . . . . . . Y** (Y or N)
    Backup Copy Technique . . . . . . S    (P, R, S, VP, VR, CP or CR)

_Figure 10-1  Management class backup attributes_

The meanings of the attributes in Figure 10-1 are explained for the management class
MC54PRIM:

- The BACKUP FREQUENCY attribute determines how frequently changed data sets are
backed up automatically. It specifies the number of days that must elapse since the last
backup. In this case, one day must elapse before it is eligible for backup again and it must
change since the last backup.

- The NUMBER of BACKUP VERS (Data Set Exists) attribute tells DFSMShsm to keep two
backup versions while the data set exists.

The correct use of the BACKUP FREQUENCY and NUMBER OF BACKUP VERSIONS
(Data Set Exists) attributes can provide added assurance that you can recover data sets.

Assume that you specify a backup frequency of one day and a maximum number of
versions of three. If the data set changes daily, the user has only two days to detect any
errors that occur in the data set before the oldest backup version contains the error. If the
error is not detected in two days, the user cannot use a backup version to recover the data
set because all backup versions now contain error data.

Now, assume that you specify a frequency of two days and a maximum number of
versions of three. If the data set changes daily, the user has six days to detect the error
before the last backup version contains the error. However, each successive backup
version contains two days’ worth of daily changes, which might not provide enough
granularity for your needs.

You can also control the number of days in which a user can detect errors by increasing
the number of retained backup versions.

- The NUMBER of BACKUP VERS (Data Set Deleted) attribute tells DFSMShsm to keep
one backup version and retain it for 30 days after the data set is deleted.

- The RETAIN DAYS EXTRA BACKUP VERS: Extra backup versions are retained for five
days after they are created.

- ADMIN or USER COMMAND BACKUP: Apart from automatic backup, users can issue
commands to back up data sets.


-----

AUTO BACKUP: Process the eligible data sets.

BACKUP COPY TECHNIQUE: Use the standard copy process. The Concurrent Copy
process is not used.

If you want to back up data sets by using Concurrent Copy, follow these conditions. The
data set must reside on level 0 volumes. The Concurrent Copy will not process either
integrated catalog facility (ICF) catalogs or partitioned data sets (PDSs). Use of
Concurrent Copy for data set backup is justified only if the data set is database-related
and significant value exists in serializing the data set for as short a duration as possible.

However, if your hardware is sufficient to support Concurrent Copy, seven options are
available to you through the management class BACKUP COPY TECHNIQUE attribute:

- **R:** CONCURRENT REQUIRED

Concurrent Copy must be used for backup. The backup will fail for
data sets that do not reside on volumes that are supported by
Concurrent Copy or otherwise unavailable for Concurrent Copy.

- **P:** CONCURRENT PREFERRED

Concurrent Copy needs to be used for backup. A data set is backed
up without Concurrent Copy if it does not reside on a volume that is
supported by Concurrent Copy or otherwise unavailable for
Concurrent Copy.

- **S:** STANDARD

Data sets are backed up without the Concurrent Copy technique.

- **VP:** VIRTUAL PREFERRED

Data is processed with virtual Concurrent Copy, if possible.
Otherwise, the data is processed by using standard I/O as though
CONCURRENT is not specified.

- **VR:** VIRTUAL REQUIRED

Data is processed with virtual Concurrent Copy, if possible.
Otherwise, the data is not processed.

- **CP:** CACHE PREFERRED

Data is processed with cache-based Concurrent Copy, if possible.
Otherwise, the data is processed by using standard I/O as though
CONCURRENT is not specified.

- **CR:** CACHE REQUIRED

Data is processed with cache-based Concurrent Copy, if possible.
Otherwise, the data is not processed.

When DFSMShsm performs either automatic, volume, or command data set backup
processing, the management class that is associated with a data set specifies to use a
Concurrent Copy BACKUP COPY TECHNIQUE and DFSMSdss as the data mover, and
then DFSMShsm invokes DFSMSdss by using the appropriate **CONCURRENT** parameter.


-----

Figure 10-2 shows how to specify the automatic backup (Auto Backup) attribute to a storage
group.

                        POOL STORAGE GROUP DEFINE 
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : SCTEST
    To DEFINE Storage Group, Specify:
    Description ==>
    ==>
    Auto Migrate . . Y (Y, N, I or P)  Migrate Sys/Sys Group Name . .
    **Auto Backup . . Y (Y or N)** Backup Sys/Sys Group Name . .
    Auto Dump . . . N (Y or N)     Dump Sys/Sys Group Name . . .
    Overflow . . . . N (Y or N)     Extend SG Name . . . . . . . .
    Copy Pool Backup SG Name . . .
    
    Dump Class . . .           (1 to 8 characters)
    Dump Class . . .           Dump Class . . .
    Dump Class . . .           Dump Class . . .
    
    DEFINE  SMS Storage Group Status . . . N  (Y or N)
    DEFINE  SMA Attributes . . . . . . . . N  (Y or N)
    Use ENTER to Perform Selection; Use DOWN Command to View next Page;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.
    
_Figure 10-2  Define the Auto Backup attribute for a storage group_

If your DFSMShsm environment consists of more than one DFSMShsm host, you can select
the system to run the backup process by specifying the system name in the BACKUP
SYS/SYS GROUP NAME field.

The value of SETSYS INCREMENTALBACKUP(ORIGINAL | CHANGEDONLY) is defined in
ARCCMDxx. If you specify ORIGINAL, DFSMShsm backs up the data sets with no backup
copies regardless of the change bit status or data sets with the change bit set to on .
Specifying CHANGEDONLY means that only data sets with the data set changed indicator
set to on are backed up.

We recommend that you specify ORIGINAL from time to time (monthly or quarterly) to ensure
that all data sets that require backup are backed up.

You can also use the GUARANTEED BACKUP FREQUENCY storage group attribute to
control the maximum period of elapsed time before a data set is backed up regardless of its
change status, as shown in Figure 10-3 on page 251.


-----

                     POOL STORAGE GROUP ALTER
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : SGTEST
    
    To ALTER Storage Group, Specify:
    
    Allocation/migration Threshold :   High  85  (1-100) Low . .   (0-99)
    Alloc/Migr Threshold Track-Managed: High  85  (1-100) Low . . 1  (0-99)
    **Guaranteed Backup Frequency . . . . . . 7** (1 to 9999 or NOLIMIT)
    BreakPointValue . . . . . . . . . . . .     (0-65520 or blank)

_Figure 10-3  Guaranteed backup frequency attribute_

GUARANTEED BACKUP FREQUENCY provides an additional criterion to create a backup
version and guarantees that a backup version of the data set is created after the specified
number of days elapses, whether the data set changed or not. The backup version is created
either during the DFSMShsm automatic backup or command volume backup incremental
processing.

If the data set did not change and the backup version is created because the GUARANTEED
BACKUP FREQUENCY was met, that backup version replaces the most recent backup
version. (The replacement is not performed for the **(H)BACKDS** and **BACKVOL TOTAL**
commands.)

GUARANTEED BACKUP FREQUENCY (as shown in Figure 10-3) takes precedence over
the management class attribute BACKUP FREQUENCY. For example, if the BACKUP
FREQUENCY is set to six days and GUARANTEED BACKUP FREQUENCY is set to four
days, the data set is backed up after four days, whether the data set changed or not. If the
attribute values are reversed, the data set is backed up after four days if changed, or after six
days if not changed.

GUARANTEED BACKUP FREQUENCY provides a way to reduce the necessary number of
tape mounts to recover a volume when a few data sets are not changed. Volume dumps copy
all data, even if nearly all of the data is already backed up by incremental backup. Having all
of the necessary data for a recovery on a few tapes can reduce the number of necessary tape
mounts to recover a volume entirely from incremental backups.

If the data set storage group attribute of AUTO BACKUP is NO, no data sets are backed up by
the automatic backup process, regardless of the setting of GUARANTEED BACKUP
FREQUENCY. However, if an SMS volume is backed up by the **BACKVOL** command,
GUARANTEED BACKUP FREQUENCY is honored for data sets that are not changed.

**Note:** No interaction occurs between GUARANTEED BACKUP FREQUENCY and the
command backup of a single data set or of a total volume.

**Storage group processing priority for automatic backup**
The ISMF Storage Group application has a processing priority setting, which is set to 50, by
default. If the client does not change this setting, DFSMShsm uses the normal processing
order.


-----

If the client changes the setting to a different, individual value on the storage groups that are
backed up, DFSMShsm uses the processing priority (possible value is 100 - 1) and processes
automatic backup on these storage groups based on priority. DFSMShsm selects the storage
group with the highest priority first and then lower priority storage groups, always selecting
the highest prioritized of the remaining storage groups next. This way, DFSMShsm backs up
the most mission-critical storage groups first.

### 10.1.2 Backing up data sets with a command

You can use a DFSMShsm command to back up data sets.

The data set backup by command function provides the following capabilities:

- Up to 64 data sets for each host can be backed up at one time.

- Data sets can be backed up directly to DASD or to tape.

- If Concurrent Copy is specified, users are notified when the data set backup is either
logically or physically complete. In addition, Concurrent Copy supports non-SMS data
sets, or if specified on the command, Concurrent Copy overrides the management class
for an SMS data set.

- Users can unmount continuously mounted backup tapes.

- Users can tailor the times when DFSMShsm unmounts a tape.

- Users can specify the number of days to retain a specific backup copy of a data set. See
10.1.4, “Backing up a data set manually and the RETAINDAYS keyword”.

- Users can create a data set backup copy and store it under a different, specified data set
name.

You can use the data set backup command to back up an individual data set:

(H)BACKDS _dsname_ | _dsn_patterns_

You can direct your backup copies to either DASD or tape:

(H)BACKDS _dsname_ TARGET(DASD) | TARGET(TAPE)

You can also back up the data set to a new name:

(H)BACKDS _dsname_ NEWNAME( _newdsname_ )

When DFSMShsm processes these commands, it first checks the management class for the
data set to determine the value of the ADMIN OR USER COMMAND BACKUP attribute. If the
value of the attribute is BOTH, a DFSMShsm-authorized user can use either of the
commands, and a non-DFSMShsm-authorized user can use the **(H)BACKDS** command to back
up the data set. If the value of the attribute is ADMIN, a DFSMShsm-authorized user can use
either of the commands to back up the data set, but a non-DFSMShsm-authorized user
cannot back up the data set. If the value of the attribute is NONE, the command backup
cannot be performed.

Because the data set backup function allows users to target their backups directly to tape, this
backup might increase the number of required tape drives and mounts. The **SETSYS** **DSBACKUP**
and the **HOLD BACKUP(DSCOMMAND)** commands manage the task levels of both tape and ML1
DASD.

The **DSBACKUP** command has three parameters, **DASDSELECTIONSIZE** , **DASD** , and **TAPE** , so that
you can control the command data set backup environment:

SETSYS DSBACKUP(DASDSELECTIONSIZE( _max_ _standard_ ) DASD(TASK( _n_ )) TAPE(TASK( _n_ )))


-----

With the **DASDSELECTIONSIZE** parameter, you can balance the workload between DASD and
tape tasks for all WAIT requests that do not target an output device. This parameter is
applicable if both tape and ML1 DASD are used for data set backup requests. The defaults for
this parameter are 3,000 KB for the _maximum_ size and 250 KB for the _standard_ size.

With the **DASD** parameter, you can specify the number of concurrent data set backup tasks that
can direct backups to ML1 DASD. The value _n_ for the **TASK** subparameter is 0 -  64 tasks.

With the **TAPE** parameter, you can specify the number of concurrent data set backup tasks that
can direct backups to tape. The value _n_ for the **TASK** subparameter is 0 -  64 tasks. Another
parameter, **DEMOUNTDELAY** , is available with the **TAPE** parameter:

SETSYS DSBACKUP(TAPE(TASK( _n_ ) DEMOUNTDELAY(MAXIDLETASKS( _drives_ )- MINUTES( _minutes_ )))

With **DEMOUNTDELAY** , you can tailor the DFSMShsm tape mounting and unmounting for
command data set backup. **MINUTES(** **_minutes_** **)** is the number of minutes that you want
DFSMShsm to wait before it deallocates the tape that is associated with continuously inactive
(idle) command data set backup tasks. _Minutes_ is a value 0 -  1440 . A value of 1440 indicates
that DFSMShsm will not unmount the tape until command data set backup tasks are held, a
SWITCHTAPES event occurs, or DFSMShsm shuts down, so this value extends across 24
hours.

If you specify **DEMOUNTDELAY(MINUTES(0))** , a tape that is mounted for command data set
backup remains mounted long enough to support a continuous stream of WAIT-type backup
requests to tape from a single job stream even if the queue of work momentarily becomes
empty. The tape remains mounted for up to 5 seconds if **MINUTES(0)** is specified.

A change to the **DEMOUNTDELAY(MINUTES)** value results in setting a new delay time. This new
time is calculated from the current time of day. The new delay does not include any time that
the tape was idle up to the issuance of this change.

**MAXIDLETASKS(drives)** is the maximum number of tape drives that **DEMOUNTDELAY** can
accommodate. _Drives_ is a value 0 -  64 . You can specify a maximum of 64 drives for
**MAXIDLETASKS** , regardless of the number of **TAPE** tasks specified. However, the effective
number of tasks for **MAXIDLETASKS** is bound by the number of **TAPE** tasks. So, if you specify
**MAXIDLETASKS(64)** and **TAPE(TASKS(5))** , the effective **MAXIDLETASKS** is 5 . If later you specify
**TAPE(TASKS(7))** again, the effective **MAXIDLETASKS** becomes 7 .

When a command data set backup task that is writing to tape completes, DFSMShsm does
not deallocate tape units until all queued command data set backup requests are processed.
This design enables all consecutive WAIT-type requests by the same batch job to be
performed without an unmount and mount sequence.

If a data set backup command does not specify a target, the data set is directed to either tape
or DASD based on the size of the data set and the availability of tasks. If the data set is
greater than a specified threshold, the data set is directed to tape. Any data set less than or
equal to a specified threshold is directed to the first available task, DASD or tape.

Tape takes longer, from initial selection, to first write than DASD, but tape is potentially faster
in data throughput. DASD takes less time to become available because no mount delay
exists, but the throughput is potentially less than for tapes.


-----

**Notes:**

- A change in **TAPE(TASKS(** **_n_** **))** requires DFSMShsm to start additional tape tasks or stop
existing tape tasks (at the end of processing the current command data set).

- Be careful when you change the **DEMOUNTDELAY** parameter. For instance, if a backup to
tape task completes its work, the value for **MAXIDLETASKS** is greater than zero, and
**DEMOUNTDELAY(10)** is in effect, DFSMShsm sets an internal timer for 10 minutes:

  - If no additional work is received after 10 minutes, the task deallocates the drive and
ends.

  - If during this 10-minute idle time, the task is selected to perform a backup, the timer
is canceled. After the task completes, DFSMShsm again sets a timer value of 10
minutes.

  - If during this 10-minute delay, the time is decreased to 5 minutes by using
**DEMOUNTDELAY(MINUTES(5))** , DFSMShsm cancels the current timer and a new timer
is established for 5 minutes (not considering any previous time).

  - If **DEMOUNTDELAY(MINUTES(3))** is in effect and you issue an increase, such as
**DEMOUNTDELAY(MINUTES(10))** , DFSMShsm cancels the current timer value and uses
the new value. This increase resets the countdown to 10 minutes from the time the
command is received.

- If a parsing error occurs, DFSMShsm fails the **SETSYS** command with the ARC1605I
message. For non-parsing errors but contextual errors, DFSMShsm fails the parameter
in error but continues to process all other parameters in the **SETSYS DSBACKUP**
command. DFSMShsm treats the **TAPE(TASKS)** and **DASD(TASKS)** parameters as a
single entity. For example, **SETSYS DSBACKUP(DASDSELECTIONSIZE(5000 300)**
**DASD(TASKS(45)) TAPE(TASKS(20)))** results in setting **DASDSELECTIONSIZE** , but
**DASD(TASKS)** and **TAPE(TASKS)** fail because the sum of both tasks exceeds 64.

During the automatic backup process, DFSMShsm recognizes two situations that involve
backup and a data set in use. First, certain data sets exist that you do not expect to be open
for update, but that might be in use when backup is attempted. For this situation, the **SETSYS**
**BACKUP** parameter has an **INUSE** subparameter to control how DFSMShsm can try a second
time, after a specified delay, to serialize on a data set before backing it up, and whether
successful serialization for a retry is required or merely preferable. The backups that the
**INUSE** subparameter allows can be placed temporarily on ML1 DASD.

In the example system, it is presumed that you want DFSMShsm to attempt a retry if needed,
after a delay of 10 minutes, and to back up the data set even if it is still (or again) in use on the
second try. The following command shows a data set that is used with the SERIALIZATION
preferred method. The following command is added to the example system:

SETSYS BACKUP(INUSE(RETRY(Y) DELAY(10) SERIALIZATION(PREFERRED)))

For the second situation, certain data sets (typically a database) are expected to be open for
update most of the time. You can use the data set backup installation exit ARCBDEXT (which
is invoked at the data set backup level for **BACKDS** command) for additional control to direct
DFSMShsm either not to serialize before making a backup or to back up the data set even if
serialization fails. The **SETSYS** **INUSE** subparameters can also apply in this situation, but the
exit can override them for a certain data set.

For more information about the installation exit, see _DFSMS Installation Exits,_ SC26-7396.


-----

**Back up data set in a batch environment**
DFSMShsm supplies a program to perform an _inline backup_ . The program that you execute is
called ARCINBAK. With ARCINBAK, you can back up data sets in the middle of a job by the
addition of a new step to the job. In Example 10-1, the DD names to identify the data sets that
you want backed up are identified as BACK _nn_ . The data set names that are associated with
these DDs will be backed up. A DD name prefix of anything other than BACK is not allowed.

_Example 10-1  Sample ARCINBAK JCL_

    //JOBNAME JOB . . . ,USER=USERID,PASSWORD=USERPSWD
    //STEP1 EXEC PGM=USERPGM
    //SYSPRINT DD SYSOUT=A
    //DSET1 DD DSN=USERID.N03.GDG(-1),DISP=OLD
    //DSET2 DD DSN=USERID.N03.PSFB,DISP=OLD
    //DSET3 DD DSN=USERID.N04.PSFB,DISP=OLD
    //DSET4 DD DSN=USERID.N03.KSDS,DISP=OLD
    /*
    //STEP2 EXEC PGM=ARCINBAK
    //ARCPRINT DD SYSOUT=A
    //ARCSNAP DD SYSOUT=A
    //* ---------------------------------------------------------------//* BACKUP OF GDG DATA SET SHOULD BE SUCCESSFUL.
    //* ---------------------------------------------------------------//BACK01 DD DSN=*.STEP1.DSET1,DISP=SHR
    //* ---------------------------------------------------------------//* BACKUP OF NON-VSAM DATA SET SHOULD BE SUCCESSFUL.
    //BACK02 DD DSN=*.STEP1.DSET2,DISP=SHR
    //* ---------------------------------------------------------------//* BACKUP OF VSAM DATA SET SHOULD BE SUCCESSFUL.
    //* ---------------------------------------------------------------//BACK03 DD DSN=*.STEP1.DSET4,DISP=SHR
    //* ---------------------------------------------------------------//* BACKUP OF GDG DATA SET SHOULD BE SUCCESSFUL.
    //* ---------------------------------------------------------------//BACK04 DD DSN=USERID.N01.GDG.G0001V00,DISP=SHR
    //* ---------------------------------------------------------------//* BACKUP OF NON-VSAM DATA SET SHOULD BE SUCCESSFUL.
    //* ---------------------------------------------------------------//BACK05 DD DSN=USERID.N01.PSFB,DISP=SHR
    //* ---------------------------------------------------------------//* BACKUP OF UNCATALOGED DATA SET SHOULD FAIL.
    //* ---------------------------------------------------------------//BACK06 DD DSN=USERID.N02.UNCAT,VOL=SER=VOL003,UNIT=3390,DISP=SHR
    //* ---------------------------------------------------------------//* BACKUP OF VSAM DATA SET SHOULD BE SUCCESSFUL.
    //* ---------------------------------------------------------------//* ---------------------------------------------------------------//BACK07 DD DSN=USERID.N01.KSDS,DISP=SHR
    //* ---------------------------------------------------------------//* BACKUP OF OPEN IN-USE VSAM DATA SET SHOULD BE SUCCESSFUL.
    //* ----------------------------------------------------------------


    
    
    //BACK08 DD DSN=USERID.N02.KSDS,DISP=SHR
    //* ---------------------------------------------------------------//* BACKUP OF RACF PROTECTED NON-VSAM DATA SET
    //* BY AN UNAUTHORIZED USER SHOULD FAIL.
    //* ---------------------------------------------------------------//BACK09 DD DSN=USERXX.N02.PSFB,DISP=SHR
    /*

**Notes:**

- Only cataloged data sets are supported for inline backup. If volume and unit information
is specified on the DD statement, an uncataloged data set is assumed, and the data set
is not processed.

- ARCINBAK does not support data sets that are allocated with any of the following
dynamic allocation options, XTIOT, UCB NOCAPTURE, and DSAB above the line,
except when the calling program supplies an open data control block (DCB).

### 10.1.3 Backup non-SMS-managed data sets

You can also use automatic backup for data sets that reside on non-SMS-managed volumes
by using the **ADDVOL** command and defining it in ARCCMDxx. The syntax is shown:

ADDVOL _volser_ UNIT( _unittype_ ) PRIMARY(AUTOBACKUP)

The primary differences in processing for SMS-managed storage and non-SMS-managed
storage occur in the following processes:

- Uncataloged data sets can be backed up and recovered.
- Users can specify the volume to which a data set is to be recovered.
- Volumes to be dumped or backed up are individually identified to DFSMShsm.

Because storage groups or management classes are not available to define how to manage
the volumes and which volumes to manage for non-SMS-managed storage, the following
subtasks provide a way to manage non-SMS-managed storage:

- Defining how frequently data sets that are managed by a system will be backed up. The
**FREQUENCY** parameter of the **SETSYS** command controls the backup frequency for the data
sets that are processed by each DFSMShsm host. The command that is added to the
ARCCMDxx member for each host is shown:

SETSYS FREQUENCY( _days_ )

- Defining how many versions to keep for each backed up data set. For non-SMS-managed
storage, the number of backup versions to retain is a DFSMShsm host-wide specification.

As with SMS-managed storage, depending on the record length that is used to define the
backup control data set (BCDS), DFSMShsm can maintain up to 29, or up to 100, backup
versions of any data set. Within that upper limit, the **VERSIONS** parameter of the **SETSYS**
command controls the number of backup versions to retain. The command syntax to add
to the ARCCMD _xx_ member for each DFSMShsm host is shown:

SETSYS VERSIONS( _limit_ )

If you use the **RETAINDAYS** keyword when you create data set backup copies, and active
backup copies with unmet RETAINDAYS values are rolled off, DFSMShsm maintains them
as retained backup copies. DFSMShsm can maintain more than enough retained copies
for each data set to meet all expected requirements.


-----

The **FREQUENCY** and **VERSION** parameters of the **SETSYS** command control how often data sets
are backed up and how many versions are kept for all of the non-SMS-managed data sets.

You can control the frequency and versions for individual data sets with the **(H)ALTERDS**
command. You can also control the frequency and the versions independently of each other.

The command syntax is shown:

(H)ALTERDS _dsname_ FREQUENCY( _days_ ) VERSIONS( _limit_ )

### 10.1.4 Backing up a data set manually and the RETAINDAYS keyword

If the data set’s management class allows users to issue a backup command, users can back
up their data sets whenever they want, regardless of the change bit status, by using the
**(H)BACKDS** command. The **RETAINDAYS** keyword in the **(H)BACKDS** command allows users to
specify the number of days that a backup version will be retained for cataloged data sets. This
**RETAINDAYS** value is applied when a backup copy is rolled off.

The value of **RETAINDAYS** can be in the range of 0 -  50000 , which corresponds to the maximum
of 136 years. If you specify 99999 , the data set backup version is treated as never expiring.
Any value greater than 50000 (and other than 99999 ) causes a failure with an error message
ARC1605I. A **RETAINDAYS** value of 0 indicates the following conditions:

- The backup version expires when the next backup copy is created.

- The backup version might expire within the same day that it was created if EXPIREBV
processing takes place.

- The backup version is kept as an active copy before roll-off occurs.

- The backup version is not managed as a retained copy.

The command syntax for **BACKDS** with the **RETAINDAYS** keyword is shown:

(H)BACKDS _dsname_ RETAINDAYS( _days_ )

By using this feature, DFSMShsm backup copies are managed as either active copies or
retained copies:

- Active copies are a set of backup copies that are not yet rolled off. Active copies are
determined by either the SMS management class or SETSYS value. Depending on the
record length that is used to define the BCDS, DFSMShsm can keep up to 29, or up to
100, versions of any data set.

- Retained copies are backup copies that rolled off from a set of active copies and did not
reach their retention period. A data set can have an unlimited number of retained backup
copies.


-----

**Backup versions roll-off process**
The roll-off processing checks all of the active copies to see whether any active copy has
specified retained days and if so, if the retained days were met:

- If more than one of the active backup copies met their retain days, the oldest active
backup copy is rolled off, and the rest of the active backup copies are maintained as active
copies even though their retain days were already met.

- If one or more of the active backup copies specified retain days, but none of them met their
retain days, one of the following actions occurs:

  - If the oldest active backup copy has a RETAINDAYS value, it is rolled off and replaced
by the new backup copy but it is still maintained as a retained backup copy.

  - If the oldest active backup copy does not have a RETAINDAYS value, it is rolled off as
normal.

- If none of the active backup copies have RETAINDAYS values, roll off processing is
processed as normal.

**_Roll-off example_**
By using the **HLIST DSNAME(MHLRES4.JCL.CNTL) BCDS TERMINAL** DFSMShsm command, the
MHLRES4.JCL.CNTL data set shows five backup versions and the maximum number of
active copies. The NUMBER of BACKUP VERS (Data Set Exists) field in ISMF is 5 . See
Example 10-2.

_Example 10-2  Output from the HLIST command before the creation of another backup version_

    DSN=MHLRES4.JCL.CNTL               BACK FREQ = *** MAX ACTIVE B
    ACKUP VERSIONS = ***
    
    BDSN=HSM.BACK.T180320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:18 CAT=YES GEN=000 VER=005 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00015
    
    BDSN=HSM.BACK.T140320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:14 CAT=YES GEN=001 VER=004 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T080320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:08 CAT=YES GEN=002 VER=003 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00005
    
    BDSN=HSM.BACK.T040320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:04 CAT=YES GEN=003 VER=002 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    **BDSN=HSM.BACK.T280220.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A**
    **BACKDATE=12/08/20 BACKTIME=20:02:28 CAT=YES GEN=004 VER=001 UNS/RET= NO**
    **RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00001**
    
    TOTAL BACKUP VERSIONS = 0000000005
    
    ARC0140I LIST COMPLETED,    12 LINE(S) OF DATA OUTPUT
    COMMAND REQUEST 00000078 SENT TO DFSMSHSM
    ***


-----

The next day after the creation of the new backup version (VER=006), Example 10-3 shows
that the backup copy version 001 (VER=001) is rolled off because its RETAINDAYS value is 1
day and it is no longer shown in the output.

_Example 10-3  Output from HLIST command after the new backup version is created_

    DSN=MHLRES4.JCL.CNTL               BACK FREQ = *** MAX ACTIVE B
    ACKUP VERSIONS = ***
    
    BDSN=HSM.BACK.T222712.MHLRES4.JCL.A2234     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/21 BACKTIME=12:27:22 CAT=YES GEN=000 VER=006 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T180320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:18 CAT=YES GEN=001 VER=005 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00015
    
    BDSN=HSM.BACK.T140320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:14 CAT=YES GEN=002 VER=004 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T080320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:08 CAT=YES GEN=003 VER=003 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00005
    
    **BDSN=HSM.BACK.T040320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A**
    **BACKDATE=12/08/20 BACKTIME=20:03:04 CAT=YES GEN=004 VER=002 UNS/RET= NO**
    **RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*******
    
    TOTAL BACKUP VERSIONS = 0000000005
    
    ARC0140I LIST COMPLETED,    12 LINE(S) OF DATA OUTPUT
    COMMAND REQUEST 00000085 SENT TO DFSMSHSM
    ***

The next creation of a new backup version (VER=007) has backup copy version 2
(VER=002), which has no RETAINDAYS value, to be rolled off as shown in Example 10-4.

_Example 10-4  Output from HLIST command_

    DSN=MHLRES4.JCL.CNTL               BACK FREQ = *** MAX ACTIVE B
    ACKUP VERSIONS = ***
    
    BDSN=HSM.BACK.T025012.MHLRES4.JCL.A2234     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/21 BACKTIME=12:50:02 CAT=YES GEN=001 VER=007 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T014812.MHLRES4.JCL.A2234     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/21 BACKTIME=12:48:01 CAT=YES GEN=002 VER=006 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T180320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:18 CAT=YES GEN=001 VER=005 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00015
    
    BDSN=HSM.BACK.T140320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    
    BACKDATE=12/08/20 BACKTIME=20:03:14 CAT=YES GEN=002 VER=004 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    **BDSN=HSM.BACK.T080320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A**
    **BACKDATE=12/08/20 BACKTIME=20:03:08 CAT=YES GEN=003 VER=003 UNS/RET= NO**
    **RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00005**
    
    TOTAL BACKUP VERSIONS = 0000000005
    
    ARC0140I LIST COMPLETED,    12 LINE(S) OF DATA OUTPUT
    COMMAND REQUEST 00000085 SENT TO DFSMSHSM
    ***

The oldest active copy is VER=003 and its RETAINDAYS (RETDAYS= 00005 ) values are not
met. After the creation of a new backup version, VER=003 is rolled off but it is kept as the
RETAINED COPIES version. This rolled-off backup version has no GEN and VER values
anymore, as shown in Example 10-5. Also, the TOTAL BACKUP VERSIONS field is 6 .

_Example 10-5  Output from (H)LIST command_

    DSN=MHLRES4.JCL.CNTL               BACK FREQ = *** MAX ACTIVE B
    ACKUP VERSIONS = ***
    
    BDSN=HSM.BACK.T275912.MHLRES4.JCL.A2234     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/21 BACKTIME=12:59:27 CAT=YES GEN=000 VER=008 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T025012.MHLRES4.JCL.A2234     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/21 BACKTIME=12:50:02 CAT=YES GEN=001 VER=007 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T014812.MHLRES4.JCL.A2234     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/21 BACKTIME=12:48:01 CAT=YES GEN=002 VER=006 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T180320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:18 CAT=YES GEN=001 VER=005 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00015
    
    BDSN=HSM.BACK.T140320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:14 CAT=YES GEN=002 VER=004 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    **BDSN=HSM.BACK.T080320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A**
    **BACKDATE=12/08/20 BACKTIME=20:03:08 CAT=YES GEN=*** VER=*** UNS/RET= NO**
    **RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00005**
    
    **TOTAL BACKUP VERSIONS = 0000000006**
    
    ARC0140I LIST COMPLETED,    14 LINE(S) OF DATA OUTPUT
    ***

A new **SELECT(ACTIVE/RETAINDAYS)** parameter was added to the DFSMShsm **(H)LIST**
command to list either active copies or only retained copies as shown:

(H)LIST DSNAME( _dsn_ ) BOTH | BCDS SELECT(ACTIVE | RETAINDAYS)


-----

The output of the **LIST DSNAME(MHLRES4.JCL.CNTL) BCDS SELECT(ACTIVE) TERMINAL**
command is shown in Example 10-6.

_Example 10-6  Output from (H)LIST command for ACTIVE copies_

DSN=MHLRES4.JCL.CNTL               BACK FREQ = *** MAX ACTIVE B
    ACKUP VERSIONS = ***
    
    BDSN=HSM.BACK.T275912.MHLRES4.JCL.A2234     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/21 BACKTIME=12:59:27 CAT=YES GEN=000 VER=008 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T025012.MHLRES4.JCL.A2234     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/21 BACKTIME=12:50:02 CAT=YES GEN=001 VER=007 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T014812.MHLRES4.JCL.A2234     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/21 BACKTIME=12:48:01 CAT=YES GEN=002 VER=006 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    BDSN=HSM.BACK.T180320.MHLRES4.JCL.A22333     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:18 CAT=YES GEN=003 VER=005 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00015
    
    BDSN=HSM.BACK.T140320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:14 CAT=YES GEN=002 VER=004 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=*****
    
    TOTAL BACKUP VERSIONS = 0000000005
    
    ARC0140I LIST COMPLETED,    12 LINE(S) OF DATA OUTPUT
    ***

The output of the **LIST DSNAME(MHLRES4.JCL.CNTL) BCDS SELECT(RETAINDAYS) TERMINAL**
command is shown in Example 10-7.

_Example 10-7  Output from the (H)LIST command for RETAIN copies_

    DSN=MHLRES4.JCL.CNTL               BACK FREQ = *** MAX ACTIVE B
    ACKUP VERSIONS = ***
    
    BDSN=HSM.BACK.T180320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:18 CAT=YES GEN=000 VER=005 UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00015
    
    BDSN=HSM.BACK.T080320.MHLRES4.JCL.A2233     BACKVOL=SBXHS6  FRVOL=MHLS1A
    BACKDATE=12/08/20 BACKTIME=20:03:08 CAT=YES GEN=*** VER=*** UNS/RET= NO
    RACF IND=NO BACK PROF=NO NEWNM=NO NOSPH=*** GVCN=*** RETDAYS=00005
    
    TOTAL BACKUP VERSIONS = 0000000002
    ARC0140I LIST COMPLETED,    6 LINE(S) OF DATA OUTPUT


-----

### 10.1.5 Using DSSXMMODE

The backup and migration throughput can be maximized by using the DSSXMMODE feature.
In this mode, DFSMSdss is loaded in its own address space, or in the DFSMShsm address
space, for backup, CDS backup, dump, migration, and full-volume and data set recovery
functions. This parameter can be issued only from the ARCCMDxx PARMLIB member. This
parameter does not affect the FRBACKUP and FRRECOV use of the DFSMSdss
cross-memory interface.

The syntax command is shown:

SETSYS DSSXMMODE(Y|N BACKUP(Y|N) CDSBACKUP(Y|N) DUMP(Y|N) MIGRATION(Y|N)
RECOVER(Y|N))

The following parameters are optional parameters of the **DSSXMMODE** parameter:- 

-: **N:** DFSMSdss is not loaded into its own address space for the
DFSMShsm function that is invoked. Instead, DFSMSdss is loaded in
the DFSMShsm address space, which is the default.

- **Y:** DFSMSdss is loaded into a new address space for the DFSMShsm
function that is invoked and retained for future invocations of that
function until DFSMShsm is shut down or restarted.

- **BACKUP:** This value controls where DFSMSdss for the DFSMShsm backup
function is loaded. It can be in its own address space ( Y ) or in the
DFSMShsm address space ( N ). The default is N for no.

- **CDSBACKUP:** This value controls where DFSMSdss for the DFSMShsm CDS
backup function is loaded. It can be in its own address space ( Y ) or in
the DFSMShsm address space ( N ). The default is N for no.

- **DUMP:** This value controls where DFSMSdss for the DFSMShsm dump
function is loaded. It can be in its own address space ( Y ) or in the
DFSMShsm address space ( N ). The default is N for no.

- **MIGRATION:** This value controls where DFSMSdss for the DFSMShsm migration
function is loaded. It can be in its own address space ( Y ) or in the
DFSMShsm address space ( N ). The default is N for no.

- **RECOVERY:** This value controls where DFSMSdss for the DFSMShsm full-volume
and data set recovery functions are loaded. They can be in their own
address space ( Y ) or in the DFSMShsm address space ( N ). The default
is N for no.

Table 10-1 on page 263 defines the DFSMSdss address space identifiers for address spaces
that are started by DFSMShsm.


-----

_Table 10-1  DFSMSdss address space identifiers that are started by DFSMShsm functions_

|Function|Address space identifier|
|---|---|
|Backup|ARCnBKUP|
|CDS backup|ARCnCDSB|
|Dump|ARCnDUMP|
|Fast replication backup|DSSFRBxx|
|Fast replication recovery (copy pool and volume)|DSSFRRxx|
|Fast replication recovery (data set)|DSSFRDSR|
|Migration|ARCnMIGR|
|Recovery from backup (data set and full volume)|ARCnRCVR|
|Recovery from a dump tape (data set)|ARCnREST|
|Recovery from a dump tape (full volume)|ARCnRSTy|



DFSMSdss requires that the ARC* and DFSSFR* address spaces run at the same or higher
priority as DFSMShsm. You can create Workload Manager (WLM) profiles so that these
address spaces are assigned to the correct service classes.

**Notes:**

- Each DFSMSdss address space requires the correct setup in the same manner as
DFSMShsm started task control (STC).

- Use of the DFSMSdss cross-memory API for these functions might result in an
increase in CPU time.

- When DFSMShsm starts one or more of these address spaces, the address spaces
remain active until DFSMShsm is terminated. When DFSMShsm terminates, all of the
started DFSMSdss address spaces automatically terminate.

- The **FRBACKUP** and **FRRECOV** commands always use DFSMSdss cross-memory support.

- The DFSMSdss address space identifiers that are started for the dump, data set
recovery from dump, data set recovery from backup, migration, backup, and CDS
backup functions are optional and are controlled by using the SETSYS DSSXMMODE.

### 10.1.6 Interrupting and restarting backup processing

To stop the first phase of automatic backup, which is the backup of the CDSs, you must use
the **STOP** command.

You can use the following methods to interrupt automatic backup processing during its last
three phases. The last three phases of automatic backup processing consist of the following
functions:

- Movement of backup versions
- Backup of migrated data sets
- Backup of volumes that are managed by DFSMShsm that have the automatic backup
attribute


-----

Four methods exist that you can use to interrupt automatic backup processing during its last
three phases:

- Holding backup processing: You can hold backup processing by specifying the **HOLD**
command for **BACKUP** or **BACKUP(AUTO)** , or **ALL** parameters. If you hold backup processing
with the **ENDOFDATASET** parameter while DFSMShsm is moving backup versions or backing
up migrated data sets, processing ends after DFSMShsm finishes processing the current
backup version.

  If you hold backup processing with the **ENDOFVOLUME** parameter while DFSMShsm is
moving backup versions or backing up migrated data sets, processing ends after the
current phase of automatic backup completes. If you hold backup processing with the
**ENDOFVOLUME** parameter while DFSMShsm is backing up managed volumes, DFSMShsm
continues processing the current volumes and does not process any new volumes.
Otherwise, DFSMShsm finishes processing the current data set. DFSMShsm does not
start any new volume backup tasks.

- Disabling backup processing: You can disable backup processing by specifying the **SETSYS**
**NOBACKUP** command. If you disable backup processing while DFSMShsm is moving backup
versions or backing up migrated data sets, processing ends after DFSMShsm finishes
processing the current backup version. If you disable backup processing while
DFSMShsm is backing up volumes that are managed by DFSMShsm, DFSMShsm
finishes processing the current data set and does not start any new volume backup tasks.

- Placing DFSMShsm in emergency mode: You can place DFSMShsm in emergency mode
by specifying the **SETSY** **EMERGENCY** command. If you place DFSMShsm in emergency
mode while DFSMShsm is moving backup versions or backing up migrated data sets,
processing ends after DFSMShsm finishes processing the current data set.

  If you are in a multiple DFSMShsm-host environment and enabled the secondary host
promotion function, another DFSMShsm host takes over the unique functions of the
primary DFSMShsm host that you put in emergency mode. Those functions include the
automatic movement of backup versions from ML1 to tape, and the automatic backup of
migrated data sets. If promotion occurs during an automatic backup window (as defined
on the primary host), and it is not past the latest start time, the promoted host takes over
where the primary host left off.

  If the promoted host was an automatic backup host before being promoted, it performs
backups of DFSMShsm-owned volumes after performing the automatic movement of
backup versions from ML1 to tape, and the automatic backup of migrated data sets. If the
promoted host was not an automatic backup host before being promoted, it does not back
up DFSMShsm-owned volumes.

  If you place DFSMShsm in emergency mode while DFSMShsm is backing up volumes
that are managed by DFSMShsm, DFSMShsm finishes processing the current data set
before ending the volume backup tasks that are in progress and it does not start any new
volume backup tasks.

- Stopping DFSMShsm: You can stop DFSMShsm by entering the MVS or DFSMShsm
**STOP** command. If you shut down DFSMShsm while DFSMShsm is moving backup
versions or backing up migrated data sets, processing ends after DFSMShsm finishes
processing the current data set.

  If you shut down DFSMShsm while DFSMShsm is backing up volumes that are managed
by DFSMShsm, DFSMShsm finishes processing the current data set before ending the
volume backup tasks that are in progress and does not start any new volume backup
tasks.

  If you issue the DFSMShsm **STOP** command while DFSMShsm is backing up the CDSs or
the journal data set, DFSMShsm finishes processing the current data set and does not
back up any other CDSs or the journal data set.




  If you are in a multiple DFSMShsm-host environment and you enabled secondary host
promotion and issued a **STOP PROMOTE** command, another DFSMShsm host takes over the
unique functions of the primary DFSMShsm host that you stopped. Those functions
include the automatic movement of backup versions from ML1 to tape, and the automatic
backup of migrated data sets.

  If promotion occurs during an automatic backup window (as defined on the primary host),
and it is not past the latest start time, the promoted host takes over where the primary host
left off. If the promoted host was an automatic backup host before being promoted, it
performs backups of DFSMShsm-owned volumes after performing the automatic
movement of backup versions from ML1 to tape, and automatic backup of migrated data
sets. If the promoted host was not an automatic backup host before being promoted, it
does not back up DFSMShsm-owned volumes.

You can restart automatic backup processing by entering the corresponding commands if
certain conditions are met:

- RELEASE BACKUP or RELEASE ALL
- SETSYS BACKUP
- SETSYS NOEMERGENCY
- START DFSMShsm (MVS command)

If secondary host promotion did not continue backup processing where the primary
DFSMShsm host left off, the automatic backup restarts at the point of interruption if the date
of the defined start time for the current automatic backup window is the same as the date that
automatic backup last started from the beginning (same cycle date). When automatic backup
restarts, it uses the same set of daily backup volumes it was using when it first started. Those
daily backup volumes are the volumes that are assigned to the day in the backup cycle that
was in effect when the automatic backup started.

After you restart automatic backup processing, DFSMShsm continues with the next volume
that is managed by DFSMShsm under automatic backup control. However, a few
considerations exist:

- If you issued a DFSMShsm **STOP** command and then issued a **START** DFSMShsm
command, DFSMShsm can start automatic backup at the point of interruption if the
following conditions are met:

  - If you add the primary volumes under automatic backup control in the same order in
each startup of DFSMShsm

  - If you do not change the volumes that are associated with the storage groups

- If you interrupted a **BACKVOL PRIMARY** command, DFSMShsm can resume backup at the
start of the next volume only if you issued HOLD BACKUP or SETSYS NOBACKUP . You
must issue RELEASE BACKUP , RELEASE ALL, or SETSYS BACKUP during the same
startup of DFSMShsm if you want the **BACKVOL PRIMARY** command to resume processing.

- DFSMShsm does not restart a **BACKVOL** command that is issued for one volume.

## 10.2 Dump availability management

The process of copying all data from a DASD volume to a tape volume is called _volume dump_ .
The dump of the entire volume is performed by DFSMShsm invoking DFSMSdss through an
application interface. Volumes that are not in storage groups or not set as ADDVOL to
DFSMShsm are dumped only by command.


-----

### 10.2.1 Setting up automatic full volume dump in the DFSMShsm environment

You must define the full volume dump window and the full volume dump cycle for DFSMShsm
to perform the auto dump process. You specify the **SETSYS** **AUTODUMPSTART** and **DEFINE**
commands in the ARCCMDxx PARMLIB, as shown:

SETSYS AUTODUMPSTART( _StartTime_ _LatestStartTime_ _QuiesceTime_ )

The command syntax is defined:

- _StartTime_ : Automatic dump start time.
- _LatestStartTime_ : Start automatic dump no later than the specified time.
- _QuiesceTime_ : Additional volumes for full volume dump are not allowed after this time.

If you omit the quiesce time, auto dump processes all eligible volumes.

DFSMShsm always allocates the source and dump volumes for the full-volume dump before it
invokes DFSMSdss. DFSMSdss performs only volume-level serialization during full-volume
dump processing. Because DFSMShsm does not perform any data set serialization during
full-volume dump processing, activity against a volume that is being dumped needs to be kept
at an absolute minimum during the full-volume dump process, or errors occur.

You also need to define the backup cycle and the date that it will start the backup process.
This command is also in ARCCMDxx and the syntax is shown:

DEFINE DUMPCYCLE(NNNNNNY CYCLESTARTDATE(2012/08/06))

The preceding command establishes a seven-day dump cycle that starts on Monday, 6
August 2012. Automatic dump will run on Sunday only. Specifying CYCLESTARTDATE
means that the cycle stays the same through each initialization of DFSMShsm. We
recommend that you code your DEFINE similar to the example. You might find that it is easier
to work with a seven-day cycle than a long string of Ys and Ns.

**Notes:**

- The date that is specified for the cycle start date cannot be a date in the future. The
date must always fall on or before the date that the **DEFINE** command is issued.

- When you redefine the dump class, be careful if you use the **DAY** parameter to define
the days that a certain dump class is active. The day that is specified with the **DAY**
parameter needs to be a Y day in the DUMPCYCLE. If the day that is specified is an N
day in the DUMPCYCLE, automatic dump processing does not target the dump class.

- Dump cleanups run on N days in the dump cycle. Cleanup functions include the
expiration of expired dump copies and the deletion of excess dump VTOC copy data
sets.

DFSMShsm can run up to 64 concurrent dump tasks for each host, which does not limit the
number of dump copies because each dump task can create up to five dump copies. The
syntax for this command is shown. You need to add it to your ARCCMDxx member.

SETSYS MAXDUMPTASKS( _nn_ )

You must consider the number of tape units that you expect to have available during
automatic dump processing. To determine the number of necessary tape drives, multiply the
number of dump classes for each volume by the number of dump tasks. If this number
exceeds the number of tape drives that you expect to have available, you must lower your
**MAXDUMPTASKS** parameter. If this number is under the number of tape drives that you expect to
have available, you can raise your **MAXDUMPTASKS** parameter.


-----

You also need to specify the DFSMSdss DASD I/O buffering technique to use for dump. With
a single START I/O instruction, DFSMSdss can read one, two, or five tracks at a time, or a
complete cylinder from the DASD that is being dumped. The syntax to set up the I/O buffer is
shown:

SETSYS DUMPIO( _n_ , _m_ )

The _n_ variable indicates the DFSMSdss DASD I/O buffering technique for the physical volume
dump (by using the **BACKVOL** command with the **DUMP** parameter or during automatic dump).
The _m_ variable indicates the value that is used for the DFSMSdss logical dump (with
DFSMSdss specified as the data mover on the **SETSYS** command).

The values that are used for the _n_ and _m_ variables are shown in Table 10-2.

_Table 10-2  SETSYS DUMPIO command parameter values_

|Value|Meaning|
|---|---|
|1|DFSMSdss reads one track at a time|
|2|DFSMSdss reads two tracks at a time|
|3|DFSMSdss reads five tracks at a time|
|4|DFSMSdss reads one cylinder at a time|



If you specify DUMPIO without a value for the _m_ variable, the _m_ variable defaults to the value
that is specified for the _n_ variable. If you do not specify the **DUMPIO** parameter on any **SETSYS**
command, the DFSMShsm default for the _n_ variable is 1 and for the _m_ variable is 4 .

The full volume dump process for each volume is based on information from DUMPCLASS,
which is a predefined DFSMShsm-named set of characteristics that describes how volume
dumps are managed. At least one DUMPCLASS must be defined in ARCCMDxx so that
automatic dump uses that information in the full volume dump process. Each storage group
that is eligible for full volume dump is assigned with a valid DUMPCLASS and this
DUMPCLASS is passed to all volumes that belong to that storage group.

DUMPCLASS is created by using the DFSMShsm **DEFINE DUMPCLASS** command in
ARCCMDxx. Because many considerations exist, we briefly explain what you can achieve
with the dump class definition, the parameters that you can specify, and the DFSMShsm
default (highlighted in bold) on the **DEFINE DUMPCLASS** command.

We show a sample definition that we use throughout this book.

For our examples, we define a dump class called _ONSITE_ . With the **DEFINE DUMPCLASS**
command, you can specify the following definitions:

- You define the name of the DUMPCLASS:

  DEFINE DUMPCLASS(ONSITE)

- You define whether dump volumes are available for reuse when they are invalidated:

  DEFINE DUMPCLASS(ONSITE AUTOREUSE | **NOAUTOREUSE** )

- You define whether data set restore is allowed from a full-volume dump copy:

  DEFINE DUMPCLASS(ONSITE DATASETRESTORE | **NODATASETRESTORE** )


-----

- You define that a dump will be taken only on a particular day in the dump cycle:

  DEFINE DUMPCLASS(ONSITE DAY( _day_ ))

  If DAY is not specified, the **FREQUENCY** parameter is met, and it is a Y day in the dump cycle,
the volume is dumped.

- You define the minimum number of days between volume dumps to a class:

  DEFINE DUMPCLASS(ONSITE FREQUENCY( _days_ | **7** ))

- You define whether DFSMSdss resets the “data set changed” indicator for each data set
after a full-volume dump:

  DEFINE DUMPCLASS(ONSITE RESET | **NORESET** )

  Do not use **RESET** if you want incremental backup to make a backup version. **RESET** is not
used if the DASD volume is also managed by DFSMShsm backup.

- You define how long you want to keep this full volume dump version:

  DEFINE DUMPCLASS(ONSITE RETENTIONPERIOD( _days_ | **NOLIMIT** ))

- You define the number of dump copies that DFSMShsm places on a dump volume that is
assigned during one invocation of automatic dump:

  DEFINE DUMPCLASS(ONSITE STACK( _nn_ ))

  The _nn_ variable is a value 1 -  255 .

   you plan to use full volume dump with the stacking feature, it is better to group all
volumes with the same volume capacity in the same storage group and assign
DUMPCLASS with the appropriate stack number.

  For example, if you set up a pool of 100 volumes to have full volume dump, which consists
of 80 model 9 volumes and 20 model 27 volumes, the dumpclass is defined with
DUMPSTACK= 20 and also MAXDUMPTASKS is 5 . DFSMShsm possibly will assign all or
most of 20 volumes of model 27 into one dump task and the others are divided evenly to
the remaining four dump tasks. As a result, it takes a longer time to finish the dump for all
20 model 27 volumes. Therefore, auto dump might not complete within its defined window.

  Alternatively, you can put all of the 80 model 9 volumes into one storage group and assign
DUMPCLASS= FDVMOD9 with DUMPSTACK= 20 and put all of the 20 model 27 volumes into
another storage group and assign DUMPCLASS= FDVMOD27 with DUMPSTACK= 5 . Then,
each of the dump tasks either gets 20 model 9 volumes or 5 model 27 volumes. Therefore,
the dump window is shorter.

- You define the type of tape unit to use for the dump tapes:

  DEFINE DUMPCLASS(ONSITE UNIT( _unittype_ ))

- You can use a generic name or an esoteric name that is defined with the **SETSYS**
**USERUNITTABLE** command in place of the unit type.

- You define the number of generations for which copies of the VTOC of dumped volumes
will be kept:

  DEFINE DUMPCLASS(ONSITE VTOCCOPIES( _copies_ | **2** ))

From the **DEFINE DUMPCLASS** parameters, we can define an example **DUMPCLASS** name _ONSITE_
in one command:

DEFINE DUMPCLASS(ONSITE UNIT(3590-1)

-  RETPD(15) AUTOREUSE STACK(13)

-  DATASETRESTORE VTOCCOPIES(4) - DAY(7))


-----

If Concurrent Copy capability is set up and you want to use it for volume dumps, use this
command syntax to make DFSMSdss use Concurrent Copy when it performs full-volume
dumps:

SETSYS VOLUMEDUMP(CC)

Contention can occur between DFSMShsm full-volume dump and DFP scratch, rename, and
allocate services. Major resource name ADRLOCK and minor resource name NONSPEC are
allocated as exclusive (EXC) when the volume is not known.

Because this process can affect dump performance, you can bypass it by using the patch:

PATCH .MCVT.+38F X’10’ VERIFY(.MCVT.+38F X’00’)

**10.2.2 Setting up automatic full-volume dump at the volume level**

For SMS-managed volumes, only volumes that belong to a storage group with an AUTO
DUMP value of Y and that are assigned a valid DUMPCLASS are eligible for full-volume
dumps.

Figure 10-4 shows the Pool Storage Group Alter panel to specify whether you want to perform
an automatic dump. Entering Y tells DFSMShsm to perform an automatic dump for all
volumes in the storage group. You assign the dump class on this panel.

                    POOL STORAGE GROUP ALTER
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : SGTEST
    To ALTER Storage Group, Specify:
    Description ==>
    ==>
    Auto Migrate . . N (Y, N, I or P)  Migrate Sys/Sys Group Name . .
    Auto Backup . . N (Y or N)     Backup Sys/Sys Group Name . .
    **Auto Dump . . . Y (Y or N)** Dump Sys/Sys Group Name . . .
    Overflow . . . . N (Y or N)     Extend SG Name . . . . . . . .
    Copy Pool Backup SG Name . . .
    
    **Dump Class . . . ONSITE** (1 to 8 characters)
    Dump Class . . .           Dump Class . . .
    Dump Class . . .           Dump Class . . .
    
    ALTER   SMS Storage Group Status . . . N  (Y or N)
    ALTER   SMA Attributes . . . . . . . . N  (Y or N)
    Use ENTER to Perform Selection; Use DOWN Command to View next Page;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.

_Figure 10-4  Pool Storage Group Alter panel for auto dump and assigning dump class_

For a non-SMS-managed volume, you can use the **ADDVOL** command:

ADDVOL _volser_ UNIT( _unittype_ ) PRIMARY(AUTODUMP ( _dumpclass_ ))

If your DFSMShsm environment consists of more than one DFSMShsm host, you can choose
the system to run the full-volume dump process by entering the system name in the Dump
Sys/Sys Group Name field in Figure 10-4.


-----

### 10.2.3 Dumping volumes by command

Issue one of the commands that are shown in Example 10-8 to cause a command dump of
one or more volumes.

_Example 10-8  Dumping volumes by command_

    BACKVOL VOLUMES( _volser,...,volser_ ) DUMP
    
    BACKVOL VOLUMES( ) DUMP(DUMPCLASS( _class,class,class,class,class_ _volser,...,volser_ ))
    
    BACKVOL VOLUMES( ) DUMP(DUMPCLASS( ) RETENTIONPERIOD( _class, . . .,class_ _volser,...,volser_ _days_ |*|NOLIMIT, . . ., _days_ |*|NOLIMIT))

To dump all of the volumes in one or more storage groups, use the following command:

BACKVOL STORAGEGROUP( _sgname_ ) DUMP(DUMPCLASS( _class_ ) STACK( _nn_ ))

### 10.2.4 Interrupting and restarting automatic dump processing

You can use the following methods to interrupt automatic dump processing during the three
phases of automatic dump:

- Automatic deletion of expired dump copies
- Automatic dump of volumes that are managed or owned by DFSMShsm
- Deletion of excess VTOC copy data sets

You can use the following four methods to interrupt automatic dump processing:

- Hold dump processing: You can hold dump processing by specifying the **HOLD** command
for the **DUMP** , **DUMP(AUTO)** , or **ALL** parameter. If you hold dump processing with the
**ENDOFDATASET** parameter while DFSMShsm is automatically deleting expired dump copies
or excess dump VTOC copy data sets, processing ends after DFSMShsm finishes
processing the current dump generation.

  If you hold dump processing with the **ENDOFVOLUME** parameter while DFSMShsm is
automatically deleting expired dump copies or excess VTOC copies, processing ends after
the current phase of automatic dump completes. If you hold dump processing with the
**ENDOFVOLUME** parameter, DFSMShsm continues processing the current volumes and does
not process any new volumes. Otherwise, DFSMShsm dump processing ends the next
time that DFSMSdss reads or writes a record. DFSMShsm does not start any new volume
dump tasks.

- Disable dump processing: You can disable dump processing by specifying the **SETSYS**
**NOBACKUP** command. If you disable dump processing while DFSMShsm is automatically
deleting expired dump copies or excess dump VTOC copy data sets, processing ends
after DFSMShsm finishes processing the current dump generation. If you disable dump
processing while DFSMShsm is dumping volumes that are managed by DFSMShsm,
DFSMShsm dump processing ends the next time that DFSMSdss reads or writes a
record. DFSMShsm does not start any new volume dump tasks.

- Place DFSMShsm in emergency mode: You can place DFSMShsm in emergency mode
by specifying the **SETSYS** **EMERGENCY** command. If you place DFSMShsm in emergency
mode while DFSMShsm is automatically deleting expired dump copies or excess dump
VTOC copy data sets, processing ends after DFSMShsm finishes processing the current
dump generation.




  If you are in a multiple DFSMShsm-host environment and enabled the secondary host
promotion function, another DFSMShsm host takes over the unique functions of the
primary DFSMShsm host that you put in emergency mode. Those functions include
automatic deletion of expired dump copies and excess dump VTOC copy data sets. If
promotion occurs during an automatic dump window (as defined on the primary host), and
it is not past the latest start time, the promoted host restarts automatic dump processing.

  If you place DFSMShsm in emergency mode while DFSMShsm is dumping volumes that
are managed by DFSMShsm, DFSMShsm dump processing ends the next time that
DFSMSdss reads or writes a record. DFSMShsm does not start any new volume dump
tasks.

- Stop DFSMShsm processing: You can stop DFSMShsm by entering the MVS or
DFSMShsm **STOP** command. If you shut down DFSMShsm while DFSMShsm is deleting
expired dump copies or excess dump VTOC copy data sets, processing ends after
DFSMShsm finishes processing the current dump generation. If you shut down
DFSMShsm while DFSMShsm is dumping volumes that are managed by DFSMShsm,
DFSMShsm dump processing ends the next time that DFSMSdss reads or writes a
record. DFSMShsm does not start any new volume dump tasks.

  If you are in a multiple DFSMShsm-host environment, enabled secondary host promotion,
and issued a **STOP PROMOTE** command, another DFSMShsm host takes over the unique
functions of the primary DFSMShsm host that you stopped. Those functions include the
automatic movement of backup versions from ML1 to tape, and the automatic backup of
migrated data sets.

  If promotion occurs during an automatic backup window (as defined on the primary host),
and it is not past the latest start time, the promoted host takes over where the primary host
left off. If the promoted host was an automatic backup host before it was promoted, it
performs backups of DFSMShsm-owned volumes after it performs the automatic
movement of backup versions from ML1 to tape, and automatic backup of migrated data
sets. If the promoted host was not an automatic backup host before it was promoted, it
does not back up DFSMShsm-owned volumes.

You can restart automatic dump processing by entering the corresponding commands:

- RELEASE DUMP or RELEASE ALL
- SETSYS BACKUP
- SETSYS NOEMERGENCY
- START DFSMShsm (MVS command)

After you restart automatic dump processing, DFSMShsm continues with the next volume
that is managed by DFSMShsm under automatic dump control. The automatic dump restarts
at the point of interruption if less than 24 hours passed since automatic dump was last started
from the beginning and if the time is still within the dump start window.

## 10.3 Recovering data from the DFSMShsm backup

The recovery and restore process, which can be considered the opposite of backup and
dump, is not an automatic process. DFSMShsm will not automatically recover or restore data
if it becomes damaged. The recover and restore process is driven by commands. You use the
recovery and restore process to perform the following tasks:

- Recover a data set that was lost or damaged
- Access an earlier version of the data set without deleting the current version


-----

The recovery process has two restrictions:

- DFSMShsm cannot recover a data set to a migration volume.

- DFSMShsm cannot recover a data set that is marked as migrated in the computing system
catalog, _unless_ the data set is non-VSAM, a recover data set name is issued with
NEWNAME specified, and the NEWNAME data set is not a migrated data set.

In this topic, we show how to perform the following functions:

- Recover the most recent version of a data set or dump volume
- Restore a volume from a dump copy and update it from incremental backup versions
- Restore a volume from a full-volume dump copy
- Recover a volume from DFSMShsm backup versions

You need to determine the maximum number of recovery tasks that your systems can afford,
based on the amount of disk storage and the number of available tape drive resources. The
actual number of effective tasks might be limited if the number of backup resources is less
than the total tasks specified. The command syntax to define this parameter is shown:

MAXDSRECOVERTASKS( _nn_ )

The _nn_ variable is a value 0 -  64 .

The value that you specify for **MAXDSRECOVERTASKS** applies to data set recovery tasks for both
DASD and tape. To limit tape requests to a subset of this value, use the keyword
**MAXDSTAPERECOVERTASKS** instead. The maximum number of tasks from DASD is the value of
**MAXDSRECOVERTASKS** minus the current number of tape tasks that are specified by
**MAXDSTAPERECOVERTASKS** . If no active tape tasks exist, the number of disk tasks can reach the
value that is specified by **MAXDSRECOVERTASKS** . To prevent tape tasks, specify **SETSYS**
**MAXDSTAPERECOVERTASKS(0)** .

### 10.3.1 Recover a data set from a backup

The data set recovery function refers to the process of recovering a data set to its condition as
of a specified date. You can recover individual data sets by entering an HRECOVER line
operator on an ISMF panel or by issuing the **HRECOVER** command. DFSMShsm can recover
data sets from a DFSMShsm backup version or from a DFSMShsm dump copy. DFSMShsm
automatically chooses the most recent copy of the data set unless directed otherwise by
options you specify with the **HRECOVER** command.

If the data set is SMS-managed at the time of recovery, the target volume is determined by
the data set’s storage class and storage group. If the data set is not SMS-managed, the target
volume is selected in the following order:

- The specified target volume
- The volume on which the target data set is cataloged
- The volume from which the data set was originally backed up or dumped

There are two ways to recover a data set:

- Recover by using the ISMF panel
- Recover by using the TSO **(H)RECOVER** command

**Recover data set by using the ISMF panel**
The following steps present an example of how to use the HRECOVER line operator to
recover a cataloged data set. In our ISMF panel example, we used MHLRESA.REPORT.JCL
as a sample data set name.


-----

From option 1 of the ISMF panel, we generate a list of data set patterns of MHLRESA.** as
shown in Figure 10-5.

                        DATA SET SELECTION ENTRY PANEL
    Command ===>
    
    For a Data Set List, Select Source of Generated List . . 2 (1 or 2)
    
    1 Generate from a Saved List     Query Name To
    List Name . .          Save or Retrieve
    
    2 Generate a new list from criteria below
    **Data Set Name . . . 'MHLRESA.**'**
    Enter "/" to select option   Generate Exclusive list
    Specify Source of the new list . . 2  (1 - VTOC, 2 - Catalog)
    1 Generate list from VTOC
    Volume Serial Number . . .      (fully or partially specified)
    Storage Group Name . . . .      (fully specified)
    2 Generate list from Catalog
    Catalog Name . . .
    Volume Serial Number . . .      (fully or partially specified)
    Acquire Data from Volume . . . . . . . N (Y or N)
    Acquire Data if DFSMShsm Migrated . . N (Y or N)
    Use ENTER to Perform Selection; Use DOWN Command to View next Selection Panel;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-5  ISMF data set selection panel_

Enter the HRECOVER line operator in the line operator column next to MHLRESA.REPORT.JCL,
as described in Figure 10-6.


-----

    DATA SET LIST
    Command ===>                         Scroll ===> HALF
    Entries 1-15 of 15
    Enter Line Operators below:               Data Columns 3-5 of 42
    
    LINE                  ALLOC   ALLOC   % NOT
    OPERATOR     DATA SET NAME     SPACE   USED   USED
    ---(1)---- ------------(2)------------ ---(3)--- ---(4)--- -(5)-
    MHLRESA           --------- ---------  ---
    MHLRESA.ABARS        --------- ---------  ---
    MHLRESA.ABARS.INSTRUCT   --------- ---------  ---
    MHLRESA.ABARS.OUTPUT    --------- ---------  ---
    MHLRESA.ABARS.SELECT    --------- ---------  ---
    MHLRESA.AUDIT.MCDS     --------- ---------  ---
    MHLRESA.BRODCAST      --------- ---------  ---
    MHLRESA.LOG.MISC      --------- ---------  ---
    **HRECOVER  MHLRESA.REPORT.JCL** --------- ---------  ---
    MHLRESA.REPORT.LIB     --------- ---------  ---
    MHLRESA.RMMREP.XMIT     --------- ---------  ---
    MHLRESA.SC64.ISPF42.ISPPROF --------- ---------  ---
    MHLRESA.SC64.SPFLOG1.LIST  --------- ---------  ---
    MHLRESA.SC70.ISPF42.ISPPROF --------- ---------  ---
    MHLRESA.VALIDATE      --------- ---------  ---
    ---------- ------ ----------- BOTTOM OF DATA ----------- ------ ----

_Figure 10-6  ISMF Data Set List panel_

Pressing Enter opens the next panel, which shows all available backup versions of data set
MHLRESA.REPORT.JCL, as shown in Figure 10-7.

                          HRECOVER ENTRY PANEL
    Command ===>
    
    Specify Y to recover a Backup Version for
    Data Set: MHLRESA.REPORT.JCL
    Recover                 Recover
    Version Date  TIME  Gen#  (Y/N) | Version Date  TIME  Gen#  (Y/N)
    --------------------------------------|---------------------------------------
    **008   2012237 141952 00   y** | 007   2012234 162156 01   N
    006   2012234 134813 02   N  | 005   2012233 193354 03   N
    004   2012233 193350 04   N  |
    |
    
    Use ENTER to Continue; Use DOWN to select other versions;
    Use HELP Command for Help; Use END Command to Cancel the HRecover.

_Figure 10-7  HRECOVER_

In Figure 10-7, we chose to recover from backup version 008. Therefore, we enter Y in the
RECOVER column.

With the next panel, you can specify several options, including whether you want to recover
the data set to a new name or you want to replace the current version on disk by a backup
version. We chose to replace the current version on disk by the most recent backup version.


-----

Therefore, we entered Y , as shown in Figure 10-8. Press Enter to perform the data set
recovery.

                           HRECOVER ENTRY PANEL
    Command ===>
    
    Optionally specify one or more for
    Data Set: MHLRESA.REPORT.JCL
    For the Backup Data Set, Specify:
    Backup Generation Number . . 00     (0 to 99)
    Backup Date . . . . . . . .       (yyyy/mm/dd)
    Backup Time . . . . . . . .       (hhmmss)
    Data Set Password . . . . .       (if password protected)
    Note: Dataset Password ignored when in ADMIN mode.
    To Recover to a specific volume, Specify:
    Volume Serial Number . . . .       (target volume)
    Device Type . . . . . . . .       (target device type)
    To rename recovered Data Set, Specify:
    New Data Set Name . . . . . 'MHLRESA.REPORT.JCL'
    Password of New Data Set . .       (if password protected)
    **Replace existing Data Set . . Y** (Y or N)
    DA Access Option . . . . . . .       (1, 2 or 3)
    Wait for completion . . . . . N      (Y or N)
    Use ENTER to perform HRecover;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-8  HRECOVER ENTRY PANEL_

**Recover by using the TSO (H)RECOVER command**
Another way to recover a data set is to issue the TSO **(H)RECOVER** command. If no data set of
the same name exists in the system catalog, you can use the following command:

H)RECOVER _dsname_

The _dsname_ is the name of the data set that you want to recover. DFSMShsm chooses the
most recently created version of the data set for you. You do not have to identify where the
most recent copy is yourself before you put the command together.

The following command recovers a specific data set but gives it a new name:

(H)RECOVER _dsname_ NEWNAME( _newdsname_ )

The _dsname_ is the name of the data set that you want to recover and _newdsname_ is the new
name for the data set that you want to recover.

You can replace an existing, cataloged data set with the recovered version by using the
following command:

(H)RECOVER _dsname_ REPLACE

If a backup exists of a data set in full-volume dump version, you can recover it by using the
following command to restore the latest dump copy from available dump volumes. If the data
set exists, you add the **REPLACE** optional parameter to restore the data set successfully.

(H)RECOVER _dsname_ FROMDUMP


-----

You can issue the following additional commands to recover a non-SMS-managed data set:

- To recover an uncataloged data set to the volume from which it was backed up, execute
the following command:

(H)RECOVER _dsname_ FROMVOLUME( _volser_ )

- You can also add the **REPLACE** parameter to the command to replace the existing
uncataloged data set of the same name on that volume. Or, add the **NEWNAME(** **_newdsname_** **)**
parameter to give it a new name.

- To recover a data set to a specified volume that is different from the original volume,
execute the following command:

(H)RECOVER _dsname_ TOVOLUME( _volser_ ) - UNIT( _unittype_ )

You can also use the preceding command to recover an SMS-managed data set to a
non-SMS-managed volume by adding **FORCENONSMS** parameter.

- To recover a data set to the specified volume and give the data set a new name, execute
the following command:

(H)RECOVER _dsname_ NEWNAME( _newdsname_ ) - TOVOLUME( _volser_ ) UNIT( _unittype_ )

You can also use the preceding command to recover an SMS-managed data set to a
non-SMS-managed volume by adding **FORCENONSMS** parameter.

### 10.3.2 Restoring a volume from a full volume dump

The methods to restore volumes from a full volume restore are described.

**Full-volume restore with incremental backup**
When you recover a volume with full-volume restore and incremental backup versions, one of
your users might use a data set in the interval between the restore and the incremental
recovery. Avoid this situation by putting the SMS-managed volume in DISALL status. You
need to be aware of two considerations before you place the volume in DISALL status:

- With the DISALL status, data sets might be recovered (as directed by your automatic class
selection (ACS) routines) to different volumes in the storage group that contain the volume
that is being recovered.

- If you are recovering a catalog, or a volume that belongs to a storage group that uses the
guaranteed space attribute, the volume must be in the enabled (ENABLE) status during
the recovery process.

Ensure that the SMS-managed volume is in the disabled (DISALL) status before you issue
the **RECOVER** command, unless you are recovering a catalog, or a volume from a storage group
that uses the guaranteed space attribute.

After the recovery process completes, be sure to review the messages for failures or data
sets that were not recovered. For example, if the volume that you are recovering contains part
of a multivolume data set, that partial data set is not recovered. After the recovery process
completes, the data set is listed as not being recovered, which gives you the opportunity to
recover the entire data set, including the parts of the data set that reside on volumes that are
not being recovered. You can issue individual **RECOVER** **_dsn_** commands for these data sets.


-----

Example 10-9 shows the commands that you can use to combine a full-volume restore with a
full-volume recovery.

_Example 10-9  Commands to combine a full-volume restore with a full-volume recovery_

    RECOVER * TOVOLUME(original_volser) UNIT(unittype) - FROMDUMP(DUMPVOLUME(tape_volser) APPLYINCREMENTAL)
    
    RECOVER * TOVOLUME(original_volser) UNIT(unittype) - FROMDUMP(DUMPCLASS(class) APPLYINCREMENTAL)
    
    RECOVER * TOVOLUME(original_volser) UNIT(unittype) - FROMDUMP(DUMPGENERATION(dgennum) APPLYINCREMENTAL)


When **APPLYINCREMENTAL** is specified on a **RECOVER FROMDUMP** command, DFSMShsm first
performs a full-volume restore. After the volume restore completes, DFSMShsm performs the
following steps:

- If ICF catalogs reside on the volume, DFSMShsm recovers any catalog for which a backup
version exists that was created following the dump. All of these catalogs must be
successfully recovered for the APPLYINCREMENTAL process to continue. The recovery
of the catalog fails and the APPLYINCREMENTAL process will not continue for either or
both of the following conditions:

  - If any catalog was backed up by DFSMShsm operating with a version of MVS/DFP that
is earlier than version 2 release 3.0

  - If any catalog was dumped before it was exported or backed up by DFSMShsm, which
uses export

- DFSMShsm creates a list of candidate data sets from either or both the dump and backup
VTOC copy data sets to control the volume recovery process.

- DFSMShsm applies incremental data set backups from the list of candidate data sets with
the following exceptions:

  - Catalogs.

  - Data sets that are cataloged as multiple-volume, including multiple-stripe data sets. If
the target volume is SMS-managed, the data set is not currently cataloged on the
volume that is being processed, and the data set was restored to the volume,
DFSMShsm scratches it from the restored volume.

  - VSAM data sets that are currently cataloged as MIGRAT. Cluster, index, and data
components are scratched from the restored volume.

  - Non-VSAM data sets that are currently cataloged as MIGRAT but the dump VTOC
copy data set is more recent than the backup VTOC copy data set. If the data set was
restored to the volume, DFSMShsm scratches it from the restored volume.

  - VSAM volume data sets (VVDSs).

  - VTOC index data sets.

  - Data sets that are not on the backup VTOC copy data set but the backup VTOC copy
data set is more recent than the dump VTOC copy data set. If the data set was
restored to the volume, DFSMShsm scratches it from the restored volume.

  - Currently uncataloged VSAM data sets. If the data set was restored to the volume,
DFSMShsm scratches it from the restored volume.

  - Currently uncataloged non-VSAM data sets that are targeted to an SMS-managed
volume. If the data set was restored to the volume, DFSMShsm scratches it from the
restored volume.


-----

  - ICF VSAM component data sets.

  - Data sets that are currently cataloged as MIGRAT, that were cataloged when they were
backed up, and that are selected for recovery. If the data set was restored to the
volume, DFSMShsm scratches it from the restored volume.

  - Data sets that are currently cataloged as MIGRAT, that were uncataloged when they
were backed up, that are selected for recovery, and that were migrated after the backup
version of the uncataloged data set was made from the same volume. If the data set
was restored to the volume, DFSMShsm scratches it from the restored volume.

  - Data sets that are currently cataloged on another volume but the selected data set was
cataloged when it was backed up. If the data set was restored to the volume,
DFSMShsm scratches it from the restored volume.

  - Data sets that were last backed up when they were cataloged, but a backup version of
a data set that was uncataloged when it was backed up is selected for recovery. If the
data set was restored to the volume, DFSMShsm scratches it from the restored
volume.

  - Data sets that are cataloged on the non-SMS-managed target volume but the backup
version was made when the data set was uncataloged. (If the data set was
uncataloged when it was backed up from the target volume and is now cataloged on
the target volume, and the volume is now SMS-managed, recovery is allowed.) If the
data set was restored to the volume, DFSMShsm scratches it from the restored
volume.

  - Data sets that are not SMS-managed that are being recovered to SMS-managed
volumes.

**Full-volume restore from a dump copy**
_Full-volume restore_ is the process of recovering the entire physical contents of a volume.
Full-volume restore, an alternative to the DFSMShsm volume recover function, invokes Data
Facility Storage Management Subsystem Data Set Services (DFSMSdss) to perform
full-volume restores.

DFSMShsm uses the dump volumes of a dump copy as input to a DFSMSdss full-volume
restore request. If the dump copy starts in file two or beyond on a stacked dump tape,
DFSMShsm supplies the file block ID for high-speed positioning to the start of the dump copy.
Example 10-10 shows the commands that can be used to request a full-volume restore.

_Example 10-10  Commands to request a full-volume dump_

    RECOVER * TOVOLUME(original_volser) UNIT(unittype) - FROMDUMP
    
    RECOVER * TOVOLUME(original_volser) UNIT(unittype) - FROMDUMP DATE(date)
    
    RECOVER * TOVOLUME(original_volser) UNIT(unittype) - FROMDUMP(DUMPVOLUME(tape_volser))
    
    RECOVER * TOVOLUME(original_volser) UNIT(unittype) - FROMDUMP(DUMPCLASS(class))
    
    -----
    
    RECOVER * TOVOLUME(original_volser) UNIT(unittype) - FROMDUMP(DUMPCLASS(class)) DATE(yyyy/mm/dd)
    
    RECOVER * TOVOLUME(original_volser) UNIT(unittype) - FROMDUMP(DUMPGENERATION(dgennum)) -

    TARGETVOLUME(spare_volser)

### 10.3.3 Volume recovery from incremental backup versions

Normally, volume recovery is run when an entire DASD pack is lost or damaged or a
significant amount of the data is inaccessible. A copy of the VTOC that was made during the
incremental backup of the volume is used to drive the individual incremental recovery of data
sets for which DFSMShsm has backup versions. DFSMShsm volume recovery is also
referred to as _incremental volume recovery_ . After DFSMShsm successfully recovers a
volume, each supported data set on the recovered volume is at the level of its latest backup.
This process recovers the data set to the most recent level unless someone changed the data
set since the last time DFSMShsm backed it up.

The following commands cause volume recovery from incremental backup copies, as shown
in Example 10-11.

_Example 10-11  Volume recovery from incremental backup copies_

    RECOVER * TOVOLUME( _volser_ ) UNIT( _unittype_ )
    RECOVER * TOVOLUME( _volser_ ) UNIT( _unittype_ ) DATE( _date_ )

Volume recovery is similar to data set recovery in that DFSMShsm recovers each data set;
DFSMShsm does not recover data track by track. First, a single task builds the queue of data
sets that need to be recovered, then multitasking is used during the actual recovery of the
data sets. If recovering multiple volumes, it is most efficient to first put a hold on tape data set
recovers, create the queue of data sets to be recovered, and then release the tape data set
recovery processing.

Because data sets from different volumes might reside on one tape, recovering data sets for
multiple volumes at the same time, rather than volume-by-volume, reduces the number of
required tape mounts, and therefore speeds processing. However, because DFSMShsm
recovers each data set and because the backup operation uses backup volumes from
different days, volume recovery can require access to many backup volumes only to recover
one level 0 volume. As a result, volume recovery can be time-consuming. Consider the use of
the DFSMShsm full-volume dump restore facility to reduce the time that is required for
recovery.

You can specify that backup versions be at least as recent as a particular date by specifying
the **DATE** parameter of the **RECOVER** command. You must use the **TOVOLUME** parameter to
identify an entire volume that you want to recover.

DFSMShsm uses the _latest_ VTOC copy data set to recover the volume unless an error occurs
in allocating or opening the latest VTOC copy data set. If the error occurs, DFSMShsm
attempts to recover the volume by using the _next latest_ copy of the VTOC copy data set. If a
similar error occurs again, the volume recovery fails.


-----

## 10.4 Aggregate backup and recovery support

Aggregate backup and recovery support (ABARS) is a component of DFSMShsm that
performs data backup processes on a predefined set of data that is known as an _aggregate_
and recovers at another computer site or at the same site. ABARS can be used to back up
and recover both SMS and non-SMS-managed data.

In general, to use aggregate backup, you first specify the data sets to be backed up, and then
use the **ABACKUP** command or ISMF panels to back up the data sets to tape files. These tape
files can then be physically transported or transmitted by using a transmission program, such
as the IBM Tivoli NetView® for z/OS File Transfer Program (FTP), to a recovery site where the
data sets can be recovered.

Only a host that is designated with the HOSTMODE=MAIN attribute can execute aggregate
backup or aggregate recovery.

The following functions are shown:

- Define an aggregate group
- Define a selection data set
- Define an instruction data set
- Set your ABARS-related **SETSYS** commands
- Perform an aggregate backup (ABACKUP)
- Perform an aggregate recover (ARECOVER)

### 10.4.1 Define an aggregate group

Before you can run aggregate backup, you need to create one or more selection data sets
and define an aggregate group and related management class to specify exactly which data
sets to back up. You can also create an instruction data set to contain any information you
want conveyed to the recovery site. One instruction data set name and up to five selection
data set names can be defined within one aggregate group.

An aggregate group can be created through ISMF with your selections for data set name,
instruction data set name, and the management control information that is required to perform
ABACKUP .

Figure 10-9 on page 281 shows the ISMF Aggregate Group Application Selection panel that
is displayed after you enter option 9 on the ISMF main entry panel.


-----

                       AGGREGATE GROUP APPLICATION SELECTION
    Command ===>
    
    To Perform Aggregate Group Operations, Specify:
    CDS Name . . . . . . . . 'SYS1.SMS.MHLRES3.SCDS'
    (1 to 44 Character Data Set Name or 'Active')
    Aggregate Group Name . . **MHLRESA** (for Aggregate Group List, fully or
    Partially Specified or * for All)
    Select one of the following Options:
    **3** 1. List   - Generate a list of Aggregate Groups
          2. Display - Display an Aggregate Group
          3. Define  - Define an Aggregate Group
          4. Alter  - Alter an Aggregate Group
          5. Abackup - Backup an Aggregate Group
          6. Arecover - Recover an Aggregate Group
    
    If List Option is Chosen,
    Enter "/" to select option   Respecify View Criteria
    Respecify Sort Criteria
    
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-9  ISMF Aggregate Group Application Selection panel_

We chose to call our aggregate group, _MHLRESA_ . After you type a name for your aggregate
group and press Enter, you are presented with the panels that are shown in Figure 10-10 and
Figure 10-11 on page 282.

                            AGGREGATE GROUP DEFINE
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Aggregate Group Name : MHLRESA
    
    To DEFINE Aggregate Group, Specify:
    Description ==> ABARS example
    ==>
    
    Backup Attributes
    Number of Copies . . . . . . . 1        (1 to 15)
    Management Class Name . . . . . MNDB22     (1 to 8 characters, to be
    defined in current SCDS)
    
    Output Data Set Name Prefix . . . MHLRESA
    (1 to 33 Characters)
    Account . . . . . . . . . . . . .
    (1 to 32 Characters)
    
    Use ENTER to Perform Verification; Use DOWN Command to View next Panel;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.

_Figure 10-10  Aggregate Group Define (Page 1 of 2)_


-----

The management class specifies the parameters, such as the maximum number of aggregate
versions to keep, the aggregate group retention parameters, the serialization option, and
whether Concurrent Copy is requested.

The prefix identifies the output data sets that are created by aggregate backup. These output
data sets are created with the following naming conventions:

- prefix.D.C _nn_ V _nnnn_ and outputdatasetprefix.O.C _nn_ V _nnnn_ for the data file
- prefix.C.C _nn_ V _nnnn_ for the control file
- prefix.I.C _nn_ V _nnnn_ for the instruction/activity log file

Because they all share a common output data set prefix and version number, it is easier to
identify all of the output data sets from one aggregate backup.

In our example, the prefix is MHLRESA. The data, control, and instruction/activity log files are
named MHLRESA.D.C01V0001 (for the data file), MHLRESA.O.C01V0001 (for the control file), and
MHLRESA.C.C01V0001 and MHLRESA.I.C01V0001 (for the instruction/activity files). _Cnn_ is the
copy number and _Vnnnn_ is the version number that are generated during the ABACKUP
operation.

Figure 10-11 shows the second panel of the Aggregate Group Define process.

                                 AGGREGATE GROUP DEFINE
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Aggregate Group Name : MHLRESA
    To Edit a Data Set, Specify Number . . **1** (1, 2, 3, 4, 5, or 6)
    Selection Data Sets:      (1 to 44 characters)
    1 . . **'MHLRESA.ABARS.SELECT'**
    Member Name . . **GROUP1** (1 to 8 characters)
    2 . .
    Member Name . .       (1 to 8 characters)
    3 . .
    Member Name . .       (1 to 8 characters)
    4 . .
    Member Name . .       (1 to 8 characters)
    5 . .
    Member Name . .       (1 to 8 characters)
    
    Instruction Data Set:      (1 to 44 characters)
    6 . . 'MHLRESA.ABARS.INSTRUCT'
    Use ENTER to Perform Verification; Use UP Command to View previous Panel;

_Figure 10-11  Aggregate Group Define (Page 2 of 2)_

In Figure 10-11, we chose to describe our aggregate group, MHLRESA, make one copy of
the output files, take our management class definitions from management class MNDB22,
and use MHLRESA as the data set prefix for the output data sets that are created by
ABACKUP .


-----

### 10.4.2 Define selection and instruction data set

A _selection data set_ lists the names of the data sets to be processed during ABACKUP . You
can create a selection data set by using the Aggregate Group Alter panel, or you can
predefine one yourself. Your selection data set must have at least one INCLUDE, ALLOCATE,
or ACCOMPANY data set. We show how to set up a basic aggregate with an INCLUDE
statement.

In Figure 10-11 on page 282, we chose to call our selection data set,
_MHLRESA.ABARS.SELECT_ and our instruction data set is called
_MHLRESA.ABARS.INSTRUCT_ .

To create the data set MHLRESA.ABARS.SELECT and edit member GROUP1, enter option
1 in the “To Edit a Data Set, Specify Number” field, as shown in Figure 10-11 on page 282.
ISMF creates the data set and displays member GROUP1 for you to edit, as shown in
Figure 10-12.

    EDIT    MHLRESA.ABARS.SELECT(GROUP1) - 01.00      Columns 00001 00072
    Command ===>                         Scroll ===> PAGE
    ****** ***************************** Top of Data ******************************
    ==MSG> -Warning- The UNDO command is not available until you change
    ==MSG>      your edit profile using the command RECOVERY ON.
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ''''''
    ****** **************************** Bottom of Data ****************************

_Figure 10-12  Editing panel in ISMF_

We selected the data set pattern of MHLRESA.REPORT.** as the candidates for the ABARS
process. Therefore, we create an INCLUDE statement in that member, as shown in
Figure 10-13.

    EDIT    MHLRESA.ABARS.SELECT(GROUP1) - 01.00      Columns 00001 00072
    Command ===>                         Scroll ===> PAGE
    ****** ***************************** Top of Data ******************************
    000100 INCLUDE(MHLRES%.REPORT.**)
    ****** **************************** Bottom of Data ****************************

_Figure 10-13  Editing the member in ISMF_


-----

ABACKUP uses this statement to select cataloged data sets with a high-level qualifier (HLQ)
of MHLRESA and a second-level qualifier of REPORT. The instruction data set is an optional
data set that is free-form text that is meant to include the necessary information to assist with
the recovery process at the recovery site.

It might be useful to include the following types of information:

- SMS attributes
- Description of the application
- RACF environment
- Software requirements
- Hardware requirements
- Unique application dependencies

You can create and edit the MHLRESA.ABARS.INSTRUCT data set in the same way as the
preceding selection data set.

You can specify other parameters in your selection data set. For more information, see
_DFSMShsm Storage Administration Guide,_ SC26-0421, for comprehensive details about
additional optional parameters, such as the following parameters:

- EXCLUDE
- ACCOMPANY
- ACCOMPANYEXCLUDE
- ALLOCATE
- ALLOCATEEXCLUDE

### 10.4.3 Defining SETSYS parameters for aggregate backup

We do not go into detail about each option. Example 10-12 shows the considerations if you
decide that you want to implement ABARS.

_Example 10-12  Example of SETSYS parameters of ABARS_

    SETSYS                       /* START ONLY ONE SECONDARY ADDRESS */ -
      MAXABARSADDRESSSPACE(1)    /* SPACE FOR BACKING UP AND     */
                                 /* RECOVERING AGGREGATED DATA SETS  */
    
    SETSYS                       /* START THE SECONDARY ADDRESS    */ -
      ABARSPROCNAME(DFHSMABR)    /* SPACE WITH THE STARTUP PROCEDURE */
                                 /* NAMED DFHSMABR.          */
    
    SETSYS                       /* WRITE THE ABARS ACTIVITY LOG TO  */ -
      ABARSACTLOGTYPE(DASD)      /* DASD               */
    
    SETSYS                       /* LOG ALL ABARS MESSAGES      */ -
      ABARSACTLOGMSGLVL(FULL)
    
    SETSYS                       /* RECOVER ML2 DATA SETS TO TAPE.  */ -
      ARECOVERML2UNIT(3590-1)
    
    SETSYS                       /* USE 90% OF THE AVAILABLE TAPE FOR */ -
      ARECOVERPERCENTUTILIZED(090) /* ARECOVERY TAPES.        */
    
    SETSYS                       /* BACKUP AGGREGATES TO 3590-1    */ -
      ABARSUNITNAME(3590-1)      /* DEVICES.
    
    
    -----
    
    SETSYS                     /* BACKUP ABARS DATA SETS WITH TWO  */ -
      ABARSBUFFERS(2)          /* DATA MOVEMENT BUFFERS.      */
    
    SETSYS                     /* SPECIFY ABARS TO STACK THE    */ -
      ABARSTAPES(STACK)        /* ABACKUP OUTPUT ONTO A MINIMUM   */
                               /* NUMBER OF TAPE VOLUMES      */
    
    SETSYS                     /* ABARS ACTIVITY LOG WILL NOT BE  */ -
      ABARSDELETEACTIVITY(N)   /* AUTOMATICALLY DELETED DURING   */
                               /* ABARS PROCESSING         */
    
    SETSYS                     /* SET PERFORMANCE OF BACKING UP   */ -
      ABARSOPTIMIZE(3)         /* LEVEL 0 DASD DATASETS       */
    
    SETSYS                     /* TARGET DATASET IS TO BE ASSIGNED */ -
      ARECOVERTGTGDS(SOURCE)   /* SOURCE STATUS           */
    
    SETSYS                     /* ALLOWS RECOVERY OF A LEVEL 0   */ -
      ABARSVOLCOUNT(ANY)       /* DASD DATA SET UP TO 59 VOLUMES  */
                               /*                                */
    
    SETSYS            /* SKIP L0 DATA SETS PROTECTED    */  ABARSKIP(PPRC XRC)     /* BY PPRC OR XRC          */

For more information about the **SETSYS** parameters for ABARS, see _DFSMShsm_
_Implementation and Customization Guide,_ SC35-0418, and _DFSMShsm Storage_
_Administration Guide,_ SC26-0421.

You must also define a procedure for DFSMShsm to use. You must add this procedure to
SYS1.PROCLIB. It is not necessary to issue an **MVS** **START** command for the procedure
(DFHSMABR in our case). DFSMShsm schedules the start internally when an ABACKUP or
ARECOVER task starts. Example 10-13 shows an ABARS procedure JCL.

_Example 10-13  ABARS procedure example_

     //*****************************************************************/
     //* ABARS SECONDARY ADDRESS SPACE STARTUP PROCEDURE */
     //*****************************************************************/
     //*
     //DFHSMABR PROC
     //DFHSMABR EXEC PGM=ARCWCTL,REGION=0M
     //SYSUDUMP DD SYSOUT=A
     //MSYSIN DD DUMMY
     //MSYSOUT DD SYSOUT=A
     /*

After you define the data sets that you want to back up in your selection data set, you can use
the DFSMShsm **ABACKUP** command or ISMF panels to back up the data sets to tape.

To activate the new definitions, you must activate the newly configured SCDS by copying the
SCDS into the ACDS. Enter option 5 from the CDS Application Selection panel, as shown in
Figure 10-14 on page 286.


-----

                          CDS APPLICATION SELECTION
    Command ===>
    
    To Perform Control Data Set Operations, Specify:
    CDS Name . . 'SYS1.SMS.MHLRES3.SCDS'
                              (1 to 44 Character Data Set Name or 'Active')
    
    Select one of the following Options:
    **5** 1. Display    - Display the Base Configuration
          2. Define    - Define the Base Configuration
          3. Alter     - Alter the Base Configuration
          4. Validate   - Validate the SCDS
          5. **Activate   - Activate the CDS**
          6. Cache Display - Display CF Cache Structure Names for all CF Cache Sets
          7. Cache Update - Define/Alter/Delete CF Cache Sets
          8. Lock Display - Display CF Lock Structure Names for all CF Lock Sets
          9. Lock Update  - Define/Alter/Delete CF Lock Sets
    If CACHE Display is chosen, Enter CF Cache Set Name . . *
    If LOCK Display is chosen, Enter CF Lock Set Name . . . *
                                  (1 to 8 character CF cache set name or * for all)
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-14  CDS Application Selection panel_

### 10.4.4 Perform an aggregate backup (ABACKUP)

After you successfully activate the new definition, you can use ABARS functions for the
aggregate group MHLRESA.

To perform an aggregate backup, enter option 5 , as shown in Figure 10-15 on page 287.


-----

                  AGGREGATE GROUP APPLICATION SELECTION
    Command ===>
    
    To Perform Aggregate Group Operations, Specify:
       CDS Name . . . . . . . . 'ACTIVE'
                                      (1 to 44 Character Data Set Name or 'Active')
       Aggregate Group Name . . MHLRESA  (for Aggregate Group List, fully or
                                         Partially Specified or * for All)
    Select one of the following Options:
    **5** 1. List   - Generate a list of Aggregate Groups
          2. Display - Display an Aggregate Group
          3. Define  - Define an Aggregate Group
          4. Alter  - Alter an Aggregate Group
          5. **Abackup - Backup an Aggregate Group**
          6. Arecover - Recover an Aggregate Group
    
    If List Option is Chosen,
      Enter "/" to select option   Respecify View Criteria
                                   Respecify Sort Criteria
    
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-15  Perform ABACKUP by using the ISMF panel_

The Aggregate Group Backup panel displays, as shown in Figure 10-16.

                       AGGREGATE GROUP BACKUP
  Command ===>
  
  CDS Name : ACTIVE
  
    Aggregate Group Name . . MHLRESA
  
    Unit Name . . . . . . . . 3590-1
    Processing Option . . . . 2          (1=Verify, 2=Execute)
    Wait for Completion . . . N          (Y or N)
    Stack / Nostack . . . . . N          (S=STACK, N=NOSTACK or blank)
    Optimize . . . . . . . .           (1 to 4 or blank)
    Delete Data Sets After Abackup . . N     (Y or N)
  
    Filter Output Data Set Name (1 to 44 Characters)
      ===> ABARS
  
    Enter "/" to select option
    Process only         L0     ML1    ML2    USERTAPE
  
  Use ENTER to Continue;
  Use HELP Command for Help; Use END Command to Exit.

_Figure 10-16  Aggregate Group Backup (Page 1 of 2)_

The second Aggregate Group Backup panel displays, as shown in Figure 10-17.


-----

                         AGGREGATE GROUP BACKUP
    Command ===>
    
    DFSMShsm Command and Processing Option:
    NOWAIT ABACKUP MHLRESA UNIT(3590-1) EXECUTE NOSTACK
    FILTEROUTPUTDATASET('MHLRESA.ABARS')
    
    Enter 1 to Submit DFSMShsm ABACKUP COMMAND
    Enter 2 to Save Generated ABACKUP PARAMETERS
    
     Select Option . . 1             (1=SUBMIT, 2=SAVE)
    
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-17  Aggregate Group Backup (Page 2 of 2)_

In Figure 10-17, we chose to run ABACKUP directly by entering option 1 in the Select Option
field to submit the DFSMShsm **ABACKUP** command.

After the ABACKUP process completes, you can view the ABACKUP log data set, as shown
in Example 10-14.

_Example 10-14  ABACKUP log_

    PAGE 0001 Z/OS DFSMSHSM 1.13.0 DATA FACILITY HIERARCHICAL STORAGE MANAGER 12.2
    ARC6000I ABACKUP MHLRESA EXECUTE UNIT(3590-1) FILTEROUTPUTDATASET('MHLRESA.ABARS
    ARC6054I AGGREGATE BACKUP STARTING FOR AGGREGATE GROUP MHLRESA, AT 12:16:39, STA
    ARC6030I ACTIVITY LOG FOR AGGREGATE GROUP MHLRESA WILL BE ROUTED TO HSMACT.H2.AB
    ARC6379I THE MANAGEMENT CLASS CONSTRUCTS USED IN THE AGGREGATE GROUP, MHLRESA, A
    CLASS NAME : MCDB22
    DESCRIPTION:
    MGMT CLASS FOR DB2 TS
    EXPIRATION ATTRIBUTES
      EXPIRE AFTER DAYS NON-USAGE: NOLIMIT
      EXPIRE AFTER DATE/DAYS   : NOLIMIT
    RETENTION LIMIT: NOLIMIT
    PARTIAL RELEASE: N
    MIGRATION ATTRIBUTES
      PRIMARY DAYS NON-USAGE: 7
      LEVEL 1 DAYS NON-USAGE: 20
      COMMAND/AUTO MIGRATE : BOTH
    BACKUP ATTRIBUTES
      BACKUP FREQUENCY         : 1
      # OF BACKUP VERSIONS (DS EXISTS) : 1
      # OF BACKUP VERSIONS (DS DELETED): 0
      RETAIN DAYS ONLY BACKUP VERSION : 1
    
    
    -----
    
          (DS DELETED)
    RETAIN DAYS EXTRA BACKUP VERSIONS: 1
      ADMIN OR USER COMMAND BACKUP   : BOTH
      AUTO BACKUP           : Y
      AGGREGATE BACKUP COPY TECHNIQUE : S
    
    CLASS NAME : MC54NMIG
    DESCRIPTION:
    MANAGEMENT CLASS FOR NO MIGRATION
    EXPIRATION ATTRIBUTES
      EXPIRE AFTER DAYS NON-USAGE: NOLIMIT
      EXPIRE AFTER DATE/DAYS   : NOLIMIT
    RETENTION LIMIT: NOLIMIT
    PARTIAL RELEASE: C
    MIGRATION ATTRIBUTES
      PRIMARY DAYS NON-USAGE: 0
      LEVEL 1 DAYS NON-USAGE: 9999
      COMMAND/AUTO MIGRATE : BOTH
    BACKUP ATTRIBUTES
      BACKUP FREQUENCY         : 1
      # OF BACKUP VERSIONS (DS EXISTS) : 2
      # OF BACKUP VERSIONS (DS DELETED): 1
      RETAIN DAYS ONLY BACKUP VERSION : 60
           (DS DELETED)
      RETAIN DAYS EXTRA BACKUP VERSIONS: 30
    ADMIN OR USER COMMAND BACKUP   : BOTH
      AUTO BACKUP           : Y
      AGGREGATE BACKUP COPY TECHNIQUE : S
    
    ARC6379I THE STORAGE CLASS CONSTRUCTS USED IN THE AGGREGATE GROUP, MHLRESA, ARE:
    CLASS NAME : STANDARD
    DESCRIPTION:
    PAGE 0002 Z/OS DFSMSHSM 1.13.0 DATA FACILITY HIERARCHICAL STORAGE MANAGER 12.2
    STORAGE CLASS FOR AVERAGE RESPONSE TIME
    AVAILABILITY: STANDARD
    ACCESSIBILITY: S
    GUARANTEED SPACE: N
    GUARANTEED SYNCHRONOUS WRITE: N
    
    ARC6004I 00D0 ABACKUP PAGE 0001   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVIC
    ARC6004I 00D0 ABACKUP ADR035I (SCH)-PRIME(06), INSTALLATION EXIT ALTERED BYPASS
    ARC6004I 00D0 ABACKUP DUMP DATASET(FILTERDD(SYS00005)) -
    ARC6004I 00D0 ABACKUP OUTDDNAME(SYS00004 -
    ARC6004I 00D0 ABACKUP         ) OPTIMIZE(3) SPHERE
    ARC6004I 00D0 ABACKUP ALLDATA(*) FORCECP(0) -
    ARC6004I 00D0 ABACKUP SHARE TOLERATE(ENQFAILURE)
    ARC6004I 00D0 ABACKUP ADR101I (R/I)-RI01 (01), TASKID 001 HAS BEEN ASSIGNED TO C
    ARC6004I 00D0 ABACKUP ADR109I (R/I)-RI01 (01), 2012.240 12:16:59 INITIAL SCAN OF
    ARC6004I 00D0 ABACKUP ADR050I (001)-PRIME(01), DFSMSDSS INVOKED VIA APPLICATION
    ARC6004I 00D0 ABACKUP ADR016I (001)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    ARC6004I 00D0 ABACKUP ADR006I (001)-STEND(01), 2012.240 12:16:59 EXECUTION BEGIN
    ARC6004I 00D0 ABACKUP ADR411W (001)-DTDSC(01),
    ARC6004I 00D0 ABACKUP DATA SET MHLRESA.SC64.SPFLOG1.LIST IN CATALOG UCAT.VSBOX01
    ARC6004I 00D0 ABACKUP REQUEST
    ARC6004I 00D0 ABACKUP ADR804W (001)-DTDSC(01),
    
    
    -----
    
    ARC6004I 00D0 ABACKUP EOF FOR DATA SET MHLRESA.SC64.SPFLOG1.LIST IN CATALOG UCAT
    ARC6004I 00D0 ABACKUP ALL
    ARC6004I 00D0 ABACKUP             ALLOCATED SPACE WILL BE PROCESSED
    ARC6004I 00D0 ABACKUP ADR801I (001)-DTDSC(01),
    ARC6004I 00D0 ABACKUP 2012.240 12:17:10 DATA SET FILTERING IS COMPLETE. 14 OF 14
    ARC6004I 00D0 ABACKUP SERIALIZATION
    ARC6004I 00D0 ABACKUP             AND 0 FAILED FOR OTHER REASONS
    ARC6004I 00D0 ABACKUP ADR454I (001)-DTDSC(01), THE FOLLOWING DATA SETS WERE SUCC
    ARC6004I 00D0 ABACKUP              MHLRESA.AUDIT.MCDS
    ARC6004I 00D0 ABACKUP              MHLRESA.SC70.ISPF42.ISPPROF
    ARC6004I 00D0 ABACKUP              MHLRESA.ABARS.INSTRUCT
    ARC6004I 00D0 ABACKUP              MHLRESA.LOG.MISC
    ARC6004I 00D0 ABACKUP              MHLRESA.BRODCAST
    ARC6004I 00D0 ABACKUP              MHLRESA.REPORT.JCL
    ARC6004I 00D0 ABACKUP              MHLRESA.ABARS.SELECT
    ARC6004I 00D0 ABACKUP              MHLRESA.SC64.ISPF42.ISPPROF
    ARC6004I 00D0 ABACKUP              MHLRESA.REPORT.LIB2
    ARC6004I 00D0 ABACKUP              MHLRESA.SC64.SPFLOG1.LIST
    ARC6004I 00D0 ABACKUP              MHLRESA.ABARS.OUTPUT
    ARC6004I 00D0 ABACKUP              MHLRESA.ABARS
    ARC6004I 00D0 ABACKUP              MHLRESA.RMMREP.XMIT
    ARC6004I 00D0 ABACKUP              MHLRESA.VALIDATE
    ARC6004I 00D0 ABACKUP ADR006I (001)-STEND(02), 2012.240 12:17:16 EXECUTION ENDS
    ARC6004I 00D0 ABACKUP ADR013I (001)-CLTSK(01), 2012.240 12:17:21 TASK COMPLETED
    ARC6004I 00D0 ABACKUP ADR012I (SCH)-DSSU (01),
    ARC6004I 00D0 ABACKUP 2012.240 12:17:21 DFSMSDSS PROCESSING COMPLETE. HIGHEST RE
    ARC6004I 00D0 ABACKUP             TASK  001
    ARC6004I 00D0 ABACKUP PAGE 0001   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVIC
    ARC6004I 00D0 ABACKUP ADR035I (SCH)-PRIME(06), INSTALLATION EXIT ALTERED BYPASS
    ARC6004I 00D0 ABACKUP DUMP DATASET(INCLUDE( -
    PAGE 0003 Z/OS DFSMSHSM 1.13.0 DATA FACILITY HIERARCHICAL STORAGE MANAGER 12.2
    ARC6004I 00D0 ABACKUP  MHLRESA.ABARS.INSTRUCT           ,  -
    ARC6004I 00D0 ABACKUP  HSMACT.H2.ABACKUP.MHLRESA.D12240.T121639   )) -
    ARC6004I 00D0 ABACKUP OUTDDNAME(SYS00017 -
    ARC6004I 00D0 ABACKUP         ) OPTIMIZE(3) SPHERE
    ARC6004I 00D0 ABACKUP ALLDATA(*) FORCECP(0) -
    ARC6004I 00D0 ABACKUP SHARE TOLERATE(ENQFAILURE)
    ARC6004I 00D0 ABACKUP ADR101I (R/I)-RI01 (01), TASKID 001 HAS BEEN ASSIGNED TO C
    ARC6004I 00D0 ABACKUP ADR109I (R/I)-RI01 (01), 2012.240 12:17:41 INITIAL SCAN OF
    ARC6004I 00D0 ABACKUP ADR050I (001)-PRIME(01), DFSMSDSS INVOKED VIA APPLICATION
    ARC6004I 00D0 ABACKUP ADR016I (001)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    ARC6004I 00D0 ABACKUP ADR006I (001)-STEND(01), 2012.240 12:17:41 EXECUTION BEGIN
    ARC6004I 00D0 ABACKUP ADR801I (001)-DTDSC(01),
    2012.240 12:17:48 DATA SET FILTERING IS COMPLETE. 2 OF 2 DATA SETS WERE SELECTE
    ARC6004I 00D0 ABACKUP             AND 0 FAILED FOR OTHER REASONS
    ARC6004I 00D0 ABACKUP ADR454I (001)-DTDSC(01), THE FOLLOWING DATA SETS WERE SUCC
    ARC6004I 00D0 ABACKUP              MHLRESA.ABARS.INSTRUCT
    ARC6004I 00D0 ABACKUP              HSMACT.H2.ABACKUP.MHLRESA.D12240
    ARC6004I 00D0 ABACKUP ADR006I (001)-STEND(02), 2012.240 12:17:53 EXECUTION ENDS
    ARC6004I 00D0 ABACKUP ADR013I (001)-CLTSK(01), 2012.240 12:17:58 TASK COMPLETED
    ARC6004I 00D0 ABACKUP ADR012I (SCH)-DSSU (01),
    2012.240 12:17:58 DFSMSDSS PROCESSING COMPLETE. HIGHEST RETURN CODE IS 0000
    ARC6382I ACTIVITY LOG HSMACT.H2.ABACKUP.MHLRESA.D12240.T121639 HAS BEEN SUCCESSF
    ARC6382I INSTRUCTION DATA SET MHLRESA.ABARS.INSTRUCT HAS BEEN SUCCESSFULLY BACKE
    ARC6369I STORAGE REQUIREMENTS FOR AGGREGATE GROUP MHLRESA ARE: L0=2109K, ML1=0,
    
    
    -----
    
    ARC6061I VOLUMES USED FOR CONTROL FILE MHLRESA.C.C01V0001 DURING AGGREGATE BACKU
    THS014
    ARC6060I VOLUMES USED FOR DATA FILE MHLRESA.D.C01V0001 DURING AGGREGATE BACKUP F
    THS006
    ARC6071I VOLUMES USED FOR INSTRUCTION/ACTIVITY LOG FILE MHLRESA.I.C01V0001 DURIN
    MHLRESA ARE:
    THS007
    ARC6055I AGGREGATE BACKUP HAS COMPLETED FOR AGGREGATE GROUP MHLRESA, AT 12:18:33
    
### 10.4.5 Perform an aggregate recover (ARECOVER)

We describe how to recover a single data set that was backed up previously. On the
Aggregate Group Application Selection panel, enter option 6 to recover an aggregate group,
as shown in Figure 10-18.

                     AGGREGATE GROUP APPLICATION SELECTION
    Command ===>
    
    To Perform Aggregate Group Operations, Specify:
      CDS Name . . . . . . . . 'ACTIVE'
                                     (1 to 44 Character Data Set Name or 'Active')
      Aggregate Group Name . . MHLRESA  (for Aggregate Group List, fully or
                                        Partially Specified or * for All)
    Select one of the following Options:
    **6** 1. List   - Generate a list of Aggregate Groups
          2. Display - Display an Aggregate Group
          3. Define  - Define an Aggregate Group
          4. Alter  - Alter an Aggregate Group
          5. Abackup - Backup an Aggregate Group
        **6. Arecover - Recover an Aggregate Group**
    
    If List Option is Chosen,
      Enter "/" to select option   Respecify View Criteria
                                   Respecify Sort Criteria
    
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-18  Perform ARECOVER by using ISMF_

Pressing Enter will open the next panel, which is shown in Figure 10-19 


-----

    AGGREGATE GROUP RECOVER        Page 1 of 8
    Command ===>
    
    Abackup Control Dataset . .
    (1 to 44 Characters)
    Xmit . . . . . . . . . . . N      (Y or N)
    Stack / Nostack . . . . .       (S=STACK, N=NOSTACK or blank)
    Aggregate Group Name . . . . MHLRESA
    Date . . . . . . . . . . . 2012/08/27 (yyyy/mm/dd)
    Version . . . . . . . . .       (1 to 9999)
    
    Processing Option . . . . . 3      (1=Prepare, 2=Verify, 3=Execute)
    Wait for Completion . . . . N      (Y or N)
    Target GDG Data Set Status        (A=ACTIVE, D=DEFERRED, R=ROLLEDOFF,
    S=SOURCE or blank)
    Volume Count . . . . . . . .       (A=ANY or blank)
    Recover Instruction Data Set . . N   (Y or N)
    Recover Activity Log . . . . . . N   (Y or N)
    **Recover Individual Data Sets . . Y** (Y or N)
    
    Use ENTER to Continue; Use DOWN to View Additional Options;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-19  ABARS Group Recover panel_

We chose to recover a single data set, 'MHLRESA.REPORT.JCL', on the same system
(assuming it was lost and no longer cataloged) by entering option Y in the Recover Individual
Data Sets field, as shown in Figure 10-19.

The next panel requires you to enter the data set name that you want to recover, which, in our
case, is 'MHLRESA.REPORT.JCL', as shown in Figure 10-20 on page 293.


-----

                          AGGREGATE GROUP RECOVER
    Command ===>
    
    AGGREGATE GROUP NAME:
    MHLRESA
    
    Specify Data Set Name for Individual Recovery:
    
    **Single Data Set Name . . . . 'MHLRESA.REPORT.JCL'**
    (1 to 44 Characters)
    List Of Names Data Set . . .
    (1 to 44 Characters)
    
    Use ENTER to Continue;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-20  ABARS recover single data set by using ISMF_

Pressing Enter opens the Aggregate Group Recover panel that is shown in Figure 10-21.

                         AGGREGATE GROUP RECOVER
    Command ===>
    
    DFSMShsm Command and Processing Option:
    NOWAIT ARECOVER AGGREGATE(MHLRESA) DATE(2012/08/27)
    EXECUTE
    MIGRATEDDATA(ML1)
    ONLYDATASET(NAME('MHLRESA.REPORT.JCL'))
    
    Enter 1 to Submit DFSMShsm ARECOVER COMMAND
    Enter 2 to Save Generated ARECOVER PARAMETERS
    
    Select Option . . 1             (1=SUBMIT, 2=SAVE)
    
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-21  ABARS recover single data set second panel_

We chose to run the job, so we entered option 1 . DFSMShsm starts ABARS to recover the
data set.


-----

After the ARECOVER process completes, you can view the ARECOVER log data set as
shown in Example 10-15.

_Example 10-15  ARECOVER log data set_

    PAGE 0001 Z/OS DFSMSHSM 1.13.0 DATA FACILITY HIERARCHICAL STORAGE MANAGER 12.2
    ARC6000I ARECOVER AGGREGATE(MHLRESA) DATE(2012/08/27) EXECUTE MIGRATEDDATA(ML1)
    ARC6102I AGGREGATE RECOVERY STARTING USING CONTROL FILE DATA SET MHLRESA.C.C01V0
    STARTED TASK = DFHSMABR.ABAR0154
    ARC6030I ACTIVITY LOG FOR CONTROL FILE DATA SET MHLRESA.C.C01V0001 WILL BE ROUTE
    HSMACT.H2.ARECOVER.MHLRESA.D12240.T123654
    ARC6308I CONFLICT RESOLUTION DATA SET HSM.MHLRESA.CONFLICT.D12240.T121639 WILL B
    ARC6115I AGGREGATE RECOVERY USING CONTROL FILE DATA SET MHLRESA.C.C01V0001 WILL
    THS006
    THS007
    ARC6119I STORAGE CLASS NAMES EXISTED DURING AGGREGATE BACKUP OF AGGREGATE GROUP
    MHLRESA.C.C01V0001
    STANDARD
    ARC6119I MANAGEMENT CLASS NAMES EXISTED DURING AGGREGATE BACKUP OF AGGREGATE GRO
    MHLRESA.C.C01V0001
    MC54NMIG
    MCDB22
    ARC6004I 00D0 ARECOVER PAGE 0001   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVI
    ARC6004I 00D0 ARECOVER ADR035I (SCH)-PRIME(06), INSTALLATION EXIT ALTERED BYPASS
    ARC6004I 00D0 ARECOVER RESTORE DATASET(FILTERDD(SYS00007)) -
    ARC6004I 00D0 ARECOVER INDDNAME(SYS00006) -
    ARC6004I 00D0 ARECOVER OUTDYNAM( -
    ARC6004I 00D0 ARECOVER  (ML940D) -
    ARC6004I 00D0 ARECOVER  ) -
    ARC6004I 00D0 ARECOVER PERCENTUTILIZED( -
    ARC6004I 00D0 ARECOVER 090 -
    ARC6004I 00D0 ARECOVER  ) -
    ARC6004I 00D0 ARECOVER VOLCOUNT(ANY) -
    ARC6004I 00D0 ARECOVER SPHERE -
    ARC6004I 00D0 ARECOVER TGTGDS(SOURCE  ) -
    ARC6004I 00D0 ARECOVER CATALOG FORCE FORCECP(0)
    ARC6004I 00D0 ARECOVER ADR101I (R/I)-RI01 (01), TASKID 001 HAS BEEN ASSIGNED TO
    ARC6004I 00D0 ARECOVER ADR109I (R/I)-RI01 (01), 2012.240 12:38:05 INITIAL SCAN O
    ARC6004I 00D0 ARECOVER ADR050I (001)-PRIME(01), DFSMSDSS INVOKED VIA APPLICATION
    ARC6004I 00D0 ARECOVER ADR016I (001)-PRIME(01), RACF LOGGING OPTION IN EFFECT FO
    ARC6004I 00D0 ARECOVER ADR006I (001)-STEND(01), 2012.240 12:38:05 EXECUTION BEGI
    ARC6004I 00D0 ARECOVER ADR780I (001)-TDDS (01),
    ARC6004I 00D0 ARECOVER THE INPUT DUMP DATA SET BEING PROCESSED IS IN LOGICAL DAT
    ARC6004I 00D0 ARECOVER DFSMSDSS VERSION
    ARC6004I 00D0 ARECOVER             1 RELEASE 13 MODIFICATION LEVEL
    ARC6004I 00D0 ARECOVER ADR711I (001)-NEWDS(01), DATA SET MHLRESA.REPORT.JCL HAS
    ARC6004I 00D0 ARECOVER NO DATACLAS, AND MGMTCLAS MC54NMIG
    ARC6004I 00D0 ARECOVER ADR474I (001)-TDNVS(01),
    ARC6004I 00D0 ARECOVER DATA SET MHLRESA.REPORT.JCL CONSISTS OF 00000010 TARGET T
    ARC6004I 00D0 ARECOVER ADR489I (001)-TDLOG(01), DATA SET MHLRESA.REPORT.JCL WAS
    ARC6004I 00D0 ARECOVER ADR454I (001)-TDLOG(01), THE FOLLOWING DATA SETS WERE SUC
    ARC6004I 00D0 ARECOVER              MHLRESA.REPORT.JCL
    ARC6004I 00D0 ARECOVER ADR006I (001)-STEND(02), 2012.240 12:38:07 EXECUTION ENDS
    ARC6004I 00D0 ARECOVER ADR013I (001)-CLTSK(01), 2012.240 12:38:12 TASK COMPLETED
    PAGE 0002 Z/OS DFSMSHSM 1.13.0 DATA FACILITY HIERARCHICAL STORAGE MANAGER 12.2
    ARC6004I 00D0 ARECOVER ADR012I (SCH)-DSSU (01),
    
    
    -----
    
    ARC6004I 00D0 ARECOVER 2012.240 12:38:12 DFSMSDSS PROCESSING COMPLETE. HIGHEST R
    ARC6116I THE FOLLOWING DATA SETS WERE SUCCESSFULLY RECOVERED USING AGGREGATE GRO
    MHLRESA.REPORT.JCL
    ARC6103I AGGREGATE RECOVERY HAS COMPLETED FOR AGGREGATE GROUP MHLRESA, USING CON
    AT 12:38:12, RETCODE = 000

You can use the same process to recover the entire data sets that were backed up by ABARS
in the same way that you recover a single data set.

### 10.4.6 Interrupting and restarting ABARS processing

Three methods are available to interrupt automatic dump processing:

- Hold ABARS processing: You can hold ABARS processing by specifying the **HOLD**
command for the **ABACKUP** , **ARECOVER** , or **ALL** parameter. If you hold ABARS processing with
the **ENDOFDATASET** parameter, processing ends after DFSMShsm finishes processing the
current data set.

- Place DFSMShsm in emergency mode: You can place DFSMShsm in emergency mode
by specifying the **SETSYS EMERGENCY** command.

- Stop DFSMShsm processing: You can stop DFSMShsm by entering the MVS or
DFSMShsm **STOP** command.

You can restart automatic dump processing by entering the corresponding commands:

- RELEASE ABACKUP , ARECOVER , or RELEASE ALL
- SETSYS NOEMERGENCY
- START DFSMSHSM (MVS command)

## 10.5 Fast replication backup and recovery

_Fast replication_ is a function that uses volume-level fast replication to create backup versions
for sets of storage groups. It uses IBM FlashCopy®, which is a point in time copy of a volume.
A point in time copy gives the appearance of an almost instantaneous volume copy.

The process of fast data replication occurs so fast because it builds a map, with pointers, to
the source volume tracks or extents. You no longer need to wait for the physical copy to
complete before applications can resume the access to the data. Both the source and target
data are available for read/write access almost immediately, while the copy process continues
in the background. This process guarantees that the contents of the target volume are an
exact duplicate of the source volume at that point in time.

You define a set of storage groups with the SMS “copy pool” construct. Fast replication target
volumes contain the fast replication backup copies of volumes that are managed by
DFSMShsm. Fast replication target volumes are defined with the SMS “copy pool backup”
storage group type. The fast replication backup versions on DASD can then be dumped to
tape by using either the **FRBACKUP** command or with Automatic Dump.

Recovery from the fast replication backup versions can be performed at the data set, volume,
or copy pool level. The entire copy pool, individual volumes, and data sets within a copy pool
can be recovered from the fast replication backup versions on DASD or tape. Individual data
sets are recovered to the volume or volumes that they existed on at the time of the backup.


-----

The following functions are described:

- Define a copy pool backup storage group
- Assign a storage group to copy pool backup and copy pool construct
- Perform a simple fast replication backup
- Perform a simple fast replication recover

### 10.5.1 Define copy pool backup storage group

The SMS copy pool backup storage group type contains the candidate target volumes for
DFSMShsm fast replication requests. The volumes that are associated with a copy pool
backup storage group are for DFSMShsm use. If you need to restrict the use of a volume in a
copy pool backup storage group, vary the volume offline.

For each source volume in a storage group to be copied, enough eligible target volumes must
exist in the copy pool backup storage group to satisfy the number of volumes that are required
by the number of specified backup versions.

An eligible target volume must meet the following requirements:

- Have the same track format as the source volume.

- Be the same size as the source volume.

- For FlashCopy:

  - Not be a primary or secondary volume in a Global Mirror (XRC) volume pair.

  - For FlashCopy version 1:

-  Reside in the same logical subsystem (LSS) as the source volume

-  Not be in a FlashCopy relationship at the time of the backup

In our example, we chose the name of _DB0BCPB_ for the copy pool. From the ISMF main
menu, enter option 6 (Storage Group Application Selection). We entered the name DB0BCPB in
the Storage Group Name field, and we entered COPY POOL BACKUP in the Storage Group Type
field. We entered option 3 to define a new storage group, as shown in Figure 10-22


-----

                        STORAGE GROUP APPLICATION SELECTION
    Command ===>
    
    To perform Storage Group Operations, Specify:
    CDS Name . . . . . . 'SYS1.SMS.MHLRES3.SCDS'
    (1 to 44 character data set name or 'Active' )
    **Storage Group Name  DB0BCPB** (For Storage Group List, fully or
    partially specified or * for all)
    **Storage Group Type  COPY POOL BACKUP** (VIO, POOL, DUMMY, COPY POOL BACKUP,
    OBJECT, OBJECT BACKUP, or TAPE)
    Select one of the following options :
    **3** 1. List  - Generate a list of Storage Groups
          2. Display - Display a Storage Group (POOL only)
        **3. Define - Define a Storage Group**
          4. Alter  - Alter a Storage Group
          5. Volume - Display, Define, Alter or Delete Volume Information
    
    If List Option is chosen,
    Enter "/" to select option   Respecify View Criteria
    Space Info in GB      Respecify Sort Criteria
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-22  ISMF Storage Group Application Selection panel_

When you press Enter, ISMF opens the next panel. We entered the description for this copy
pool, as shown in Figure 10-23.

                      COPY POOL BACKUP STORAGE GROUP DEFINE
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : DB0BCPB
    
    To DEFINE Storage Group, Specify:
    
    **Description ==> DB0B COPY POOL BACKUP**
    ==>
    
    DEFINE  SMS Storage Group Status . . . Y (Y or N)
    
    Use ENTER to Perform Verification;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.

_Figure 10-23  Copy Pool Backup Storage Group Define panel_


-----

The next panel allows us to define SMS storage group status for all systems in the SMSplex.
In our case, we enabled this storage group to all four systems in the SMSplex by entering
ENABLE , as shown in Figure 10-24.

                         SMS STORAGE GROUP STATUS DEFINE
    Command ===>
    
    SCDS Name . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : DB0BCPB
    Storage Group Type : COPY POOL BACKUP
    To DEFINE Storage Group System/          ( Possible SMS SG Status
    Sys Group Status, Specify:             for each:
    -  Pool SG Type
    System/Sys   SMS SG  System/Sys   SMS SG    NOTCON, ENABLE, DISALL
    Group Name   Status  Group Name   Status    DISNEW, QUIALL, QUINEW
    ----------   ------  ----------   ------  - Tape SG Type
    SC63    ===> **ENABLE** SC64    ===> **ENABLE** NOTCON, ENABLE,
    SC65    ===> **ENABLE** SC70    ===> **ENABLE** DISALL, DISNEW
    ===>           ===>     - Copy Pool Backup SG Type
    ===>           ===>       NOTCON, ENABLE )
    ===>           ===>     * SYS GROUP = sysplex
    ===>           ===>      minus Systems in the
    ===>           ===>      Sysplex explicitly
    ===>           ===>      defined in the SCDS
    Use ENTER to Perform Verification; Use DOWN Command to View next Panel;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.

_Figure 10-24  SMS Storage Group Status Define panel_

Press PF3 twice to save the new definition. ISMF returns to the first panel.

The next step is to define DASD volumes for this storage group, as shown in Figure 10-25


-----

    STORAGE GROUP APPLICATION SELECTION
    Command ===>
    
    To perform Storage Group Operations, Specify:
    CDS Name . . . . . . 'SYS1.SMS.MHLRES3.SCDS'
    (1 to 44 character data set name or 'Active' )
    Storage Group Name **DB0BCPB** (For Storage Group List, fully or
    partially specified or * for all)
    Storage Group Type           (VIO, POOL, DUMMY, COPY POOL BACKUP,
    OBJECT, OBJECT BACKUP, or TAPE)
    Select one of the following options :
    **5** 1. List  - Generate a list of Storage Groups
          2. Display - Display a Storage Group (POOL only)
          3. Define - Define a Storage Group
          4. Alter  - Alter a Storage Group
        **5. Volume - Display, Define, Alter or Delete Volume Information**
    
    If List Option is chosen,
    Enter "/" to select option   Respecify View Criteria
    Space Info in GB      Respecify Sort Criteria
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-25  Storage Group Application Selection panel_

On the next panel, we entered the volsers or volume patterns to define them to be part of this
storage group, as shown in Figure 10-26.

                          STORAGE GROUP VOLUME SELECTION
    Command ===>
    
    CDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : DB0BCPB
    Storage Group Type : COPY POOL BACKUP
    
    Select One of the following Options:
    **2** 1. Display - Display SMS Volume Statuses (Pool & Copy Pool Backup only)
        **2. Define - Add Volumes to Volume Serial Number List**
          3. Alter  - Alter Volume Statuses (Pool & Copy Pool Backup only)
          4. Delete - Delete Volumes from Volume Serial Number List
    
    Specify a Single Volume (in Prefix), or Range of Volumes:
    Prefix  From   To  Suffix Type
    ______ ______ ______ _____  _
    ===> SBOXK  A    H        A
    ===> SBOXK  0    1        X
    ===>
    ===>
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-26  Storage Group Volume Selection panel_


-----

In our example, we defined volumes SBOXKA - SBOXKH and SBOXK0 - SBOXK1 to be part
of this copy pool storage group.

On the next panel, we defined the volume status on each system of the SMSplex, as shown in
Figure 10-27. We chose to enable all volumes to all systems.

                              SMS VOLUME STATUS DEFINE
    Command ===>
    
    SCDS Name . . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name . : DB0BCPB
    Volume Serial Numbers : SBOXKA - SBOXKH    SBOXK0 - SBOXK1
    
    To DEFINE SMS Volume Status, Specify:
    
    System/Sys   SMS Vol   System/Sys   SMS Vol  ( Possible SMS Vol
    Group Name   Status   Group Name   Status    Status for each:
    ----------   -------   ----------   -------   NOTCON, ENABLE,
    SC63    ===> **ENABLE** SC64    ===> **ENABLE** DISALL, DISNEW,
    SC65    ===> **ENABLE** SC70    ===> **ENABLE** QUIALL, QUINEW )
    ===>            ===>
    ===>            ===>      * SYS GROUP = sysplex
    ===>            ===>       minus systems in the
    ===>            ===>       sysplex explicitly
    ===>            ===>       defined in the SCDS
    ===>            ===>
    Use ENTER to Perform Verification; Use DOWN Command to View next Panel;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.

_Figure 10-27  SMS Volume Status Define panel_

We pressed PF3 twice to save the new configuration and to complete the copy pool backup
definition process.

The next step is to define this copy pool backup and copy pool construct to any storage group
where it is wanted as a copy pool.

### 10.5.2 Assign storage groups to copy pool backup and copy pool construct

In our example, we chose storage group, _DB0BARCH_ .

From the Storage Group List panel, we entered the ALTER command against the DB0BARCH
storage group, as shown in Figure 10-28 on page 301.


-----

                             STORAGE GROUP LIST
    Command ===>                        Scroll ===> CSR
    Entries 1-8 of 8
    Data Columns 3-6 of 48
    CDS Name : MHLRESA.SMS.SCDS
    
    Enter Line Operators below:
    
    LINE    STORGRP SG        VIO   VIO  AUTO
    OPERATOR  NAME   TYPE       MAXSIZE UNIT MIGRATE
    ---(1)---- --(2)--- -------(3)------ --(4)-- (5)- --(6)---
    **ALTER    DB0BARCH POOL** ------- ---- NO
    DB0BCPB COPY POOL BACKUP ------- ---- --------
    DB0BDATA POOL       ------- ---- NO
    DB0BIMAG POOL       ------- ---- NO
    DB0BLOG1 POOL       ------- ---- NO
    DB0BLOG2 POOL       ------- ---- NO
    DB0BMISC POOL       ------- ---- NO
    DB0BTARG POOL       ------- ---- NO
    ---------- -------- ------ BOTTOM OF DATA ------ -------- ----------

_Figure 10-28  Storage Group List panel_

We then typed the copy pool DB0BCPB in the Copy Pool Backup SG Name field, as shown in
Figure 10-29, and saved it.

                          POOL STORAGE GROUP ALTER
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : DB0BARCH
    To ALTER Storage Group, Specify:
    Description ==> DB0B ARCHIVE LOGS
    ==>
    Auto Migrate . . N (Y, N, I or P)  Migrate Sys/Sys Group Name . .
    Auto Backup . . N (Y or N)     Backup Sys/Sys Group Name . .
    Auto Dump . . . N (Y or N)     Dump Sys/Sys Group Name . . .
    Overflow . . . . N (Y or N)     Extend SG Name . . . . . . . .
    **Copy Pool Backup SG Name . . . DB0BCPB**
    
    Dump Class . . .           (1 to 8 characters)
    Dump Class . . .           Dump Class . . .
    Dump Class . . .           Dump Class . . .
    
    ALTER   SMS Storage Group Status . . . N  (Y or N)
    ALTER   SMA Attributes . . . . . . . . N  (Y or N)
    Use ENTER to Perform Selection; Use DOWN Command to View next Page;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.

_Figure 10-29  Pool Storage Group Alter panel_

You also need to define storage group candidates to the copy pool construct.


-----

From the ISMF primary panel, we entered option P Copy Pool (Specify Pool Storage Groups
for Copies), as shown in Figure 10-30.

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
    **P Copy Pool         - Specify Pool Storage Groups for Copies**
    R Removable Media Manager  - Perform Functions Against Removable Media
    X Exit           - Terminate ISMF
    Use HELP Command for Help; Use END Command or X to Exit.

_Figure 10-30  ISMF Primary Option Menu panel_

The copy pool construct that relates to DB2 must point to the SMS storage groups that will be
processed for fast replication operations.

The BACKUP SYSTEM utility requires that at least a database copy pool, which consists of
all database objects (DB2 catalog, directory, and all user data), exists to take a data-only
backup. If you plan to take full system backups, a log copy pool that consists of the active logs
and bootstrap data sets (BSDSs) are required. The DB2 RESTORE SYSTEM utility uses only
the database copy pool. A copy pool can contain up to 256 SMS storage groups to be
processed for fast replication operations.

The copy pool naming convention must be in the following form:

DSN$ _locn-name_ $ _cp-type_

- DSN is the unique DB2 product identifier.
- $ is a required delimiter.
- _locn-name_ is the DB2 location name.
- $ is a required delimiter.
- _cp-type_ is the copy pool type:
  - DB for database
  - LG for logs

In our example, we chose the _locn-name_ , _DB0B_ . Therefore, we defined the copy pool
construct as _DSN$DB0B$DB_ .

Figure 10-31 shows how to define a new copy pool construct.


-----

    COPY POOL APPLICATION SELECTION
    Command ===>
    
    To perform Copy Pool Operations, Specify:
    CDS Name . . . . 'SYS1.SMS.MHLRES3.SCDS'
    (1 to 44 character data set name or 'Active' )
    Copy Pool Name **DSN$DB0B$LG** (For Copy Pool List, fully
    or partially specified or * for all)
    
    Select one of the following options :
    **3** 1. List  - Generate a list of Copy Pools
          2. Display - Display a Copy Pool
        **3. Define - Define a Copy Pool**
          4. Alter  - Alter a Copy Pool
    
    If List Option is chosen,
    Enter "/" to select option   Respecify View Criteria
    Respecify Sort Criteria
    
    Use ENTER to Perform Selection;
    Use HELP Command for Help; Use END Command to Exit.

_Figure 10-31  Copy Pool Application Selection panel_

Next, specify the options for dump, as shown in Figure 10-32. We chose to use this copy pool
construct with automatic dump. We entered DB0BTST1 for the Dump Class.

                          COPY POOL DEFINE
    Command ===>
    
    SCDS Name . . : SYS1.SMS.MHLRES3.SCDS
    Copy Pool Name : **DSN$DB0B$LG**
    
    To DEFINE Copy Pool, Specify:
    Description ==> **DB0B ARCH AND LOGS**
    ==>
    **Auto Dump . . . Y** (Y or N)    Dump Sys/Sys Group Name . . .
    Dump Class . . **DB0BTST1** Dump Class . .
    Dump Class . .           Dump Class . .
    Dump Class . .
    
    Number of DASD Fast Replication Backup
    Versions with Background Copy . . . . . 1    (0 to 85 or blank)
    FRBACKUP to PPRC Primary Volumes allowed . . .   (NO, PN, PP, PR or blank)
    FRRECOV to PPRC Primary Volumes allowed . . .   (NO, PN, PP, PR or blank)
    
    Use ENTER to Perform Verification; Use DOWN Command to View next Panel;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.

_Figure 10-32  Copy Pool Define panel_

In the next panel, we defined the catalog name that resides in the storage group DB0BDATA,
as shown in Figure 10-33.


-----

                          COPY POOL DEFINE
    Command ===>
    
    SCDS Name . . : SYS1.SMS.MHLRES3.SCDS
    Copy Pool Name : DSN$DB0B$LG
    
    To DEFINE Copy Pool, Specify:
    Catalog Name ==> **UCAT.DB0BLOGS**
    ==>
    ==>
    ==>
    ==>
    ==>
    ==>
    ==>
    ==>
    ==>
    Capture Catalog Information for Data Set Recovery . . . R  (R, P or N)
    Allow Fast Reverse Restore . . . . . . . . . . . . . . Y  (Y or N)
    
    Use ENTER to Perform Verification; Use UP/DOWN Command to View previous Panel;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.

_Figure 10-33  Copy Pool Define next panel_

In Figure 10-33, we chose to capture the information of catalog UCAT.DB0BLOGS for the
FRBACKUP function. If the FRBACKUP process failed to capture this catalog information, it
fails the backup version.

We also allowed the Fast Reverse Restore function. No additional keywords need to be
specified to recover from a DASD copy pool fast reverse restore-eligible version. If the state of
the FlashCopy relationships meets the fast reverse restore requirements, DFSMShsm uses
fast reverse restore to recover the copy pool. Otherwise, DFSMShsm uses regular FlashCopy
if the background copy completed.

After a successful fast reverse restore, the contents of the DASD backup volume are invalid.
DFSMShsm invalidates and initializes the individual DASD backup volume so it is ready to be
reused. When the entire copy pool is successfully recovered, DFSMShsm invalidates the
DASD copy pool backup version.

If several of the volumes fail to recover, the FASTREPLICATIONSTATE of the DASD version
becomes FCFRRINCOMPLETE. You can issue the **FRRECOV COPYPOOL(** **_cpname_** **)** command
again after you address the reason for the failure. DFSMShsm tries to recover the remaining
volumes again by using fast reverse restore.

Fast reverse restore is not supported with multiple FlashCopy targets. For a successful fast
reverse restore, verify that no multiple FlashCopy targets exist before you issue the **FRRECOV**
**COPYPOOL(** **_cpname_** **)** command. To remove unneeded FlashCopy targets that are managed by
DFSMShsm, issue **FRBACKUP COPYPOOL(** **_cpname_** **) WITHDRAW** to withdraw the most recent DASD
backup copy, or issue **FRDELETE COPYPOOL(** **_cpname_** **) VERSION(** **_n_** **)** to delete a specific backup
version. For FlashCopy targets that are not managed by DFSMShsm, issue **ICKDSF FLASHCPY**
**WITHDRAW** or **SDM FCWITHDR** to withdraw the unneeded FlashCopy relationships.


-----

To determine whether a copy pool is defined to allow fast reverse restore and to select a fast
reverse restore eligible backup version, issue **LIST COPYPOOL (** **_cpname_** **)** . If the copy pool is
defined to allow fast reverse restore at the time of the backup, the output displays the backup
version with FCFRR=Y . Otherwise, the output contains FCFRR=N .

Example 10-16 shows the copy pool construct DSN$DB0B$LG, which is defined to allow fast reverse restore.

_Example 10-16  DSN$DB0B$LG information from the LIST CP command_

    1-- DFSMShsm CONTROL DATASET --COPY POOL--LISTING --------- AT 12:32:34 ON
    12/08/31 FOR SYSTEM=SC64
    
    0COPYPOOL=DSN$DB0B$LG
    ALLOWPPRCP FRB=NO FRR=NO
    
    VERSION VTOCENQ   DATE      TIME     FASTREPLICATIONSTATE DUMPSTATE
       007    Y     2012/08/29  18:03:38     RECOVERABLE        NONE
    TOKEN(C)=C'MHLRESA'
    TOKEN(H)=X'D4C8D3D9C5E2C1'
    TOTAL NUM OF VOLUMES=00004,INCREMENTAL=N,CATINFO=Y, **FCFRR=Y** ,RECOVERYINCOMPLETE=N
    
    SGNAME  SOURCE - TARGET SOURCE - TARGET SOURCE - TARGET SOURCE - TARGET
    DB0BARCH SBOXJ0 - SBOXK0 SBOXJ1 - SBOXK1
    DB0BLOG1 SBOXJ8 - SBOXKA
    DB0BLOG2 SBOXJ9 - SBOXKB
    
    0----- END OF -- COPY POOL -- LISTING -----

To determine whether fast reverse restore can be used for a copy pool version, issue the
**QUERY COPYPOOL** command. When fast reverse restore can be used, the **QUERY COPYPOOL**
command output displays “background copy percent-complete” (PCT-COMP) information
other than “***” . Percent-complete information (a percentage) is available for full-volume
FlashCopy pairs with an incomplete background copy only. A full-volume FlashCopy
relationship is established when the FlashCopy technique (such as fast reverse restore or
incremental) designates it, or when SETSYS FASTREPLICATION(FCRELATION(FULL)) was
specified.

Because fast reverse restore invalidates the entire DASD backup version after a successful
recovery, you can use the PCT-COMP percentage to determine the progress of the
background copy and decide whether to use fast reverse restore for recovery.

Example 10-17 shows the output from the **QUERY COPYPOOL(DSN$DB0B$LG)** command.

_Example 10-17  Output from the QUERY COPYPOOL(DSN$DB0B$LG) command_

    ARC1820I THE FOLLOWING VOLUMES IN COPY POOL DSN$DB0B$LG, VERSION 008, HAVE AN
    ARC1820I (CONT.) ACTIVE FLASHCOPY BACKGROUND COPY
    ARC1820I (CONT.) SGNAME  FR-PRIMARY FR-BACKUP PCT-COMP
    ARC1820I (CONT.) DB0BARCH SBOXJ0   SBOXK0   008
    ARC1820I (CONT.) DB0BARCH SBOXJ1   SBOXK1   009
    ARC1820I (CONT.) DB0BLOG1 SBOXJ8   SBOXKA   068
    ARC1820I (CONT.) DB0BLOG2 SBOXJ9   SBOXKB   001
    ***


-----

**Notes:**

1. A DASD backup version can be used for fast reverse restore only one time because the
process invalidates the data on the backup volume. Keep dump tape copies in addition
to DASD backup copies when you plan to use the fast reverse restore function.

2. VERIFY(N) cannot be specified when the copy pool is defined to allow fast reverse
restore unless a previous FRRECOV COPYPOOL operation determined that the copy
pool backup version is no longer eligible to use fast reverse restore. Regular fast
replication recovery can be used instead.

3. When a recovery operation, such as fast reverse restore from a DASD backup, will
invalidate an existing DASD copy with an incomplete dump copy, the recovery operation
fails. To force the use of a DASD copy with an incomplete dump copy, specify the **FORCE**
parameter on the **FRRECOV** command.

4. The preserve mirror operation cannot be used in combination with fast reverse restore.
Do not specify PMPREF or PMREQ in the “FRBACKUP to PPRC Primary Volumes
allowed” or “FRRECOV to PPRC Primary Volumes allowed” fields for a copy pool that is
defined to allow fast reverse restore.

In the next panel, we entered the storage group name DB0BARCH to be copied to this copy pool
construct, as shown in Figure 10-34.

                             COPY POOL DEFINE
    Command ===>
    
    SCDS Name . . : SYS1.SMS.MHLRES3.SCDS
    Copy Pool Name : DSN$DB0B$LG
    
    To DEFINE Copy Pool, Specify:
    Storage Group Names: (specify 1 to 256 names)
    ==> **DB0BARCH**
    ==>
    ==>
    ==>
    ==>
    ==>
    
    Use ENTER to Perform Verification; Use UP/DOWN Command to View other Panels;
    Use HELP Command for Help; Use END Command to Save and Exit; CANCEL to Exit.

_Figure 10-34  Copy Pool Define next panel_

We pressed PF3 to save the new definition, which completed the define storage group to
copy pool backup storage group and copy pool construct.


-----

### 10.5.3 Perform fast replicate backup (FRBACKUP)

The _fast replication backup_ function is not available as part of automatic backup or dump. You
can invoke the **FRBACKUP** command from a system console, with a TSO command ( **HSENDCMD** ),
with a batch job, or by using the ARCHSEND macro interface. DFSMShsm invokes the
DFSMSdss COPY FULL function to create a fast replication backup for each volume in the
specified copy pool.

The simple syntax command is shown:

FRBACKUP COPYPOOL( _cpname_ ) EXECUTE

In our example, we chose to use FRBACKUP with a batch job and to use the DUMP option to
also copy those volumes to tapes after the fast replication successfully completed, as shown
in Example 10-18.

_Example 10-18  Sample JCL for FRBACKUP with DUMP options_

    //MHLRESAY JOB (999,POK),'MHLRESA',CLASS=A,MSGCLASS=X,
    // NOTIFY=&SYSUID,TIME=1440,REGION=0M
    /*JOBPARM SYSAFF=*
    //HSMFR EXEC PGM=IKJEFT01
    //SYSPRINT DD SYSOUT=*
    //PRINTER DD SYSOUT=*
    //INDEX  DD SYSOUT=*
    //SYSIN  DD SYSOUT=*
    //SYSTSPRT DD SYSOUT=*
    //SYSTSIN DD *
    **HSEND FRBACKUP COPYPOOL(DSN$DB0B$LG) TOKEN(MHLRESA) DUMP -**
    **DUMPCLASS(DB0BTST1) EXECUTE**

The **FRBACKUP COPYPOOL** command is successful only after DFSMSdss successfully
establishes a fast replication relationship for each source volume. If one of the volumes fails,
the entire backup version is failed with the following results:

- DFSMShsm continues to process volumes to determine whether any remaining volumes
will also encounter an error.

- For each volume in error, the target volume is disassociated from the source volume. If a
problem exists with the target volume, the target volume needs to be removed from the
copy pool backup storage group. Otherwise, it can be reselected during the next **FRBACKUP**
command.

- FlashCopy relationships that are successfully established are withdrawn after all volumes
are processed.

- The backup version is marked as invalid after all volumes are processed.

After the batch job is complete, you can check in the DFSMShsm log to verify the FRBACKUP
process, as shown in Example 10-19.

_Example 10-19  Sample of FRBACKUP activities log_

    ARC1801I FAST REPLICATION BACKUP DUMP IS STARTING FOR COPY POOL DSN$DB0B$LG, AT
    13:38:55 ON 2012/08/31, TOKEN='MHLRESA'
    ARC0640I ARCFRTM - PAGE 0001   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVICES
    2012.244 13:38
    ARC0640I ARCFRTM - ADR035I (SCH)-PRIME(06), INSTALLATION EXIT ALTERED BYPASS FAC
    CLASS CHK DEFAULT TO YES
    ARC0640I ARCFRTM - PARALLEL
    
    
    -----
    
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 001 HAS BEEN ASSIGNED TO
    COMMAND 'PARALLEL '
    ARC0640I ARCFRTM - COPY IDY(SBOXJ0) ODY(SBOXK0) DUMPCOND FR(REQ) PUR ALLX
    ALLD(*)    ARC0640I ARCFRTM - FCFVR ARC0640I ARCFRTM - DEBUG(FRMSG(DTL))
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 002 HAS BEEN ASSIGNED TO
    COMMAND 'COPY '
    ARC0640I ARCFRTM - COPY IDY(SBOXJ1) ODY(SBOXK1) DUMPCOND FR(REQ) PUR ALLX
    ALLD(*)    ARC0640I ARCFRTM - FCFVR ARC0640I ARCFRTM - DEBUG(FRMSG(DTL))
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 003 HAS BEEN ASSIGNED TO
    COMMAND 'COPY '
    ARC0640I ARCFRTM - COPY IDY(SBOXJ8) ODY(SBOXKA) DUMPCOND FR(REQ) PUR ALLX
    ALLD(*)    ARC0640I ARCFRTM - FCFVR ARC0640I ARCFRTM - DEBUG(FRMSG(DTL))
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 004 HAS BEEN ASSIGNED TO
    COMMAND 'COPY '
    ARC0640I ARCFRTM - COPY IDY(SBOXJ9) ODY(SBOXKB) DUMPCOND FR(REQ) PUR ALLX
    ALLD(*)    ARC0640I ARCFRTM - FCFVR ARC0640I ARCFRTM - DEBUG(FRMSG(DTL))
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 005 HAS BEEN ASSIGNED TO
    COMMAND 'COPY '
    ARC0640I ARCFRTM - ADR109I (R/I)-RI01 (01), 2012.244 13:38:55 INITIAL SCAN OF
    USER CONTROL STATEMENTS COMPLETED
    ARC0640I ARCFRTM - ADR014I (SCH)-DSSU (02),
    2012.244 13:38:55 ALL PREVIOUSLY SCHEDULED TASKS COMPLETED. PARALLEL MODE NOW IN
    EFFECT
    ARC0640I ARCFRTM - ADR050I (002)-PRIME(02), DFSMSDSS INVOKED VIA CROSS MEMORY
    APPLICATION INTERFACE
    ARC0640I ARCFRTM - ADR016I (002)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCFRTM - ADR050I (003)-PRIME(02), DFSMSDSS INVOKED VIA CROSS MEMORY
    APPLICATION INTERFACE
    ARC0640I ARCFRTM - ADR016I (003)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCFRTM - ADR050I (004)-PRIME(02), DFSMSDSS INVOKED VIA CROSS MEMORY
    APPLICATION INTERFACE
    ARC0640I ARCFRTM - ADR016I (004)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCFRTM - ADR050I (005)-PRIME(02), DFSMSDSS INVOKED VIA CROSS MEMORY
    APPLICATION INTERFACE
    ARC0640I ARCFRTM - ADR016I (005)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCFRTM - ADR006I (005)-STEND(01), 2012.244 13:38:55 EXECUTION BEGINS
    ARC0640I ARCFRTM - ADR241I (005)-DDTFP(01), TARGET VTOC BEGINNING AT 000000005:00
    AND ENDING AT 000000016:14 IS OVERLAID
    ARC0640I ARCFRTM - ADR806I (005)-T0MI (02), VOLUME SBOXJ9 WAS COPIED USING A FAST
    REPLICATION FUNCTION
    ARC0640I ARCFRTM - ADR006I (005)-STEND(02), 2012.244 13:38:55 EXECUTION ENDS
    ARC0640I ARCFRTM - ADR013I (005)-CLTSK(01), 2012.244 13:38:55 TASK COMPLETED WITH
    RETURN CODE 0000
    
    
    -----
    
    ARC0640I ARCFRTM - ADR006I (004)-STEND(01), 2012.244 13:38:55 EXECUTION BEGINS
    ARC0640I ARCFRTM - ADR241I (004)-DDTFP(01), TARGET VTOC BEGINNING AT 000000005:00
    AND ENDING AT 000000016:14 IS OVERLAID
    ARC0640I ARCFRTM - ADR806I (004)-T0MI (02), VOLUME SBOXJ8 WAS COPIED USING A FAST
    REPLICATION FUNCTION
    ARC0640I ARCFRTM - ADR006I (004)-STEND(02), 2012.244 13:38:55 EXECUTION ENDS
    ARC0640I ARCFRTM - ADR013I (004)-CLTSK(01), 2012.244 13:38:56 TASK COMPLETED WITH
    RETURN CODE 0000
    ARC0640I ARCFRTM - ADR006I (003)-STEND(01), 2012.244 13:38:55 EXECUTION BEGINS
    ARC0640I ARCFRTM - ADR241I (003)-DDTFP(01), TARGET VTOC BEGINNING AT 000000005:00
    AND ENDING AT 000000016:14 IS OVERLAID
    ARC0640I ARCFRTM - ADR806I (003)-T0MI (02), VOLUME SBOXJ1 WAS COPIED USING A FAST
    REPLICATION FUNCTION
    ARC0640I ARCFRTM - ADR006I (003)-STEND(02), 2012.244 13:38:55 EXECUTION ENDS
    ARC0640I ARCFRTM - PAGE 0003   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVICES
    2012.244 13:38
    ARC0640I ARCFRTM - ADR013I (003)-CLTSK(01), 2012.244 13:38:56 TASK COMPLETED WITH
    RETURN CODE 0000
    ARC0640I ARCFRTM - PAGE 0002   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVICES
    2012.244 13:38
    ARC0640I ARCFRTM - ADR006I (002)-STEND(01), 2012.244 13:38:55 EXECUTION BEGINS
    ARC0640I ARCFRTM - ADR241I (002)-DDTFP(01), TARGET VTOC BEGINNING AT 000000005:00
    AND ENDING AT 000000016:14 IS OVERLAID
    ARC0640I ARCFRTM - ADR806I (002)-T0MI (02), VOLUME SBOXJ0 WAS COPIED USING A FAST
    REPLICATION FUNCTION
    ARC0640I ARCFRTM - ADR006I (002)-STEND(02), 2012.244 13:38:55 EXECUTION ENDS
    ARC0640I ARCFRTM - ADR013I (002)-CLTSK(01), 2012.244 13:38:57 TASK COMPLETED WITH
    RETURN CODE 0000
    ARC0640I ARCFRTM - ADR012I (SCH)-DSSU (01), 2012.244 13:38:57 DFSMSDSS PROCESSING
    COMPLETE. HIGHEST RETURN CODE IS 0000
    ARC1805I THE FOLLOWING 00004 VOLUME(S) WERE SUCCESSFULLY PROCESSED BY FAST
    REPLICATION BACKUP OF COPY POOL DSN$DB0B$LG
    ARC1805I (CONT.) SBOXJ0
    ARC1805I (CONT.) SBOXJ1
    ARC1805I (CONT.) SBOXJ8
    ARC1805I (CONT.) SBOXJ9
    **ARC1802I FAST REPLICATION BACKUP DUMP HAS COMPLETED FOR COPY POOL DSN$DB0B$LG, AT**
    **13:42:25 ON 2012/08/31,**
    **FUNCTION RC=0000, MAXIMUM VOLUME RC=0000, CAPTURE CATALOG RC=0000**

The log also shows the DUMP activities, as shown in Example 10-20.

_Example 10-20  Sample of full volume dump activities for FRBACKUP with the DUMP option_

    ARC0622I FULL VOLUME DUMP STARTING ON VOLUME SBOXJ8(SMS) AT 13:42:07 ON
    2012/08/31, SYSTEM SC64, TASK ID=ARCDVOL1 ,
    TO DUMP CLASS(ES)=  DB0BTST1
    ARC0728I VTOC FOR VOLUME SBOXJ8 COPIED TO DATA SET
    HSM.DUMPVTOC.T553813.VSBOXJ8.D12244 ON VOLUME SBXHS6
    ARC0640I ARCDVOL1 - PAGE 0001   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVICES
    2012.244 13:42
    ARC0640I ARCDVOL1 - ADR035I (SCH)-PRIME(06), INSTALLATION EXIT ALTERED BYPASS FAC
    CLASS CHK DEFAULT TO YES
    ARC0640I ARCDVOL1 - ADR035I (SCH)-PRIME(03), INSTALLATION EXIT ALTERED WORKUNIT
    DEFAULT TO
    ARC0640I ARCDVOL1 - DUMP FULL INDDNAME(SYS01492)     -
    
    
    -----
    
    ARC0640I ARCDVOL1 - OUTDDNAME(SYS01488) ARC0640I ARCDVOL1 - ALLEXCP ALLDATA(*) OPTIMIZE(3) TOLERATE(IOERROR)
    ARC0640I ARCDVOL1 - ADR101I (R/I)-RI01 (01), TASKID 001 HAS BEEN ASSIGNED TO
    COMMAND 'DUMP '
    ARC0640I ARCDVOL1 - ADR109I (R/I)-RI01 (01), 2012.244 13:42:07 INITIAL SCAN OF
    USER CONTROL STATEMENTS COMPLETED
    ARC0640I ARCDVOL1 - ADR050I (001)-PRIME(01), DFSMSDSS INVOKED VIA APPLICATION
    INTERFACE
    ARC0640I ARCDVOL1 - ADR016I (001)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCDVOL1 - ADR006I (001)-STEND(01), 2012.244 13:42:07 EXECUTION BEGINS
    ARC0640I ARCDVOL1 - ADR006I (001)-STEND(02), 2012.244 13:42:16 EXECUTION ENDS
    ARC0640I ARCDVOL1 - ADR013I (001)-CLTSK(01), 2012.244 13:42:16 TASK COMPLETED
    WITH RETURN CODE 0000
    ARC0640I ARCDVOL1 - ADR012I (SCH)-DSSU (01), 2012.244 13:42:16 DFSMSDSS
    PROCESSING COMPLETE. HIGHEST RETURN CODE IS 0000
    ARC0637I DUMP COPY OF VOLUME SBOXJ8 COMPLETE, DCLASS=DB0BTST1, EXPDT=2012/09/01
    ARC0623I FULL VOLUME DUMP OF VOLUME SBOXJ8 ENDING AT 13:42:17, PROCESSING
    SUCCESSFUL
    ARC0622I FULL VOLUME DUMP STARTING ON VOLUME SBOXJ9(SMS) AT 13:42:17 ON
    2012/08/31, SYSTEM SC64, TASK ID=ARCDVOL1 ,
    TO DUMP CLASS(ES)=  DB0BTST1
    ARC0728I VTOC FOR VOLUME SBOXJ9 COPIED TO DATA SET
    HSM.DUMPVTOC.T553813.VSBOXJ9.D12244 ON VOLUME SBXHS6
    ARC0640I ARCDVOL1 - PAGE 0001   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVICES
    2012.244 13:42
    ARC0640I ARCDVOL1 - ADR035I (SCH)-PRIME(06), INSTALLATION EXIT ALTERED BYPASS FAC
    CLASS CHK DEFAULT TO YES
    ARC0640I ARCDVOL1 - ADR035I (SCH)-PRIME(03), INSTALLATION EXIT ALTERED WORKUNIT
    DEFAULT TO
    ARC0640I ARCDVOL1 - DUMP FULL INDDNAME(SYS01494)      ARC0640I ARCDVOL1 - OUTDDNAME(SYS01488) ARC0640I ARCDVOL1 - ALLEXCP ALLDATA(*) OPTIMIZE(3) TOLERATE(IOERROR)
    ARC0640I ARCDVOL1 - ADR101I (R/I)-RI01 (01), TASKID 001 HAS BEEN ASSIGNED TO
    COMMAND 'DUMP '
    ARC0640I ARCDVOL1 - ADR109I (R/I)-RI01 (01), 2012.244 13:42:17 INITIAL SCAN OF
    USER CONTROL STATEMENTS COMPLETED
    ARC0640I ARCDVOL1 - ADR050I (001)-PRIME(01), DFSMSDSS INVOKED VIA APPLICATION
    INTERFACE
    ARC0640I ARCDVOL1 - ADR016I (001)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCDVOL1 - ADR006I (001)-STEND(01), 2012.244 13:42:17 EXECUTION BEGINS
    ARC0640I ARCDVOL1 - ADR006I (001)-STEND(02), 2012.244 13:42:25 EXECUTION ENDS
    ARC0640I ARCDVOL1 - ADR013I (001)-CLTSK(01), 2012.244 13:42:25 TASK COMPLETED
    WITH RETURN CODE 0000
    ARC0640I ARCDVOL1 - ADR012I (SCH)-DSSU (01), 2012.244 13:42:25 DFSMSDSS
    PROCESSING COMPLETE. HIGHEST RETURN CODE IS 0000
    **ARC1802I FAST REPLICATION BACKUP DUMP HAS COMPLETED FOR COPY POOL DSN$DB0B$LG, AT**
    **13:42:25 ON 2012/08/31,**
    **FUNCTION RC=0000, MAXIMUM VOLUME RC=0000, CAPTURE CATALOG RC=0000**
    ARC0637I DUMP COPY OF VOLUME SBOXJ9 COMPLETE, DCLASS=DB0BTST1, EXPDT=2012/09/01
    ARC0623I FULL VOLUME DUMP OF VOLUME SBOXJ9 ENDING AT 13:42:34, PROCESSING
    SUCCESSFUL


-----

**Note:** When a DFSMShsm Fast Replication backup is initiated by issuing an **FRBACKUP**
command to pair a primary volume with a secondary volume in the backup storage group,
target volumes that were already processed potentially will be re-examined. Therefore,
multiple ARC1809I return code 2 messages result for the same target volume. The
suppression of ARC1809I messages is possible with a patch, which means that _all_
ARC1809I messages are omitted.

To manage the behavior with the **SETSYS** command instead, use a new **SETSYS** parameter:

FASTREPLICATION(VOLUMEPAIRMESSAGES(YES|NO))

Setting this parameter to YES will limit the informational message to be issued one time for
each target volume for each source storage group. Changing the setting to NO causes only
one ARC1809I message for any secondary volume.

The current patch to disable the occurrence of ARC1809I messages will continue to be
supported.

You can also check the data sets that were backed up by using **FRBACKUP** by using this
command:

LIST COPYPOOL( _cpname_ ) DATASETS

The sample output of the **LIST** command is shown in Example 10-21.

_Example 10-21  Sample output of the LIST COPYPOOL(DSN$DB0B$LG) DATASETS command_

    -- DFSMShsm CONTROL DATA SET -- COPY POOL -- LISTING -- AT 14:05:36 ON 12/08/31
    FOR SYSTEM=SC64
    
    COPYPOOL=DSN$DB0B$LG          ,VER=008,GEN=000,CATINFO=Y
    CATALOG INFORMATION DATA SET NAME=HSM.HSMCIDS.D12244.T133855.C001
    TKN(C)=C'MHLRESA'
    TKN(H)=X'D4C8D3D9C5E2C1'
    
    CATALOG NAME = UCAT.DB0BLOGS
    
    DATA SET NAME
    DB0BA.ARCHLOG1.A0000001
    DB0BA.ARCHLOG1.A0000002
    DB0BA.ARCHLOG1.A0000003
    DB0BA.ARCHLOG1.A0000004
    DB0BA.ARCHLOG1.A0000005
    DB0BA.ARCHLOG1.A0000006
    DB0BA.ARCHLOG1.A0000007
    DB0BA.ARCHLOG1.A0000008
    DB0BA.ARCHLOG1.A0000009
    DB0BA.ARCHLOG1.A0000010
    DB0BA.ARCHLOG1.A0000011
    DB0BA.ARCHLOG1.A0000012
    DB0BA.ARCHLOG1.A0000013
    DB0BA.ARCHLOG1.A0000014
    DB0BA.ARCHLOG1.A0000015
    DB0BA.ARCHLOG1.A0000016
    DB0BA.ARCHLOG1.A0000017
    DB0BA.ARCHLOG1.A0000018
    DB0BA.ARCHLOG1.A0000019
    
    
    -----
    
    DB0BA.ARCHLOG1.B0000001
    DB0BA.ARCHLOG1.B0000002
    DB0BA.ARCHLOG1.B0000003
    DB0BA.ARCHLOG1.B0000004
    DB0BA.ARCHLOG1.B0000005
    DB0BA.ARCHLOG1.B0000006
    DB0BA.ARCHLOG1.B0000007
    DB0BA.ARCHLOG1.B0000008
    DB0BA.ARCHLOG1.B0000009
    DB0BA.ARCHLOG1.B0000010
    DB0BA.ARCHLOG1.B0000011
    DB0BA.ARCHLOG1.B0000012
    DB0BA.ARCHLOG1.B0000013
    DB0BA.ARCHLOG1.B0000014
    DB0BA.ARCHLOG1.B0000015
    DB0BA.ARCHLOG1.B0000016
    DB0BA.ARCHLOG1.B0000017
    DB0BA.ARCHLOG1.B0000018
    DB0BA.ARCHLOG1.B0000019
    DB0BA.ARCHLOG2.A0000001
    1
    -- DFSMShsm CONTROL DATA SET -- COPY POOL -- LISTING -- AT 14:05:36 ON 12/08/31
    FOR SYSTEM=SC64
    
    CATALOG NAME = UCAT.DB0BLOGS                (CONTINUED)
    
    DATA SET NAME
    DB0BA.ARCHLOG2.A0000002
    DB0BA.ARCHLOG2.A0000003
    DB0BA.ARCHLOG2.A0000004
    DB0BA.ARCHLOG2.A0000005
    DB0BA.ARCHLOG2.A0000006
    DB0BA.ARCHLOG2.A0000007
    DB0BA.ARCHLOG2.A0000008
    DB0BA.ARCHLOG2.A0000009
    DB0BA.ARCHLOG2.A0000010
    DB0BA.ARCHLOG2.A0000011
    DB0BA.ARCHLOG2.A0000012
    DB0BA.ARCHLOG2.A0000013
    DB0BA.ARCHLOG2.A0000014
    DB0BA.ARCHLOG2.A0000015
    DB0BA.ARCHLOG2.A0000016
    DB0BA.ARCHLOG2.A0000017
    DB0BA.ARCHLOG2.A0000018
    DB0BA.ARCHLOG2.A0000019
    DB0BA.ARCHLOG2.B0000001
    DB0BA.ARCHLOG2.B0000002
    DB0BA.ARCHLOG2.B0000003
    DB0BA.ARCHLOG2.B0000004
    DB0BA.ARCHLOG2.B0000005
    DB0BA.ARCHLOG2.B0000006
    DB0BA.ARCHLOG2.B0000007
    DB0BA.ARCHLOG2.B0000008
    DB0BA.ARCHLOG2.B0000009
    DB0BA.ARCHLOG2.B0000010
    
    
    -----
    
    DB0BA.ARCHLOG2.B0000011
    DB0BA.ARCHLOG2.B0000012
    DB0BA.ARCHLOG2.B0000013
    DB0BA.ARCHLOG2.B0000014
    DB0BA.ARCHLOG2.B0000015
    DB0BA.ARCHLOG2.B0000016
    DB0BA.ARCHLOG2.B0000017
    DB0BA.ARCHLOG2.B0000018
    DB0BA.ARCHLOG2.B0000019
    DB0BB.BSDS01
    DB0BB.BSDS02
    DB0BL.LOGCOPY1.DS01
    DB0BL.LOGCOPY1.DS02
    DB0BL.LOGCOPY1.DS03
    DB0BL.LOGCOPY2.DS01
    DB0BL.LOGCOPY2.DS02
    1
    -- DFSMShsm CONTROL DATA SET -- COPY POOL -- LISTING -- AT 14:05:36 ON 12/08/31
    FOR SYSTEM=SC64
    
    CATALOG NAME = UCAT.DB0BLOGS                (CONTINUED)
    
    DATA SET NAME
    DB0BL.LOGCOPY2.DS03
    
    TOTAL NUMBER OF DATA SETS =     84

### 10.5.4 Perform fast replicate recover (FRRECOV)

You can use the **FRRECOV** command to recover a copy pool or individual volumes and data sets
from the managed copy pool copies. The backup copy to be recovered can reside on either
DASD or tape. If the backup copy resides on both DASD and tape, the default is to use the
DASD backup copy.

To restrict the recovery to backup copy versions that reside on DASD or tape, use the
FROMDASD (for DASD) or FROMDUMP (for tape) options. If the backup copy version is not
on either DASD or tape, the recovery request fails.

When DATE, GENERATION, TOKEN, or VERSION is specified, the corresponding backup
copy is recovered. If no specific backup copy is specified, an attempt to recover generation
zero occurs. If no valid backup copy (either the indicated or implicit) is found, on DASD or
tape, the recovery request fails.

A specific dump class to recover the version from can be specified when you are recovering
from a dump copy on tape. When recovery is performed at the copy pool level, and the dump
copy to recover is a partial dump, the recovery request will fail unless the PARTIALOK option
is specified.

You can either recover an individual data set or the whole copy pool. We show how to recover
for both cases.

**Using FRRECOV to recover a data set**
You can recover an individual data set by using the **FRRECOV** command. The syntax is shown:

FRRECOV DSNAME( _dsn_ ) FROMDASD | FROMDUMP


-----

Again, we use **FRRECOV** with a batch job.

In our example, we try to recover data set DB0BA.ARCHLOG1.A0000001 from a full volume
dump. The JCL is shown in Example 10-22.

_Example 10-22  Sample JCL for a data set recover by using FRRECOV from a dump_

    //MHLRESAY JOB (999,POK),'MHLRES1',CLASS=A,MSGCLASS=X,
    // NOTIFY=&SYSUID,TIME=1440,REGION=0M
    /*JOBPARM SYSAFF=*
    //HSMFR EXEC PGM=IKJEFT01
    //SYSPRINT DD SYSOUT=*
    //PRINTER DD SYSOUT=*
    //INDEX  DD SYSOUT=*
    //SYSIN  DD SYSOUT=*
    //SYSTSPRT DD SYSOUT=*
    //SYSTSIN DD *
    HSEND FRRECOV DSNAME(DB0BA.ARCHLOG1.A0000001) REPLACE FROMDUMP

After the job completes, you can verify the recovery process in the DFSMShsm log, as shown
in Example 10-23.

_Example 10-23  DFSMShsm log_

    ARC1801I FAST REPLICATION DATA SET RECOVERY IS STARTING FOR DATA SET
    DB0BA.ARCHLOG1.A0000001, AT 14:13:45 ON 2012/08/31
    ARC1861I THE FOLLOWING 0001 DATA SET(S) WERE SUCCESSFULLY PROCESSED DURING FAST
    REPLICATION DATA SET RECOVERY:
    ARC1861I (CONT.) DB0BA.ARCHLOG1.A0000001, COPYPOOL=DSN$DB0B$LG, DEVTYPE=TAPE
    ARC1802I FAST REPLICATION DATA SET RECOVERY HAS COMPLETED FOR DATA SET
    DB0BA.ARCHLOG1.A0000001,
    AT 14:14:06 ON 2012/08/31, FUNCTION RC=0000, MAXIMUM DATA SET RC=0000

The recovery activities from the DFSMShsm dump log are shown in Example 10-24.

_Example 10-24  DFSMShsm dump log_

    DFSMSHSM DUMP LOG, TIME 13:42:34, DATE 12/08/31
    ARC0640I GDSN01 - PAGE 0001   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVICES
    2012.244 14:13
    ARC0640I GDSN01 - ADR035I (SCH)-PRIME(06), INSTALLATION EXIT ALTERED BYPASS FAC
    CLASS CHK DEFAULT TO YES
    ARC0640I GDSN01 - ADR035I (SCH)-PRIME(03), INSTALLATION EXIT ALTERED WORKUNIT
    DEFAULT TO
    ARC0640I GDSN01 - RESTORE DS(INCLUDE(DB0BA.ARCHLOG1.A0000001
    )) ARC0640I GDSN01 - INDDNAME(SYS01524) OUTDYNAM(SBOXJ0,3390  ) REPLACE
    CANCELERROR ARC0640I GDSN01 - BYPASSACS(DB0BA.ARCHLOG1.A0000001           ) ARC0640I GDSN01 - STORCLAS(DB0BARCH           ) CATALOG FORCECP(0) ARC0640I GDSN01 - MGMTCLAS(MCDB22            )
    ARC0640I GDSN01 - ADR101I (R/I)-RI01 (01), TASKID 001 HAS BEEN ASSIGNED TO
    COMMAND 'RESTORE '
    ARC0640I GDSN01 - ADR109I (R/I)-RI01 (01), 2012.244 14:13:45 INITIAL SCAN OF USER
    CONTROL STATEMENTS COMPLETED
    ARC0640I GDSN01 - ADR050I (001)-PRIME(01), DFSMSDSS INVOKED VIA APPLICATION
    INTERFACE
    
    
    -----
    
    ARC0640I GDSN01 - ADR016I (001)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR THIS
    TASK
    ARC0640I GDSN01 - ADR006I (001)-STEND(01), 2012.244 14:13:45 EXECUTION BEGINS
    ARC0640I GDSN01 - ADR780I (001)-TDDS (01),
    THE INPUT DUMP DATA SET BEING PROCESSED IS IN FULL VOLUME FORMAT AND WAS CREATED
    BY DFSMSDSS VERSION 1
    ARC0640I GDSN01 -             RELEASE 13 MODIFICATION LEVEL 0 ON
    2012.244 13:39:32
    ARC0640I GDSN01 - ADR378I (001)-TDDS (01), THE FOLLOWING DATA SETS WERE
    SUCCESSFULLY PROCESSED FROM VOLUME SBOXJ0
    ARC0640I GDSN01 -              DB0BA.ARCHLOG1.A0000001
    RESTORED ON SBOXJ0
    ARC0640I GDSN01 - ADR006I (001)-STEND(02), 2012.244 14:14:06 EXECUTION ENDS
    ARC0640I GDSN01 - ADR013I (001)-CLTSK(01), 2012.244 14:14:06 TASK COMPLETED WITH
    RETURN CODE 0000
    ARC0640I GDSN01 - ADR012I (SCH)-DSSU (01), 2012.244 14:14:06 DFSMSDSS PROCESSING
    COMPLETE. HIGHEST RETURN CODE IS 0000
    DFSMSHSM MIGRATION LOG, TIME 14:05:00, DATE 12/08/31

**Using FRRECOV to recover a copy pool**
This option indicates to DFSMShsm to recover all source volumes that are associated with
the named copy pool.

The command syntax to use the **FRRECOV** command to recover a copy pool is shown:

FRRECOV COPYPOOL( _cpname_ ) FROMDASD | FROMDUMP

The sample JCL to recover the copy pool DSN$DB0B$LG is shown in Example 10-25.

_Example 10-25  Sample JCL to recover a copy pool by using FRRECOV_

    //MHLRESAR JOB (999,POK),'MHLRES1',CLASS=A,MSGCLASS=X,
    // NOTIFY=&SYSUID,TIME=1440,REGION=0M
    /*JOBPARM SYSAFF=*
    //HSMFR EXEC PGM=IKJEFT01
    //SYSPRINT DD SYSOUT=*
    //PRINTER DD SYSOUT=*
    //INDEX  DD SYSOUT=*
    //SYSIN  DD SYSOUT=*
    //SYSTSPRT DD SYSOUT=*
    //SYSTSIN DD *
    HSEND FRRECOV CP(DSN$DB0B$LG) FROMDASD

Before the recovery of a copy pool, DFSMShsm deallocates the specified catalogs. If
DFSMShsm cannot deallocate any of the catalogs, the **FRRECOV** operation might fail. **CATALOG**
reallocates the catalogs when it is used. Example 10-26 shows that the system automatically
deallocates the catalog that is involved.

_Example 10-26  Syslog activities for FRRECOV_

    **STC28942 00000290 F CATALOG,UNALLOCATE(UCAT.DB0BLOGS**
    INTERNAL 00000090 IEC351I CATALOG ADDRESS SPACE MODIFY COMMAND ACTIVE
    00000290 IEF196I IGD104I UCAT.DB0BLOGS
    00000290 IEF196I DDNAME=SYS00177
    INTERNAL 00000090 IEC352I CATALOG ADDRESS SPACE MODIFY COMMAND COMPLETED
    STC28942 00000090 IGD103I SMS ALLOCATED TO DDNAME SYS01528
    
    
    -----
    
    STC28942 00000090 IGD103I SMS ALLOCATED TO DDNAME SYS01529
    STC28942 00000090 IGD103I SMS ALLOCATED TO DDNAME SYS01530
    STC28942 00000090 IGD103I SMS ALLOCATED TO DDNAME SYS01531
    STC28942 00000090 ARC1805I THE FOLLOWING 00004 VOLUME(S) WERE 945
    945 00000090 ARC1805I (CONT.) SUCCESSFULLY PROCESSED BY FAST REPLICATION R
    945 00000090 ARC1805I (CONT.) OF COPY POOL DSN$DB0B$LG
    STC28942 00000090 ARC1805I (CONT.) SBOXJ0
    STC28942 00000090 ARC1805I (CONT.) SBOXJ1
    STC28942 00000090 ARC1805I (CONT.) SBOXJ8
    STC28942 00000090 ARC1805I (CONT.) SBOXJ9
    **STC28942 00000090 ARC1802I FAST REPLICATION RECOVERY HAS COMPLETED FOR 950**
    **950 00000090 ARC1802I (CONT.) COPY POOL DSN$DB0B$LG, AT 14:26:12 ON 2012/0**
    **950 00000090 ARC1802I (CONT.) FUNCTION RC=0000, MAXIMUM VOLUME RC=0000**

When the recovery process completes, you can also verify it in the DFSMShsm log, as shown in Example 10-27.

_Example 10-27  DFSMShsm backlog activities for the copy pool recover with FRRECOV_

    ARC1801I FAST REPLICATION RECOVERY IS STARTING FOR COPY POOL DSN$DB0B$LG, AT
    14:26:10 ON 2012/08/31
    ARC0640I ARCFRTM - PAGE 0001   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVICES
    2012.244 14:26
    ARC0640I ARCFRTM - ADR035I (SCH)-PRIME(06), INSTALLATION EXIT ALTERED BYPASS FAC
    CLASS CHK DEFAULT TO YES
    ARC0640I ARCFRTM - PARALLEL
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 001 HAS BEEN ASSIGNED TO
    COMMAND 'PARALLEL '
    ARC0640I ARCFRTM - COPY IDY(SBOXK0) ODY(SBOXJ0) DUMPCOND FR(REQ) PUR ALLX
    ALLD(*)    ARC0640I ARCFRTM - DEBUG(FRMSG(DTL))
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 002 HAS BEEN ASSIGNED TO
    COMMAND 'COPY '
    ARC0640I ARCFRTM - COPY IDY(SBOXK1) ODY(SBOXJ1) DUMPCOND FR(REQ) PUR ALLX
    ALLD(*)    ARC0640I ARCFRTM - DEBUG(FRMSG(DTL))
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 003 HAS BEEN ASSIGNED TO
    COMMAND 'COPY '
    ARC0640I ARCFRTM - COPY IDY(SBOXKA) ODY(SBOXJ8) DUMPCOND FR(REQ) PUR ALLX
    ALLD(*)    ARC0640I ARCFRTM - DEBUG(FRMSG(DTL))
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 004 HAS BEEN ASSIGNED TO
    COMMAND 'COPY '
    ARC0640I ARCFRTM - COPY IDY(SBOXKB) ODY(SBOXJ9) DUMPCOND FR(REQ) PUR ALLX
    ALLD(*)    ARC0640I ARCFRTM - DEBUG(FRMSG(DTL))
    ARC0640I ARCFRTM - ADR101I (R/I)-RI01 (01), TASKID 005 HAS BEEN ASSIGNED TO
    COMMAND 'COPY '
    ARC0640I ARCFRTM - ADR109I (R/I)-RI01 (01), 2012.244 14:26:10 INITIAL SCAN OF
    USER CONTROL STATEMENTS COMPLETED
    ARC0640I ARCFRTM - ADR014I (SCH)-DSSU (02),
    2012.244 14:26:10 ALL PREVIOUSLY SCHEDULED TASKS COMPLETED. PARALLEL MODE NOW IN
    EFFECT
    ARC0640I ARCFRTM - ADR050I (002)-PRIME(02), DFSMSDSS INVOKED VIA CROSS MEMORY
    APPLICATION INTERFACE
    
    
    -----
    
    ARC0640I ARCFRTM - ADR016I (002)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCFRTM - ADR050I (003)-PRIME(02), DFSMSDSS INVOKED VIA CROSS MEMORY
    APPLICATION INTERFACE
    ARC0640I ARCFRTM - ADR016I (003)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCFRTM - ADR050I (004)-PRIME(02), DFSMSDSS INVOKED VIA CROSS MEMORY
    APPLICATION INTERFACE
    ARC0640I ARCFRTM - ADR016I (004)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCFRTM - ADR050I (005)-PRIME(02), DFSMSDSS INVOKED VIA CROSS MEMORY
    APPLICATION INTERFACE
    ARC0640I ARCFRTM - ADR016I (005)-PRIME(01), RACF LOGGING OPTION IN EFFECT FOR
    THIS TASK
    ARC0640I ARCFRTM - ADR006I (002)-STEND(01), 2012.244 14:26:10 EXECUTION BEGINS
    ARC0640I ARCFRTM - ADR241I (002)-DDTFP(01), TARGET VTOC BEGINNING AT 000000005:00
    AND ENDING AT 000000016:14 IS OVERLAID
    ARC0640I ARCFRTM - ADR806I (002)-T0MI (02), VOLUME SBOXK0 WAS COPIED USING A FAST
    REPLICATION FUNCTION
    ARC0640I ARCFRTM - ADR006I (002)-STEND(02), 2012.244 14:26:10 EXECUTION ENDS
    ARC0640I ARCFRTM - ADR013I (002)-CLTSK(01), 2012.244 14:26:10 TASK COMPLETED WITH
    RETURN CODE 0000
    ARC0640I ARCFRTM - ADR006I (004)-STEND(01), 2012.244 14:26:10 EXECUTION BEGINS
    ARC0640I ARCFRTM - ADR241I (004)-DDTFP(01), TARGET VTOC BEGINNING AT 000000005:00
    AND ENDING AT 000000016:14 IS OVERLAID
    ARC0640I ARCFRTM - ADR806I (004)-T0MI (02), VOLUME SBOXKA WAS COPIED USING A FAST
    REPLICATION FUNCTION
    ARC0640I ARCFRTM - ADR006I (004)-STEND(02), 2012.244 14:26:10 EXECUTION ENDS
    ARC0640I ARCFRTM - ADR013I (004)-CLTSK(01), 2012.244 14:26:11 TASK COMPLETED WITH
    RETURN CODE 0000
    ARC0640I ARCFRTM - ADR006I (003)-STEND(01), 2012.244 14:26:10 EXECUTION BEGINS
    ARC0640I ARCFRTM - ADR241I (003)-DDTFP(01), TARGET VTOC BEGINNING AT 000000005:00
    AND ENDING AT 000000016:14 IS OVERLAID
    ARC0640I ARCFRTM - ADR806I (003)-T0MI (02), VOLUME SBOXK1 WAS COPIED USING A FAST
    REPLICATION FUNCTION
    ARC0640I ARCFRTM - ADR006I (003)-STEND(02), 2012.244 14:26:10 EXECUTION ENDS
    ARC0640I ARCFRTM - ADR013I (003)-CLTSK(01), 2012.244 14:26:11 TASK COMPLETED WITH
    RETURN CODE 0000
    ARC0640I ARCFRTM - ADR006I (005)-STEND(01), 2012.244 14:26:10 EXECUTION BEGINS
    ARC0640I ARCFRTM - PAGE 0002   5695-DF175 DFSMSDSS V1R13.0 DATA SET SERVICES
    2012.244 14:26
    ARC0640I ARCFRTM - ADR241I (005)-DDTFP(01), TARGET VTOC BEGINNING AT 000000005:00
    AND ENDING AT 000000016:14 IS OVERLAID
    ARC0640I ARCFRTM - ADR806I (005)-T0MI (02), VOLUME SBOXKB WAS COPIED USING A FAST
    REPLICATION FUNCTION
    ARC0640I ARCFRTM - ADR006I (005)-STEND(02), 2012.244 14:26:10 EXECUTION ENDS
    ARC0640I ARCFRTM - ADR013I (005)-CLTSK(01), 2012.244 14:26:12 TASK COMPLETED WITH
    RETURN CODE 0000
    ARC0640I ARCFRTM - ADR012I (SCH)-DSSU (01), 2012.244 14:26:12 DFSMSDSS PROCESSING
    COMPLETE. HIGHEST RETURN CODE IS 0000
    ARC0400I VOLUME SBOXJ0 IS 96% FREE, 0000000014 FREE TRACK(S), 000029087 FREE
    CYLINDER(S), FRAG .000
    ARC0401I LARGEST EXTENTS FOR SBOXJ0 ARE CYLINDERS   29087, TRACKS   436315
    ARC0402I VTOC FOR SBOXJ0 IS 00180 TRACKS(0009000 DSCBS), 0008959 FREE DSCBS(99%
    OF TOTAL)
    
    
    -----
    
    ARC0400I VOLUME SBOXJ1 IS 96% FREE, 0000000014 FREE TRACK(S), 000028991 FREE
    CYLINDER(S), FRAG .000
    ARC0401I LARGEST EXTENTS FOR SBOXJ1 ARE CYLINDERS   28991, TRACKS   434875
    ARC0402I VTOC FOR SBOXJ1 IS 00180 TRACKS(0009000 DSCBS), 0008957 FREE DSCBS(99%
    OF TOTAL)
    ARC0400I VOLUME SBOXJ8 IS 97% FREE, 0000000005 FREE TRACK(S), 000009727 FREE
    CYLINDER(S), FRAG .000
    ARC0401I LARGEST EXTENTS FOR SBOXJ8 ARE CYLINDERS   9727, TRACKS   145905
    ARC0402I VTOC FOR SBOXJ8 IS 00180 TRACKS(0009000 DSCBS), 0008989 FREE DSCBS(99%
    OF TOTAL)
    ARC0400I VOLUME SBOXJ9 IS 98% FREE, 0000000005 FREE TRACK(S), 000009847 FREE
    CYLINDER(S), FRAG .000
    ARC0401I LARGEST EXTENTS FOR SBOXJ9 ARE CYLINDERS   9847, TRACKS   147705
    ARC0402I VTOC FOR SBOXJ9 IS 00180 TRACKS(0009000 DSCBS), 0008991 FREE DSCBS(99%
    OF TOTAL)
    ARC1805I THE FOLLOWING 00004 VOLUME(S) WERE SUCCESSFULLY PROCESSED BY FAST
    REPLICATION RECOVERY OF COPY POOL DSN$DB0B$LG
    ARC1805I (CONT.) SBOXJ0
    ARC1805I (CONT.) SBOXJ1
    ARC1805I (CONT.) SBOXJ8
    ARC1805I (CONT.) SBOXJ9
    **ARC1802I FAST REPLICATION RECOVERY HAS COMPLETED FOR COPY POOL DSN$DB0B$LG, AT**
    **14:26:12 ON 2012/08/31, FUNCTION RC=0000,**
    MAXIMUM VOLUME RC=0000


-----

