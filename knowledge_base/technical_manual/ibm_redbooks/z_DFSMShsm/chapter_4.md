# 4. SMS and its interactions with DFSMShsm

## 4.1 DFSMShsm in an SMS environment

To implement DFSMShsm into an existing SMS environment or from scratch, you must
change your SMS configuration. If you plan to implement DFSMShsm in a non-SMS
environment, consider also converting the data that DFSMShsm manages to SMS control as
part of the DFSMShsm implementation. You have more options for and granularity in
enforcing those options when data is managed by SMS.

In a DFSMShsm environment, data can be divided into the following categories:

- Data that is managed by DFSMShsm
This group includes all of the user and application data that DFSMShsm manages according to the parameters that are coded in the SMS management classes. Certain DFSMShsm system data sets are also included in this group.
- Data that is owned by DFSMShsm
User data that is now on DFSMShsm-owned volumes, for example, migration level 2 (ML2) data or backup copies. The data in this group belongs to DFSMShsm. SMS must be updated so that it recognizes the DFSMShsm data and excludes it from SMS management.

- DFSMShsm control data
Data, such as control data sets (CDSs) that are required for the operation of DFSMShsm.

The description in this chapter assumes that the data your DFSMShsm is managing as well
as potentially a subset of DFSMShsm control data and DFSMShsm-owned data can be
SMS-managed.

DFSMShsm migration level 1 (ML1) volumes and direct access storage device (DASD)
backup volumes _cannot_ be SMS-managed.

DFSMShsm CDSs, their backups, log data sets and ML2, tape backup volumes, and DUMP
volumes can be SMS-managed if they reside on DASD or reside in SMS-managed tape
libraries.

We start by describing how the SMS values (groups and classes) that are assigned to data
sets determine how DFSMShsm manages the data sets. These topics are not intended to
provide an in-depth description about converting data to SMS management, but instead focus
on the attributes of SMS classes and groups that interact with DFSMShsm.

Next, we describe how you can use SMS attributes to manage the DFSMShsm resources.
This description includes how you can use access control list (ACS) routines to ensure that
the correct classes and groups are assigned to data that is both managed and owned by
DFSMShsm.

This topic is restricted to SMS-managed data sets unless otherwise stated. Several of the
same attributes can be assigned to non-SMS-managed data sets by **SETSYS** commands.

We start with the SMS attributes that can be assigned to a data set, describing them in thereverse order to the order they are assigned:
- Pool storage groups
- Tape storage groups
- Management classes
- Storage classes
- Data classes


-----

Then, we describe assigning these constructs by using ACS routines for data that is both
DFSMShsm-managed and DFSMShsm-owned.

All of these groups and classes are defined to your SMS configuration by using Interactive
Storage Management Facility (ISMF). For detailed instructions about defining and maintaining
an SMS configuration, see _z/OS V1R10.0 DFSMS Storage Administration Reference (for_
_DFSMSdfp, DFSMSdss, DFSMShsm),_ SC26-7402.

## 4.2 POOL storage group

A POOL storage group describes a grouping of DASD on which data sets reside. Figure 4-1
shows the ISMF Pool Storage Group Alter panel for the DB8B storage group in our
environment.

    DGTDCSG2          POOL STORAGE GROUP ALTER     Page 1 of 2
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : DB8B
    To ALTER Storage Group, Specify:
    Description ==> DASD FOR DB2 V8 DB8B
    ==>
    Auto Migrate . . N (Y, N, I or P)  Migrate Sys/Sys Group Name . .
    Auto Backup . . N (Y or N)     Backup Sys/Sys Group Name . .
    Auto Dump . . . N (Y or N)     Dump Sys/Sys Group Name . . .
    Overflow . . . . N (Y or N)     Extend SG Name . . . . . . . .
    Copy Pool Backup SG Name . . . SGCPB1
    
    Dump Class . . .           (1 to 8 characters)
    Dump Class . . .           Dump Class . . .
    Dump Class . . .           Dump Class . . .
    
    ALTER   SMS Storage Group Status . . . N  (Y or N)
    ALTER   SMA Attributes . . . . . . . . N  (Y or N)
    Use ENTER to Perform Selection; Use DOWN Command to View next Page;

_Figure 4-1  Storage group attributes_

Several DFSMShsm functions are managed at a storage group level:
- Space management
- Availability management:
  - Automatic backup processing
  - Automatic dump processing

You can also control which system performs what functions against the volumes of a storage
group at the system or system group level.

The storage group specifications determine only how DFSMShsm automated processing
processes the volume. They have no impact on how commands that are directed to the
volume or against data sets that are allocated on the volume are processed.


-----

**4.2.1 Space management**

Both primary space management and interval migration are determined by the storage group
that is assigned to the first extent of the data set. This determination occurs whether or not a
data set is eligible for processing by space management. If a data set extends over more than
one storage group (for example, a primary storage group and an extend storage group), the
settings of the storage group where the first extent of the data set resides are critical.

Two separate parameters determine whether DFSMShsm processes data from the volumes
in a storage group. Both of these parameters are set at the storage group level (not the
volume level).

The first parameter that you define is the migration attribute of a storage group with the “Auto
Migrate” attribute. The second attribute consists of a high and a low allocation threshold that
determines whether any particular volume is targeted by a particular run of space
management.

**Auto Migrate attribute**
The following options are available when you determine whether DFSMShsm space
management processes data:

- **N:** DFSMShsm will not process this volume during space management.

- **Y:** This setting is the default setting for a pool storage group. DFSMShsm
processes data sets from volumes in this storage group during primary
space management. Volumes are targeted by primary space
management until either the volume falls below its “MIGR LOW”
threshold or no more data sets that are eligible to be migrated are on
the volume.

  If SETSYS INTERVALMIGRATION was issued, volumes in a storage
group are also eligible for interval migration processing.

- **I:** Data sets on volumes in a storage group that are defined as AM= I are
eligible for processing by both primary space management and
interval migration regardless of the setting of SETSYS
INTERVALMIGRATION.

- **P:** Data sets are eligible for primary space management but not interval
space management regardless of the setting of SETSYS
INTERVALMIGRATION.

**Thresholds:**
Storage groups are assigned both a low and a high allocation threshold. Thresholds apply at
the volume level. Thresholds are specified as a percentage of the space in the storage group.

Thresholds are applied as a percentage of the space for each volume so where storage
groups consist of volumes of different capacities, thresholds are applied according to each
volume’s capacity.

A volume that is over its high allocation threshold can still be targeted for new allocations if no
other volume is available in the eligible storage groups with sufficient space to accommodate
the data set.

**_Low allocation threshold_**
The default low allocation threshold is 1 .


-----

During primary space management, data sets are processed until either the allocated space
on the volume drops below the storage group low allocation threshold or until DFSMShsm
can find no more data sets on the volume that are eligible for processing.

**_High allocation threshold_**
The default high allocation threshold is 85 . During interval migration, DFSMShsm moves
eligible data sets from each volume with allocation at or above the MIGR HIGH value until the
allocation reaches the MIGR LOW value. When Auto Migrate is I , migration is performed
when the space that is used exceeds the halfway mark between the HIGH and LOW
thresholds.

**_Migrate system/system group name_**
The default value is blank. By specifying a system or system group, you can restrict
DFSMShsm space management to only process volumes in this storage group on the system
or system group that is specified.

### 4.2.2 Availability management

When you define a pool storage group, you define settings for both automatic backup and
automatic dump processing.

**Automatic backup processing**
Enabling automatic backup processing for a storage group is a binary choice. By default, the
following Auto Backup attributes are enabled for pool storage groups:

- **N:** DFSMShsm availability management does not target data sets whose
primary extent is on volumes in this storage group during automatic
backup processing.

- **Y:** DFSMShsm availability management targets data sets whose primary
extent is on volumes in this storage group during automatic backup
processing.

You can also limit processing to a single system or system group by specifying a system or
system group name, which is similar to the configuration for space management.

**Automatic dump processing**
Although the automatic dump process is part of DFSMShsm availability management, you
can specify the Auto Dump attribute of a storage group independently of the Auto Backup
attribute.

**_Auto Dump attribute_**
Two settings are possible for the Auto Dump attribute: Y to enable automatic dump processing
and N to disable it. The default setting for the Auto Dump attribute for a pool storage group is **N**.

If you use overflow and extend storage groups for automatic dump processing, these groups
must be managed in the same way as the primary storage group. This approach ensures that
all data that was targeted to this storage group is managed in the same way.

**_Dump classes_**
Each pool storage group can be assigned 1 - 5 DFSMShsm dump classes. As with automatic
migration and automatic backup, you can also limit automatic dump processing of a storage
group to a specific system or system group.


-----

## 4.3 Tape storage group

_Tape storage groups_ are used to logically group automated or manual tape libraries. If you
plan to use SMS-managed tape and direct DFSMShsm tape allocations to an SMS-managed
library, you need to define at least one tape storage group.

If you plan to create off-site copies of DFSMShsm media, or to separate the location of
migrated data from backup or dump copies, more than one tape storage group might be
required. The only entries that are required in a tape storage group are 1 - 8 tape library
names. Figure 4-2 shows an example of defining a tape storage group.

    DGTDCSGA         TAPE STORAGE GROUP ALTER
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Group Name : SGLIB1
    
    To ALTER Storage Group, Specify:
    
    Description ==>
    ==>
    
    Library Names (1 to 8 characters each):
    ===> LIB1   ===>      ===>      ===>
    ===>      ===>      ===>      ===>
    
    ALTER   SMS Storage Group Status . . N (Y or N)
    
    Use ENTER to Perform Verification and Selection;

_Figure 4-2  Define tape storage group_

Tape libraries are defined by using ISMF Option 10, Library Management, where library
names are registered. For more information about defining tape libraries, see _z/OS DFSMS_
_OAM Planning, Installation, and Storage Administration Guide for Tape Libraries_ , SC35-0427.

## 4.4 Management class

The _management class_ that is assigned to a data set is the primary factor that determines how
DFSMShsm treats a data set. Only SMS-managed DASD data sets are assigned
management classes. Management classes define the following attributes for a data set:
- Expiration
- Partial release
- Migration
- Special processing for generation data sets (GDSs)
- Backup
- Aggregate backup and recovery support (ABARs)


-----

### 4.4.1 Expiration

Three attributes together determine whether a data set is eligible to be expired. These values
apply to DASD data sets based on the management class that is assigned to the data set.
These values can also influence the life of tape data sets. For DASD data sets, expiration
processing is performed as part of primary space management. For tape data sets, expiration
is performed by your tape management system.

The following attributes define whether a data set expires:
- Expire after days non-usage
- Expire after date/days
- Retention limit

The default value for each of these settings is NOLIMIT .

Figure 4-3 shows the first page of the Management Class Alter panel.

    DGTDCMC1         MANAGEMENT CLASS ALTER        Page 1 of 5
    Command ===>
    
    SCDS Name . . . . . . : SYS1.SMS.MHLRES3.SCDS
    Management Class Name : MC365
    
    To ALTER Management Class, Specify:
    
    Description ==>
    ==>
    
    Expiration Attributes
    Expire after Days Non-usage . . 365     (1 to 93000 or NOLIMIT)
    Expire after Date/Days . . . . . NOLIMIT   (0 to 93000, yyyy/mm/dd or
    NOLIMIT)
    
    Retention Limit . . . . . . . . . NOLIMIT   (0 to 93000 or NOLIMIT)
    
    Use ENTER to Perform Verification; Use DOWN Command to View next Panel;

_Figure 4-3  ISMF Management Class Alter panel_

DFSMShsm uses a combination of the three values to determine whether or when a DASD
data set is eligible for expiration.

**Expire after days non-usage**
This attribute determines how many days will elapse after the last use of the data set before it
is eligible to be expired. The data set’s last use is determined by the last reference date in the
volume table of contents (VTOC) (DS1REFD in the format1 or format8 data set control block
(DSCB)).

**Expire after date/days**
This attribute defines either the actual date on which a data set becomes eligible for
expiration, or the absolute number of days after its initial allocation that a data set becomes
eligible for expiration.


-----

**Retention limit**
If you specify a value for the retention limit attribute other than NOLIMIT , you can restrict the
values that users can specify for data sets at allocation time. A user-specified value for
retention period (RETPD) or expiration date (EXPDT) can be accepted, reduced, or removed
entirely. If NOLIMIT is specified for the retention period, any RETPD or EXPDT that is assigned
to a data set is accepted. If the user-specified value for RETPD or EXPDT is less than the
value that is specified in the retention limit, it is accepted and assigned to the data set. If the
retention limit is set to zero , any RETPD or EXPDT value that the user assigns to a data set is
overridden.

**Interaction between expiration and retention**
The following rules are followed for both tape and DASD data sets:

- If both expiration attributes are NOLIMIT , the data set never expires. The setting of RETPD
has no impact.

- If the retention period is zero, neither of the settings for expiration has any impact and the
data set will not expire.

- If the retention period is not zero and one of the expiration attributes is NOLIMIT and the
other is specified, the specified value is used and the data set expires.

- If the date that is determined by the expiration values is sooner than the retention period,
both expiration attributes must be satisfied. The data set expires on the latter of the dates
that are determined by the expiration attributes.

**Recommendations**
In general, we recommend not using expiration dates for DASD data sets and setting RETPD
to zero in management classes. This approach ensures that data sets are managed by
DFSMShsm while they are cataloged, and that any EXPDT or RETPD values that are coded
at data set allocation are ignored.

For DFSMShsm tape data sets, we recommend not specifying a retention period or expiration
date unless your tape management system treats this date as a non-expiration flag. This topic
is described more in Chapter 7, “Tape considerations” on page 117.

### 4.4.2 Partial release

This attribute determines whether DFSMShsm releases allocated but unused space for data
sets that are assigned to this management class. This attribute is only relevant to non-Virtual
Storage Access Method (VSAM) data sets and to extended format VSAM data sets.

The following values are valid for the partial release attribute:

- **N:** Partial release is not performed. N is the default value.

- **Y:** Space is released during primary space management.

- **C:** Partial release is performed only if a secondary allocation exists for the
data set.

- **YI:** Partial release is performed either during primary space management
or whenever the data set is closed.

- **CI;** Partial release is performed either during primary space management
or whenever the data set is closed, but only if a secondary allocation
exists for the data set.


-----

If you plan to use partial release, consider using the options that occur only if a secondary
allocation is present. If a data set with only a primary allocation is processed by partial
release, any attempt to later extend the data set fails.

### 4.4.3 Migration

Your management classes define how DFSMShsm targets data sets during space
management, determining what data sets are migrated and to which level they are migrated.
You can control DFSMShsm migration processing by using management class attributes.
Figure 4-4 shows the ISMF Management Class Alter panel where these attributes are
defined.

    DGTDCMC2         MANAGEMENT CLASS ALTER        Page 2 of 5
    Command ===>
    
    SCDS Name . . . . . . : SYS1.SMS.MHLRES3.SCDS
    Management Class Name : MC365
    
    To ALTER Management Class, Specify:
    
    Partial Release . . . . . . . . . YI     (Y, C, YI, CI or N)
    
    Migration Attributes
    Primary Days Non-usage . . . . 10     (0 to 9999 or blank)
    Level 1 Days Non-usage . . . . 30     (0 to 9999, NOLIMIT or blank)
    Command or Auto Migrate . . . . BOTH    (BOTH, COMMAND or NONE)
    
    GDG Management Attributes
     GDG Elements on Primary . . . 2      (0 to 255 or blank)
    Rolled-off GDS Action . . . . . EXPIRE   (MIGRATE, EXPIRE or blank)
    
    Use ENTER to Perform Verification; Use UP/DOWN Command to View other Panels;
    
_Figure 4-4  ISMF Management Class Alter panel_

**Primary days non-usage**
This attribute determines how many days after a data set is not accessed, based on
DS1LREFD, until a data set is eligible to be migrated. You can specify a value 0 -  9999 . A
value for this field is required if the Command or Auto Migrate attribute is set to Yes . The
default value is 2 .

**Level 1 days non-usage**
This setting determines whether data sets are migrated to DFSMShsm ML1 media, and if so,
how long they remain on level 1 media. The default value is 60 . The following values are valid:

**Zero** A 0 indicates no migration to DFSMShsm ML1. If migration occurs, data
sets are migrated directly to DFSMShsm ML2.

**A number of days 1 - 9999**
The total number of days since the last access to the data set before the
data set is eligible to be moved to ML2. This period includes any time that
is spent on primary DASD before the data set was moved to ML1.


-----

**NOLIMIT** Data sets will not move from ML1 to ML2 during automated DFSMShsm
processing. They remain on ML1. Data sets can be moved with storage
administrator commands, such as a **FREEVOL** command.

For more information about using the **FREEVOL** command, see “Converting
level 1 DASD volumes” on page 334.

**Command or Auto Migrate attribute**
This attribute determines whether data sets are eligible to be processed by DFSMShsm
automated functions or by command only. Commands can be generated either by storage
administrators or users unless migration is prohibited for this data set. The possible values
are BOTH , COMMAND , or NONE . If NONE is specified, data sets will not migrate and you can leave
the “primary days non-usage” and “level 1 days non-usage” fields blank.

### 4.4.4 GDG management

The following generation data group (GDG) management criteria apply only for GDS data
sets. They apply in addition to the normal migration attributes. You can specify the following
attributes in ISMF . Figure 4-4 on page 67 shows the ISMF Management Class Alter panel
where these options are defined.

**GDG Elements on Primary attribute**
This attribute determines the number of GDS data sets for the GDG base on the primary
DASD. Additional GDS data sets are targeted by DFSMShsm for migration. Possible values
are 0 -  255 , or blank . The default is blank . DFSMShsm targets older GDS data sets for
migration. Older generations are migrated even if they do not meet the normal migration
criteria for the management class.

**Rolled-off GDS Action attribute**
If a GDG is defined with the NOSCRATCH option, older GDS data sets will not be scratched
as they are rolled out of the GDG base. Rolled-off GDS action applies only to those data sets.
The following values are valid:

- **MIGRATE:** Rolled-off GDS data sets are targeted for migration by using the
management class Migration attributes.

- **EXPIRE:** Rolled-off GDS data sets are expired.

- **None:** No action is taken.

If a GDG is defined with the SCRATCH attribute, these settings have no meaning.

### 4.4.5 Backup attributes**

The following attributes of a management class are used by DFSMShsm backup processing:

- Backup Frequency
- Number of Backup Vers (Data Set Exists)
- Number of Backup Vers (Data Set Deleted)
- Retain days only Backup Ver (Data Set Deleted)
- Retain days extra Backup Vers
- Admin or User Command Backup
- Auto Backup
- Backup Copy Technique


-----

Figure 4-5 shows the portion of the ISMF Management Class Alter panel where these values
are specified.

    DGTDCMC3         MANAGEMENT CLASS ALTER        Page 3 of 5
    Command ===>
    
    SCDS Name . . . . . . : SYS1.SMS.MHLRES3.SCDS
    Management Class Name : MC365
    
    To ALTER Management Class, Specify:
    Backup Attributes
    Backup Frequency . . . . . . . . 1    (0 to 9999 or blank)
    Number of Backup Vers . . . . . . 10    (1 to 100 or blank)
    (Data Set Exists)
    Number of Backup Vers . . . . . . 2    (0 to 100 or blank)
    (Data Set Deleted)
    Retain days only Backup Ver . . . 60    (1 to 9999, NOLIMIT or blank)
    (Data Set Deleted)
    Retain days extra Backup Vers . . 20    (1 to 9999, NOLIMIT or blank)
    Admin or User command Backup . . BOTH   (BOTH, ADMIN or NONE)
    Auto Backup . . . . . . . . . . . Y    (Y or N)
    Backup Copy Technique . . . . . . S    (P, R, S, VP, VR, CP or CR)

_Figure 4-5  ISMF Management Class Alter panel_

**Backup Frequency attribute**
This attribute determines how many days can elapse before DFSMShsm backs up a changed
data set. The default value is 1 . To back up a changed data set every backup cycle, specify
the backup frequency as zero .

**Number of Backup Vers (Data Set Exists) attribute**
This attribute specifies the maximum number of backup copies for a data set that can exist
while the source data set exists. The default value is 2 .

**Number of Backup Vers (Data Set Deleted) attribute**
This attribute specifies the maximum number of backup copies that can be kept when
EXPIREBV processing determines that the source data set was deleted. For more
information about EXPIREBV processing, see 11.1.6, “Expiring backup versions” . The default value is 1 .

**Retain days only Backup Ver (Data Set Deleted) attribute**
This attribute determines how long DFSMShsm keeps the most recent backup version of a
data set when EXPIREBV processing determines that the original data set was deleted. If the
value that is specified for “retain days extra backup versions” is NOLIMIT , all backup versions
are kept for this period. The possible values for this field are a number of days ( 1 -  9999 ) or
NOLIMIT . If NOLIMIT is set, backups are kept until they are manually deleted. The default value
is 60 .


-----

**Retain days extra Backup Vers attribute**
This field applies to other than the most recent backup copy of a data set. It determines how
long to keep backup copies other than the most recent backup copy. The value is specified as
a number of days ( 1 -  9999 ). Or, NOLIMIT can be specified, in which case no backup copies are
expired until the number of backups exceeds the specified number of backup versions. The
default value is 30 .

If a value other than NOLIMIT is specified, fewer than the expected number of backup versions
for a fairly static data set are possible because older versions are expired and no new backup
is created if the data set is not changed.

**Admin or User command Backup attribute**
This setting determines whether users, DFSMShsm administrators, or both, can explicitly
issue data set level backups against the data sets. The following values are valid for this
setting:

- **NONE:** Data set backup commands that are issued against data sets that are
assigned to this management class are not processed.

- **ADMIN:** Data set backup commands that are issued against data sets by
DFSMShsm storage administrators are processed. Commands from
users are failed.

- **BOTH:** Both storage administrators and users can issue DFSMShsm backup
commands against data sets that are assigned to this management
class.

The default value is **BOTH** .

**Auto Backup attribute**
This parameter specifies whether or not DFSMShsm Autobackup processing targets data
sets with this management class. The possible values are Y or N . Y is the default value.

**Backup Copy Technique attribute**
This parameter works together with the storage class accessibility value and the physical
DASD where the data set exists. These values are used for both automatic and
user-generated backup copies. Nine values are valid:

- **R:** Concurrent Required. Concurrent Copy must be used for data set
backup. Backups fail if Concurrent Copy cannot be used.

- **P:** Concurrent Preferred. DFSMShsm first attempts to perform the
backup by using Concurrent Copy. If this attempt fails, it then selects a
non-Concurrent Copy technique to back up the data set.

- **S:** Standard. Data sets are backed up without using Concurrent Copy.

- **VP:** Virtual Preferred. Virtual Concurrent Copy is preferred. If virtual
Concurrent Copy is not available, standard copy techniques are used.

- **VR:** Virtual Required. Virtual Concurrent Copy is to be used. If virtual
Concurrent Copy fails, the data set backup is failed.

- **CP:** Cache Preferred. Data is processed with cache-based Concurrent
Copy if available; otherwise, standard copy techniques are used.

- **CR:** Cache Required. Data is processed with cache-based Concurrent
Copy if available; otherwise, the data set is not backed up.

The default value is Standard .


-----

### 4.4.6 ABARS attributes

The following management class attributes apply only if you plan to use the DFSMShsm
ABARs facility:

- \# Versions
- Retain Only Version
- Retain Extra Versions
- Copy Serialization
- Abackup Copy Technique

Figure 4-6 is a sample ISMF Management Class Alter panel with specified ABARS attributes.

    DGTDCMC7         MANAGEMENT CLASS ALTER        Page 5 of 5
    Command ===>
    
    SCDS Name . . . . . . : SYS1.SMS.MHLRES3.SCDS
    Management Class Name : MC365
    
    To ALTER Management Class, Specify:
    AGGREGATE Backup Attributes:
    \.# Versions . . . . . . . .     (1 to 9999, NOLIMIT or blank)
    Retain Only Version . . .     (1 to 9999, NOLIMIT or blank)
    Unit . . . . . . . . . .     (D=days, W=weeks, M=months, Y=years or
    blank)
    Retain Extra Versions . .     (1 to 9999, NOLIMIT or blank)
    Unit . . . . . . . . . .     (D=days, W=weeks, M=months, Y=years or
    blank)
    Copy Serialization . . . .     (C=continue, F=fail or blank)
    Abackup Copy Technique . . S    (P, R, S, VP, VR, CP or CR)
    
    Use ENTER to Perform Verification; Use UP Command to View previous Panel;

_Figure 4-6  ISMF Management Class Alter panel_

You can define the following attributes. These attributes apply to the aggregate group, not to
the data sets that are part of the group.

**# Versions attribute**
This attribute determines the number of versions of an aggregate group that DFSMShsm
retains. The field is not required. The specified value can be a number ( 1 -  9999 ) or NOLIMIT .
By default, no value is specified.

**Retain Only Version attribute**
This attribute defines how long the only version of an Abackup is retained. The value is a
number ( 1 -  9999 ) or NOLIMIT , which means that the most recent version of an aggregate is
retained for an unlimited time. The default value is a blank, which results in the most recent or
only version of an aggregate being expired. If you specify a numeric value, you can also
specify a unit: days, weeks, months, or years that are applied to the numeric value. If a
non-blank value is specified, “Retain Only Version, Unit” becomes a required field.


-----

**Retain Extra Versions attribute**
This attribute defines how long aggregate versions other than the most recent version are
retained. The value is a number ( 1 -  9999 ) or NOLIMIT , which means that the most recent
version of an aggregate is retained for an unlimited time. The default value is a blank, which
results in the most recent, or only version of an aggregate being expired. If you specify a
numeric value, you can also specify a unit: days, weeks, months, or years that are applied to
the numeric value. If a non-blank value is specified, “Retain Extra Versions, Unit” becomes a
required field.

**Copy Serialization attribute**
The following values can be specified:

- **C:** Continue. Allows aggregate backup to continue if a requested shared
enqueue cannot be obtained.

- **F:** Fail. Aggregate backup fails if a requested enqueue cannot be
obtained.

- **Blank:** Aggregate backup fails.

- **Abackup Copy Technique attribute:**
This parameter specifies the copy technique to use during ABACKUP processing. This value
is independent of the copy technique that is used for other DFSMShsm backups. Nine values
are valid:

- **R:** Concurrent Required. Concurrent Copy must be used for data set
backup. Backups fail if Concurrent Copy cannot be used.

- **P:** Concurrent Preferred. DFSMShsm first attempts to perform the
backup by using Concurrent Copy. If Concurrent Copy fails, it selects a
non-Concurrent Copy technique to back up the data set.

- **S:** Standard. Data sets are backed up without using Concurrent Copy.

- **VP:** Virtual Preferred. Virtual Concurrent Copy is preferred. If virtual
Concurrent Copy is not available, standard copy techniques are used.

- **VR:** Virtual Required. Virtual Concurrent Copy is to be used. If virtual
Concurrent Copy fails, the data set backup is failed.

- **CP:** Cache Preferred. Data is processed with cache-based Concurrent
Copy if available; otherwise, standard copy techniques are used.

- **CR:** Cache Required. Data is processed with cache-based Concurrent
Copy. If cache-based Concurrent Copy fails, the data set is not backed
up.

The default value is Standard .

## 4.5 Storage class

Assigning a storage class to a DASD data set is the trigger for a data set to be
SMS-managed. So, the act of assigning the storage class not only assigns the attributes of
the storage class to the data set, but also affects how DFSMShsm processes the data set.
When a data set is SMS-managed, the attributes that are assigned by the storage class,
management class, and storage group data sets determine how DFSMShsm processes the
data set rather than the DFSMShsm options that are established by using **SETSYS** commands.


-----

Storage classes also specify several attributes that can directly affect how DFSMShsm
processes a data set, for example:

- Guaranteed space
- Accessibility
- 
St:orage class attributes are defined in ISMF option 5. See Figure 4-7.

    DGTDCSC1           STORAGE CLASS ALTER       Page 1 of 2
    Command ===>
    
    SCDS Name . . . . . : SYS1.SMS.MHLRES3.SCDS
    Storage Class Name : GSSTD
    To ALTER Storage Class, Specify:
    Description ==> STORAGE CLASS FOR AVERAGE RESPONSE TIME
    ==>
    Performance Objectives
    Direct Millisecond Response . . . .       (1 to 999 or blank)
    Direct Bias . . . . . . . . . . . .       (R, W or blank)
    Sequential Millisecond Response . .       (1 to 999 or blank)
    Sequential Bias . . . . . . . . . .       (R, W or blank)
    Initial Access Response Seconds . .       (0 to 9999 or blank)
    Sustained Data Rate (MB/sec) . . .       (0 to 999 or blank)
    OAM Sublevel . . . . . . . . . . .       (1, 2 or blank)
    Availability . . . . . . . . . . . . S      (C, P ,S or N)
    Accessibility . . . . . . . . . . . S      (C, P ,S or N)
    Backup . . . . . . . . . . . . . .       (Y, N or Blank)
    Versioning . . . . . . . . . . . .       (Y, N or Blank)

_Figure 4-7  ISMF Storage Class Alter panel_

**Guaranteed space**
Guaranteed space allows hand placement of data sets. It also changes the way that primary
and secondary space allocations are distributed on multiple volume data sets. Guaranteed
space also permits pre-allocation of space for data sets. Primary space is allocated on each
volume. After primary space on a specific volume is used, as many secondary extents as
possible are added to the volume before space on a different volume is used.

If you are allocating data sets with the guaranteed space attribute, you might want to limit
DFSMShsm space management of this data.

**Accessibility**
This attribute defines whether Concurrent Copy can be used for backups of data sets. The
following values are valid:

- **C:** Continuous data sets must be allocated on volumes that support
Concurrent Copy.

- **C:P** Continuous preferred data sets are to be allocated on volumes that
support Concurrent Copy, if possible. If not, allocation is allowed on
other volumes.

- **S:** Volumes that do not support Concurrent Copy are preferred for
allocation.

- **N:** No preference. Concurrent Copy capability is not considered during
allocation.


-----

## 4.6 Values for SMS classes and groups

We do not describe absolute values in this section because requirements vary depending on
the system. We describe several factors that you need to consider when you determine how
much data to migrate and how many data set backups to take.

Several of these values need to be determined with the owners of the data that you are
managing. Certain values are based on how the cost of data on both primary DASD and
migrated or backed up data is passed back to users.

In a perfect world, data does not need to be migrated and an almost infinite number of
backups can be kept. In the real world, the trick is to minimize the impact to users who have to
wait while data sets are recalled while also minimizing the work that DFSMShsm performs.
The number of backups created is a trade-off between users wanting to keep many backups
and the space and cost of maintaining backups.

You might also be able to identify classes of data where backups or dumps taken by using
DFSMShsm availability management might not be the best way to ensure that data is
recoverable. For example, almost any database where multiple data sets are required to be
backed up at a specific point in time are in this class.

If you run a multisite environment or if your disaster recovery depends on a second site, you
need to consider whether both DFSMShsm migrated and backup data needs to be replicated
to all sites.

### 4.6.1 Naming classes and groups

Where possible, we recommend meaningful names for classes and groups. Names are
restricted to eight alphanumeric characters with the first character alphabetic. Consider that
names for data classes, storage classes, and management classes can be specified by
users. Therefore, select names that are meaningful to them. You are not required to indicate
the type of class in the name of the class. Also, management, storage, and data class names
can be identical.

**SMS class names**
Because users see the names that are assigned to their data sets, choose names that tell the
user how their data will look or will be managed. Also, try to choose names that do not
disparage the service that is provided. No one wants to see a class that is called “slow”
assigned to their data.

If you allow users to hardcode class names in their JCL, the suggested data class names
include the following names:

- **FB80S:** For RECFM=FB, LRECL=80 DSORG=PS

- **VB255PE:** For RECFM=VB, LRECL=255, DSORG=PO, DSNTYPE=LIBRARY

- **KS:** For a key-sequenced data set (KSDS) with no other specified
attributes

- **TAPE3592:** For tape data sets

You can also create certain data classes for specific requirements, for example:

- **LOGGER:** To ensure that IXGLOGR data sets get the required share options

- **DEFAULT:** To ensure that all allocated data sets get a complete data set control
block (DSCB) and can therefore be managed by DFSMShsm


-----

For storage classes, use names that describe the service that is provided, for example:

- **STANDARD:** A class for most data sets

- **GSSTD:** The same as standard but with the guaranteed space attribute

- **CDSRLS:** Used for record-level sharing (RLS)

- **NONSMS:** As a flag in the ACS routines to ensure that data is not SMS-managed

- **SMSTAPE:** To direct tape data sets to SMS

For management classes, also use names that describe how the data set is managed, for example:

- **STANDARD:** Most data sets

- **NOMIG:** Data sets that must not be processed by DFSMShsm space
management

- **NOMGMT:** Data sets that must not be processed at all by DFSMShsm

- **PRODGDS:** Production GDS

**Storage group names**
Unlike SMS classes, users cannot directly specify the storage group name in the JCL, and
storage group names are not externalized in allocation messages, for example, IGD101I.
Therefore, storage group names must be chosen so that they are the most meaningful to
storage administrators. Usually, storage group names are selected so they reflect the pool of
DASD that is maintained in the storage group, for example:

- **PRIMARY:** The primary storage group

- **TAPE3592:** A tape storage group

### 4.6.2 How many classes and groups

The number of classes and groups depends on the interaction between the different types of
data that you might need to manage and the different DFSMShsm management criteria that
you use for that data.

We recommend that you define as few classes and groups as possible, by using criteria for
most data, and managing the rest by exception. SMS makes this process easier by enabling
data with different management criteria to reside on the same volume. For example, data sets
that must not be migrated can reside on volumes that are processed by DFSMShsm space
management, that is, if the correct management class is assigned to them.

Depending on the number of DASD volumes that are managed, separating data into separate
storage groups that are based on the data’s management requirements might be beneficial.
For example, separate a group of volumes that are not processed by availability
management. However, you then need to ensure that no data sets are allocated on volumes
in this storage group with management classes where backup is expected.

## 4.7 Automatic class selection

In addition to updating your automatic class selection (ACS) routines to ensure that the
correct classes and groups are assigned to user data, you also need to ensure that
DFSMShsm-owned data is managed correctly. For a description of the data sets that
constitute DFSMShsm data, see 3.1, “DFSMShsm components” on page 30.


-----

### 4.7.1 Coding for data that is owned by DFSMShsm and control data

Most data that is managed by DFSMShsm has names in fixed formats. These names can be
used as filters in ACS routines to control the allocation of data sets.

The following filters are a starting point. The filters assume that the default high-level qualifier
(HLQ) of hierarchical storage management (HSM) is used to manage DFSMShsm data sets.
For more information and examples, see the _DFSMShsm Implementation and Customization_
_Guide_ , SC35-0418.

The first group of data sets is data that must be directed to non-SMS DASD. This data
includes DFSMShsm backup and migration copies, VTOC copies, DUMP VTOC copies, and
small-data-set packing (SDSP) data sets. See Example 4-1.

_Example 4-1  Sample ACS filter list for non-SMS data_

    FILTLIST &HSM_NONSMS_DATA_DSN
    INCLUDE(HSM.SMALLDS.**,HSM.BACK.**,HSM.HMIG.**,HSM.MDB.**,
    HSM.VCAT.**,HSM.VTOC.**,HSM.DUMPVTOC.**)

The next set of data sets consists of the HSM CDSs. These CDSs must be DASD resident but
can be SMS-managed or non-SMS-managed. See Example 4-2.

_Example 4-2  Sample ACS filter for DFSMShsm CDSs_

    FILTLIST &HSM_CONTROL_DATA INCLUDE(HSM.%CDS.*, HSM.JRNL.*
    INCLUDE(HSM.%CDS.*,HSM.JRNL.*)

HSM logs are next. HSM logs include both the ARCLOG data sets and the HSM problem
determination aid (PDA) data sets, and potentially data that is offloaded from these data sets.
Again, these data sets can be SMS-managed or non-SMS-managed. If you decide that these
HSM logs are not managed by SMS, they can be written to DFSMShsm ML1 volumes. Use
the following sample ACS filter for DFSMShsm log data sets:

    FILTLIST &HSM_LOG_DATA INCLUDE(HSM.EDITLOG?, HSM.HSMLOG*,HSM.HSMPDO.*)

If you are writing HSM activity logs to DASD by using SETSYS ACTLOGTYPE(DASD) , the following
filter manages the activity log output data sets. If these data sets are used, they can be either
managed by SMS or not. See Example 4-3.

_Example 4-3  Sample ACS filter for DFSMShsm activity logs_

    FILTLIST &HSM_ACT_DATA_DSN INCLUDE
    (HSMACT.H%.*.D*.T*)

The next filter is for CDS backups. See Example 4-4.

_Example 4-4  Sample ACS filter for DFSMShsm CDS backup copies_

    FILTLIST &HSM_BKCONT1_DATA_DSN INCLUDE
    (HSM.%CDS.BACKUP.*,
    HSM.JRNL.BACKUP.*)


-----

The last filter is for ABARs control files. See Example 4-5.

_Example 4-5  Sample ACS filter for DFSMShsm ABARS CDSs_

    FILTLIST &HSM_ABARS_DATA_DSN INCLUDE
    (ABARS.*.INSTRUCT)

### 4.7.2 ACS execution for data that is managed by DFSMShsm

Coding is reviewed for user data, which is the data that DFSMShsm manages.

**ACS environments**
Execution of ACS routines in both the order that the routines are executed and the variables
that are available for evaluation uses several environments. For a DFSMShsm environment,
three ACS environments need to be considered. Other values for _&ACSENVIR_ exist and are
described in _z/OS DFSMS Storage Administration Reference,_ SC35-7402. Each of these
values is returned as a different value for the _&ACEENVIR_ ACS variable. The values are
shown:

- **ALLOC:** This value is the environment that is present when a data set is first
allocated.

- **RECALL:** This environment is present when a data set is being recalled by
DFSMShsm.

- **RECOVER:** This value exists when a data set is being recovered by DFSMShsm. If
DFSMSdss is the data mover, ACSENVIR= RECOVER is passed to the
storage group ACS routine for a data set recall.

For initial allocations, all of the data class ACS routine is executed, followed by the storage
class routine. If no storage class is assigned, no further ACS routines are executed and the
data set becomes non-SMS. If a storage class is assigned, the management class ACS
routine is executed, followed by the storage group ACS routine. The management class ACS
routine might assign a management class to a data set, but the storage group ACS routine
must assign a storage group to a data set or the data set allocation fails. The class values that
are set in previous ACS routines are available as read-only variables in subsequent routines.

When _ACSENVIR_ is RECOVER , the storage class ACS routine is executed first, followed by the
management class, and then the storage group routines. The data class ACS routine is not
executed.

We do not recommend adding special code for changing ACS execution that is based on
_ACSENVIR_ . Instead, we recommend that you redrive ACS routines and allow ACS constructs
to be redetermined regardless of the ACS environment. This way, even if classes change over
time if a data set is recalled or recovered, the data set is treated according to your current
policies.

**ACS design**
Designing your ACS routines is beyond the scope of this book. However, if you have a robust
data set naming standard, use it to provide a framework for the decisions in your ACS
routines. You can use your data set naming standard as the basis of ACS FILTERLISTs in
your ACS routines.

If you have no data set naming standard or an installation with many naming standards, it
might be easier to assign default values to profiles by using RACF or an equivalent function
and enable these defaults to be used by setting PARMLIB IGDSMSxx ACSDEFAULTS( YES ).


-----

This method requires additional administration because new RACF profiles need to be
provided with an SMS segment with the appropriate STORCLAS and MGMTCLAS values.
You use the values that are passed in DEF_MGMTCLAS and DEF_STORCLAS as input to
the ACS routines. However, because no default storage group is assigned, you cannot use
the assigned values in the storage group ACS routine.

**Testing changes**
Whenever you change your ACS environment, you must test the changes before you activate
them. ISMF provides options to both generate and run test cases. For more information, see
_z/OS DFSMS Storage Administration Reference_ , SC35-7402, and _Maintaining Your SMS_
_Environment_ , SG24-5484.

## 4.8 SMS and non-SMS configuration considerations

You might not define copypool storage groups for your installation and therefore none are
eligible for processing. When DFSMShsm does not find any copypools, message ARC0570I
is issued with return code 36, which indicates that the information is unavailable. It appears as
though something is wrong with the configuration when nothing is wrong in the environment.

If you are not using copypool storage groups, you can use the **PATCH** command, which will
stop this message from being issued. The following **PATCH** command turns message
ARC0570I with return code 36 off:
    
F hsm,PATCH .MCVT.+297 BITS(.....1..) VERIFY(.MCVT.+297 BITS(.....0..))

Also, you might not define any storage groups that are eligible for processing. In this case,
DFSMShsm issues an ARC0570I RC17 message, which can appear to be a problem when it
is not. If you do not want to receive the ARC0570I RC17 message, use the following **PATCH**
command:

F hsm,PATCH .MCVT.+297 BITS(....1...) VERIFY(.MCVT.+297 BITS(....0...))

You can reverse the suppression of these messages by using the following patches, as shown
in Example 4-6.

_Example 4-6  TURN ARC0570I MESSAGES ON_

    F hsm,PATCH .MCVT.+297 BITS(.....0..) VERIFY(.MCVT.+297 BITS(.....1..))
    F hsm,PATCH .MCVT.+297 BITS(....0...) VERIFY(.MCVT.+297 BITS(....1...))


-----

