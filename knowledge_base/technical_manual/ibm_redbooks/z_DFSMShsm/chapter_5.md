
# 5. Implementing DFSMShsm security

The purpose of the implementation process is to select and implement options so that the
resultant customized product can be used effectively. It is important to consider the
modifications that must be applied to the existing system (for example, Resource Access
Control Facility (RACF) and Data Facility Storage Management Subsystem (DFSMS)) as
prerequisites before you start any implementation activity.


-----

## 5.1 Providing security for DFSMShsm resources

RACF can be used to provide security for your DFSMShsm environment. You define
DFSMShsm resources to RACF for use during authorization checking. RACF controls the
users that can issue DFSMShsm commands and access DFSMShsm data sets and
DFSMShsm-owned data sets. DFSMShsm requests RACF authorization before it allows
access to data sets by non-DFSMShsm authorized users.

In this section, we explain how to define the DFSMShsm started task to RACF, create RACF
profiles to protect DFSMShsm resources, and authorize access to those resources.

### 5.1.1 Defining a DFSMShsm RACF user ID

For a started procedure to access any of the system resources, a user ID must be associated
with that task. The user ID must have all of the correct authorizations for accessing the
system resources. Therefore, it must be assigned the RACF OPERATIONS attribute. We
recommend defining a new user ID for each started procedure rather than using the default
started task user ID (for example, STCUSER). In our system, we associated user ID _HSM1_
with the DFSMShsm started procedure, and user ID _ABR1_ with the ABARS started
procedure. We used the following RACF commands to define the DFSMShsm user IDs:
- ADDUSER hsm1 OPERATIONS DFLTGRP(SYS1) NAME('ITSO DFSMShsm Userid')
- ADDUSER abr1 OPERATIONS DFLTGRP(SYS1) NAME('ITSO ABARS Userid')

**Note:** If you do not specify the **DFLTGRP** parameters on the **ADDUSER** command, the
current connect group of the user that is issuing the **ADDUSER** command is used as the
default.

You can also create RACF user IDs by submitting a batch job by using the JCL that is shown
in Figure 5-1.

    //RACFADDU JOB ,'ADD STC RACF USERS',
    //     NOTIFY=&SYSUID,
    //     MSGCLASS=X,
    //     CLASS=A
    //*****************************************************************
    //*** THIS JOB WILL CREATE THE DFSMSHSM AND ABARS STC USER IDS **
    //*****************************************************************
    //S0000010 EXEC PGM=IKJEFT01
    //SYSTSPRT DD SYSOUT=*
    //SYSTSIN DD *
    ADDUSER HSM1 DFLTGRP(SYS1) NAME('DFSMSHSM STC USERID') OPERATIONS
    ADDUSER ABR1 DFLTGRP(SYS1) NAME('ABARS STC USERID') OPERATIONS
    /*

_Figure 5-1  Sample JCL to create a RACF user ID_


-----

### 5.1.2 Identifying started procedures to RACF

Before RACF 2.1, the only way to associate a started procedure with a RACF user ID was by
coding the RACF started procedures table, ICHRIN03. With RACF 2.1, assigning RACF
identities to started procedures is greatly simplified by the introduction of the RACF STARTED
class. You can add or modify security definitions for new and existing started procedures by
issuing the **RDEFINE** and **RALTER** commands.

Even though the use of the RACF STARTED class is the preferred way of identifying started
procedures to RACF, it is still mandatory to have the ICHRIN03 module. RACF cannot be
initialized if ICHRIN03 is not present in the system. A dummy ICHRIN03 is shipped and
installed with RACF.

**RACF STARTED class**
Use the STARTED class to assign RACF identities to started procedures dynamically by
using the **RDEFINE** and **RALTER** commands. Resource names in the STARTED class have the
format _membername.jobname_ . You assign identities, such as the RACF user ID and group ID,
by using fields in the **STDATA** segment. You can define the started procedure resource by
using either a generic profile name or a discrete profile name. A _RACF generic profile_
describes one or more data sets with a similar name structure. A _RACF discrete profile_
describes a specific data set on a specific volume. In our system, we created generic profiles
for started procedures. Issue the following RACF commands to assign RACF identities to the
DFSMShsm and ABARS started procedures:

- RDEFINE STARTED (HSM1.*) UACC(NONE) STDATA(USER(HSM54) GROUP(SYS1))
- RDEFINE STARTED (ABR1.*) UACC(NONE) STDATA(USER(ABR54) GROUP(SYS1))
- SETROPTS RACLIST(STARTED) REFRESH
- SETROPTS GENERIC(STARTED) REFRESH

**Note:** After you add profiles to the RACF STARTED class, refresh the in-storage
profiles by using the **SETROPTS REFRESH** command. The **SETROPTS GENERIC** command is
needed only when you define generic profiles.

**Listing the STARTED class profiles**
To verify the STARTED class definitions, use the **RLIST** command:

- RLIST STARTED (HSM1.*)
- RLIST STARTED (ABR1.*)

Figure 5-2 shows the output of the first command.


-----

    READY
    rlist started (hsm1.*)
    CLASS NAME
    ----- ---STARTED HSM1.* (G)
    LEVEL OWNER UNIVERSAL ACCESS YOUR ACCESS WARNING
    ----- -------- ---------------- ----------- ------00 MHLRES3 NONE NONE NO
    INSTALLATION DATA
    ----------------NONE
    APPLICATION DATA
    ---------------NONE
    AUDITING
    -------FAILURES(READ)
    NOTIFY
    -----NO USER TO BE NOTIFIED

_Figure 5-2  Output of the RLIST command_

**ICHRIN03 started procedures table**
We do not recommend that you use the RACF started procedures table (ICHRIN03). The
preferred way of adding started procedure users to the RACF database is by using the
STARTED class.

The sample JCL in Figure 5-3 shows how to identify the DFSMShsm and
aggregate backup and recovery support (ABARS) started procedures to RACF by
assembling ICHRIN03. You can enter a generic profile name for the started task table with an
asterisk (*), this approach has limited flexibility. These limitations might affect how useful it
can be in certain installations. For more information, see the _z/OS Security Server RACF_
_System Programmer’s Guide_ , SA22-7681. Started class profiles provide the capability to
define specific and appropriate authority flexibly and dynamically.


-----

    /ICHRIN03 JOB 'STARTED PROC TAB',MSGLEVEL=(1,1),REGION=4096K
    //*******************************************************************//
    //* ICHRIN03:                            *//
    //*                                 *//
    //*  RACF INSTALLATION PROCEDURE STEP: CREATE/UPDATE      @03C*//
    //*  STARTED PROCEDURES TABLE.                 @P2C*//
    //*                                 *//
    //* NOTE: Please read the section titled "Coding the Started    *//
    //*    Procedures Module" in the System Programming       *//
    //*    Library: RACF. It is important that you are familiar   *//
    //*    with this section before attempting to use or modify   *//
    //*    this sample.                       *//
    //*                                 *//
    //*******************************************************************//
    //STEP1   EXEC HLASMCL PARM.L=(RENT,XREF,LIST,LET,NCAL)
    //*                              @04C
    //C.SYSIN DD *
    ICHRIN03 CSECT
    TITLE 'ICHRIN03 - STARTED PROCEDURES TABLE'
    EJECT
    DC  XL2'80xx'     CHANGE 'xx' TO THE HEX VALUE FOR
    THE NUMBER OF ENTRIES IN THE TABLE
    
    - 
    DC  CL8'HSM1  '   PROCNAME - SPECIFY YOUR DFSMShsm
    DC  CL8'HSM1  '   STARTED TASK USER ID
    DC  CL8'SYS1  '   GROUP
    DC  XL1'00'      NOT PRIVELEDGED OR TRUSTED
    DC  XL7'00'      RESERVED
    
    - 
    DC  CL8'ABR1  '   PROCNAME - SPECIFY YOUR ABARS
    DC  CL8'ABR1  '   STARTED TASK USER ID
    DC  CL8'SYS1  '   GROUP
    DC  XL1'00'      NOT PRIVELEDGED OR TRUSTED
    DC  XL7'00'      RESERVED
    
    - 
    .          .
    .          .
    .          .
    .          .
    END
    /*
    //L.SYSLMOD DD DSN=SYS1.LPALIB,       ** MUST BE LIBRARY WITH**
    //       DISP=SHR,UNIT=YYYY,     ** NEW RELEASE OF RACF **
    //       VOL=SER=XXXXXX
    //L.SYSIN  DD *
    ENTRY ICHRIN03
    NAME ICHRIN03(R)
    /*
    //*

_Figure 5-3  ICHRIN03 sample source_


-----

### 5.1.3 Protecting DFSMShsm data sets

You must protect DFSMShsm resources, such as the control data sets (CDSs), journals, logs,
and backed up data sets from unauthorized access. In member STARTER of data set
HSM.SAMPLE.CNTL, you specify a high-level qualifier (HLQ) (parameter **UID** ) for the journal,
control, and log data sets. You can define a generic RACF data set profile to protect the
DFSMShsm data sets. In our system, we chose an HLQ of hierarchical storage management
(HSM). The following RACF command defines a generic data set profile with a universal
access of NONE :

ADDSD 'HSM.**' UACC(NONE)

After you create the RACF generic profile for protecting the DFSMShsm data sets, you must
permit users or groups access to the RACF profile based on their requirements. For example,
your storage administrator needs RACF ALTER access to the CDSs to move them to another
volume or to increase their space allocations.

You can give user ID HSMSTOR ALTER access to the DFSMShsm CDSs. You can use a
different qualifier for CDSs, such as HSMCDS, so that you can provide access to only those
data sets with the following command:

PERMIT 'HSMCDS.**' ID(HSMSTOR) ACC(ALTER)

**Listing the generic data set profile**
To verify the DFSMShsm generic data set profile, use the **LISTDSD** command:

LISTDSD DA('hsm.**') GENERIC

Figure 5-4 shows the output of the command.

    READY
    INFORMATION FOR DATASET HSM.** (G)
    LEVEL OWNER UNIVERSAL ACCESS WARNING ERASE
    ----- -------- ---------------- ------- ----00 SYS1 NONE NO NO
    AUDITING
    -------FAILURES(READ)
    NOTIFY
    -------NO USER TO BE NOTIFIED
    YOUR ACCESS CREATION GROUP DATASET TYPE
    ----------- -------------- -----------ALTER SYS1 NON-VSAM
    NO INSTALLATION DATA

_Figure 5-4  LISTDSD command output_

**Protecting DFSMShsm activity logs**
DFSMShsm writes its activity logs to direct access storage device (DASD) if you specify
SETSYS ACTLOGTYPE(DASD) in the ARCCMDxx member of PARMLIB. DFSMShsm
allocates the activity logs with an HLQ of HSMACT. The following RACF command defines a
generic data set profile to protect the activity logs with a universal access of NONE :

ADDSD 'HSMACT.**' UACC(NONE)


-----

After you create the RACF generic profile for protecting the DFSMShsm activity logs, you
must permit users or groups access to the RACF profile based on their requirements.
Consider that someone might use a batch job to access data in the activity logs.

The following RACF command can be used to give user ID HSMUSR1 READ access to the
activity logs:

PERMIT 'HSMACT.**' ID(HSMUSR1) UACC(READ)

**Protecting DFSMShsm tapes**
To protect DFSMShsm-managed tapes with RACF-protected data sets on them, follow these
steps:

1. Install and activate RACF.

2. Define the tapes that you want to protect to RACF by defining the TAPEVOL resource
class in the RACF class descriptor table.

3. Specify the **SETSYS TAPESECURITY(RACF)** command.

You define the RACF environment to DFSMShsm when you specify the **SETSYS**
**TAPESECURITY(RACF)** command. DFSMShsm protects each backup, migration, and dump tape
with RACF.

The way that you define your RACF TAPEVOL resource class is determined by the number of
tapes you want to protect.

**_Protecting up to 10,000 tapes_**
If you are protecting up to a maximum of 10,000 tapes, you define two RACF resource names
in the TAPEVOL resource class:

- HSMABR is the name for aggregate backup and recovery tapes.
- HSMHSM is the name for all other DFSMShsm tapes.

Issue the following RACF commands:

- RDEFINE TAPEVOL HSMABR
- RDEFINE TAPEVOL HSMHSM

You can add tapes to RACF before DFSMShsm uses them. If you choose this approach, you
must use the appropriate TAPEVOL resource class. Use the following command:

RALTER TAPEVOL HSMHSM ADDVOL( _volser_ )

IBM suggests that clients _not add tapes to RACF before DFSMShsm uses them_ . Instead, let
DFSMShsm add to the TAPEVOL for you automatically as tapes are encountered.

**_Protecting more than 10,000 tapes_**
To RACF-protect more than 10,000 tapes, you define multiple RACF resource names for
DFSMShsm tape volume sets in the TAPEVOL resource class. Use the following resource
names:

- HSMHSM (must be defined)
- HSMABR for ABARS tapes
- DFHSM _x_

In DFHSM _x_ , _x_ is a non-blank character (alphanumeric, national, or the hyphen) that
corresponds to the last non-blank character of the tape volume serial number. You need to
define a DFHSM _x_ resource name for each _x_ value that exists, based on your installation
naming standards.


-----

The following RACF commands add resource names to the TAPEVOL class for HSMHSM
(required), HSMABR (for aggregate backup and recovery tapes), and DFHSMA (for all tapes
with a volume serial number that ends with the letter A):

- RDEFINE TAPEVOL HSMHSM
- RDEFINE TAPEVOL HSMABR
- RDEFINE TAPEVOL DFHSMA

To activate the RACF protection of tape volumes by using the DFHSMx resource names that
are defined, you must issue the following RACF command on each system in the sysplex:

RALTER TAPEVOL HSMHSM ADDVOL(DFHSM)

You can add RACF protection to the DFSMShsm tape volumes before DFSMShsm uses
them, except for the HSMABR tapes. You must add the tape volume serial number to the
appropriate DFHSMx tape volume set, which is based on the last non-blank character of the
tape volume serial number. To protect a tape with a volume serial of POK33H , you use the
following RACF command:

RALTER TAPEVOL DFHSMH ADDVOL(POK33H)

Tapes that are already protected in the tape volume set of HSMHSM continue to be protected.

### 5.1.4 Controlling the use of DFSMShsm operator-entered commands

No RACF authorization checking is performed for DFSMShsm commands that are entered at
the system console if LOGON(OPTIONAL) is specified for consoles in the CONSOLxx
PARMLIB member. In this case, the authority of the AUTH attribute is used. Consoles with
master authority can enter any command.

To protect DFSMShsm commands that are entered from a console, two levels of authorization
processing are required. When commands are entered at the console, they are entered in the
form of a **MODIFY** command against the DFSMShsm task or job name. The console address
space can check whether the operator has authority to enter this command against an
OPERCMDS profile. However, this check requires that the console has a RACF user ID that
is associated with it. In CONSOLxx member, LOGON(AUTO) or LOGON(REQUIRED) must
be specified.

For the first check, the RACF OPERCMDS class is used by the console address space
COMTASK to verify that the user ID of the console has authority to issue the **MODIFY**
command against DFSMShsm. The second check is then performed by DFSMShsm to see
whether the user ID that is passed is authorized for the relevant FACILITY class profile for the
command. Alternatively, this second check can be performed by using the AUTH table
constructed from the **AUTH** commands in the ARCCMDxx member, although this approach is
not recommended. RACF is preferred.

**Associating a user ID with a console**
You can use a RACF profile in the CONSOLE class to determine which user IDs are
authorized to log on to a particular console. The commands in Example 5-1
define a RACF profile for console CON1 (CON1 is defined in CONSOLxx), and authorize user
ID CONSID1 to log on to that console.


-----

_Example 5-1  Defining consoles and authorizing users_

    RDEF CONSOLE CON1 UACC(NONE)
    PERMIT CON1 CLASS(CONSOLE) ID(CONSID1) ACCESS(READ)
    PERMIT CON1 CLASS(CONSOLE) ID(OPERGRP) ACCESS(READ)
    SETROPTS CLASSACT(CONSOLE)

**OPERCMDS profile for the MODIFY command**
Table 5-1 lists the profile names to use to authorize the **MVS** **MODIFY** command, depending on
whether the target is a job, started task, or user. DFSMShsm can be run as a job or a started
task. The first row is for a job. The second row is for a Time Sharing Option (TSO) user (not
relevant in this instance for DFSMShsm). The last two rows are for a started task, depending
on whether it is started with an identifier, for example, START DFHSM.HSM , or just START DFHSM .

_Table 5-1  OPERCMDS MODIFY command resource names_


|Command|Authority|Resource|
|---|---|---|
|MODIFY jobname|UPDATE|MVS.MODIFY.JOB.jobname|
|MODIFY userid|UPDATE|MVS.MODIFY.JOB.userid|
|MODIFY jobname MODIFY jobname.id MODIFY id|UPDATE|MVS.MODIFY.STC.member.id|
|MODIFY jobname|UPDATE|MVS.MODIFY.STC.member. jobname|


For the example that is shown in Example 5-2, assume that DFSMShsm is run as a started
task named _DFHSM_ , and its JCL is in a member called _DFHSM_ . Our console, CON1, is
defined with LOGON(AUTO) and so is given authority to issue the **MODIFY** command against
DFHSM. In addition, an operator with the user ID of CONSID1 and members of the
OPERGRP are also permitted access.

_Example 5-2  Creating profiles to allow the DFSMShsm MODIFY command_

    SETROPTS GENERIC(OPERCMDS) REFRESH
    REDFINE OPERCMDS MVS.MODIFY.STC.DFHSM.DFHSM UACC(NONE)
    PERMIT MVS.MODIFY.STC.DFHSM.DFHSM CLASS(OPERCMDS) ID(CON1) ACCESS(UPDATE)
    PERMIT MVS.MODIFY.STC.DFHSM.DFHSM CLASS(OPERCMDS) ID(OPERGRP) ACCESS(UPDATE)
    PERMIT MVS.MODIFY.STC.DFHSM.DFHSM CLASS(OPERCMDS) ID(CONSID1) ACCESS(UPDATE)
    SETROPTS CLASSACT(OPERCMDS)
    SETROPTS RACLIST(OPERCMDS) REFRESH
    SETROPTS GENERIC(OPERCMDS) REFRESH

**When the OPERCMDS class is activated**
If RACF is active, MVS uses the z/OS CMDAUTH function, that is, OPERCMDS profiles, to
verify that the operator console that is used to issue the **DFSMShsm** command is authorized to
issue that **DFSMShsm** command.

If RACF is not active at DFSMShsm startup, MVS does not verify any commands that are
issued from an operator console. In this case, authorization falls back to the user IDs that are
given authority by using the **AUTH** command in ARCCMDxx.

If you stop RACF while DFSMShsm is active, MVS fails all DFSMShsm commands that are
issued from an operator console.


-----

### 5.1.5 Authorizing and protecting commands by using AUTH

We recommend that you use RACF to protect DFSMShsm commands and resources.
However, we include information about the DFSMShsm **AUTH** command for completeness. If
DFSMShsm is initialized before RACF, or RACF or another external security software product
is not present, authorities that are granted by the **AUTH** command in ARCCMDxx will be used.

The **AUTH** command is used to identify users who can issue authorized DFSMShsm
commands and users who can issue DFSMShsm-authorized commands. The **AUTH** command
is also used to add, delete, and change the authority of other DFSMShsm users. The
DFSMShsm storage administrator must be identified as the user who can change the
authority of other DFSMShsm users. A user that is defined as authorized through the **AUTH**
command can issue DFSMShsm commands, bypassing RACF authorization checking.

The **AUTH** command is specified in the ARCCMDxx member of PARMLIB during DFSMShsm
startup or entered by DFSMShsm users who have the database authority control attribute.

**CONTROL authority**
The following command allows user ID ITSOHSM to add, delete, or change the DFSMShsm
authorization of other users. This command can be placed in the ARCCMDxx member of
PARMLIB or issued by a user with the database control attribute. The following **AUTH**
command grants CONTROL authority:

AUTH ITSOHSM DATABASEAUTHORITY(CONTROL)

**USER authority**
The following command allows user ID ITSOHSM to issue authorized commands except for
the **AUTH** command. This command can be placed in the ARCCMDxx member of PARMLIB or
issued by a user with the database control attribute. This **AUTH** command grants USER
authority:

AUTH ITSOHSM DATABASEAUTHORITY(USER)

**Revoking authority**
The command to revoke the authority of a DFSMShsm authorized user can be issued by a
user with the database control attribute or placed in the ARCCMDxx member of PARMLIB.
Use the following **AUTH** command to revoke the DFSMShsm authority of user ITSOHSM:

AUTH ITSOHSM REVOKE

### 5.1.6 Authorizing and protecting commands by using RACF

DFSMShsm uses the following set of the RACF FACILITY class profiles to protect
commands:

- STGADMIN.ARC.command : The DFSMShsm storage administrator command protection is a
profile for a specific DFSMShsm storage administrator command.

- STGADMIN.ARC.command.parameter : This profile protects a specific DFSMShsm
administrator command with a specific parameter.

- STGADMIN.ARC.ENDUSER.h_command : This profile is for DFSMShsm user command
protection. This profile protects a specific DFSMShsm user command.

- STGADMIN.ARC.ENDUSER.h_command.parameter : This profile protects a specific DFSMShsm
user command with a specific parameter.


-----

For a complete list of commands and parameters that you can protect, see “Protecting
DFSMShsm storage administrator commands with RACF FACILITY class profiles” in the
_z/OS DFSMShsm Implementation and Customization Guide_ , SC35-0418.

The RACF commands in Example 5-3 show how to grant broad access to the storage
administrators group to issue all commands by using a generic profile.

_Example 5-3  Granting access to all commands_

    SETROPTS CLASSACT(FACILITY)
    SETROPTS GENERIC(FACILITY)
    SETROPTS RACLIST(FACILITY)
    REDFINE FACILITY STGADMIN.ARC.* UACC(NONE)
    PERMIT STGADMIN.ARC.* CLASS(FACILITY) ID(STGADGRP) ACCESS(READ)
    SETROPTS RACLIST(FACILITY) REFRESH
    SETROPTS GENERIC(FACILITY) REFRESH

After you define the generic profile and permitted access, you can display it by using the
**RLIST** RACF command. Figure 5-5 shows a sample listing.

Other groups and users will require various accesses and each command can be specifically
permitted and restricted. Section 5.1.7, “RACF protection for ABARS” on page 91 illustrates
the use of more specific profiles to protect the ABARS commands.


-----

Figure 5-5 shows how to display the STGADMIN.ARC.* profile.

    RL FACILITY (STGADMIN.ARC.*) ALL
    CLASS   NAME
    -----   ----
    FACILITY  STGADMIN.ARC.* (G)
    
    LEVEL OWNER   UNIVERSAL ACCESS YOUR ACCESS WARNING
    ----- --------  ---------------- ----------- -------
    00  MHLRES3     NONE        NONE  NO
    
    INSTALLATION DATA
    -----------------
    NONE
    
    APPLICATION DATA
    ----------------
    NONE
    SECLEVEL
    --------
    NO SECLEVEL
    
    CATEGORIES
    ----------
    NO CATEGORIES
    
    SECLABEL
    --------
    NO SECLABEL
    
    AUDITING
    --------
    FAILURES(READ)
    
    NOTIFY ------
    NO USER TO BE NOTIFIED
    
    CREATION DATE LAST REFERENCE DATE LAST CHANGE DATE
    (DAY) (YEAR)    (DAY) (YEAR)   (DAY) (YEAR)
    ------------- ------------------- ----------------
    205  03     205  03     205  03
    
    ALTER COUNT  CONTROL COUNT  UPDATE COUNT  READ COUNT
    -----------  -------------  ------------  ----------
    NOT APPLICABLE FOR GENERIC PROFILE
    
    USER   ACCESS
    ----   ------
    MHLRES3  READ
    MHLRES1  READ
    MARY   ALTER

_Figure 5-5  Listing the STGADMIN.ARC.* profile_


-----

### 5.1.7 RACF protection for ABARS

Users that are authorized by DFSMShsm (specified in the **SETSYS AUTH** command in the
ARCCMDxx member of PARMLIB) can issue **ARECOVER** and **ABACKUP** commands. ABARS also
uses RACF FACILITY class profiles to permit certain operators and users to issue the
**ABACKUP** and **ARECOVER** commands.

You define RACF FACILITY class profiles and authorize users based on the level of
authorization that the user requires. Comprehensive authorization allows a user to issue the
**ABACKUP** and **ARECOVER** commands for all aggregates. RACF will not check the authority of the
user to access each data set in a specific aggregate.

Restricted authorization restricts a user to issuing **ABACKUP** and **ARECOVER** commands for only
the single aggregate that is specified in the ABARS FACILITY class profile name.

The RACF FACILITY class profiles have names that begin with STGADMIN (storage
administration). These FACILITY profiles are used to protect ABARS functions and many
other storage management subsystem (SMS) functions.

Define the profiles for comprehensive command authority with the following RACF
commands, as shown in Example 5-4.

_Example 5-4  Defining FACILITY profiles to protect ABARS commands_

RDEFINE FACILITY STGADMIN.ARC.ABACKUP UACC(NONE)
RDEFINE FACILITY STGADMIN.ARC.ARECOVER UACC(NONE)

The following command authorizes a user (MHLRES5) to issue the **ABACKUP** command for all
aggregate groups:

PERMIT STGADMIN.ARC.ABACKUP CLASS(FACILITY)

-  ID(MHLRES5) ACCESS(READ)

More restricted aggregate backup authority can be defined with profiles
(STGADMIN.ARC.ABACKUP . _agname_ ) for each aggregate. Issue the following command to
define a FACILITY class profile for the ITSOU001 backup aggregate:

RDEFINE FACILITY STGADMIN.ARC.ABACKUP.ITSOU001 UACC(NONE)

Authority to issue an **ABACKUP** command for aggregate ITSOU001 is given to user MHLRES5.
The following command permits access to the **ABACKUP** command for a specific aggregate:

PERMIT STGADMIN.ARC.ABACKUP.ITSOU001 CLASS(FACILITY)

-  ID(MHLRES5) ACCESS(READ)

Users with this restricted authority must have a minimum of READ access to all data sets that
are protected by RACF in the aggregate group. If the users do not have this level of access to
the data sets, the **ABACKUP** command fails.

As with the **ABACKUP** commands, **ARECOVER** commands can also be restricted with a profile for
each aggregate, STGADMIN.ARC.ARECOVER. _agname_ . The use of
DSCONFLICT(REPLACE), REPLACE as a conflict resolution data set action, or REPLACE
when it is specified by ARCCREXT, can also be restricted by using the RACF FACILITY class
profile STGADMIN.ARC.ARECOVER. _agname_ .REPLACE. The following command shows
how to define the profile to protect **ARECOVER** for an aggregate and the use of **REPLACE** :

RDEFINE FACILITY STGADMIN.ARC.ARECOVER.ITSOU001.REPLACE UACC(NONE)


-----

### 5.1.8 RACF protection for Concurrent Copy

DFSMShsm uses the STGADMIN.ADR.DUMP .CNCURRNT FACILITY class to authorize the
use of the Concurrent Copy options on the data set **backup** commands. Checking for
authorization occurs before DFSMSdss is invoked. If RACF indicates a lack of authority,
DFSMShsm will only fail the data set backup request if the Concurrent Copy request was
REQUIRED . If REQUIRED was not specified and RACF indicates a lack of authority, DFSMShsm
continues to back up the data set as though the Concurrent Copy keyword was not specified
on the **backup** command.

Define the profiles for comprehensive command authority with the following RACF command.
This command protects Concurrent Copy:

RDEFINE FACILITY STGADMIN.ADR.DUMP.CNCURRNT UACC(NONE)

The following command authorizes a user to issue the **backup** or **dump** command with
Concurrent Copy options:

PERMIT STGADMIN.ADR.DUMP.CNCURRNT CLASS(FACILITY)

-  ID(MHLRES5) ACCESS(READ)

### 5.1.9 ARCCATGP group

Issuing the **UNCATALOG** , **RECATALOG** , or **DELETE** **NOSCRATCH** command against a migrated data set
causes the data set to be recalled before the operation is performed. You can authorize
certain users to issue these commands without recalling the migrated data sets by connecting
the user to the RACF group ARCCATGP . When a user is logged on under RACF group
ARCCATGP , DFSMShsm bypasses the automatic recall for UNCATALOG, RECATALOG, and
DELETE NOSCRATCH requests for migrated data sets. The following tasks are used to
enable DFSMShsm to bypass automatic recall during catalog operations:

1. Define RACF group ARCCATGP by using the following RACF command:

ADDGROUP (ARCCATGP)

2. Connect users who need to perform catalog operations without automatic recall to
ARCCATGP by using the following RACF command:

CONNECT (userid1,. . .,userid _n_ ) GROUP(ARCCATGP) AUTHORITY(USE)

3. Each user who needs to perform catalog operations without automatic recall must log on
to TSO and specify the **GROUP(ARCCATGP)** parameter on the TSO logon window
(Figure 5-6) or the **GROUP=ARCCATGP** parameter on the JOB statement of a batch job
(Figure 5-7 ).

//HSMCAT  JOB(accounting information),'ARCCATGP Example',
//     USER=ITSOHSM,GROUP=ARCCATGP,PASSWORD=password
//STEP1  EXEC PGM=....

_Figure 5-6  JCL specifying RACF group ARCCATGP_


-----
    ------------------------------- TSO/E LOGON --------------------------------

    Enter LOGON parameters below:          RACF LOGON parameters:
    
    Userid  ===> MHLRES5
    
    Password ===>                 New Password ===>
    
    Procedure ===> IKJACCNT             Group Ident ===>
    
    Acct Nmbr ===> ACCNT#
    
    Size   ===> 6072
    
    Perform  ===>
    
    Command  ===>
    
    Enter an 'S' before each option desired below:
    -Nomail     -Nonotice    -Reconnect    -OIDcard
    
    PF1/PF13 ==> Help  PF3/PF15 ==> Logoff  PA1 ==> Attention  PA2 ==>
    Reshow
    You may request specific help information by entering a '?' in any entry
    field

_Figure 5-7  TSO/E LOGON panel_

### 5.1.10 Protecting migration and backup data sets

When a data set is migrated or backed up by DFSMShsm, the data set is given a name that is
based on the prefix that you specify in the ARCCMDxx PARMLIB member and DFSMShsm
constants.

The migration copy of a data set has the following name:

_prefix_ .HMIG.Tssmmhh.user1.user2.Xydd

In this name, _prefix_ is the prefix that you specify on the **SETSYS MIGRATEPREFIX** command.

The backup version of a data set has the following name:

_prefix_ .BACK.Tssmmhh.user1.user2.Xydd

In this name, _prefix_ is the prefix that you specify on the **SETSYS BACKUPPREFIX** command.

Migrated and backed up data sets must not be accessed as regular MVS data sets. Use
generic profiles that are based on the prefix that you specify on **SETSYS MIGRATEPREFIX** and
**SETSYS BACKUPPREFIX** to RACF to protect migrated and backed up data sets. The RACF
profiles must be created with a universal access authority (UACC) of NONE . Only DFSMShsm
and your storage administrator need to access them. The DFSMShsm started task does not
need to be on the access lists because DFSMShsm sets itself up as a privileged user to
RACF. Users who have the RACF OPERATIONS attribute automatically can access the
profiles.


-----

In our system, migrated data sets have the prefix _HSM_ . Use the following RACF command to
define a generic data set profile to protect all migrated data sets with a universal access of
NONE . The following command defines generic profile protection of migration and backup data
sets:

ADDSD 'HSM.**' UACC(NONE)

After you create the RACF generic profile to protect all migrated data sets, you must permit
users access to the RACF profile based on their requirements.


-----

