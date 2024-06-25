# 8. HSMplexes, GRSplexes, and sysplexes
In this chapter, we describe the concepts of and relationships among hierarchical storage management complexes (HSMplexes), global resource serialization complexes (GRSplexes), and sysplexes.

## 8.1 Introduction to the HSMplex

So far, the information in this book focused on the implementation of a single HSM
environment. Depending on the number of systems in your environment that share data and
the amount of data that you want DFSMShsm to manage, you might want to implement an
HSMplex.

### 8.1.1 Sample HSMplex

An _HSMplex_ is two or more instances of DFSMShsm that share the same set of control data
sets (CDSs) and journal. These instances of DFSMShsm can be installed on the same z/OS
image or logical partition (LPAR), on multiple LPARs, or both. Multiple instances of
DFSMShsm that are installed on the same LPAR are called a _multi-address space_
_DFSMShsm_ (MASH) LPAR.

An HSMplex can contain up to 39 instances of DFSMShsm, which is the number of unique
host IDs that exist within DFSMShsm. For a visual representation of the HSMplex concept,

For an Example, the HSMplex consists of six DFSMShsm hosts: Three on System A, one on System B,
and two on System C. All six of these hosts share access to the same set of CDSs and journal.
Each LPAR has one MAIN host. The MAIN host on each LPAR can perform the following
unique functions:

- Process implicit requests, such as recalls (in a non-common recall queue (CRQ)
environment) and delete migrated data sets, from user address spaces

- Process explicit commands from Time Sharing Option (TSO), such as **HSEND** and **HMIGRATE**

- Manage aggregate backup and recovery support (ABARS) secondary address spaces


-----

Hosts A, D, and E are MAIN hosts. If an LPAR has more than one DFSMShsm host (MASH
LPAR), the hosts other than the MAIN host are designated as auxiliary (AUX) hosts. AUX
hosts and MAIN hosts can perform the following functions:

- Allow modify commands from a console
- Run automatic space management, backup, and dump

Hosts B, C, and F are AUX hosts. Each HSMplex has one host that is designated as the
PRIMARY host. The PRIMARY host performs certain functions on behalf of all of the hosts in
an HSMplex to avoid duplicate effort. See 8.3.1, “Defining a primary DFSMShsm” on
page 198 for a list of the duties that a PRIMARY host performs. Host B is the PRIMARY host.

## 8.2 Data integrity and resource serialization

When a single-host DFSMShsm environment is used, DFSMShsm protects the integrity of its
owned and managed resources by serializing access to data within the DFSMShsm address
space. In an HSMplex, you must choose one of the following methods of resource
serialization:

- _Global enqueue_ : User data sets are protected with the ENQ and DEQ macros.

- _Volume reserve_ : DFSMShsm issues reserves against the source volume to protect data
sets during volume processing. The protection is requested by the RESERVE macro.

Global enqueues use the ENQ macro to obtain access to a resource and the DEQ macro to
free the resource. The ENQ macro serializes resources between tasks within the same
address space on the same LPAR. If you choose to employ this method of serialization in a
multi-LPAR HSMplex, you need to activate the global resource serialization (GRS) element of
z/OS or an equivalent cross-system enqueue product.

Volume reserves use the RESERVE macro to obtain access to a resource and the DEQ
macro to free the resource. The RESERVE macro serializes an entire volume against
updates by other LPARs, but allows shared access between tasks within the same address
space, or between address spaces on the owning LPAR.

### 8.2.1 Global resource serialization

GRS is a z/OS product that is designed to protect the integrity of resources in a multiple
DFSMShsm host environment. When the systems that share resources are combined into a
global resource serialization complex (GRSplex), GRS serializes access to shared resources.
User data sets are protected by associating them with the SYSDSN resource and then
passing the SYSDSN token to the other LPARS in the GRSplex. The SYSDSN resource is
passed to cross-system (global) enqueues. DFSMShsm shares its resources according to
ranges of control that are defined by z/OS that are known as _scopes_ in GRS terminology.

These scopes, in association with GRS resource name lists (RNLs), define to the entire
complex the resources that are local and the resources that are global. The GRS scopes are
listed:

**STEP** Scope within a z/OS address space
**SYSTEM** Scope within a single z/OS image
**SYSTEMS** Scope across multiple z/OS images


-----

The GRS RNLs are listed:

- SYSTEM inclusion RNL

  This RNL lists the resources that are requested with a scope of SYSTEM that you want
GRS to treat as global resources.

- SYSTEMS exclusion RNL

  This RNL lists the resources that are requested with a scope of SYSTEMS that you want
GRS to treat as local resources.

- RESERVE conversion RNL

  This RNL lists resources that are requested on RESERVE macro instructions for which
you want GRS to suppress the hardware reserve.

For the full list of resources that must be considered when you configure your DFSMShsm
serialization parameters, see Table 8-1. In this table, each pair of QNAME and RNAME
values uniquely identifies a resource; the SCOPE indicates the range of control. The DISP
(disposition) column indicates whether DFSMShsm or the system requests the resource for
exclusive (EXCL) or shared (SHR) use. The significance of the “Can convert from RESERVE”
column is explained in 8.2.3, “Using GRS resource name lists”. The “Must
propagate” column indicates whether GRS must communicate the ENQ to all shared
systems.

_Table 8-1  DFSMShsm GSRs_


|QNAME (Major)|RNAME (Minor)|SCOPE|DISP|Can convert from RESERVE|Must propagate|Description|
|---|---|---|---|---|---|---|
|Serialization of control data sets (CDSs)|||||||
|ARCAUDITa|ARCBCDS ARCMCDS ARCOCDS|SYSTEMS|SHR|NO|N/Ab|The associated volume reserves of the CDS volumes prevent updates from other LPARS while AUDIT FIX is running in the LPAR that is issuing the reserve.|
|ARCBACKa|ARCBCDS ARCMCDS ARCOCDS|SYSTEMS|SHR|NO|N/A|The associated volume reserves of the CDS volumes prevent access or updates to the CDSs from other LPARs while the CDSs are backed up in the LPAR that is issuing the reserve.|
|ARCGPAa|ARCBCDS ARCMCDS ARCOCDS|SYSTEMS|SHR|NO|N/A|The associated volume reserves of the CDS volumes prevent access or updates to the CDSs from other LPARs while CDSs are updated.|

-----

|QNAME (Major)|RNAME (Minor)|SCOPE|DISP|Can convert from RESERVE|Must propagate|Description|
|---|---|---|---|---|---|---|
|ARCGPA|ARCRJRN|SYSTEMS|EXCL|YES|YES|The associated volume reserve of the journal volume prevents access to the journal from other DFSMShsm hosts while the journal is being read or written in the DFSMShsm host that is issuing the reserve.|
|ARCGPALa|ARCMCDS|SYSTEMS|SHR|NO|N/A|The associated volume reserve of the migration control data set (MCDS) volume prevents access to the level 2 control record (L2CR) from other LPARs while the L2CR is updated in the LPAR that is issuing the reserve.|
|ARCUPDTa|ARCBCDS ARCMCDS ARCOCDS|SYSTEMS|SHR|NO|N/A|The associated volume reserves of the CDS volumes prevent access to the CDS from other LPARs while they are being recovered by UPDATEC processing in the LPAR that is issuing the reserve.|
|ARCENQGc|ARCBCDS ARCMCDS ARCOCDS|SYSTEMS|EXCL|N/A|YES|This enqueue is issued in CDSQ=YES environments and is held while accessing the CDSs from the DFSMShsm host that is issuing the enqueue.|
|ARCENQG|ARCCDSVF|SYSTEMS|EXCL|N/A|YES|This enqueue is issued to ensure that only one CDS version backup function is running in the HSMplex.|
|ARCENQG|ARCCDSVD|SYSTEMS|EXCL|N/A|YES|This enqueue is issued to ensure that the data area that is used in CDS version backup is not updated by any other DFSMShsm host while the function is running.|
|ARCENQG (VSAM record-level sharing (RLS) mode)|ARCCAT|SYSTEMS|SHR|N/A|YES|This enqueue is issued as a CDS update resource only in RLS mode.|
|Serialization of functions|||||||

-----

|QNAME (Major)|RNAME (Minor)|SCOPE|DISP|Can convert from RESERVE|Must propagate|Description|
|---|---|---|---|---|---|---|
|ARCENQG|ARCBMBC|SYSTEMS|EXCL|N/A|YES|This enqueue is issued in SETSYS USERSERIALIZATION environments and ensures that only one instance of move backup versions is running in the HSMplex.|
|ARCENQG|ARCL1L2|SYSTEMS|EXCL|N/A|YES|This enqueue is issued in SETSYS USERSERIALIZATION environments and ensures that only one instance of level 1 to level 2 migration is running in the HSMplex.|
|ARCENQG|ARCMCLN|SYSTEMS|EXCL|N/A|YES|This enqueue is issued in SETSYS USERSERIALIZATION environments and ensures that only one instance of migration cleanup is running in the HSMplex.|
|ARCENQG|EXPIREBV|SYSTEMS|EXCL|N/A|YES|Only one instance of the EXPIREBV command is allowed in an HSMplex.|
|ARCENQG|RECYC-L2|SYSTEMS|EXCL|N/A|YES|This enqueue is issued during RECYCLE to establish a limit of one instance of recycling migration-level 2 (ML2) tapes in the HSMplex.|
|ARCENQG|RECYC-DA|SYSTEMS|EXCL|N/A|YES|This enqueue is issued during RECYCLE to establish a limit of one instance of recycling daily backup tapes in the HSMplex.|
|ARCENQG|RECYC-SP|SYSTEMS|EXCL|N/A|YES|This enqueue is issued during RECYCLE to establish a limit of one instance of recycling spill backup tapes in the HSMplex.|

-----

|QNAME (Major)|RNAME (Minor)|SCOPE|DISP|Can convert from RESERVE|Must propagate|Description|
|---|---|---|---|---|---|---|
|ARCENQG|COPYPOOL I I cpname|SYSTEMS|EXCL|N/A|YES|The scope of the fast replication copy pool extends beyond an HSMplex because a copy pool is defined at the SMSplex level. All DFSMShsm hosts, regardless of which HSMplex they reside in, are prevented from processing the same copy pool. The resource is obtained unconditionally and if the resource is not immediately available, it waits.|
|ARCENQG|CPDUMP&& cpname&& Vnnn|SYSTEMS|EXCL|N/A|YES|This enqueue is used for the dumping of copy pools.|
|ARCBTAPE|volser|SYSTEMS|EXCL|N/A|YES|This enqueue is used for the Recover Tape Takeaway function.|
|ARCBTAPE|volser. TAKEAWAY|SYSTEMS|EXCL|N/A|YES|This enqueue is used for the Recover Tape Takeaway function.|
|SERIALIZATION of user data sets by DFSMShsm|||||||
|ARCDSN|DSNAME|SYSTEMS|d|N/A|YES|For SETSYS USERSERIALIZATION environments, this enqueue enables DFSMShsm to protect the integrity of the data set that relates to concurrent processing in an HSMplex. For SETSYS HSERIALIZATION environments, this enqueue enables DFSMShsm to protect the integrity of the data set that relates to concurrent processing only within the DFSMShsm host that is issuing the enqueue.|

-----

|QNAME (Major)|RNAME (Minor)|SCOPE|DISP|Can convert from RESERVE|Must propagate|Description|
|---|---|---|---|---|---|---|
|ARCENQG|dsname|SYSTEMS|e|N/A|YES|This ENQ prevents catalog locate requests from getting a “not cataloged” response in that interval of time during migration or recall when the volser is being changed from MIGRAT to non-MIGRAT or from non-MIGRAT to MIGRAT. It is also used to determine whether a recall with an “in process” flag set on really means “in process” or is a residual condition after a system outage.|
|ARCBACV|volserxf|SYSTEMS|EXCL|YES|YES|This reserve is issued only when you are running in a SETSYS HSERIALIZATION environment when you perform volume backup. The associated volume reserve of the user volume prevents updates of a user data set from other DFSMShsm hosts while it is being copied by the DFSMShsm host that is issuing the reserve.|
|ARCMIGV|volserxf|SYSTEMS|EXCL|YES|YES|This reserve is issued only when you run in a SETSYS HSERIALIZATION environment when you perform volume migration. The associated volume reserve of the user volume prevents updates of a user data set from other DFSMShsm hosts while it is being copied by the DFSMShsm host that is issuing the reserve.|
|Serialization of user data sets by the system|||||||
|SYSDSN|dsname|g|e|N/A|YES|This enqueue is the method MVS allocation uses to provide data integrity when it allocates data sets.|

-----

|QNAME (Major)|RNAME (Minor)|SCOPE|DISP|Can convert from RESERVE|Must propagate|Description|
|---|---|---|---|---|---|---|
|SYSVSAM|dsname|SYSTEMS|e|N/A|YES|This enqueue is the method that Virtual Storage Access Method (VSAM) uses to provide data integrity commensurate with the share options of VSAM data sets.|
|SYSVTOC|volser|SYSTEMS|e|YES|YES|This enqueue is the method that DFSMSdfp direct access device space management (DADSM) uses to provide the integrity of a volume’s volume table of contents (VTOC). Note: For more information about journal volume dumps, see 8.2.3, “Using GRS resource name lists”.|

a. These resources are requested only in CDSR= YES environments. In CDSQ= YES environments, serialization is achieved through a global enqueue that uses the Qname of ARCENQG.
b. N/A means “not applicable”.
c. Valid if CDSQ= YES .
d. EXCL is used for migration, recall, recover, and ARECOVER. SHR is used for backup and ABACKUP.
e. The DISP of this resource can be either EXCL or SHR and requests are both types.
f. Used only in SETSYS DFHSMDATASETSERIALIZATION.
g. The SCOPE of the SYSDSN resource is SYSTEM only but is automatically propagated by GRS, if the default GRSRNL00 member that is supplied by Multiple Virtual Storage (MVS) is used. GRS-equivalent products also need to propagate this resource.

### 8.2.2 Serialization groups

Serialization within DFSMShsm is divided into four groups:

- Serialization of control data sets (CDSs)
- Serialization of user data sets by DFSMShsm
- Serialization of user data sets by the system
- Serialization of DFSMShsm functions

**Control data set serialization**
DFSMShsm provides two parameters that can be included in the startup procedure in
SYS1.PROCLIB to control the method of serialization to be used against the CDSs by
DFSMShsm:

- CDSQ: Serialize the CDSs by using global enqueues
- CDSR: Serialize the CDSs by using volume reserves


-----

**_CDSQ_**
By specifying CDSQ=YES and CDSR=NO in the DFSMShsm startup procedure, you instruct
DFSMShsm to serialize the CDSs by using global enqueues and that you have a
cross-system serialization product to propagate enqueues across the systems in this
HSMplex. Do not specify CDSQ=YES unless you have a cross-system serialization product,
such as GRS, to propagate the enqueues.

**Note:** All hosts in the HSMplex must implement the same serialization method.

If you want to start an instance of DFSMShsm in AUX mode, you must specify this method of
serialization (or CDSSHR=RLS).

**_CDSR_**
By specifying CDSR=YES and CDSQ=NO in the DFSMShsm startup procedure, you instruct
DFSMShsm to serialize the CDSs by using volume reserves. All hosts in the HSMplex must
use the same serialization method. If this method of serialization is used, all hosts within the
HSMplex must be started in MAIN mode.

**_Default_**
If you do not specify a serialization method in the startup procedure, the serialization
parameters that are used are CDSR=YES, CDSQ=NO. If an instance of DFSMShsm is
started in AUX mode by using these defaults, DFSMShsm will issue message ARC0006I and
shut down.

**_CDSSHR_**
The **CDSSHR** parameter in the startup procedure is used with your specifications for the **CDSQ**
and **CDSR** parameters. Table 8-2 shows the serialization that is achieved when
you use different combinations of these keywords.

_Table 8-2  DFSMShsm serialization with startup procedure keywords_

|CDSQ keyword|CDSR keyword|CDSSHR keyword|Serialization|
|---|---|---|---|
|YES|YES|YES|Both CDSQ and CDSR options are used.|
|YES|NO or not specified|YES|Only the CDSQ option is used.|
|With any other combination of specifications||YES|Only the CDSR option is used.|
|N/A|N/A|RLS|Uses VSAM RLS.|
|With any other combination of specifications||NO|No multiple host serialization.|
|With any other combination of specifications||Not specified|Performs multi-host serialization of type CDSQ and CDSR (whichever is specified) only if the DASD volume where the index of the MCDS resides was generated by using SYSGEN as SHARED or SHAREDUP.|

No default exists for the **CDSSHR** parameter. We recommend that you use Table 8-2 to
determine the appropriate CDSSHR setting for your environment.

-----

**_Sharing the CDS in an HSMplex_**
DFSMShsm determines that it is in a shared CDS environment by performing the following
steps:

1. Recognizing the CDSSHR=YES or CSDSHR=RLS startup parameters

2. Determining whether the index component of the migration control data set (MCDS)
resides on a DASD volume that was generated (SYSGEN) with a SHARED or
SHAREDUP device attribute

If either of these conditions are met, DFSMShsm performs the appropriate serialization.

**Note:** The one exception to this rule is if CDSSHR=NO is specified. If CDSSHR=NO is
specified, DFSMShsm will _not_ perform multiple-host serialization regardless of how the
DASD volume that contains the index component of the MCDS was generated (SYSGEN).

For example, if the DASD volume that contains the index component of the MCDS was
generated with SYSGEN as SHARED and CDSSHR=NO is specified, DFSMShsm will not
perform any multiple-host serialization. However, if CDSSHR=YES and the DASD volume
where the index component of the MCDS was not generated (SYSGEN) as SHARED,
DFSMShsm will perform the appropriate serialization for a shared CDS environment.

Specifying CDSSHR=YES or CDSSHR=RLS in an HSMplex will avoid any damage if the
DASD volume is specified incorrectly as non-shared on one or more of the sharing systems.

If the CDSs reside on a SHARED DASD volume but are only used by a single DFSMShsm
host, specifying CDSSHR=NO eliminates unnecessary serialization overhead.

**Note:** Each DFSMShsm host in the HSMplex needs the same setting for the **CDSSHR**
startup parameter.

**Serialization of user data sets by DFSMShsm**
DFSMShsm provides two **SETSYS** commands that can be included in the ARCCMDxx startup
PARMLIB member to control the method of serialization to be performed against user data
sets by DFSMShsm:

- **USERDATASETSERIALIZATION** : Serialize the data sets by using global enqueues
- **DFHSMDATASETSERIALIZATION** : Serialize the data sets by using volume reserves

**_USERDATASETSERIALIZATION_**
By specifying SETSYS USERDATASETSERIALIZATION, you are instructing DFSMShsm to
serialize data sets that are processed by volume migration and volume backup at the data set
level rather than the volume level during data set processing. You can use this specification in
an HSMplex where either the instances of DFSMShsm do not share volumes or you have a
cross-system serialization product, such as GRS, to propagate the enqueues across
systems.

-----

**_DFHSMDATASETSERIALIZATION_**
By specifying SETSYS DFHSMDATASETSERIALIZATION, you are instructing DFSMShsm to
serialize data sets that are processed by volume migration and volume backup by reserving
the source volume during data set processing. DFSMShsm releases the volume after the
data is copied because each volume’s data sets are migrated or backed up.

Although volume reserves ensure data integrity, they also prevent users on other systems
from accessing other data sets on the reserved volume while DFSMShsm is processing a
data set on that volume. Also, if one system issues multiple reserves for a device, that system
can tie up the device. Other systems cannot access the shared device until the reserve count
on the owning system is zero and the system releases the shared device.

The following considerations must be made when you select the serialization method of the
user data set by DFSMShsm:

- The setting of this parameter affects the value of the DFSMShsm integrity age.

- In DFSMShsm Version 1 Release 5, the incremental backup function was restructured to
improve the performance of that function. This improvement is effective only when
USERDATASETSERIALIZATION is specified. Use the **SETSYS** **DFHSMDATASETSERIALIZATION**
command only if it is required by your environment. Otherwise, it is recommended that you
use the **SETSYS USERDATASETSERIALIZATION** command.

- The Fast Subsequent Migration function supports reconnection only in a
USERDATASETSERIALIZATION environment.

- Certain data sets, such as multivolume physical sequential data sets, are processed with
**SETSYS USERDATASETSERIALIZATION** only.

**_Default_**
If you do not specify a serialization method in the startup PARMLIB member,
DFHSMDATASETSERIALIZATION is used.

**Serialization of user data sets by the system**
The system serializes user data sets by using three resources:

**SYSDSN** This enqueue is the method that MVS allocation uses to provide data
integrity when it allocates data sets.

**SYSVSAM** This enqueue is the method that VSAM uses to provide data integrity
commensurate with the share options of VSAM data sets.

**SYSVTOC** This enqueue is the method that DFSMSdfp DADSM uses to provide
integrity of a volume’s VTOC.

Non-VSAM user data sets are protected by associating them with the SYSDSN resource.
VSAM user data sets are protected by associating them with both the SYSDSN and
SYSVSAM resources. The SYSVTOC resource is used to protect the integrity of a volume’s
VTOC.

In an HSMplex, the SYSDSN and SYSVSAM resources must be propagated across all of the
systems by using GRS or an equivalent cross-system enqueue product. The SYSVTOC
resource needs to be propagated only if a global enqueue is used to serialize it.

-----

**Serialization of DFSMShsm functions**
Certain DFSMShsm functions in an HSMplex environment can be executed only by one
DFSMShsm host at a time. To ensure that this requirement is met, certain resources need to
be propagated across all systems in an HSMplex by using GRS or an equivalent
cross-system enqueue product. Table 8-3 displays these resource names and a description of
the function that each resource name provides.

_Table 8-3  GRS resource names_

|Major (qname) resource name|Minor (rname) resource name|Serialization result|
|---|---|---|
|ARCENQG|ARCL1L2|Allows only one DFSMShsm host to perform level 1 to level 2 migration.|
||ARCMCLN|Allows only one DFSMShsm host to perform migration cleanup.|
||ARCBMBC|Allows only one DFSMShsm host to move backup versions.|
||RECYC-L2|Allows only one DFSMShsm host to perform recycle on ML2 tape volumes.|
||RECYC-SP|Allows only one DFSMShsm host to perform recycle on spill tape volumes.|
||RECYC-DA|Allows only one DFSMShsm host to perform RECYCLE on daily tape volumes.|
||EXPIREBV|Allows one instance of the EXPIREBV command in an HSMplex.|



### 8.2.3 Using GRS resource name lists

You can use three GRS resource name lists to configure your environment:

- SYSTEM inclusion RNL
- SYSTEMS exclusion RNL
- RESERVE conversion RNL

You can use the RNLs to modify the default serialization method for each resource, as
described in Table 8-1 on page 186. Each resource is defined by using a major name
(qname) and a minor name (rname). Each RNL definition must be included in the GRSRNLxx
member of SYS1.PARMLIB.

**_SYSTEM inclusion RNL_**
This RNL is used to list the resources that are requested with a scope of SYSTEM that you
want GRS to treat as global resources. If you are in a multiple-LPAR DFSMShsm
environment, you must be sure to include the SYSDSN resource in this RNL to ensure that
the resource is propagated across all systems. (The SYSDSN resource is included, by
default, in the GRSRNL00 member that is supplied in z/OS.) You must include this resource in
the GRSRNLxx:

RNLDEF RNL(INCL) TYPE(GENERIC) QNAME(SYSDSN)


-----

**_SYSTEMS exclusion RNL_**
This RNL is used to list the resources that are requested with a scope of SYSTEMS that you
want GRS to treat as local resources. If you are in a CDSR=YES and
DFHSMDATASETSERIALIZATION environment, you need to include the resources that are
shown in Example 8-1 in your GRSRNLxx to ensure that the resources are treated as local
resources (SCOPE=SYSTEM).

_Example 8-1  Include these resources in your GRSRNLxx_

    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(ARCAUDIT)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(ARCBACK)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(ARCGPA)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(ARCGPAL)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(ARCUPDT)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(ARCBACV)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(ARCMIGV)

If you are in a CDSQ=YES and USERDATASETSERIALIZATION environment, you do not
need to include any of the previous resources in this RNL because they are used only in a
CDSR=YES and SETSYS DFHSMDATASETSERIALIZATION environment. The exception to
this rule is the ARCGPA/ARCRJRN resource. This resource is also used when you use global
enqueues but you do not need to specify it in any RNL because it is propagated as a
SYSTEMS enqueue, by default.

**_RESERVE conversion RNL_**
This RNL is used to list the resources that are requested on RESERVE macro instructions for
which you want GRS to convert hardware reserves to global enqueues.

If you are in a CDSR=YES and DFHSMDATASETSERIZATION environment, you can convert
one or more of the following volume reserves to global enqueues.

_Table 8-4  Resources that can be converted from volume reserves to global enqueues_

|Major (qname) resource name|Minor (rname) resource name|
|---|---|
|ARCGPA|ARCRJRN|
|ARCBACV|volserx|
|ARCMIGV|volserx|


If you choose to convert one or more of the volume reserves to global enqueues, the required
conversion entries for each of these volume reserves in the GRSRNLxx member are shown in
Example 8-2.

_Example 8-2  Required conversion entries to convert volume reserves to global enqueues_

    RNLDEF RNL(CON) TYPE(SPECIFIC) QNAME(ARCGPA) RNAME(ARCRJRNL)
    RNLDEF RNL(CON) TYPE(GENERIC) QNAME(ARCBACV)
    RNLDEF RNL(CON) TYPE(GENERIC) QNAME(ARCMIGV)

**Note:** If the volume that contains the DFSMShsm journal data set is dumped by invoking
DFSMSdss for a full volume dump, the ARCGPA/ARCRJRN resource and
SYSVTOC/volser resource must both be treated consistently by either treating both as
volume reserves or both as global enqueues. Failure to follow this rule can result in
lockouts or long blockages of DFSMShsm processing.

-----

For more information about GRS RNLs, see _z/OS MVS Planning: Global Resource_
_Serialization_ , SA22-7600.

### 8.2.4 VSAM SHAREOPTIONS for CDSs

The two methods that can be used to define the share options for the DFSMShsm CDSs are
described. You can use the first method if you start only one DFSMShsm host for each LPAR
in an HSMplex. You must use the second method if any of the LPARs in the HSMplex contain
more than one DFSMShsm host.

**_Method 1: VSAM SHAREOPTIONS(2 3)_**
If you start only one DFSMShsm host for each LPAR, the following share option strategy will
provide maximum protection against accidental, non-DFSMShsm concurrent updates:

- Define the CDSs with VSAM SHAREOPTIONS(2 3).
- Use the GRS RNL exclusion capability to avoid propagating the VSAM resource of
SYSVSAM for the CDS components to the other systems.

Cross-region share option 2 allows only one job at a time to open a data set for output. If that
data set is in the SYSTEMS exclusion list, the open is limited to one LPAR. This combination
sets a limit of one open for each LPAR with the expectation that the one open will be
DFSMShsm. When DFSMShsm is active on each LPAR in the HSMplex, no jobs other than
DFSMShsm will be able to update the CDSs. This method is an additional level of protection
that is not afforded by SHAREOPTION(3 3).

Add the statements that are shown in Example 8-3 to the GRSRNLxx PARMLIB member to
avoid propagating the SYSVSAM resource for the CDS components to the other systems.

_Example 8-3  GRS RNLDEF statements for SHAREOPTIONS(2 3)_

    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(SYSVSAM) RNAME(MCDS index name)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(SYSVSAM) RNAME(MCDS data name)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(SYSVSAM) RNAME(BCDS index name)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(SYSVSAM) RNAME(BCDS data name)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(SYSVSAM) RNAME(OCDS index name)
    RNLDEF RNL(EXCL) TYPE(GENERIC) QNAME(SYSVSAM) RNAME(OCDS data name)

If you want to implement this sharing option and start multiple instances of DFSMShsm in the
same LPAR (MASH), you must use VSAM record-level sharing (RLS) to access the CDSs
(CDSSHR=RLS).

**Note:** The GRS RNLDEF statements cannot be used with method 1 when you use RLS.

Additional considerations apply when you use this sharing strategy:

- Reserve contentions can occur when an installation does not use a global serialization
product and that site processes DFSMShsm and applications concurrently with VSAM
data sets on the same volume.

- Do not attempt to reorganize your CDSs while DFSMShsm is running on any LPAR that
uses those CDSs.

- You can use this share option to access the CDSs in RLS mode. When you use RLS in a
VSAM SHAREOPTIONS(2 3) mode, VSAM allows non-RLS access that is limited to read
processing.


-----

**_Method 2: VSAM SHAREOPTIONS(3 3)_**
If you start more than one DFSMShsm host for each LPAR or want to use the default starter
set to allocate the CDSs, you can use this share option.

The DFSMShsm starter set allocates the CDSs with VSAM SHAREOPTIONS(3 3) to allow
multiple DFSMShsm hosts to be started in one or more LPARs in an HSMplex. This method
relies on DFSMShsm to provide an appropriate serialization protocol to ensure the read and
update integrity of the CDSs.

This share option can be used if you want to use RLS (CDSSHR=RLS) to access the CDSs.

**Note:** Do not attempt to reorganize your CDSs while DFSMShsm is running on any LPAR
that uses those CDSs.

**Important integrity exposure** : Even with one of these share options chosen, a data
integrity exposure might exist if DFSMShsm is not active on all LPARS in the HSMplex.
Because of this exposure, you must strictly control any CDS reorganization or
maintenance procedure so that the utility job is allocated with a disposition of OLD
(DISP=OLD). A disposition of OLD causes an exclusive enqueue to be issued against the
SYSDSN resource for the CDS cluster name. Because any active DFSMShsm in an
HSMplex holds a shared enqueue on this resource, the job cannot process until each of
the active DFSMShsm hosts in the HSMplex is shut down.

## 8.3 Starting DFSMShsm in an HSMplex

The tasks to start DFSMShem in an HSMplex are described.

### 8.3.1 Defining a primary DFSMShsm

In an environment with multiple DFSMShsm hosts (in one or more LPARS), define one host
as the PRIMARY DFSMShsm. This host automatically performs backup and dump functions
that do not relate to one data set or volume. The following “level functions” are included:

- Automatic backup processing:

  - Phase 1: Backing up CDSs
  - Phase 2: Backing up data sets that migrated before being backed up
  - Phase 3: Moving backup versions of data sets from migration-level 1 (ML1) volumes to
backup volumes

- Automatic dump processing:

  - Phase 1: Deleting expired dump copies
  - Phase 2: Deleting excess dump VTOC copy data sets

The storage administrator must specify the primary DFSMShsm in the DFSMShsm startup
procedure, or as a parameter on the **START** command. If no primary DFSMShsm is specified,
DFSMShsm does not perform the level functions that are listed. If you start more than one
primary DFSMShsm, DFSMShsm might process the level functions more than once a day.

The primary host can be either a MAIN or an AUX host. Designating an AUX host as the
primary host reduces contention between its level functions and the responsibilities that are
unique to the MAIN host, such as recalls and deletes.

-----

### 8.3.2 Defining all DFSMShsm hosts in a multihost environment

In a multiple DFSMShsm host environment, ensure that the host identifier for each host is
unique by considering how you specify the **HOST=x** keyword of the DFSMShsm startup
procedure. The **x** keyword represents the unique host identifier for each host. For more
information and examples, see Chapter 3, “Getting started” on page 29.

If you choose to use a single startup procedure to start multiple DFSMShsm hosts in a single
LPAR, two alternatives exist to identify these startups for subsequent **MODIFY** commands, as
shown in Example 8-4.

_Example 8-4  Two alternatives to identify startup procedures_

    S procname.id1,HOST=A,HOSTMODE=MAIN,other parms
    S procname.id2,HOST=B,PRIMARY=Y,HOSTMODE=AUX,other parms
    
     or
    
    S procname,JOBNAME=id1,HOST=A,HOSTMODE=MAIN,other parms
    S procname,JOBNAME=id2,HOST=B,PRIMARY=Y,HOSTMODE=AUX,other parms

The common procedure needs to specify PRIMARY= N so that you need to override it only for
the one primary host.

If you need to issue the same command to multiple DFSMShsm hosts that are started with
identifiers with the same set of leading characters, you can use an asterisk wildcard with the
**MODIFY** command, as shown in Example 8-5.

_Example 8-5  Modify command with an asterisk wildcard_

    F DFHSM*,Q ACT

## 8.4 HSMplex considerations

The considerations for implementing an HSMplex are described.

### 8.4.1 CDS backup version considerations

The CDS backup version considerations vary depending on the devices to which you back up
your CDSs.

**CDS backup to DASD**
The CDS backup function is executed by the PRIMARY host as the first phase of automatic
backup. If you want a secondary DFSMShsm host to back up the CDSs by using the **BACKVOL**
**CDS** command, you need to ensure that the pre-allocated backup versions are accessible to
the secondary DFSMShsm host.

**CDS backup to tape**
If your CDSs are not in a shared user catalog, the backup versions will not “roll off” if they are
created on one LPAR and deleted on another LPAR.

-----

### 8.4.2 Problem determination aid considerations

You must allocate unique PDOX and PDOY data set names for each DFSMShsm host if the
data sets are allocated to a catalog that is shared between the DFSMShsm hosts. Allocating
the data sets on a volume that is managed by another DFSMShsm host can cause
performance degradation or a lockout.

We recommend that you create the problem determination aid (PDA) data set names to
reflect the DFSMShsm host that they correspond to. For example, if the host ID is 1 , the PDA
data set names are allocated in the following manner:

- _qualifier_ .HSM1.HSMPDOX
- _qualifier_ .HSM1.HSMPDOY

**8.4.3 Small-data-set-packing data set considerations**

You need to define a (2 x) share option for your small-data-set packing (SDSP) data sets. In
an HSMplex that uses a cross-system enqueue product, this option prevents any job on the
LPARs in the HSMplex from writing to an SDSP while a DFSMShsm host has an SDSP open
for output. A (3 x) option allows programs other than DFSMShsm to write to an SDSP while a
DFSMShsm host has an SDSP open for output.

### 8.4.4 Volume considerations

Consider the following factors when you implement DFSMShsm in a multiple DFSMShsm
host environment:

- DFSMShsm does not reserve the volume that contains a user’s data set if the user issues
a request to migrate or back up the data set, but depends upon global serialization of the
SYSDSN resource.

- While DFSMShsm calculates the free space of a volume, it reserves the volume. This task
can momentarily interfere with the response time for the functions that require access to
that volume.

- Run automatic primary space management, backup, and dump during periods of low
system use and low interactive user activity to reduce contention for data sets among
DFSMShsm hosts. When a volume that is managed by DFSMShsm is being processed by
space management or backup in a DFHSMDATASETSERIALIZATION environment, other
DFSMShsm hosts can experience performance problems if they attempt to access the
volume. To eliminate these performance problems, consider the use of
USERDATASETSERIALIZATION as an alternative.

### 8.4.5 JES3 considerations

In a JES3 environment, the storage management subsystem (SMS) volumes and the
non-SMS volumes that are managed by DFSMShsm in the DFSMShsm general pool must be
shared by all processors. Furthermore, if several of the volumes in a data set or volume pool
are also in the general pool, all volumes in both pools must be shared.


-----

### 8.4.6 Multitasking considerations

When you run multiple tasks in a multiple DFSMShsm host configuration, consider the most
effective use of your LPARs and the number of DFSMShsm hosts that run within each of
those LPARs. For example, you might ask whether it is more efficient to perform eight tasks
with one LPAR or four tasks with two LPARs. For space management and backup, it is
generally better to perform eight tasks with one LPAR and distribute those tasks across
multiple hosts that run in that LPAR. One LPAR offers better performance than two LPARs as
a result of the DFSMShsm CDS sharing protocol.

For performance reasons, run several migration or backup tasks in one LPAR versus running
a few tasks in each LPAR of an HSMplex; however, for an RLS environment (CDSSHR=RLS),
it is better to spread out the tasks across multiple LPARs.

## 8.5 DFSMShsm in a sysplex environment

A _sysplex_ is a collection of LPARs that work together by using certain hardware and software
products to process workloads. The products that make up a sysplex provide greater
availability, easier system management, and improved growth potential over a conventional
computer system of comparable processing power. Two types of sysplexes exist: base and
parallel. A _base sysplex_ is a sysplex implementation without a coupling facility. A _parallel_
_sysplex_ is a sysplex implementation with a coupling facility.

Systems in a _base_ sysplex communicate by using channel-to-channel (CTC) communications.
In addition to CTC communications, systems in a _parallel_ sysplex use a coupling facility (CF),
which is a microprocessor unit that enables high performance sysplex data sharing. Because
parallel systems allow faster data sharing, workloads are processed more efficiently.

If you run DFSMShsm in a sysplex environment, the following functions can enhance your
ability to successfully manage that environment:

- Single GRSplex serialization

  Allows each HSMplex, within a single GRSplex, to operate without interfering with any
other HSMplex.

- Secondary host promotion

  Allows one DFSMShsm host to automatically assume the unique functions of another
DFSMShsm host that failed.

- Control DS extended addressability

  Allows CDSs to grow beyond the 4 GB size limit.

- Record-level sharing

  Allows CDSs to be accessed in RLS mode. For information about RLS, see 6.1,
“Record-level sharing” on page 96.

-----

- Common recall queue

  Balances the recall workload among the hosts in an HSMplex by implementing a common
queue.

- ARCCAT release

  Allows a CDS backup host to communicate through cross-system coupling facility (XCF)
to other hosts in an HSMplex to tell them to release the necessary resources to allow CDS
backup to begin.

### 8.5.1 Configuring multiple HSMplexes in a sysplex

If you want to define multiple HSMplexes in a sysplex, you must include the following
command in each DFSMShsm host’s ARCCMD _xx_ member:

SETSYS PLEXNAME( _HSMplex_name_suffix_ )

The **PLEXNAME** keyword distinguishes the separate HSMplexes within a single sysplex. If only
one HSMplex is in a sysplex, you do not need to specify this command. By default, the
HSMplex name is ARCPLEX0; the suffix is PLEX0, with a prefix of ARC.

If you specify any HSMplex name other than the default name on one host, you must also
specify that name on all other DFSMShsm hosts in that HSMplex.

### 8.5.2 Single GRSplex serialization in a sysplex environment

If two HSMplexes exist within a sysplex (GRSplex) environment, one HSMplex interferes with
the other HSMplex whenever DFSMShsm tries to update CDSs in non-RLS mode or when
DFSMShsm performs other functions, such as level 1 to level 2 migration. This interference
occurs because each HSMplex, even though each HSMplex has unique resources, uses the
same resource names for global serialization. This interference can be avoided by using
single GRSplex serialization.

Table 8-5 shows the resource names that, when enqueued upon, cause
interference between HSMplexes in a sysplex.

-----

_Table 8-5  Resources that cause interference between HSMplexes in a sysplex_

|Major (qname) resource name|Minor (rname) resource name|Serialization result|
|---|---|---|
|ARCENQG|ARCBMBC|Enqueues during the attach of the ARCBMBC subtask, which moves backup copies from ML1 to backup tapes.|
||ARCCDSVF|Serializes a CDS backup function to ensure that only one CDS backup is running within one HSMplex.|
||ARCCDSVD|Enqueues while copying CDSVDATA.|
||ARCL1L2|Enqueues L1 to L2 migration. L1 to L2 migration is a function of secondary space management.|
||ARCMCLN|Enqueues migration cleanup. Migration cleanup is a function of secondary space management.|
||RECYC_L2|Prevents two hosts from recycling ML2 tapes concurrently.|
||RECYC_SP|Prevents two hosts from recycling backup spill tapes concurrently.|
||RECYC_DA|Prevents two hosts from recycling backup daily tapes concurrently.|
||ARCBCDS ARCMCDS ARCOCDS|Enqueues CDSs (not obtained in RLS mode). Note: If CDSQ is specified, ARCGPA, ARCxCDS, SYSTEMS, SHARE translates to ARCENQG, ARCxCDS, SYSTEMS, EXCLUSIVE.|
||ARCCAT|In RLS mode, enqueues change from ARCGPA/ARCCAT STEP to ARCENQG/ARCCAT SYSTEMS to prevent CDS updates during CDS backup.|
||HOST||Hostid|Ensures that only one host is started with this host identifier.|
||EXPIREBV|Ensures that only one EXPIREBV command runs within an HSMplex.|
||COPYPOOL||cpname|SYSTEMS enqueue Note: The scope of the fast replication copy pool extends beyond an HSMplex because a copy pool is defined at the SMSplex level. All DFSMShsm hosts, regardless of which HSMplex they reside in, are prevented from processing the same copy pool. The resource is obtained unconditionally and if the resource is not immediately available, it waits.|
||CPDUMP||cpname||Vnnn|SYSTEMS enqueue Note: The scope of the fast replication copy pool extends beyond an HSMplex because a copy pool is defined at the SMSplex level. All DFSMShsm hosts, regardless of which HSMplex they reside in, are prevented from processing the same copy pool. The resource is obtained unconditionally and if the resource is not immediately available, it waits.|
|ARCGPA|ARCRJRN|The volume reserve of the journal volume.|
|ARCBTAPE|volser.TAKEAWAY|Allows recover tape takeaway.|
|ARCBTAPE|volser|Allows recover tape takeaway.|

-----

Single GRSplex serialization allows DFSMShsm to translate minor global resource names to
unique values within an HSMplex, therefore avoiding interference between the HSMplexes in
a sysplex.

To use this new translation technique, you must specify the following setting in your
DFSMShsm startup procedure:

RNAMEDSN=YES

When this setting is specified, DFSMShsm will invoke the new method of translation and will
append the CDS and journal data set names that are associated with the function, CDS, or
journal that is being serialized to the minor (rname) resource name that is used for that
serialization. The result is a unique resource name because the CDS and journal data set
names are unique to each HSMplex. Table 8-6 shows the translated minor (rname) resource
names when RNAMEDSN=YES is specified.

_Table 8-6  Minor (rname) resource translations_

|Current minor (rname) resource name|Translated minor (rname) resource name|
|---|---|
|ARCBMBC|ARCBMBC&bcsdsn|
|ARCCDSVF|ARCCDSVF&mcdsdsn|
|ARCCDSVD|ARCCDSVD&mcdsdsn|
|ARCL1L2|ARCL1L2&mcdsdsn|
|ARCMCLN|ARCMCLN&mcdsdsn|
|RECYC_L2|RECYC_L2&ocdsdsn|
|RECYC_SP|RECYC_SP&ocdsdsn|
|RECYCL2_DA|RECYC_DA&ocdsdsn|
|ARCxCDS|ARCxCDS&cdsdsn|
|ARCCAT|ARCCAT&mcdsdsn|
|ARCRJRN|ARCRJRN&jrnldsn|
|HOST | | Hostid|HOST | | Hostid&mcdsdsn|
|EXPIREBV|EXPIREBV&bcdsdsn|
|volser|volser&bcdsdsn|
|volser.TAKEAWAY|volser.TAKEAWAY&bcdsdsn|


**Default**
If RNAMEDSN=NO is specified or is not present in the DFSMShsm startup procedure, the
new translation method will _not_ be invoked.

**Note:** All DFSMShsm hosts within an HSMplex must use the same translation technique. If
a host detects an inconsistency in the translation technique, the detecting host immediately
shuts down.

-----

**Compatibility considerations**
Consider the following coexistence issues before you run DFSMShsm within an HSMplex:

- If all DFSMShsm hosts within an HSMplex are running at DFSMS/MVS Version 1
Release 5 or later, all DFSMShsm hosts must use the same serialization method. If not, at
least one of the hosts will shut down (that is, each host that detects a mismatch will shut
down).

- If an HSMplex has both Version 1 Release 5 and pre-Version 1 Release 5 running
concurrently, the Version 1 Release 5 hosts cannot specify RNAMEDSN=Y. If
RNAMEDSN=Y is specified, hosts that detect the mismatched serialization method will
shut down.

- If two or more HSMplexes run concurrently in a sysplex, each HSMplex that uses an old
serialization method will interfere with other HSMplexes. HSMplexes that use the new
serialization method will not interfere with other HSMplexes. However, in a two-HSMplex
environment, one HSMplex can use the old method and the other HSMplex can use the
new method; neither one will interfere with the other.

### 8.5.3 Secondary host promotion

DFSMShsm allows secondary hosts to take over functions for a failed primary host or
secondary space management (SSM) from a failed host. Secondary host promotion ensures
continuous availability of DFSMShsm functions.

The following definitions are key to understanding the concept of secondary host promotion:

- An _original host_ is a host that is assigned to perform primary host or SSM responsibilities.

- A _secondary host_ is a host that is not assigned to perform primary host or SSM
responsibilities.

- A _primary host_ is a host that performs primary-level functions, such as the following tasks:

  - Hourly space checks (for interval migration and recall of non-SMS data)
  - During autobackup: Automatic CDS backup
  - During autobackup: Automatic movement of backup versions from ML1 to tape
  - During autobackup: Automatic backup of migrated data sets on ML1
  - During autodump: Expiration of dump copies
  - During autodump: Deletion of excess dump VTOC copy data sets

- An _SSM host_ is generally the only host that performs SSM functions.

- A host is said to be _promoted_ when that host takes over the primary or SSM (or both) host
responsibilities from an original host.

- A host is said to be _demoted_ when its primary or SSM (or both) host responsibilities are
taken over by another host. Each promoted host always has a corresponding demoted
host, and vice versa.

**When a host is eligible for demotion**
For either a base or parallel sysplex, DFSMShsm, by using XCF, can enable secondary hosts
to take over any unique functions that are performed by a primary host or SSM host when one
of the following conditions occurs:

- The primary or SSM host goes into emergency mode.
- The primary or SSM host is stopped with the **DUMP** or **PROMOTE** keyword.
- The primary or SSM host is stopped while in emergency mode.
- The primary or SSM host is canceled, DFSMShsm fails, or the system fails.
- A promoted host is stopped or failed by any means.


-----

**Notes:** If an active primary or SSM host was demoted and it did not take back its
responsibilities (for example, it is in emergency mode), you can invoke any type of
shutdown and the host remains in its demoted status.

Do not change the host ID of a host that was demoted.

**Enabling secondary host promotion**
To enable secondary host promotion, specify the **SETSYS PROMOTE** command:

SETSYS PROMOTE(PRIMARYHOST(YES | NO) SSM(YES | NO))

The following options are valid for the **SETSYS PROMOTE** command:

**PRIMARYHOST(YES | NO)**
Indicates whether you want this host to take over primary host
responsibilities for a failed host. The default value is NO .

**SSM(YES | NO)** Indicates whether you want this host to take over the SSM
responsibilities for a failed host. The default value is NO .

Consider the following factors when you enable secondary host promotion:

- Only those DFSMShsm hosts that run on DFSMS/MVS Version 1 Release 5 and later are
eligible to use secondary host promotion functions.

- This parameter is ignored when the system runs in LOCAL mode. If the system runs in
MONOPLEX mode, the secondary host promotion function is active, but it is unable to
perform actions because cross-host connections are not enabled.

- Cross-system coupling facility (XCF) must be configured on the system that DFSMShsm
is active on and must be running in multisystem mode.

- An SSM host is not eligible to be promoted for another SSM host.

- The **SETSYS** command does not trigger promotion. That is, a host can be eligible to be
promoted only for hosts that fail after the **SETSYS** command is issued.

- Do not make a host eligible for promotion if its workload conflicts with responsibilities of
the original host or if it is active on a significantly slower processor.

- If the ARCCBEXT exit is used by the primary host, it must be available for use on all hosts
that are eligible to be promoted for the primary host. If the ARCMMEXT exit is used by the
SSM host, it must be available for use on all hosts that are eligible to be promoted for the
SSM host.

- The CDS backup data sets must be cataloged on all systems that are eligible to be
promoted for primary host responsibilities.

- In a multisystem environment, DFSMShsm always sets the SETSYS NOSWAP option.

**How secondary host promotion works**
When a primary or SSM host becomes disabled, all DFSMShsm hosts in the HSMplex are
notified through XCF. Any host that is eligible to perform the functions of the failed host will
attempt to take over for the failed host. The first host that successfully takes over for the failed
host becomes the promoted host. No method exists to assign an order to which hosts take
over the functions of a failed host.

If an original host is both a primary host and an SSM host, its responsibilities can be taken
over by two separate hosts.


-----

If the promoted host itself fails, any remaining host that is eligible for promotion takes over. If
additional failures occur, promotion continues until no hosts remain that are eligible for
promotion.

If a secondary host fails while it is promoted for an original host and no remaining active hosts
are eligible for promotion, out of any of the secondary hosts that become reenabled before
the original host, only the host that was last promoted for the original host can become the
promoted host.

**Note:** For secondary host promotion to work at its highest potential, do not use system
affinity. All systems must have connectivity to all storage groups. If system affinity is used,
storage groups that are only associated with one system are not processed when that
system is not available.

**Configuring automatic backup hosts in an HSMplex**
If you use secondary host promotion, be careful when you configure automatic backup in an
HSMplex:

- More than one automatic backup host needs to be available to ensure that volume
backups of managed volumes are performed even when the primary host is disabled.

**Note:** Promoted hosts take over only the unique functions of the original host. They do
not take over functions that can be performed by other hosts.

- If a secondary automatic backup host is eligible to be promoted for the primary host, its
backup window must be offset from the original primary host’s window in a way that it can
take over from where the original primary host left off. For example, its start time can
correspond with the average time that the primary host finishes its unique automatic
backup functions.

**Note:** These scenarios assume that the primary host is always an automatic backup
host.

**Configuring automatic dump hosts in an HSMplex**
We recommend to always have more than one automatic dump host in an HSMplex to ensure
that volume dumps are performed even if the primary host is disabled.

**How the take-back function works**
When an original host is reenabled to perform its unique responsibilities (through a restart or
by leaving emergency mode), the take-back process begins. The take-back process involves
the following procedures:

- The promoted host recognizes that the original host is enabled and gives up the
responsibilities that it took over (by resetting its windows, its cycles, and its exit settings to
the values that existed before it was promoted).

**Note:** Any changes that are made to the window and the cycle while the host was
promoted will be lost.

- Until the promoted hosts give back the responsibilities, the original host does not perform
any of the responsibilities that were taken over by the promoted hosts.


-----

**Considerations for implementing XCF for secondary host promotion**
The cross system coupling facility (XCF) component of MVS provides simplified multisystem
management. XCF services allow authorized programs on one system to communicate with
programs on the same system or on different systems. If a system ever fails, XCF services
allow the restart of applications on this system or on any other eligible system in the sysplex.

Before you configure XCF in support of the secondary host promotion function, consider the
following information:

- Only one DFSMShsm XCF group exists for each HSMplex. The XCF group name is the
HSMplex name, with the default name of ARCPLEX0 .

- One XCF group member exists for each DFSMShsm host in the HSMplex.

- DFSMShsm does not use the XCF messaging facilities.

### 8.5.4 Control data set extended addressability in a sysplex

As HSMplexes and sysplexes grow larger to accommodate the amount of data that can exist
in an environment, the CDS will likely grow beyond the 16 GB size limit for multicluster
migration control data sets (MCDSs) and backup control data sets (BCDSs) (each with four
clusters with 4 GB for each cluster) and the 4 GB size limit for offline control data sets
(OCDSs). With VSAM extended addressability, you can define each CDS so that it can grow
beyond these size limitations.

DFSMShsm supports the VSAM key-sequenced data set (KSDS) extended addressability
(EA) capability that uses RLS, CDSQ, or CDSR to access its CDSs.

**Extended addressability considerations**
The following considerations or requirements can affect extended addressability for your
CDSs:

- Mixing extended function (EF) clusters and non-EF clusters is permissible because each
cluster is treated as a separate entity. However, if any cluster is accessed in RLS mode, all
clusters must be accessed in RLS mode.

- Because EF data sets might contain compressed data, DFSMShsm issues warning
message ARC0130I RC16 whenever it detects this condition. RC16 means that a certain
CDS contains compressed data, which can affect performance.

### 8.5.5 Common recall queue

The main purpose of using common recall queue (CRQ) in an HSMplex is to balance the
recall workload among all of the hosts in an HSMplex. The use of CRQ enables the following
alternative configurations:

- Recall servers: Certain hosts can be configured to place recall requests on the CRQ and
select recall requests from the CRQ while other hosts are configured to place only recall
requests on the CRQ. Issue the **HOLD COMMONQUEUE(RECALL(SELECTION))** command on
those hosts that you do not want to select recall requests from the CRQ.

When used in an HSMplex that contains multiple DFSMShsm hosts on the same LPAR,
the use of a CRQ can increase the total number of concurrent recall tasks in an LPAR.
Without a CRQ, only the MAIN host in an LPAR can process recalls. With CRQ, the total
number of recalls that can be processed concurrently on an LPAR is 15x the number of
DFSMShsm hosts on that LPAR.

-----

- Non-ML2 tape only: If certain hosts are not connected to tape drives, they can be
configured to select only recalls that do not require ML2 tape. Issue the **HOLD**
**RECALL(TAPE)** command on those hosts.

- Nonparticipating hosts: If a host within an HSMplex is not connected to the CRQ in that
HSMplex, it will be able to process only the recalls that are submitted on that system. It will
not be able to process the recalls that are submitted on other hosts.

**CRQ considerations**
If the hosts within an HSMplex are on systems that cannot share data between them, they
must not share the same CRQ. If sets of hosts cannot share data, each of those sets of hosts
can share a unique CRQ in the HSMplex. When more than one CRQ is used in an HSMplex,
each CRQ and the set of hosts that connect to it are referred to as a _CRQplex_ .

**Note:** Although it is possible for multiple CRQplexes among hosts in an HSMplex to share
data, this type of configuration is discouraged because most of the benefits of CRQ are
achieved as the number of participating hosts increases.

**ARCCAT release**
The CDS backup function requires an exclusive enqueue on the ARCGPA/ARCCAT resource
when it accesses the CDSs in a non-RLS environment. The CDS backup function requires an
exclusive enqueue on ARCEQNG/ARCCAT when it accesses the CDSs by using RLS. All
other functions that use these resources require only a shared enqueue on the resource.

When a long-running function holds this resource (for example, during the migration of a large
data set to tape), it prevents CDS backup from obtaining the exclusive enqueue on the
resource until that long-running function completes and dequeues the resource. Because
CDS backup requires an exclusive enqueue, any function that is queued after CDS backup is
unable to process until CDS backup completes.

To minimize this delay, DFSMShsm hosts in the same HSMplex (RLS environment) or within
the same LPAR (non-RLS environment) allow the CDS backup process to begin immediately
after all pending CDS updates complete. The host that performs the CDS backup sends a
notification to the other hosts by using XCF services. All hosts processing functions and tasks
that might delay CDS backup complete pending CDS updates, suspend new CDS updates,
and release the ARCCAT resource to allow CDS backup to begin. After CDS backup
completes, suspended functions and tasks resume.

This functionality does not apply when you run multiple instances of DFSMShsm in the same
LPAR where XCF services are not available.

**Note:** This functionality was introduced with z/OS 1.13. If any pre-z/OS 1.13 hosts are in
the HSMplex, this functionality will not occur for those hosts.


-----

