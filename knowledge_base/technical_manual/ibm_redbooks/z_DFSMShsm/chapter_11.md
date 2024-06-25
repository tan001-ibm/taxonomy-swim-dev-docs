# 11. Maintaining your DFSMShsm environment

In this chapter, we describe the main procedures that are performed by a storage manager
regularly. These procedures include holding and releasing DFSMShsm functions, canceling
tasks, and changing DFSMShsm control parameters, among other activities.

## 11.1 Maintaining DFSMShsm

To keep DFSMShsm up and running, the storage administrator needs to perform these
administrative activities regularly:

- Holding DFSMShsm functions
- Releasing DFSMShsm functions
- Canceling DFSMShsm requests
- Changing DFSMShsm control parameters
- Restarting DFSMShsm after an abnormal end
- Expiring old data set backup versions
- Recycling tapes and converting to new tape volumes
- Moving data sets to new DASD volumes
- Reorganizing the control data sets (CDSs)
- Auditing DFSMShsm

All of these storage manager activities are described. The main DFSMShsm management
concepts and examples are explained.

For detailed information about setting up your product, see Chapter 3, “Getting started".

### 11.1.1 Holding DFSMShsm functions

The **HOLD** command is used to selectively prevent or interrupt DFSMShsm functions from
running without stopping DFSMShsm. You can use the command to prevent tape-related
processing if, for example, you are drive-constrained at a particular moment or hardware
errors exist.

For functions that process on a volume basis (backup, dump, migration, and recover), you can
choose whether you want to interrupt the function at the end of the data set that is being
processed or at the end of the volume that is being processed.

The **HOLD** command attempts to terminate the process at a reasonable point, so that the
function can be restarted or reissued later. A function is held until it is released or DFSMShsm
is shut down and restarted. Depending on the **HOLD** command that you issue, you can prevent
processing of all DFSMShsm functions or selectively choose the following functions to hold:

- Command
- Automatic primary space management
- Automatic secondary space management
- Common queue
- Fast replication backup
- Fast replication recovery
- Recall or deletion of a migrated data set
- Command backup
- Automatic backup
- Aggregate backup
- Aggregate recovery
- Command dump
- Automatic dump
- Audit command processing
- List
- Report
- Recovery, and restore


-----

- Recycle
- Logging
- Tape copy
- Tape replace
- Expiring backup versions

The **HOLD** command offers flexibility. The **HOLD** commands that you will use most are
described. For more details about the **HOLD** command, see the _DFSMShsm Storage_
_Administration Guide,_ SC26-0421.

Before you submit any DFSMShsm **HOLD** commands, you must ensure that you defined the
necessary RACF or AUTH settings to your user ID. 

You can issue all of the **HOLD** commands through TSO by issuing the **HSEND** command, or
through the console with the following command:

F _hsmtask,command_

As you plan your DFSMShsm daily processing window, DFSMShsm activities might compete
for the same resources, such as DASD, data sets, tape drives, or processor time. By holding
the conflicting task instead of shutting down DFSMShsm, you can give DFSMShsm a chance
to finish the current data set or volume processing before processing stops, preventing CDS
mismatches and data loss.

All functions that are held by command are automatically released after DFSMShsm is shut
down and restarted. If you want DFSMShsm to start with any specific function held, you can
add the correct **HOLD** command in the ARCCMDxx member in the DFSMShsm PARMLIB.

If you are in the middle of space management processing, and only a few tape drives are
available for use by batch processing, it might be necessary to stop your current space
management task to free extra drives so batch processing is not affected. Hold MIGRATION,
so DFSMShsm will stop the space management task after it finishes the current data set that
is being processed by issuing one of the commands that are shown in Example 11-1.

_Example 11-1  Preventing automatic migration_

`HSEND HOLD MIGRATION(AUTO)`

`HSEND HOLD AUTOMIGRATION`

The commands that are shown in Example 11-1 do not prevent users from manually
executing migration tasks.

To prevent or hold both user and automatic migration, you can issue the following command:

`HSEND HOLD MIGRATION`

**Important:** Every time a main function, such as migration, is held, all subfunctions are
held also. In this case, both automatic and command migrations are held.

If processing peaks are caused by DFSMShsm tasks, check the specific task that is causing
the peaks, and then hold it. If you are uncertain which task is in error, another approach is to
hold all tasks, and then release one task at a time, until you can determine the task in error.
The following command holds all DFSMShsm tasks, except for the common recall queue
(CRQ):

`HSEND HOLD ALL`


-----

To hold the common recall queue, issue the following command:

`HSEND HOLD COMMONQUEUE`

For certain functions, you can control whether you want the process to stop after the current
data set processing, or after current volume processing. With the following functions, you can
make this selection:

- Space management (automatic or command)
- Backup (automatic or command)
- Dump (automatic or command)
- Recover
- Restore

If you want to stop HSM task processing after the current volume processing ends, use the
**HOLD** command and the **ENDOFVOLUME** parameter. This command holds the dump after the
current dumps that are processing finish dumping the volume:

`HSEND HOLD DUMP ENDOFVOLUME`

You can hold a task after the current data set is processed. Issue the following command to
hold the dump after the current dumps that are processing finish dumping the data set:

`HSEND HOLD DUMP ENDOFDATASET`

**Note:** If neither the **ENDOFDATASET** nor **ENDOFVOLUME** parameter is specified, the default value
is **ENDOFDATASET** to space management, backup, recover, and restore functions, and
**ENDOFVOLUME** to dump, FRBACKUP dump, and FRBACKUP dumponly functions.

You might need to hold all processing from accessing a specific type of media. In these
scenarios, you can hold all DFSMShsm tasks that use the unavailable device. If you want to
prevent tape access due to scheduled maintenance, link issues, or hardware failures, you can
issue several of the following commands to stop all tape processing and still allow users to
access data in other media, as shown in Example 11-2.

_Example 11-2  Holding DFSMShsm functions separately_

     HSEND HOLD BACKUP(AUTO DSCOMMAND(TAPE))
     HSEND HOLD MIGRATION
     HSEND HOLD RECALL(TAPE)
     HSEND HOLD RECYCLE
     HSEND HOLD TAPECOPY
     HSEND HOLD TAPEREPL
     HSEND HOLD RECOVER(TAPEDATASET)
     HSEND HOLD FRRECOV(TAPE)
     HSEND HOLD COMMONQUEUE
     HSEND HOLD DUMP
     HSEND HOLD ABACKUP
     HSEND HOLD ARECOVER

**Note:** You might not need to issue all of these commands in your environment to prevent
tape processing. For information about the tasks to hold, see the _DFSMShsm Storage_
_Administration Guide,_ SC26-0421.


-----

### 11.1.2 Releasing DFSMShsm functions

Any function that can be held can be released with the **RELEASE** command. You are able to
release all DFSMShsm tasks that you can put on hold. All of the tasks have the same name
and parameters for both hold and release.

The only time that functions will not be released is if journaling was disabled. After journaling
is disabled, it holds all DFSMShsm functions. The **RELEASE** command will not be effective until
the CDSs are successfully backed up.

To check whether any functions are on hold, issue the following command from TSO or the
console. This command queries active requests:

`HSEND QUERY ACTIVE`

The command that is shown in Example 11-3 displays all of the DFSMShsm task statuses,
including those tasks that are on hold.

_Example 11-3  Output from the QUERY ACTIVE command_

    ARC0101I QUERY ACTIVE COMMAND STARTING ON HOST=2
    
    ARC0144I AUDIT=NOT HELD AND INACTIVE, LIST=NOT HELD AND INACTIVE, **RECYCLE=HELD**
    
    ARC0144I (CONT.) AND INACTIVE, REPORT=NOT HELD AND INACTIVE
    ARC0160I MIGRATION=NOT HELD, AUTOMIGRATION=HELD, RECALL=NOT HELD,
    ARC0160I (CONT.) TAPERECALL=NOT HELD, DATA SET MIGRATION=INACTIVE, VOLUME
    ARC0160I (CONT.) MIGRATION=INACTIVE, DATA SET RECALL=INACTIVE
    ARC0163I BACKUP=NOT HELD, AUTOBACKUP=HELD, RECOVERY=NOT HELD,
    ARC0163I (CONT.) TAPEDATASETRECOVERY=NOT HELD, DATA SET BACKUP=NOT HELD, VOLUME
    ARC0163I (CONT.) BACKUP=INACTIVE, DATA SET RECOVERY=INACTIVE, VOLUME
    ARC0163I (CONT.) RECOVERY=INACTIVE
    ARC0276I DATA SET BACKUP=INACTIVE, DATA SET BACKUP ACTUAL IDLETASKS=(ALLOC=00,
    ARC0276I (CONT.) MAX=00)
    
You can see several of the DFSMShsm functions, including RECYCLE, which is held.

If the reason that the task was held is no longer valid, such as the hardware maintenance
finishes, the link was reestablished, or the processor usage decreased, you can release the
recycle task with the following command:

`HSEND RELEASE RECYCLE`

If you set a main function on hold, such as migration, you cannot release only a subfunction of
this task, such as automatic migration. Example 11-4 shows you the results of trying to
release a subfunction of a held task.

_Example 11-4  Holding and releasing functions and subfunctions_

    F HSM1,HOLD BACKUP
    ARC0100I HOLD COMMAND COMPLETED
    F HSM1,RELEASE AUTOMIGRATION
    ARC0111I SUBFUNCTION MIGRATION(AUTO) CANNOT BE 664
    ARC0111I (CONT.) RELEASED WHILE MAIN FUNCTION MIGRATION IS HELD
    ARC0100I RELEASE COMMAND COMPLETED

To release only automatic space management, you must release backup, and then hold only
the user command **BACKUP** , as shown in Example 11-5.


-----

_Example 11-5  Holding and releasing functions and subfunctions_

HSEND RELEASE BACKUP
HSEND HOLD BACKUP(DSCOMMAND)

Many variations of the **RELEASE** command exist. For more information and a comprehensive
list of the commands, see _DFSMShsm Storage Administration Reference,_ SC26-0422 _._

### 11.1.3 Canceling queued DFSMShsm requests

Use DFSMShsm to cancel queued and active requests. A _queued request_ is a request that is
not yet selected for processing by DFSMShsm. After a request is selected for processing, it
becomes an active request. Even though it can be canceled, we do not recommend canceling
it, except to prevent outages or shutting down DFSMShsm.

Several ways exist to cancel DFSMShsm requests that are waiting in the queue. You can
decide the best way to cancel all of the requests that you need to cancel without affecting
other users or requests.

Before you cancel a request, you might want to display the requests that are in the queue,
and then decide which requests to cancel. You can issue the following command to get a list
of waiting requests:

`HSEND QUERY REQUEST`

**Important** : If many requests are waiting in the queue, your window might be flooded with
requests. You can issue the **HSEND QUERY WAIT** command to check how many requests are
in the queue before you display the queue.

Example 11-6 is a sample display from the command.

_Example 11-6  Output from the QUERY REQUEST command_

    ARC0101I QUERY REQUEST COMMAND STARTING ON HOST=2
    ARC0167I MIGRATE MWE FOR DATA SET MHLRES7.ACSTEST.LISTDS FOR USER MHLRES7,
    ARC0167I (CONT.) REQUEST 00000013, WAITING TO BE PROCESSED, 00000 MWE(S) AHEAD
    ARC0167I (CONT.) OF THIS ONE
    ARC0167I MIGRATE MWE FOR DATA SET MHLRES7.BV FOR USER MHLRES7, REQUEST
    ARC0167I (CONT.) 00000014, WAITING TO BE PROCESSED, 00001 MWE(S) AHEAD OF THIS
    ARC0167I (CONT.) ONE
    ARC0167I MIGRATE MWE FOR DATA SET MHLRES7.BV.THURSDAY FOR USER MHLRES7,
    ARC0167I (CONT.) REQUEST 00000015, WAITING TO BE PROCESSED, 00002 MWE(S) AHEAD
    ARC0167I (CONT.) OF THIS ONE
    
DFSMShsm returns three lines of response to every request on the queue. This response
includes the task to be executed, the data set to be processed, the user who issued the
request, the request ID, and how many requests are ahead in the queue.

Now that you have all of the necessary information about the waiting requests, you can define
the best way to cancel the requests without affecting other requests.

One way to cancel requests is by specifying the request ID when you issue the **CANCEL**
command. This method cancels only the exact specified request. The other requests are
unchanged. The following example cancels a specific request:

`HSEND CANCEL REQ(14)`


-----

DFSMShsm returns the following output, as shown in Example 11-7.

_Example 11-7  Output from the CANCEL REQ command_

    ARC1008I MHLRES7.BV MIGRATE REQUEST 00000014 WAS CANCELLED
    
    ARC0931I (H)CANCEL COMMAND COMPLETED, NUMBER OF REQUESTS CANCELLED=1
    
    ARC1007I COMMAND REQUEST 00000031 SENT TO DFSMSHSM

Another way to cancel a request is by specifying the exact data set name for the request:

`HSEND CANCEL DATASETNAME(mhlres7.bv.thursday)`

This **HSEND CANCEL** command produces the output that is shown in Example 11-8.

_Example 11-8  HSEND CANCEL command output_

    ARC1008I MHLRES7.BV.THURSDAY MIGRATE REQUEST 00000015 WAS CANCELLED
    
    ARC0931I (H)CANCEL COMMAND COMPLETED, NUMBER OF REQUESTS CANCELLED=1
    
    ARC1007I COMMAND REQUEST 00000033 SENT TO DFSMSHSM

Another way to cancel requests is by identifying the user ID that issued the request.
Canceling requests by identifying a user ID will cancel all requests from that user, as shown in
the following example:

`HSEND CANCEL USERID(mhlres7)`

The command in Example 11-9 will cancel all requests that are associated with user ID
MHLRES7.

_Example 11-9  HSEND CANCEL output_

    ARC1008I MHLRES7.BVDEL MIGRATE REQUEST 00000016 WAS CANCELLED
    ARC1008I MHLRES7.BVDEX MIGRATE REQUEST 00000017 WAS CANCELLED
    ARC1008I MHLRES7.CMD.CLIST MIGRATE REQUEST 00000018 WAS CANCELLED
    ARC1008I MHLRES7.EXEC.RMM.CLIST MIGRATE REQUEST 00000019 WAS CANCELLED
    ARC1008I MHLRES7.EXEC.RMM.HOLDX.CLIST MIGRATE REQUEST 00000020 WAS CANCELLED
    ARC1008I MHLRES7.HSM.BCDS MIGRATE REQUEST 00000021 WAS CANCELLED
    ARC1008I MHLRES7.HSM.DUMPVOL MIGRATE REQUEST 00000022 WAS CANCELLED
    ARC1008I MHLRES7.HSM.DVOL MIGRATE REQUEST 00000023 WAS CANCELLED
    ARC1008I MHLRES7.HSM.KSDS1 MIGRATE REQUEST 00000024 WAS CANCELLED
    ARC0931I (H)CANCEL COMMAND COMPLETED, NUMBER OF REQUESTS CANCELLED=9
    ARC1007I COMMAND REQUEST 00000034 SENT TO DFSMSHSM

You can also cancel active requests. However, this method is not recommended, unless to
avoid outages and DFSMShsm shutdown. For more information about canceling active tasks,
see Chapter 14, “Problem determination”.

### 11.1.4 Changing DFSMShsm control parameters

Each time that you start DFSMShsm, a subset of parameters is established, by default. You
might want to change the defaults during normal operation, change the times that the
automatic functions are scheduled to run, or increase the number of tasks that relate to a
specific function.


-----

Use the **SETSYS** command to dynamically change the automatic functions without requiring
you to restart DFSMShsm. The **SETSYS** parameters control the main DFSMShsm functions
and schedule. Ensure that only storage administrators can access these commands. For
more information about how to control user access to DFSMShsm resources, see Chapter 5,
“Implementing DFSMShsm security”.

In certain situations, you might need to shrink your space management, or dump window, or
move it to a later time to prevent it from affecting the production environment. In these cases,
you can issue the following command to alter your primary space management window. This
command sets DFSMShsm parameters:

`HSEND SETSYS PRIMARYSPMGMSTART(0400 0500)`

This command sets the primary space management window to start at 04:00 and finish at
05:00.

If all of the processing is not complete by 05:00, DFSMShsm shuts down primary space
management and issues the following message:

`ARC0521I PRIMARY SPACE MANAGEMENT ENDING TIME REACHED`

For more information about how to set up the space management window, see Chapter 9,
“Space management”.

For automatic backup, and automatic dump processing, you must specify one more value,
which is the latest time that DFSMShsm can start the backup or dump process. If you do not
specify this value, it is set to midnight (24:00). The following command sets DFSMShsm
parameters:

`HSEND SETSYS AUTOBACKUPSTART(0600 0700 0800)`

This command sets DFSMShsm to start automatic backup at 06:00. If automatic backup is
not started until 07:00, which is our late start time, DFSMShsm will not perform automatic
backup in this cycle. The value of 08:00 is the deadline for automatic backups, so no new
backup will start after this time. Any backups that are currently running at 08:00 are not
affected.

In addition to changing primary DFSMShsm function windows, you can also define several
configurations to DFSMShsm, including how many instances of a task can run concurrently,
how many CDS backup versions to keep, monitoring DFSMShsm CDSs, monitoring control
tape units, and so on.

During your maintenance window, you might decide that it is necessary to reduce the number
of tape migration tasks so more tape drives will be available for use by other applications.
Instead of canceling or holding your space management task, you can reduce the maximum
number for migration tasks that are running concurrently in your system. Reducing the
maximum does not completely stop your migration and reduces the chance of not completing
the task within the window.

To change the number of migration tasks that are running concurrently to three, you can issue
the following command from TSO or the console:

`HSEND SETSYS MAXMIGRATIONTASKS(3)`

If more than three tasks are running when you issue the command, DFSMShsm first finishes
the current instances, and does not start new instances until it reaches the specified number
of tasks that are running.


-----

You can also control the maximum number of concurrent tasks that are running for the
following functions:

- Aggregate backup and recovery support (ABARS)
- Backup
- Dump
- Fast replication
- Recall
- Recover
- Interval migration
- Secondary space management
- Recycle

Example 11-10 shows examples of how to set the maximum number of instances that are
running for each task.

_Example 11-10  Setting DFSMShsm parameters_

    HSEND SETSYS MAXABARSADDRESSSPACE(3)
    HSEND SETSYS MAXBACKUPTASKS(3)
    HSEND SETSYS MAXDUMPTASKS(3)
    HSEND SETSYS MAXCOPYPOOLTASKS(FRBACKUP(3))
    HSEND SETSYS MAXRECALLTASKS(3)
    HSEND SETSYS MAXDSRECOVERTASKS(3)
    HSEND SETSYS MAXINTERVALTASKS(3)
    HSEND SETSYS MAXSSMTASKS(3)
    HSEND SETSYS MAXRECYCLETASKS(3)

**Important:** The values in Example 11-10 are examples only and must not be considered
as recommended values.

The **SETSYS** command has a wide range of parameters and possible configurations that are
not covered. To learn more about the **SETSYS** control parameter, see _DFSMShsm Storage_
_Administration Reference,_ SC26-0422.

### 11.1.5 Restarting DFSMShsm after an abnormal end

DFSMShsm automatically recovers after most abnormal terminations. However, if
DFSMShsm cannot recover from the abnormal end and the **RESTART** keyword was used in the
PROC statement of the startup procedure, DFSMShsm will restart itself. With the **RESTART**
keyword, you can specify a startup procedure, which can be the same or different from the
original startup procedure, and pass additional parameters to DFSMShsm.

If an abnormal end occurs that interrupts MVS processing and on the condition that the
extended common service area (ECSA) is not destroyed, DFSMShsm can continue to
process waiting requests. Additionally, if DFSMShsm is restarted, it will process any requests
that are waiting in the ECSA.

If a task that was processing within the DFSMShsm address or storage space ended
abnormally, the following message is issued:

ARC0003I taskname TASK ABENDED, CODE abendcode IN MODULE modname AT OFFSET offset,
STORAGE LOCATION location


-----

Try to understand the area of processing that was affected. DFSMShsm attempts recovery in
most cases, but the ultimate responsibility lies with the user to determine whether any further
recovery actions are required.

You need these valuable sources of information to analyze any problems:

- DFSMShsm activity logs
- DFSMShsm Problem Determination Aid (PDA) trace
- DFSMShsm-generated dumps
- System log

If a process, such as primary space management, secondary space management, automatic
backup, or automatic dump processing, ends abnormally, you might need to extend the
processing window to allow all of the work to complete. You can issue the commands that are
shown in Example 11-11 from TSO or the console to change the end time of your tasks, and
restart it.

_Example 11-11  Setting DFSMShsm parameters_

SETSYS PRIMARYSPMGMTSTART ( _starttime_ , _endingtime_ )
SETSYS SECONDARYSPMGMTSTART ( _starttime_ , _endingtime_ )
SETSYS AUTOBACKUPSTART ( _starttime_ , _latestarttime_ , _quiescetime_ )
SETSYS AUTODUMPSTART ( _starttime_ , _latestarttime_ , _quiescetime_ )

**Note:** Use the same start time that is used in your PROCLIB. Update _endingtime_ ,
_latestarttime_ , and _quiescetime_ to values that will accommodate your workload.

After the processing finishes, you can issue the **SETSYS** commands again by using the same
parameters that are used in ARCCMDxx to ensure that DFSMShsm will use the standard
values on the next cycle.


### 11.1.6 Expiring backup versions

Availability management of DFSMShsm does not automatically delete backup versions when
the related user data set is deleted. Over time, a number of unneeded backup versions can
accumulate on backup volumes. You can use the **EXPIREBV** command to identify and delete
these unnecessary backup versions based on the most recent status of the data set.

The **EXPIREBV** command causes DFSMShsm to search the backup control data set (BCDS)
for old, unwanted backup versions and to delete them (or display information about them)
based on the attributes in the management class for each data set.

You can use the **EXPIREBV** command with the **DISPLAY** parameter to see which backup copies
are eligible for deletion. The command in Example 11-12 places a list of the eligible backup
versions into the specified output data set.

_Example 11-12  Displaying old backup versions that are available for deletion_

`HSEND EXPIREBV DISPLAY ODS('output')`

If you browse the data set pointed to by the **OUTDATASET** parameter, you see the output from
this command, as shown in Example 11-13.


-----

_Example 11-13  HSEND EXPIREBV output_

    DISPLAY OF BACKUP VERSIONS ELIGIBLE FOR EXPIRATION AT 23:46:06 ON 2012/08/21 FOR
    
    COMMAND INPUT: (DEFAULTS)
    
    DSNAME = MHLRES4.SC64.SPFLOG5.LIST          DELETED*,     WAS SMS
    (* DETERMINED ON 2012/08/21)
    MANAGEMENT CLASS USED = MCDB22
    
    BACKUP VERSION DSNAME            SYS GEN    RET BACK
    CAT NMBR AGE VERS PROF
    HSM.BACK.T031418.MHLRES4.SC64.A0313     YES 000 651 NO  NO
    
    DSNAME = MHLRES4.SC70.SPFLOG1.LIST          DELETED*,     WAS SMS
    (* DETERMINED ON 2012/08/21)
    MANAGEMENT CLASS USED = MCDB22
    
    BACKUP VERSION DSNAME            SYS GEN    RET BACK
    CAT NMBR AGE VERS PROF
    HSM.BACK.T021418.MHLRES4.SC70.A0313     YES 000 651 NO  NO

To expire the backup versions, issue the following command to delete the expired backup
versions:

    HSENDCMD EXPIREBV EXECUTE

The output from the **EXPIREBV EXECUTE** command is in the backup activity log.

Use **EXPIREBV** to define how long you want to keep backups of non-SMS-managed data sets
after they are deleted. You can use the **NONSMSVERSIONS(CATALOGEDDATA(** **_xx_** **))** parameter to
specify that the backups will be kept for _xx_ days after the non-SMS-managed cataloged data
set is deleted. You can also control backup versions for uncataloged non-SMS data sets. In
this case, you use the **UNCATALOGEDDATA(** **_xx_** **)** parameter.

The following command shows you how to expire any backup version non-SMS-managed
data sets. This example will expire all backups for cataloged data sets 30 days after its
deletion, and 20 days after uncataloged data sets are deleted. Use this command to delete
expired backup versions:

`HSEND EXPIREBV NONSMSVERSIONS(CATALOGEDDATA(30) UNCATALOGEDDATA(20)) EXECUTE`

If you do not specify the retention periods for cataloged non-SMS-managed data sets,
DFSMShsm will use 60 days as the default value. If you do not specify **CATALOGEDDATA** ,
DFSMShsm will not process cataloged non-SMS expiration. The same rule applies to
**UNCATALOGEDDATA** .

When you issue the **EXPIREBV** command, DFSMShsm determines that the user data set was
scratched and stores the date of the EXPIREBV execution as the scratch date in the
DFSMShsm BCDS record (MCB). You can look at the date in the DFSMShsm BCDS record
by using the following command:

`HSEND FIXCDS B ' _dsname_ ' OUTDATASET(' _output_ ')`


-----

In the data set pointed to by the **OUTDATASET** parameter, you will see the MCB record for the
data set (see Example 11-14). The date EXPIREBV determined as the date on which the
user data set was deleted is at offset X'48'. A subsequent **EXPIREBV EXECUTE** command
deletes the unwanted backup versions when the requested time passes. The requested time
is specified in the management class for storage management subsystem (SMS) data sets, or
in the **EXPIREBV** command for non-SMS-managed data sets.

_Example 11-14  HSEND FIXCDS output_

    FIXCDS B 'MHLRES4.SC70.SPFLOG1.LIST' ODS('MHLRES7.OUTPUT.FIXCDS')
    MCH=  00D02000 CA0EE734 72CDA86B CA0EE65C CE2883EB
    +0000 E3C8E2F0 F0F0FFFF 00000000 00000000 13312752 0112015F 40006D5E 00500000
    +0020 00000001 00005E03 0000000C 0000A000 00000001 0001FFFF 0112235C 00000000
    +0040 00000000 00000000 **0112235F** 00000000 C8E2D44B C2C1C3D2 4BE3F2F7 F3F1F1F3
    +0060 4BD4C8D3 D9C5E2F4 4BE2C3F7 F04BC1F2 F0F1F540 40404040 40404040 D4D3C4F1
    +0080 F0C35200 0112015F 00000000 00000000
    ARC0197I TYPE B, KEY MHLRES4.SC70.SPFLOG1.LIST, FIXCDS DISPLAY SUCCESSFUL

We recommend that you run the **EXPIREBV EXECUTE** command regularly.

If you use ABARS in your environment, you also need to issue a specific **EXPIREBV** command
for ABARS versions to ensure that old ABARS versions are deleted following “retention only
version” and “retention extra version” parameters in the ABARS management class.

The following command deletes any ABARS data sets that are older than the retention that is
specified in the management class. Use the following command to delete expired ABARS
backup versions:

`HSEND EXPIREBV ABARSVERSIONS EXECUTE`

**Note:** You must run a specific EXPIREBV for ABARS. It is mutually exclusive with SMS,
and NONSMSVERSIONS.

In certain situations, you need to delete a valid backup version, such as a missing or broken
tape. You can delete valid backup versions of specific data sets by issuing the **BDELETE** or
**HBDELETE** command.

The following command can delete the specified backup version 001 of _dsname_ .

`HSEND BDELETE ' _dsname_ ' VERSION(001)`

If you intend to delete all backup versions of a specific data set, you can also issue the
following command:

HSEND BDELETE ' _dsname_ ' ALL

### 11.1.7 Recycling tape volumes

Over time, migrated data sets expire, are recalled, or are marked for deletion by **DELETE** and
**HDELETE** commands. Similarly, tape backup data sets are rolled off by automatic backup, or
they are marked for deletion by **BDELETE** , **HBDELETE** , and **EXPIREBV** commands. Those data sets
are still physically occupying space on the tape volumes. The percent data valid in a
DFSMShsm tape tends to decrease as backups and data sets are deleted and the data in the
tape is invalidated, resulting in inefficiently used tape media.


-----

Also, migration tapes can be requested for a recall during the migration process. In this case,
DFSMShsm allocates another tape for migration, and releases the current tape to proceed
with recall. Because the first tape was not fully filled, it is called a _partial tape._

Partial tapes can be selected by DFSMShsm to continue the migration process, but many
partial tapes mean much unused space. To reduce the number of partial tapes, remove
invalid data sets, and consolidate the valid data on fewer tape volumes. You can use the
**RECYCLE** command.

We describe the most common commands that you can use to consolidate the
DFSMShsm-owned tape data. The **RECYCLE** command can be issued from an operator
console or by using the **HSEND** command. With the **HSEND** command, you can issue the **RECYCLE**
command from any TSO user ID that was authorized through RACF, or the **AUTH** command.

Before you submit the **RECYCLE** command for processing, we recommend that you display a
list of tapes that match the requirements to be recycled. The following command displays
DFSMShsm tapes that are available for recycle:

HSEND RECYCLE DISPLAY ALL ODS(' _dsname_ ')

The display parameter causes DFSMShsm to list the tapes that are selected for recycle, but
not to execute it. Use **ODS** to direct the output of the command to a data set that you can
access later.

You can also select the tapes that will be recycled, including the following types:

- ALL
- BACKUP
- DAILY
- SPILL
- Migration-level 2 (ML2)

Each option gives you the ability to select only a specific group of tapes to recycle. This
approach can be used if you have a large tape environment, and recycling all eligible tapes at
one time might cause the command to run for an extended period, or affect the number of
tape drives that are available to other applications.

You can also limit the number of concurrent RECYCLE tasks that are running with the **SETSYS**
**MAXRECYCLETASKS** parameter. With the **SETSYS MAXRECYCLETASKS** parameter, you can
dynamically change the number of recycle tape processing tasks, even during recycle
processing, to any number 1 -  15 . The default for recycle-processing tasks is 2 . This
parameter can be added to the ARCCMDxx member and issued from TSO or the console for
special processing requirements. The following command sets the maximum number of
recycle tasks to 3 :

`HSEND SETSYS MAXRECYCLETASKS(3)`

**Note:** The previous command limits the maximum number of recycle tasks to 3 .
Remember that each instance of recycle uses two tape drives: one tape drive for input and
other tape drive for output.

You can specify to DFSMShsm when a tape becomes eligible for recycle processing by
setting a threshold value on the **SETSYS RECYCLEPERCENT** and **ML2RECYCLEPERCENT** parameters
in the ARCCMDxx member. You can also specify a value for the **PERCENTVALID** subparameter
of the **RECYCLE** command if you need to override the **SETSYS** values. If you do not set any of
these parameters, the default value of 20 % is assumed.


-----

The following command recycles DFSMShsm ML2 tapes with 10% or less valid data:

`HSEND RECYCLE ML2 PERCENTVALID(10) EXECUTE`

We recommend that you recycle your tapes regularly to avoid running out of scratch or
DFSMShsm-owned tapes. You can choose a window of less tape activity at your installation
to run the **RECYCLE** command.

You can also limit the number of tapes that are recycled by a RECYCLE process. With the
**LIMIT** parameter, you can specify RECYCLE to quiesce after the net specified number of
tapes are returned to scratch status. The following command causes RECYCLE processing
to quiesce after a net of 40 tapes is returned to scratch status. The net is determined by tapes
that are read in for input minus tapes that are written to for output.

This example releases up to 40 DFSMShsm ML2 tapes with less than 25% usage:

`HSEND RECYCLE ML2 EXECUTE PERCENTVALID(25) LIMIT(40)`

**Note:** By using our example, during the RECYCLE process, you can empty 44 ML2 tapes,
but mount four scratch tapes for output, which results in the net 40 tapes. So, RECYCLE
processing will quiesce after it processes a total of 44 tapes. The **LIMIT** parameter is
subject to the number of tapes that are allowed by the **PERCENTVALID** parameter.

In certain cases, a single tape does not have enough space to accommodate all data sets
that are being migrated, or backed up. In these cases, DFSMShsm selects a second tape to
use for migration or backup, and connects it to the first tape, creating a connected set.

Although you can create connected sets as large as 255 tapes, you can recycle only
connected sets up to 40 tapes long. In these cases, you need to break the connected set into
pieces that are smaller than 40 tapes, and then proceed with RECYCLE. 

Also, when you recycle connected sets, DFSMShsm checks the first tape in the connected
set, and compares it with the percent value that is specified either in the **RECYCLE** command,
or in the ARCCMDxx member. If the first tape does not match the requirements, the
connected set is not processed, regardless of the overall percent valid of the connected set.

To bypass this control, you can use the **CHECKFIRST(N)** parameter to make DFSMShsm check
all tapes in the connected set and calculate the overall percent valid value before it compares
it with the recycle requirements. This approach causes more tapes to be recycled because
DFSMShsm checks all connected sets to check their eligibility for recycle, but it also causes
recycle to run longer for the same reason. The following example recycles DFSMShsm tapes
with 10% or less valid data:

`HSEND RECYCLE ALL PERCENTVALID(10) CHECKFIRST(N) EXECUTE`

This command causes DFSMShsm to recycle all tapes with 10% or less valid data, and to
check all connected sets to calculate whether they are eligible for recycle.

If you use manual tape drives for the DFSMShsm workload, we recommend that you create a
list of tapes that are being recycled. Send it to the operators before you start RECYCLE
processing so that they can coordinate that the tapes are mounted promptly.


-----

With the **VERIFY** parameter of the **RECYCLE** command, you get two lists of the eligible volumes:

- A _pull list_ that operators can use to manually pull the required tapes for the RECYCLE
processing. The pull list is made up of small groups of alphabetized volume serial number
(volser) lists. The pull groups are listed in the sequence in which recycle processing will
most likely request them.

- A _mount list_ that aids operators with the mount order of tapes that are being recycled. The
mount list is in the anticipated order of the tape mounts that RECYCLE processing will
request. The list of tapes is grouped according to the requested category (for example,
ML2 and SPILL).

These lists can be helpful in an environment with many tape cartridges and without
automated tape libraries.

Both lists are dynamically allocated to either the data set with the prefix that you specified with
the **TAPELIST PREFIX(** **_tapelist_prefix_** **)** parameter or to a fully qualified data set name with
the **RECYCLE** command **TAPELIST FULLDSNAME(** **_tapelist_dsn_** **)** parameter. If none of the
previous parameters are specified in the **RECYCLE** command, DFSMShsm writes the output to
the SYSOUT class that is specified by the **SETSYS** parameter. The data set is deallocated
when the command completes.

The following command causes DFSMShsm to look for all eligible ML2 tapes and create the
pull and mount lists in a data set with prefix as its high-level qualifier (HLQ):

`HSEND RECYCLE ML2 VERIFY TAPELIST(PREFIX(prefix))`

If all of your tape volumes are in automated tape libraries, the output from the **TAPELIST** or
**VERIFY** parameters are beneficial to you as a reporting tool.

In addition to the options that are presented so far, you can also define a specific tape
volume, or tape range, for recycle. This capability gives you an extended granularity to decide
what tapes and how many tapes will be recycled when you are migrating to new tape media
or technology. You can include as many tape ranges as you need in the **RECYCLE** command.

To recycle all tapes from TP0000 to TP0099 (a specific range), you can use the following
command:

`HSEND RECYCLE ALL EXECUTE SELECT(INCLUDE(RANGE(TA0000:TA0099))) PERCENTVALID(100)`

By selecting PERCENTVALID(100), you ensure that all tapes in the specified range will be
selected for recycle.

You can also exclude ranges from processing, which makes it easier to migrate only a subset
of tapes in a determined range, for example:

    HSENDCMD RECYCLE ALL EXECUTE SELECT(INCLUDE(RANGE(TAPE00:TAPE99)) -
    EXCLUDE(RANGE(TAPE08:TAPE09))) PERCENTVALID(100)

To select multiple ranges to run a recycle, place a space between the ranges that are
selected, as shown:

    HSEND RECYCLE ALL EXECUTE SELECT(INCLUDE(RANGE(TA0000:TA0099 TB0000:TB0099))) PERCENTVALID(100)

Finally, consider the tape copies of recycled volumes. After a tape is recycled, the tape copy
is no longer useful. When you process the **RECYCLE** command, DFSMShsm invalidates
alternate or duplex tapes at the same time that it invalidates the tapes from which they were
copied.


-----

### 11.1.8 Moving data sets to new DASD volumes

As your system grows, you face situations where your current available DASD space is
insufficient to handle the amount of stored data, and you are required to move the data from
the existing DASD technology to larger DASD volumes, or even a new DASD controller.

DFSMShsm can help you with the task of migrating data over to other volumes, and replacing
existing migration-level 1 (ML1) volumes with new volumes.

**Converting level 0 DASD volumes**
DFSMSdss can be used to perform a device conversion of level 0 DASD volumes. Migration
tools use mirroring to convert the old DASD volumes to new DASD volumes. You can also use
DFSMShsm to move a single data set or an entire level 0 volume to other level 0 volumes.

When you process SMS-managed data sets, you cannot direct the movement to a specific
volume. Although you can specify a volume, SMS processing determines the actual volume to
which the data set is moved. The process for moving data sets first migrates the data sets to
ML1 volumes and then recalls them to level 0 volumes. At the beginning of processing for
**MIGRATE VOLUME** commands, DFSMShsm obtains a list of the active management classes in
the system. As DFSMShsm processes each data set, it checks the value that is specified for
the data set for the COMMAND OR AUTO MIGRATE attribute in the management class. If the
value of the attribute is BOTH, DFSMShsm processes the data set.

In recalling each data set, DFSMShsm invokes the automatic class selection (ACS) services
of DFSMSdfp. If SMS is active, the ACS routine might return a storage class to DFSMShsm
and, optionally, a management class. If the ACS routine returns a storage class, DFSMShsm
passes the data set name, with its storage class name and management class name (if
present), to DFSMSdss, which interfaces with DFSMSdfp to select a volume to receive the
data set. The following command moves all DASD data sets that are on the level 0 volume to
other level 0 volumes:

HSENDCMD MIGRATE VOLUME( _volser_ ) DAYS(0) CONVERT

**Important:** Your SMS polices for the data sets that are being converted can be altered
during the migrate and recall process, depending on how you control your ACS routines.

To prevent SMS from selecting the source volume as the target volume for the recall, change
the status attribute for the volume in the storage group. The suitable status attributes are
DISABLENEW and QUIESCENEW.

Any data set that needs to be expired is expired without being migrated, during the conversion
process.

**Converting level 1 DASD volumes**
With the **FREEVOL** command, you can empty an ML1 volume in preparation for new equipment
or to replace old or partially damaged DASD volumes. The **FREEVOL** command moves
migration copies of SMS-managed data sets from an ML1 volume based on each data set’s
management class attribute value.

You can use the following command to empty a specific volume from your ML1 pool:

`HSEND FREEVOL MIGRATIONVOLUME( _volser_ ) AGE(0)`


-----

The **AGE(0)** parameter, when it is entered for an ML1 volume, causes DFSMShsm to move
the migration copies of all SMS-managed data sets, except those copies that need backup,
from the volume and place them on other ML1 or ML2 volumes, depending on the data set’s
management class. If the management class value is specified for the
LEVEL-1-DAYS-NONUSAGE and the age is met, the data set migrates to an ML2 volume. If
the management class value is specified for LEVEL-1-DAYS-NONUSAGE and the data set
age does not meet the criterion or if the data set has an attribute of NOLIMIT, the data set
migrates to another ML1 volume. You can restrict which ML1 DASD volumes receive data
sets by using the **DRAIN** parameter of the **ADDVOL** command. **DRAIN** removes a volume from
selection candidacy.

The data sets, which need backup (that were migrated but are waiting for automatic backup),
and the backup versions, which were created by the **BACKDS** command, are not moved off the
volume. When you plan to remove an ML1 volume, you must first run AUTOBACKUP on the
primary host, and then execute the **FREEVOL AGE(0)** command.

After you empty the ML1 volume, delete the volume by using the following command:

HSEND DELVOL _volser_ MIGRATION

### 11.1.9 Auditing DFSMShsm

The **AUDIT** command detects, reports, diagnoses, and often provides repairs for discrepancies
between CDSs, catalogs, and DFSMShsm-owned volumes.

To ensure data integrity, DFSMShsm uses numerous data set records to track individual data
sets. These records are contained in the following places:

• Master catalog, which is a list of data sets for the entire system

• User catalog, which is a list of data sets that are accessible from that catalog

• Journal, which keeps a running record of backup and migration transactions

• Small-data-set packing (SDSP) data sets on migration volumes

• Migration control data set (MCDS), which is an inventory of migrated data sets and
migration volumes

• Backup control data set (BCDS), which is an inventory of backed-up data sets and
volumes, dumped volumes, and backed-up aggregates

• Offline control data set (OCDS), which contains a tape table of contents (TTOC) inventory
of migration and backup tape volumes

In normal operation, these records stay synchronized. However, because of data errors,
hardware failures, or human errors, these records can become unsynchronized. The **AUDIT**
command allows the system to cross-check the various records about data sets and
DFSMShsm resources. AUDIT can list errors and propose diagnostic actions or, at your
option, complete most repairs itself.

Consider the use of the **AUDIT** command for the following reasons:

- After any CDS restore (highly recommended)
- After an ARC184I message (error when reading or writing DFSMShsm CDS records)
- Errors on the RECALL or DELETE of migrated data sets
- Errors on BDELETE or RECOVER of backup data sets
- DFSMShsm tape selection problems
- RACF authorization failures
-  Power or hardware failure
- Periodic checks


-----

You can use the **AUDIT** command to cross-check the following sources of control information:

- MCDS or individual migration data set records
- BCDS or individual backup data set records or ABARS records
- OCDS or individual DFSMShsm-owned tapes
- DFSMShsm-owned DASD volumes
- Migration-volume records
- Backup-volume records
- Recoverable-volume records (from dump or incremental backup)
- Contents of SDSP data sets

Use the **AUDIT** command at times of low system activity because certain audit processes can
run for a long time. However, the **AUDIT** process can be used almost any time.

The following command shows how to instruct DFSMShsm to audit the BCDS and fix any
errors:

`HSEND AUDIT BCDS FIX`

See _z/OS DFSMS Storage Administration Reference (for DFSMShsm, DFSMSdss,_
_DFSMSdfp)_ , SC26-7402, which provides examples of coding the command and guidance
about interpreting its output.

## 11.2 Journal as a large format data set

The journal data set provides DFSMShsm with a record of each critical change to a CDS from
any host since the last time that CDS was successfully backed up. DFSMShsm recovers
control data sets (CDSs) by merging the journal with a backed-up version of the CDS. CDSs
cannot be recovered to the point of failure without the journal. Use of the journal is highly
recommended.

As your system and DFSMShsm-managed data grows, the number of backups, dumps,
ABARS, migrations, recalls, and recovers are expected to increase proportionally. This
increase can affect your DFSMShsm processing if your journal data set is defined with a
small allocation and becomes full before CDS backups take place.

The journal is a sequential data set that can reside only in a single volume, and it must not
have secondary allocation. In old z/OS releases, the journal data set was limited to 65,535
tracks allocation, which was the maximum number of tracks that are allocated by a sequential
data set in a single volume. Starting with V1R7, you can increase this limit by setting the
journal as a large format data set.

The large format sequential data set allows the user to set up a journal with more than 65,535
tracks, relieving the user from backing up the journal and CDSs many times a day in systems
with high DFSMShsm processing.

Before you migrate a journal to large format data set, consider the following information:

- In a multiple DFSMShsm host environment, if hosts share a single set of CDSs, they must
also share a single journal. All DFSMShsm recovery procedures are based on a single
journal to merge with a backed-up version of a CDS.

- Before you migrate the journal to large format, all DFSMShsm hosts that share the journal
must be at z/OS V1R7 or later.

- You must allocate the journal as a single volume, single extent data set that is contiguous
and non-striped.


-----

**Migrating the journal to a large format data set**
Using a larger journal data set can allow more DFSMShsm activity to take place between
journal backups and helps to avoid journal full conditions. For more information about large
format data sets, see _DFSMS Using Data Sets,_ SC26-7410.

If you decide to migrate your current journal to a large format data set, follow these steps to
avoid corrupting your CDSs during the conversion process.

As a first step, you can run an IEFBR14 program to preallocate the new journal data set, as
shown in Example 11-15. The target volume must have the contiguous space that you
specified and no secondary allocation is allowed.

_Example 11-15  Define the journal data set_

    //MHLRES71 JOB (999,POK),'HSM JOURNAL',MSGCLASS=Y,REGION=0M,TIME=10,
    // NOTIFY=&SYSUID,CLASS=A
    //************************************************************//
    //************************************************************//
    //** BEFORE SUBMITTING THE JOB, PLEASE UPDATE THE FOLLOWING **//
    //**          PARAMETERS             **//
    //**                            **//
    //** yourVOLS - The VOLSER you are allocation new journal  **//
    //** yourJRNL - The new journal name            **//
    //**                            **//
    //** You may also want to change space values to fit your  **//
    //**          environment needs          **//
    //**                            **//
    //************************************************************//
    //************************************************************//
    //ALLOCATE EXEC PGM=IEFBR14
    //JOURNAL DD DSN=yourJRNL,DISP=(,CATLG),UNIT=3390,
    //  VOL=SER=yourVOLS,SPACE=(CYL,(400),,CONTIG),DSNTYPE=LARGE
    //SYSTSIN  DD DUMMY
    //SYSTSPRT DD SYSOUT=*
    
After you preallocate the new journal, bring down all but one DFSMShsm task. Ensure that
you give DFSMShsm enough time to finish any currently running process. It might take a few
minutes to bring down DFSMShsm, depending on the process that is running.

Next, hold all DFSMShsm tasks to ensure that no new tasks come in during CDS backups.
After you put all functions on hold, and no DFSMShsm functions are running, you can issue
the command that is shown in Example 11-16 to back up CDSs and clean up the journal. Do
not forget to also hold the common queue if it is implemented in your environment.

_Example 11-16  Backing up CDSs_

    HSEND HOLD ALL
    HSEND HOLD COMMONQUEUE
    HSEND BACKVOL CDS

**Note:** It might take time for all current tasks to finish after you issue the **HOLD** command. We
recommend that you issue an **HSEND Q ACT** command to ensure that no tasks are running
before you issue the **HSEND BACKVOL CDS** command.

Example 11-17 shows the message that you receive when your **BACKVOL** **CDS**
command is complete.


-----

_Example 11-17  Output from Example 11-16_

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

After the backups finish, you can shut down the last DFSMShsm host.

Rename your current journal data set to something else, and rename the new journal you
created to the DFSMShsm journal name. Then, restart all of the DFSMShsm hosts.

You can delete your old journal data set after you checked that all DFSMShsm hosts are
working as expected.

## 11.3 Maintaining SDSP data sets: Reorganization

In most environments, a major part of the data sets that are created on DASD every day are
small data sets, occupying a few tracks or even one track in a level 0 volume. Many of these
data sets store only a few records and the rest of the tracks that are allocated are free and
wasted space.

DFSMShsm helps you to reduce the wasted space with these small data sets by giving you a
simple way to migrate them to ML1 volumes so that they allocate only the required space, and
also take advantage of data compression that is performed during this migration.

Small user data sets reside on separate tracks of level-0 user volumes. DFSMShsm can
migrate the small user data sets (as records) and store them as records in a VSAM
key-sequenced SDSP data set on a level-1 migration volume. DASD space is reduced
because multiple data sets then share the same tracks on level-1 migration volumes.

It is important to plan the number of SDSP data sets in relation to the number of concurrent
migration tasks and the amount of processing by functions with a higher usage priority for the
SDSP data sets, such as recalls or ABARS backups.

The SDSP data set is a VSAM key-sequenced data set and as the data sets are migrated to,
and recalled from SDSP , VSAM will need to be reorganized to erase old, and unnecessary
records, and to reorganize its index and data components to improve performance.


-----

You can significantly reduce the need to reorganize SDSP data sets by enabling the control
area (CA) reclaim function for them. For more information, see the topic about reclaiming CA
space in _DFSMS Using Data Sets,_ SC26-7410.

We recommend that you plan your SDSP reorganization in a window apart from the space
management window, so no migration tasks will request access to the SDSP data set.

Before you start your reorganization process, verify that no recalls are actively accessing your
SDSP . Next, hold recalls from all DFSMShsm products with access to the SDSP . Also, hold
the common request queue (CRQ) if it is implemented in your environment. Example 11-18
shows you both of the commands to hold recall and the common queue.

_Example 11-18  Holding recall and the common queue_

    HSEND HOLD RECALL
    HSEND HOLD COMMONQUEUE

You can also use the **DRAIN** parameter to drain the volume where the SDSP data set is
allocated, so no migration tasks will select this volume for migration, preserving the available
space on the volume. To drain the volume, you can issue the following command. This
command adds the migration volume to DFSMShsm:

`HSEND ADDVOL _volser_ MIGRATION(MIGRATIONLEVEL1 SDSP DRAIN) unit(3390)`

In sequence, we will export (EXPORT) the current SDSP as part of our reorganization
process. Example 11-19 shows you a sample JCL to export the SDSP data set.

_Example 11-19  Exporting SDSP data set_

    //MHLRES71 JOB (999,POK),MSGLEVEL=1,NOTIFY=&SYSUID
    //*******************************************************************//
    //*******************************************************************//
    //**  BEFORE SUBMITTING THE JOB, PLEASE CHANGE THE FOLLOWING   **//
    //**             PARAMETERS              **//
    //**                                **//
    //** yourSDSP - Your SDSP data set name              **//
    //** yourEXPT - Your temporary EXPORT copy            **//
    //** yourUNIT - Your Unit name                  **//
    //**                                **//
    //**  You may also need to change space information on ALLOCATE  **//
    //**     STEP to accommodate your SDSP entries        **//
    //**                                **//
    //*******************************************************************//
    //*******************************************************************//
    //ALLOCATE EXEC PGM=IEFBR14
    //EXPORTS DD DSN=yourEXPT,DISP=(,CATLG),
    // UNIT=yourUNIT,SPACE=(CYL,(20,2))
    //IDCAMS EXEC PGM=IDCAMS,REGION=4M
    //SYSPRINT DD SYSOUT=*
    //SYSIN DD *
    EXAMINE NAME(yourSDSP) INDEXTEST
    IF LASTCC = 0 THEN -
    EXPORT yourSDSP ODS(yourEXPT) TEMPORARY
    /*

**Note:** Your space requirements might vary depending on the size of your SDSP and the
amount of valid data.


-----

Next, we run another job to reallocate the SDSP data set on the ML1 volume, and then import
the data from the export copy. See Example 11-20.

_Example 11-20  Defining the SDSP VSAM data set and import_

    //MHLRES71 JOB (999,POK),MSGLEVEL=1,NOTIFY=&SYSUID
    //*******************************************************************//
    //*******************************************************************//
    //**  BEFORE SUBMITTING THE JOB, PLEASE CHANGE THE FOLLOWING   **//
    //**             PARAMETERS              **//
    //**                                **//
    //** yourVOLS - The ML1 volser the SDSP will be allocated     **//
    //** yourSDSP - Your SDSP data set name              **//
    //** yourEXPT - Your EXPORT data set name             **//
    //**                                **//
    //*******************************************************************//
    //*******************************************************************//
    //STEP1 EXEC PGM=IDCAMS,REGION=512K
    //SYSPRINT DD SYSOUT=*
    //SDSP1 DD UNIT=SYSDA,VOL=SER=yourVOLS,DISP=SHR
    //SYSIN DD *
    DEFINE CLUSTER (NAME(yourSDSP) VOLUMES(yourVOLS) -
    CYLINDERS(5 0) FILE(SDSP1) -
    RECORDSIZE(2093 2093) FREESPACE(0 0) -
    INDEXED KEYS(45 0) -
    SPEED BUFFERSPACE(530432) -
    UNIQUE NOWRITECHECK) -
    DATA -
    (CONTROLINTERVALSIZE(26624)) -
    INDEX -
    (CONTROLINTERVALSIZE(1024))
    IF MAXCC=0 THEN -
    IMPORT IDS(yourEXPT) ODS(yourSDSP) IEMPTY

**Note:** Your space requirements might vary depending on the size of your SDSP and the
amount of valid data.

The SDSP reorganization is complete. Now, we need to release all of the functions that we
held at the beginning of the reorganization, and issue the **ADDVOL** command with **NODRAIN** , so
DFSMShsm will set this volume as available for migration. See Example 11-21.

_Example 11-21  Releasing recall and adding ML1 volume to DFSMShsm_

    HSEND RELEASE RECALL
    HSEND RELEASE COMMON QUEUE
    HSEND ADDVOL _volser_ MIGRATION(MIGRATIONLEVEL1 SDSP NODRAIN) UNIT(3390)

For more information about the parameters to use in **DEFINE** **CLUSTER** , see the detailed
information about VSAM data sets in _VSAM Demystified_ , SG24-6105.


-----

