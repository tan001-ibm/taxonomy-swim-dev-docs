# 1. DFSMShsm overview

## 1.1  Introduction
DFSMShsm is a functional component of the Data Facility Storage Management Subsystem (DFSMS) family, which provides facilities for managing your storage devices. DFSMShsm ensures that space is available on your direct access storage device (DASD) volumes, so you can extend existing data sets and allocate new ones. DFSMShsm ensures that backup copies of your data sets are always available in case your working copies are lost or corrupted. It relieves you from manual storage management tasks and improves DASD use by automatically managing both space and data availability in a storage hierarchy. 

DFSMShsm is one of five components that, when combined, create a single, integrated software package for all of your installation’s storage management needs.

DFSMS consists of the following five components. Their functions are described:
•DFSMSdfp: Provides storage, data, program, and device management functions through the storage management subsystem (SMS).
•DFSMSrmm: Provides tape management functions for removable media, such as virtual, and physical tape cartridges.
•DFSMSdss: Provides data movement, copy, backup, and space management functions in a batch environment.
•DFSMSoam: Provides tape hardware management, such as cartridge entry, eject, and tape configuration database (TCDB) management.
•DFSMShsm: Provides backup, recovery, migration, and space management functions with optimum automation capabilities.

DFSMShsm is a policy-driven solution to storage management, removing the requirement for batch jobs to perform backup, migration, or space retrieval functions. DFSMShsm works by rules that you can apply to manage your storage. These rules are also dynamically adjustable to allow the flexibility that is required in today’s constantly changing environments. With its flexibility, you can manage your storage at the data set level, the device level, or even the device pool level. DFSMShsm provides the means to manage every data set from the time of its creation until the time that its last backup is no longer required.

In this book, we describe the major functions of DFSMShsm and how they interact with the DFSMS components, including the storage management subsystem (SMS). We also provide details, different techniques, and examples to guide you to a successful DFSMShsm implementation.

## 1.2  Storage device hierarchy
DFSMShsm uses a hierarchy of storage devices. The hierarchy is used in its automatic management of data, relieving users from manual storage management tasks. It categorizes storage devices into three levels:
•Level 0 volumes
DFSMShsm managed volumes are those level 0 volumes that are managed by the DFSMShsm automatic functions. These volumes contain data sets that are directly accessible to you and the jobs that you run. Level 0 volumes and DFSMShsm managed volumes can be any DASD that is supported by DFSMShsm. The next two types of storage are described in detail in 1.5, “Data set lifecycle with DFSMShsm” on page 5. These volume types are managed by DFSMShsm only.

•Level 1 volumes
When a data set is not referenced after a certain amount of time as defined by the installation, DFSMShsm moves the data set from the user data volume to a volume that is owned by DFSMShsm. These volumes are known as migration level 1 (ML1) volumes. These volumes are non-SMS volumes that are available for use only by DFSMShsm.

•Level 2 volumes 
When a data set resided on an ML1 volume after a determined amount of time without being referenced, DFSMShsm moves this data set to a lower-cost media that is usually high-capacity tape. These volumes are known as migration level 2 (ML2) volumes.

## 1.3  SMS and DFSMShsm relationship
With SMS, you can centrally control and direct data set allocation within your system. With SMS, you can define performance goals and data availability requirements, create model data definitions for data sets, and automate data backup, by using classes and groups. The SMS classes and groups (SMS attributes) are customized by the storage administrator based on the installation environment and storage policies. The SMS classes and groups are listed:

• Data class: A list of allocation attributes for data sets (for example, logical record length and record format). The data class simplifies and standardizes data set creation.
• Storage class: A list of storage performance and availability required by applications. The storage class contains availability and performance attributes, such as response time and cache requirements, for data sets.
• Management class: A list of backup, retention, and migration attributes for data sets.
• Storage group: A group of one or more DASD volumes that SMS uses for data set allocation.

SMS classes and storage groups can be assigned for data sets by automatic class selection (ACS). ACS routines are written by the storage administrator. They automatically assign SMS classes and groups to data sets, database data, and objects. Data allocations are processed through ACS routines. With ACS routines, you can enforce installation standards for data allocation and override user specifications for data, storage, and management classes and requests for specific DASD volumes. You can create up to four ACS routines in an SMS configuration, one for each type of SMS class and one for storage groups.

SMS and DFSMShsm work together. Using the SMS attributes specified in the management class and storage group provides space and availability management at the data set and volume level.

SMS attributes for a user or group can be defined by IBM RACF® DFP  segments (by using the ALTUSER  or ALTGROUP  command), simplifying ACS routines. For more information about these RACF commands, see z/OS Security Server RACF Command Language Reference, SA22-7687.

## 1.4  DFSMShsm control data sets 
The DFSMShsm program requires the following data sets to support full-function processing:
•Control data sets (CDSs): Migration control data set (MCDS), backup control data set (BCDS), offline control data set (OCDS), and journal data set
•Control data set and journal data set backup copies
•Problem determination aid (PDA) log data sets
•DFSMShsm logs
•Activity log data sets
•Small-data-set packing (SDSP) data sets
•System data sets

The CDSs are Virtual Storage Access Method (VSAM) key-sequenced data sets (KSDSs). 
The journal file is a physical sequence data set that is allocated to one extent. These data sets must not reside on SMS-managed volumes or volumes that contain other system data sets or catalogs. 
The following list provides the CDSs: 
•Migration control data set (MCDS):
– Contains information about the migration environment. 
– Required for DFSMShsm to function. 
•Backup control data set (BCDS): 
– Information about the backup and dump environment is stored here. 
– Only needed if you intend to use backup and dump functions.
•Offline control data set (OCDS): 
– Contains information about the backup, dump, and migration tape volumes. 
– Required if you intend to use tape devices for backup, dump, or migration volumes.
•Journal data set: 
– Records each critical update to any of the three CDSs.
– Required for DFSMShsm CDS recovery.
Next, how DFSMShsm and SMS work together to accomplish your goals for your storage 
environment is explained.

## 1.5  Data set lifecycle with DFSMShsm
When a data set is allocated on a DASD volume that is controlled and monitored by DFSMShsm (either an SMS-managed or a non-SMS-managed volume), it goes through the following different stages of its lifecycle.

### 1.5.1  Data set create
During the creation of a data set, the system performs a set of basic activities. SMS ACS routines are given control and decide whether a data set is SMS-managed: 
•For SMS-managed data sets, the data set is usually assigned to a management class that 
allows DFSMShsm to manage the volume on which it exists. 
•For non-SMS-managed data sets, the storage administrator defined the non-SMS 
volumes to DFSMShsm so that DFSMShsm manages the volume and the data set. These activities allow DFSMShsm to manage the volume and the data sets on those volumes.

### 1.5.2  Data set backup
If the volumes where the data sets exist are managed by DFSMShsm (either SMS or non-SMS volumes), DFSMShsm checks whether the volume and data set are eligible for backup. Depending on how you define your backup parameters to DFSMShsm, the volume, and the data set will be backed up.

###1.5.3  Data set update 
The next step in the data set lifecycle is usually when the data set is updated. Whenever a user, job, or task performs output processing to a DASD data set, part of this process causes an update to the FMT1 DSCB to show that the data set was changed with the last referenced date being updated.

### 1.5.4  Data set migration
The next step in the data set lifecycle is based on the duration since a data set was referenced, which is determined by your storage administrator. For SMS volumes, the duration is the Primary day parameter value in the management class. For non-SMS volumes, it is based on the value of the MIGRATE parameter when the volume is added to DFSMShsm. When the date is passed, DFSMShsm migrates the data set from the primary user volume to a lower-cost DASD volume. During this process, DFSMShsm records the updates in its CDSs and updates the volume table of contents (VTOC) and the catalog.

### 1.5.5  Data set recall 
When a data set is migrated and a user needs this data set for input or output, DFSMShsm recalls it. The DFSMShsm CDSs, catalog, and VTOCs are updated to reflect these changes.

### 1.5.6  Data set expiration
If the user never requested a recall and the data set reached its retention period, DFSMShsm expires the data set. It performs this expiration based on “Expire non-usage”, “Expire date/days” or “Ret limit” in the management class, or “DELETEBYAGE” if the volume is a non-SMS-managed volume. During this process, the DFSMShsm CDSs, the catalog, and any VTOCs are updated, as appropriate (see Figure 1-8). Depending on the retained backup versions that are set in the storage management policies, any backups of the deleted data set are still retained. 

### 1.5.7  Data set recovery 
If something happens to your user data set and you set up DFSMShsm so that it backs up your data set, you can recover it from a backup copy.

###1.5.8  Expiring backup versions
One of the last steps in a data set’s lifecycle is deleting the backup versions of a deleted data set (see Figure 1-10). This task is not performed automatically by DFSMShsm. The storage administrator handles this task by issuing the DFSMShsm EXPIREBV  command. This command expires old and unnecessary backup versions of deleted data sets. DFSMShsm updates its CDSs to reflect that these backup versions were deleted. 

## 1.6  Space management
A major function of DFSMShsm is space management. With space management, you can keep DASD space available for users on a daily and hourly basis to meet the service-level objectives of your system. The purpose of space management is to manage your DASD storage efficiently so you do not have to. 
Space management encompasses the following activities: 
• Freeing over-allocated space through partial release
• Deleting expired data setsNote:  When the backup version is deleted, the space where it was is no longer available 
and cannot be used to recover the data set. 
• Migration of data by moving eligible data sets that were not used recently to an ML1 or ML2 volume: 
– Data movement from expensive volumes to less expensive storage is called migration
Data can be migrated from user volumes that users can access to ML1 volumes. The ML1 volumes must be DASD volumes. Data can also be migrated to ML2 volumes, which are usually tape volumes. Although ML2 volumes can be DASD, SMS supports only ML2 tape.
– Recall: When the data set is needed again, DFSMShsm returns it to the user’s control. This process is called recall and is usually processed automatically by DFSMShsm. Recall is also possible by issuing an HRECALL  command from your Time Sharing Option (TSO) session or batch job. The SMS ACS routines ultimately determine the volume to which the data set is recalled.
• Extent reduction 

Space management is achieved through the following phases: 
• Primary space management
• Secondary space management
• Interval migration
• On-demand migration

This process ensures that only active data occupies space on DASD volumes. Space management parameters control how DFSMShsm makes space available on level 0 volumes. Space management is specified as a combination of parameters in the management classes and storage groups. When planning for space management, you want to strike a balance between enough available DASD space for new data set allocations and frequent recalls of migrated data sets. The management class attributes apply on a data set basis so that different data sets on the same volume can be migrated based on different criteria.

### 1.6.1  Recycle activity 
The data that DFSMShsm stored on tapes is invalidated over time by the expiration of migrated data sets or the generation of more recent backup data sets. The recycle function provides the capability of moving the valid data sets out from the original tapes and consolidating the data on another tape. One tape contains all unexpired data sets and leaves the recycled tapes available for scratching.

### 1.6.2 Primary space management
During primary space management, DFSMShsm combines a set of specific functions to accomplish DASD usage levels that are coded in storage group (SG) definitions. For non-SMS-managed volumes, are defined in the DFSMShsm PARMLIB member ARCCMDxx.
The following functions are performed by DFSMShsm during primary space management:
• Release unused over-allocated space of eligible data sets
• Delete expired and temporary data sets 
• Migrate data sets to ML1 or directly to an ML2 volume
• Whether or not to delete or migrate rolled-off generations if the data set was a member of a generation data group (GDG)

Note:  All data sets that are stored in the same non-SMS volume have the same backup frequency, number of versions to keep, migration, and deletion rules.

The first functions that are started by primary space management include checking level 0 volumes for temporary and expired data sets that are eligible for deletion. DFSMShsm also releases unused and over-allocated space from eligible data sets and migrate-eligible data sets that were recalled recently, by using the Fast Subsequent Migration function, if available. If the previous tasks were sufficient to meet the low threshold value that is coded in the SG where the volume resides or the value that is specified in the DFSMShsm PARMLIB member ARCCMDxx, the space management ends for that volume, and the next volume is selected for processing. Otherwise, DFSMShsm continues primary space management processing.

The next steps in primary space management are selecting and migrating the largest data set that is eligible for migration in the selected volume. All of the data sets are migrated to ML1 DASD, or ML2 tape, according to management class (MC) polices or ARCCMDxx specifications. Migration processing moves to the next volume when either of these conditions is met: Volume met the low threshold utilization that is coded in the SG, or no more data sets are eligible for migration. In a non-SMS environment, all data sets that are eligible are migrated, regardless of the volume space usage. A data set that is migrated from a non-SMS-managed volume can be recalled to an SMS-managed volume if SMS is active and if the data set meets the conditions for selecting SMS management for that data set. DFSMShsm recalls a non-SMS-managed data set to a non-SMS-managed volume only if the data set does not meet the conditions to become an SMS-managed data set, or if a command forces the recall to a non-SMS-managed volume.

DFSMShsm-managed volumes are usually defined in SMS storage groups. 

Non-SMS-managed volumes can be defined directly in the ARCCMDxx member in DFSMShsm PARMLIB. DFSMShsm performs space management on both SMS-managed and non-SMS-managed volumes.

### 1.6.3  Secondary space management
In secondary space management, in addition to moving data sets from ML1 volumes to ML2 volumes, DFSMShsm performs migration cleanup. These processes are performed in parallel. Migration cleanup deletes expired data sets, erases deleted data from SDSP data sets, and deletes unwanted records from the DFSMShsm MCDS.
DFSMShsm also performs these tasks:
• Schedules TAPECOPY  commands for migration tape  copy-needed (TCN) records
• Deletes expired data sets from the migration volumes
• Deletes obsolete MCDs, volume  statistics records (VSRs), and DSRs during migration cleanup
•Moves data sets (under the control of the management class) from ML1 to ML2 volumes

### 1.6.4  Interval migration
DFSMShsm interval migration ensures that a specified amount of space is available on an hourly basis. DFSMShsm performs interval migration, as needed, for all storage groups that are eligible for interval management. In interval migration, DFSMShsm performs a space check on each DFSMShsm volume that is being managed. A volume is considered eligible for interval migration based on the AUTOMIGRATE (AM=) and THRESHOLD settings of its storage group. When AM=I, the space that is used must exceed the halfway mark between high and low thresholds to make the volume eligible. When AM=Y, the space that is used must exceed the high threshold to make the volume eligible. DFSMShsm migrates eligible data sets to ML1 or ML2 volumes. This process continues until the volume that is managed by DFSMShsm reaches the low threshold or no more data sets are eligible.

### 1.6.5 On-demand migration
On-demand migration is available as an alternative to interval migration. On-demand migration performs near-immediate space management on eligible SMS-managed volumes that exceed the specified volume high threshold, instead of waiting for interval migration to run at the top of each hour. On-demand migration migrates data sets to ML1 or ML2 volumes and continues to process the volume until the low threshold is reached or no eligible data sets remain.

On-demand migration is based on an event notification from SMS. You can configure your system to manage your space needs. 

## 1.7 Availability management
The other major function that  DFSMShsm provides is availability management. Availability management enables a user to recover a lost or damaged data set easily. A storage administrator can recover a damaged volume easily at the current level. 

Availability management (backup)  automatically and periodically performs the following backups: 
• Back up CDSs and journal DFSMShsm
• Back up data sets from DASD to tape volumes
• Back up changed data sets on DASD volumes to other DASD or tape volumes 
• Back up data sets of an application to tape volumes so they can be taken offsite for recovery 

By performing these functions, automatic backup can help in these ways: 
• Prevent users from accidentally losing or incorrectly changing data sets 
• Recover data after a volume is lost due to a hardware failure 
• Protect against loss of data due to a disaster 
• Ensure that critical data is retained 

New data retention laws cause an increased need for backups of critical data.

The following functions provide valid backup or dump copies so that you can recover:
• Backup
• Dump
• Aggregate backup and recovery support (ABARS)
• Fast replication backup

### 1.7.1  Backup and recovery
Different methods are available to back up your data. DFSMShsm provides an automatic backup function, inline backup, fast replication backup, and aggregate backup.

#### Automatic backup 
The auto backup function is a data-set-level function. It relies on the guaranteed backup frequency attribute of the storage group for each data set to determine whether to copy the data set. After a data set is backed up, this function uses the data set’s management class attributes to decide how the data set is treated for the creation and retention of backup versions (how many backup versions are kept and how long they are kept).

In a non-SMS environment, the backup frequency is defined for the entire volume. All data sets that changed since the last backup process ran are eligible for a new backup. Recovery restores data sets from the daily backup volumes or the spill backup volumes to the level 0 volumes. Recovery can be performed only by command, for individual data sets or complete volumes.

#### Inline backup 
With the inline backup function, you can back up data sets in a batch or online environment. You can back up data sets in the middle of a job in a batch environment, or directly through Time Sharing Option (TSO). Inline backup writes the backup version on an ML1 volume or to a tape volume. If the data set is moved to an ML1 volume, you can minimize unplanned tape mounts. In that case, the backup version is later moved to the backup volume during automatic backup or by command.

#### ABARS
Aggregate backup and recovery support (ABARS) is a function of DFSMShsm that is designed for use in disaster recovery. ABARS facilitates a point-in-time backup of a collection of related data consistently. This group of related data is defined by ABARS as an aggregate. An aggregate is user-defined and usually a collection of data sets that are related, such as all of the specific data sets required for an application (for example, payroll). ABARS backs up the data directly from DASD (either SMS-managed or non-SMS-managed), DFSMShsm-owned ML1 DASD, or DFSMShsm-owned ML2 tape without needing intermediate staging capacity. The backup copies are created in a device-independent format.

ABARS can be used also for moving applications across non-sharing systems. If ABARS is used as a disaster recovery tool, at the recovery site, it is used to recover the data sets and allocate user-defined empty data sets that the client requires for their operating environment. 

The aggregate backup and aggregate recovery functions provide the capability to back up and recover a user-defined group of data sets. The user-defined group of data sets can belong to an application or any combination of data sets that you want to be treated as a separate entity. ABARS treats SMS and non-SMS data sets identically.

With the aggregate backup and aggregate recovery functions, you can perform the following actions:
• Define the components of an aggregate
• Back up data sets by aggregate
• Recover data sets by aggregate
• Recover a single data set
• Duplicate your aggregates at a remote site
• Resume business at a remote  location, if necessary 

#### Fast replication
Fast replication is a function that uses volume-level fast replication to create backup versions for sets of storage groups. You can define a set of storage groups with the SMS copy pool construct. Fast replication target volumes contain the fast replication backup copies of volumes that are managed by DFSMShsm. Fast replication target volumes are defined with the SMS copy pool backup storage group type.

The DFSMShsm FRBACKUP  command creates a fast replication backup version for each volume in every storage group that is defined within a copy pool. Volumes that have a fast replication backup version can be recovered either individually or at the copy pool level with the DFSMShsm FRRECOV command. This function enables the backup and recovery of a large set of volumes to occur within a short time frame. For more information about fast replication and its requirements, see the DFSMShsm Fast Replication Technical Guide, SG24-7069.

### 1.7.2  Dump
By using the automatic full volume dump function with the automatic incremental backup, the storage administrator can recover a complete volume by restoring the volume from the full volume dump, then correcting the volume for later activity by the automatic inclusion of changes taken from incremental backups. A DFSMShsm-authorized user can issue one RECOVER  command that is used to request both a volume restore and an incremental volume recovery.

To use the dump function, a dump class must be assigned to the required storage groups. Next, you need to define the dump class to DFSMShsm if it is not already set. The dump class definition includes a brief description, the dump date or day of the week to be dumped, the retention period, and the expiration date.

## 1.8  Tape processing
DFSMShsm can use tape for backup, migration, dump, and ABARS. When DFSMShsm maintains its tape environment, it performs tape copy and recycles of backup, dump, ABARS, or ML2 volumes. You implement a DFSMShsm tape processing environment by specifying SETSYS  commands in the DFSMShsm PARMLIB member ARCCMDxx.

The tape environment is determined by the definition of the library environment (tape library or non-library), tape management policy, device management policy, and performance management policy. You can also define an SMS-managed tape environment (which includes a tape management library) or a non-SMS-managed tape environment (a non-library environment).

#### DFSMShsm duplexing
The DFSMShsm duplex function allows a primary and copy tape to be created concurrently. The primary tape can be kept onsite for recovery; the copy tape can be kept at a secure offsite location. This method is the preferred method of creating copy tapes and needs to be used in preference to the TAPECOPY  command.

When DFSMShsm requests a scratch tape for output, the tape management system returns a scratch tape that is based on the type of tape that is requested and the scratch pool that is assigned. You can define a global scratch pool or a specific scratch pool. A global scratch pool is a repository of empty scratch tapes for use by any user. A specific scratch pool is a repository of empty scratch tapes whose use is restricted to a specific user or set of users. Global scratch pools are recommended because mount requests can be responded to more quickly and easily than when tapes are in a specific scratch pool. By using a global scratch pool, you can take advantage of automatic cartridge loaders easily, reducing the tape mount wait time. Global scratch pools also enable the use of a tape management product, such as DFSMSrmm.

#### DFSMSrmm and DFSMShsm interaction
Interaction between DFSMSrmm and DFSMShsm occurs during the whole tape lifecycle. SETSYS commands that are added in ARCCMDxx specify how DFSMShsm processes the tapes as they enter the scratch pool, are inventoried as active data by DFSMShsm, are recycled for backup and migration tapes, and are returned to the scratch pool.DFSMShsm is responsible for managing all of its tape retention and lets the tape management system know when the tapes can be returned to the scratch pool. If your system does not use DFSMSrmm, DFSMShsm communicates with your tape management program through the exit ARCTVEXT.

For more information about DFSMSrmm, see DFSMSrmm Primer, SG24-5983. DFSMShsm records its information about the content on the tapes that it uses in the DFSMShsm offline control data set (OCDS). It stores the information in a control data set (CDS) record that is called the tape table of contents (TTOC). 
