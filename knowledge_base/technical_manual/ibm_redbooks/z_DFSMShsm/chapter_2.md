  # 2. Planning and reviewing your environment
  
## 2.1  Considerations
Many considerations are involved as you prepare to set up your DFSMShsm environment. If you have an existing DFSMShsm environment, many of these items are valuable to review periodically because your configuration and environment changes.
Consider the following key questions and information:
- How much data do you want DFSMShsm to manage? 
  - How much available space is needed throughout the day for applications to run 
successfully? 
  - How often is the data referenced again after creation?
- How do you want DFSMShsm to manage the data?
  - Manage the space available on volumes?
Use the small-data-set packing (SDSP) facility?
  - Manage your backup and recovery data? 
  - Will you use encryption?
  - Expire the migration and backup copies of data sets?
- Data retention:
  - How long do you need to keep unreferenced data? 
  - How much data is created that has lengthy retention requirements?– If you have lengthy retention requirements, will you use high capacity tape to store this data? 
- Disaster recovery: 
  - What requirements do you have to recover critical data in a disaster?– Will you employ the DFSM Shsm duplexing function?
  - Will you use encryption?
- Is your environment storage management subsystem (SMS) or non-SMS, or a combination?
- Security aspects:
Will you use RACF or anther form of a security product?
- Interactions with othe r Data Facility Storage Manageme nt Subsystem (DFSMS) products: 
  - Will you use cross memory to invoke DSS for data movement?
  - Object access method (OAM) – RMM (or tape management system) for managing tape volumes
- Preparing for diagnostic procedures
Problem determination aid (PDA) files, log files, and dumps
- Storage requirements for DFSMShsm to manage your data:
  - Volumes for the control data sets (CDSs) and journal
  - Extra direct access storage device (DASD) for migration level 1 (ML1) volumes for 
employing an ML1 policy
  - Extra tapes and tape drives to handle DFSMShsm backup, migration, and full volume 
dump dataChapter
## 2.2  Determine how you want DFSMShsm to manage your data
DFSMShsm provides functions for th e following capabilities to manage data:
- Space management : Space management is the function of DFSMShsm to keep DASD space available for users to meet the serv ice-level objectives for your system. The purpose of space management is to manage your DASD storage efficiently. Space management automatically and periodically performs the following functions: 
  - Moves low-activity data sets from user-accessible volumes to DFSMShsm volumes. 
  - Reduces the space that is occupied by data on both the user-accessible volumes and the DFSMShsm volumes.
- Availability management : DFSMShsm backs up your data either automatically or by command to ensure availability in the accidental loss of data  sets or physical loss of volumes. DFSMShsm also allows a storage administrator to copy backup and migration tapes. These copies can be stored onsite as protection from media damage, or offsite as protection from site damage. Disaster backup and recovery are also provided for user-defined groups of data sets ( aggregates ) so that critical applications can be restored at the same location or an offsite location.

Data can be categorized in the following manner:
- System data: All system-related data that the system requires for its operation.
- Databases: Each database management system has unique da ta sets and facilities to support its online environment. These differences change the recommended storage management procedures. Database data has diverse space, performance, and availability requirements; however, dividing your database data into categories helps you identify the required storage management subsystem (SMS) services and implement a staged migration to system-managed storage. 
- Data that is created by batch jobs (generation data groups (GDGs) and Virtual Storage Access Method (VSAM) data sets).
- Data that is used by Time Sharing Option (TSO) users (user profiles and user job control language (JCL) files).
### 2.2.1  Planning for the space management process
By using space management features, you allow your data to be migrated by DFSMShsm from primary volumes (user volumes that are also known as L0 volumes ) to DFSMShsm volumes, either ML1 (L1) or ML2 volumes (L2).

Consider the following factors for the space management process:
- You want to migrate your data: Identify the data that can be migrated based on how long the data is inactive and not referenced. In general, system data and production databases must not be migrated because these types of files are referenced fairly frequently. Depending on your own service level agreements (SLAs), you develop policies to migrate data based on those needs and how often the data is referenced after it is created:
  - TSO data sets (data that is owned by TSO users or application development) are usually smaller with lower I/O activity than production data sets. These data sets might not always have predictable access or update patterns. Migration policies for these data sets can allow them to reside on user volumes longer.
  - Batch data is data that is processed regularly, usually as part of a production cycle. Most of these data sets are usually sequential data sets that are members of GDGs. You develop specific management class attributes for handling GDG roll off processing, either delete or migrate.
- Length of time to keep the migrated data: This factor depends on your installation 
requirement. Certain data might need to be retained longer than other data. You develop specific management class attributes based on these needs.For non SMS-managed volumes, this value is determined when you add the volume to the DFSMShsm configuration. 
- Time of day to schedule space management: Ideally, the migration process starts and completes before the batch window because the migration generates more free space for the batch jobs to use. The length of the migration window depends on a few factors:
  - How much data is migrated in a certain day?
  - How much of this data is migrated to DASD or tape? – If the data is being migrated to tape, how many tape drives are available for processing?

In a normal environment, data is migrated first to ML1 volumes, which are a pool of non-SMS-managed volumes that are dedicated for the use of DFSMShsm. If a data set continues to be unused, it is eventually migrated from an ML1 to an ML2 volume, which is usually high capacity tape. In certain cases, large data sets are often migrated straight to ML2 tape volumes. 

You control the automatic migration parameters. As you monitor your environment to ensure automated functions complete on time, you adjust your environment. 
### 2.2.2  Planning for availability management
Backing up your data is vital to protect your information. By using the backup functions, you allow DFSMShsm to back up your data based on the parameters that you define.Chapter 
Consider the following factors:
- Automatic backup : You can configure DFSMShsm to back up your data for you. During the automatic backup process, DFSMShsm first backs up the CDSs and journal. If you decide that you do not want to use the automatic backup process, ensure that you back up the CDSs and journal manually so that you can recover your DFSMShsm environment, if necessary.
- Time of day to schedule the backup and for how long : Choose the backup window when the system is the least active so that DF SMShsm is more likely to back up data sets that are normally in use.The length of the backup window depends on the number of data sets that are modified in a certain day and the number of tape drives that are available for the DFSMShsm automatic backup function. Y ou control the automatic backup parameters to best meet your needs. 
- Backup method (standard or Concurrent Copy) : Concurrent Copy is a combined hardware, Licensed Internal Code (LIC), and software system management solution that creates data dumps or copies while user processing continues. When DFSMShsm uses Concurrent Copy to back up the data sets, the time during which system functions are suspended is dramatically reduced. DFSMShsm reserves the data sets only for the time that it takes to initialize a Concurrent Copy session. Use of Concurrent Copy for data set back up is justified only if the data set is database-related and significant value exists in serializing the data set for the shortest possible duration.
- Data backup frequency and number of backup copies : You need to decide how frequently to back up the data and whether to back it up even if it did not change. How many backup copies do you want of your data sets? How long do you want to keep the backup copies after the data sets are delet ed? The correct answers to these questions 
can help ensure that data sets can be recovered.

**Space saving with space management and backup**
Part of DFSMShsm space management and backup management allows DFSMShsm to consolidate data while it migrates and backs up the data sets. Both functions perform cleanup activities so that space is available for migration and backup data. 

Consider implementing the following functions to save space:
- Partitioned data set (PDS) compression: For PDS data sets, the migrate and recall result in freeing embedded free space (compression), and the user information in the data-set-directory user field is retained.
- Release of over-allocated space: DFSMShsm can release unused over-allocated space during its space management process.
- Extent reduction: The migrate and recall result in reducing the number of extents for that data set.
- Reblocking: During recall and recovery, the proc ess of reblocking user data sets changes the number of records in a physical block and  therefore uses the space on the DASD user volume more efficiently. Data movement that uses DFSMShsm reblocks physical sequential data sets. Data movement that uses DFSMSdss reblocks physical sequential and PDSs. 

Use these factors to help you to determine the SMS attributes to use as you create your automatic class selection (ACS) routines.
## 2.3  Tape duplexing
The DFSMShsm duplex function allows a primary and copy tape to be created concurrently. 
The primary tape can be kept onsite for recovery and the copy tape can be kept at a secure offsite location. This method is the preferred method of creating copy tapes and needs to be used instead of the TAPECOPY  command.

If you intend to use duplex migration or to back up tape volumes, you need twice as many 
available tape drives for the use of DFSMShsm . DFSMShsm writes to two output devices in 
parallel for each function. This function is  specified in your ARCCMDxx startup PARMLIB 
member. 
## 2.4  Data encryption
Data encryption is an important tool for protecting against the possible misuse of confidential information that can occur if tapes are lost or stolen. Unless the possessor of the tape has the required key, any encrypted data on the tape remains confidential and is unreadable. Therefore, consider securing tape data through encryption in your overall security plan. 

You can secure your tape data with either tape device encryption  (with an encryption-capable 
tape drive) or through host-based encryption, that is, by requesting that DFSMSdss encrypt the data before writing it to the tape volume. In general, you use one method, not both.

When you choose a method, consider the following factors:
- Use tape device encryption if your installation includes one or more encryption-capable tape drives. Here, you specify by data class the data that is to be encrypted when stored on the tape drives.
- Use host-based encryption if you do not have an encryption-capable tape drive. Y ou can encrypt tape backups through the host-based encryption method.
### 2.4.1  zEDC hardware compression
IBM z™ Systems Enterprise Data Compression (zEDC) for z/OS hardware compression was introduced with EC12 and BC12 servers. zEDC hardware compression requires I/O compression adapters and a licensed feature (Feature Code (FC) 0420) on the server side and z/OS V2.1 (plus PTFs) and the zEDC Express for z/OS feature on the operational system side. 

zEDC compression is an effective hardware compression feature for queued sequential access method (QSAM) and bas ic sequential access method  (BSAM) that can replace tailored compression. From the DFSMSdss pers pective, support for zEDC compression was added for QSAM and BSAM, also. DFSMShsm will support zEDC compression on disk and tape for backup, migration, and dump data if DFSMSdss is the data mover. 

Activation of zEDC hardware compression in DFSMShsm, in addition to the prerequisites, requires zEDC compression to be enabled in the IGDSMSxx member in DFSMS and PARMLIB activation in DFSMShsm. For DFSMShsm migration and backup, zEDC compression can be activated by function (backup on disk, backup on tape, migration to disk, or migration to tape), or for all of these functions in one SETSYS  command. Activation of zEDC compression on the DFSMShsm DUMP function can be activated with a new ZCOMPRESS  keyword on the dump class.

For more information, see the IBM zEnterprise BC12 Technical Guide , SG24-8138, Reduce Storage Occupancy and Increase Operations Efficiency with IBM zEnterprise Data Compression , SG24-8259, and zEDC Compression: DFSMShsm Sample Implementation, REDP-5158 . 
## 2.5  Using small-data-set packing 
The small-data-set packing (SDSP) data set fa cility of DFSMShsm allows DFSMShsm to migrate small user data sets from user volumes and store them as records on an ML1 migration volume that is known as an SDSP volume . This ML1 volume is made up of one large VSAM key-sequenced data set (KSDS). The small user data sets are stored in the VSAM data set as a record , which takes up less sp ace on a DASD volume.

SDSP volumes offer the following advantages:
- Reduced fragmentation of an ML1 volume.
- Reduced use of space in the volume table of contents (VTOC). Because the small-user data sets are stored as records, ML1 VTOCs are not filled with an entry for each small user data set that is in an SDSP .
- Better use of space on the ML1 volumes because the small data sets are in the SDSP VSAM KSDS as a record. A single record wit hin a VSAM KSDS takes up less space than a data set that is stored directly on a volume.

SDSP data sets might require periodic reorganization just like any other VSAM KSDS. You can reduce the need to reorganize SDSP data se ts by enabling the control area (CA) reclaim function for them.

Although one SDSP data set can be used for each concurrent migration task, certain DFSMShsm activities have a higher usage priority for SDSP data sets, such as the following activities:
- Recall processing
- Aggregate backup and recovery  support (ABARS) processing
- FREEVOL processing
- AUDIT MEDIACONTROLS processing
- Automatic secondary space management processing

It is important to plan the number of SDSP data sets in relation to the number of concurrent migration tasks and the amount of processing that is performed by functions with a higher usage priority for the SDSP data sets.

Because of their higher usage priority, any of th ese activities can gain control of your SDSP data sets and leave you with fewer than the expected number of SDSP data sets for migration. When an activity with a higher usage priority for SDSP data sets has or requests an SDSP data set, that SDSP data set is no longer a candidate for migration.

To begin using the SDSP facility, you first need to perform the following tasks:
- Define the size of a small user data set.
- Allocate SDSP data sets.
- Specify the SDSP parameter on the ADDVOL  statement.

For more information, see the DFSMShsm Implementation and Customization Guide, SC35-0418.
## 2.6  Expired data sets and tape recycle
The following considerations relate to expired data sets and tape recycling:
- Expiring backup versions: This expi ration is accomplished through the EXPIREBV  command. The DFSMShsm  command deletes unwanted backup and expired ABARS versions of SMS-managed and non-SMS-managed data sets from storage that is owned by DFSMShsm. This command is a long-running command so most clients are selective when and how often they run this command.
- Recycle process: The RECYCLE function is a DFSMShsm method of reclaiming tape media capacity. As time passes, data on your backup and ML2 tape volumes becomes invalid. Migrated data sets expire or are marked for deletion. Backup versions roll off or are marked for deletion. To consolidate the valid data on to fewer tapes, you can use the DFSMShsm recycle process. The recycle function is considered by DFSMShsm to be a long-running task and therefore is limited to one command in execution per logical partition (LPAR). Because this function processes offline media, run this process outside of normal batch windows, either during the day or on weekends.
## 2.7  SMS classes and ACS routines 
With SMS, you can define performance goals and data availability requirements, create model data definitions for typical data sets, and automate data backup and space management. SMS can automatically assign, based on installation policy, those services and data definition attributes to data sets when they are created. IBM storage management-related products determine data placement, manage data backup, control space usage, provide data security, and perform disaster backup and recovery. 

The goals for system-managed storage are listed: 
- Improve the use of the storage media, for example, by reducing out-of-space abnormal end of tasks (abends) and providing a way to set a free-space requirement. 
- Reduce the labor that  is involved in storage manage ment by centralizing control, automating tasks, and providing interactive or batch controls for storage administrators. 
- Reduce the user’s need to be concerned with the physical details of performance, space, and device management. Users can focus on using information instead of managing data.

In the DFSMS environment, you use SMS classes and groups to set service requirements, performance goals, and data definition models for your installation. You use the Interactive Storage Managem ent Facility (ISMF) to create the appropriate classe s and groups, and automatic class selection (ACS) routines to assign them to data according to your installation’s policies. 
## 2.8  Security aspects in the DFSMShsm environment
How do you plan on securing DFSMShsm data and the DFSMShsm commands? The use of a security product, such as RACF or an equivalent, simplifies security for the storage administrator in protecting DFSMShsm resources.

Consider the following factors about security:
- Protect all CDSs with resource protection, such as RACF , from being updated by unauthorized programs and unauthorized personnel.
- DFSMShsm  and the DFSMShsm cross-memory mode started tasks must be added to the started procedures table in RACF (or the equi valent table if another security product is used).
- Consider protecting the DFSMShsm commands in the same manner as any other resources that are restricted to only storage administrators.
- Because DFSMShsm manages its own data set resources on DASD and tape, protection from unauthorized access to DFSMShsm resources is an important consideration.
The following DFSMShsm resources must be protected from unauthorized programs and unauthorized personnel:
- DFSMShsm data sets:
  - CDSs
  - journal
  - Problem determination aid (PDA) trace data sets
  - Logs
  - CDS backup versions
  - SDSP data sets
  - Migrated data sets
  - Backed-up data sets
  - ABARS SYSIN data sets
  - ABARS FILTERDD data sets
  - ABARS RESTART data sets
  - ABARS access method servic es (IDCAMS) data sets
- DFSMShsm tapes:
  - ML2 migration tapes
  - Incremental backup tapes
  - Dump tapes
  - TAPECOPY tapes
  - ABARS tapes
## 2.9  Invoking DSS to move data
In order to maximize throughput for backup, DFSMShsm CDS backup, dump, migration, and recovery function, you can use the cross-memory mode feature of DFSMSdss. The SETSYS DSSXMMODE  command allows users to specify the functions that invoke DFSMSdss in cross-memory mode as opposed to having DFSMSdss loaded in the DFSMShsm address space. 

The SETSYS DSSXMMODE  command can be issued only from the ARCCMDxx PARMLIB member. It does not affect the FRBACKUP and FRRECOV use of the DFSMSdss cross-memory interface.

Note: If you plan to use FRBACKUP DUMP to tape and FRRECOV from tape operations, the recommendation is that the DUMP and RECOVER functions use the DFSMSdss cross-memory mode to help prevent out-of-storage abends.

To implement this feature, you need to set up all security aspects for all started tasks that are involved in the same way to set up DFSMShsm STC. 
## 2.10  Interactions with object access method
The OAM is a component of DFSMSdfp, the base of the SMS of DFSMS. The OAM uses the concepts of system-managed storage, which are introduced by SMS, to manage, maintain, and verify tape volumes and tape libraries within a tape storage environment.

You need to consider the setup that is needed for DFSMShsm process tape for migration and backup. The focus is on automatic tape management.

In general, a tape library  is a set of tape volumes and the set of tape drives where those volumes can be mounted. The relationship between tape drives and tape volumes is exclusive. A tape volume that resides in a library (library-resident tape volume ) can be 
mounted only on a tape drive that is contained in that library ( library-resident tape drive ). A library-resident tape drive can be used only to mount a tape volume that resides in the same library. A tape library can consist of one or more tape systems.

When a volume is entered into a tape library, it is assigned to a tape storage group. A tape library can contain volumes from multiple storage groups, and a storage group can reside in up to eight libraries.

Tape automation provides satisfactory solutions for many of the problems that occur when tape library storage requires human intervention. Mount times are reduced from minutes to seconds. The number of lost, misfiled, or damaged tapes decreases. Security is enhanced because the tape library hardware and tape cartridges can be kept in a secure area.

The following considerations relate to implementing DFSMShsm functions in an SMS-managed tape library:
- Determine the tape functions that you want to process in a tape library: You must decide which DFSMShsm functions to process in a tape library. Each DFSMShsm function uses a unique tape data set name. An ACS routine can recognize the functions that you want to 
process in an SMS-managed tape library by the data set names.When you consider which functions to proc ess in a tape library, think in terms of 
space-management processing, availability -management processing, or CDS backup processing as indicated in the following implementation scenarios. The migration and backup conversions are implemented differently. The implementations contrast each other because, for migration, existing tapes are inserted into the library and for backup, scratch tapes are inserted into the library and the existing backup tapes are left in shelf storage.
- Set up a global scratch pool: When a DFSMShsm output function fills a tape, it requests another tape to continue output processing. Tapes are obtained from a scratch pool. The scratch pool can be either a global scratch pool or a specific scratch pool:
  - global scratch pool  is a repository of empty tapes for use by anyone. The tape volumes are not individually known by DF SMShsm while they are members of the scratch pool. When a scratch tape is mounted and written to by DFSMShsm, it becomes a private tape and is removed from the scratch pool. When tapes that are used by DFSMShsm no longer contain valid data, they are returned to the global scratch pool for use by anyone and DFSMShsm removes all knowledge of the existence of them.
  - specific scratch pool  is a repository of empty tapes that are restricted for use by a specific user or set of users. When DF SMShsm is in a specific scratch pool environment, each empty tape and each used tape is known to DFSMShsm as a result of being added to the scratch pool, generally by the ADDVOL  command. These tapes can be used by DFSMShsm only. The key characteristic of a specific scratch pool is that when a DFSMShsm tape becomes void of data, the tape is not returned to the global scratch pool but it is retained by DFSMShsm in the specific scratch pool for reuse by DFSMShsm.Global scratch pools are recommended because mount requests can be responded to more quickly and more often than when tapes reside in a specific scratch pool. The use of a global scratch pool enables the easy exploitation of cartridge loaders (including cartridge loaders in tape-library-resident devices) and works well with tape management systems, such as DFSMSrmm.

- Protecting tapes: Tape protection helps ensure that only DFSMShsm is allowed to access DFSMShsm tapes. The implementation method that is used depends on the security product that is installed.
- Tape duplexing: The DFSMShsm duplex function allows a primary tape and copy tape to be created concurrently. The primary tape can be kept onsite for recovery and the copy tape can be kept at a secure offsite location. This method is the preferred method of creating copy tapes and needs to be used instead of the TAPECOPY  command.
## 2.11  Interaction with the DFSMSrmm tape management system
DFSMSrmm provides enhanced management for tape volumes that DFSMShsm uses for each of its tape functions. Precisely how the two products work together depends on how you use each of them in your installation.

DFSMShsm can tell DFSMSrmm when it no longer requires a tape volume or when a tape volume changes status. When DFSMShsm is using DFSMSrmm, it cannot mistakenly overwrite one of its own tape volumes if an operator mounts a tape volume in response to a request for a non-specific tape volume. DFSMShsm uses  the DFSMSrmm EDGTVEXT interface to maintain the correct volume status. DFSMSrmm provides a programming 
interface so you can use the DFSMShsm exit. 
## 2.12  Preparation for problem diagnostic determination
Often, when problems occur, sufficient data for problem determination is not readily available. IBM technical support requires different data, depending on the type of problem that is encountered. The following items are often used for debugging:
- DFSMShsm activity logs
- DFSMShsm joglog
- System log
- DFSMShsm PDA traces
- DFSMShsm journal records
- System Measurement Facility (SMF) records
- Any possible system dumps that were captured at the time 
Consider the following factors when you prepare for problem diagnostic determination:
- Ensure that the PDA log is turned on and being collected.
Capture and archive the contents of PDA files when they are swapped. For more 
information, see the following link:
http://www.ibm.com/support/docview.wss?uid=isg3T1012687
- Ensure that you configured DFSMShsm for logging and log files are allocated. 
If you are concerned about DASD space and you must decide between having 
DFSMShsm log files or DFSMShsm PDA files, it is recommended to choose capturing the DFSMShsm PDA. Information that is written to the log files is also captured in the DFSMShsm PDA trace. Therefore, the PDA trace provides the logging information. 
- Ensure that DFSMShsm is configured correctly to capture SAN Volume Controller dumps 
if an abend occurs.
- Ensure that your z/OS system is able to capture a dump that is generated by DFSMShsm.
