# flowchart
flowchart

APPHGuide
From BexWiki
Jump to: navigation, search
Contents
[hide] [hide]

    1 Abstract
    2 Terminology
    3 APPH module details
        3.1 Coding Practices
        3.2 Build
    4 High Level Message Flow
        4.1 Browse
        4.2 Backup
        4.3 Restore
    5 Cluster Support
        5.1 Browse
        5.2 Backup
        5.3 Restore
    6 General Concepts
        6.1 Backup Document Construction
        6.2 Content Files
        6.3 Application versioning
        6.4 ah_config.xml
        6.5 APPS Task
    7 Implementation Details
        7.1 Cancel Handler
    8 Features
        8.1 Script Support
        8.2 Log and Temp file Cleanup
        8.3 ExpressDR
    9 Links

[edit]
Abstract

This page describes details about APPH module and also describes high level message flow involved in OSSV/XRS operations.

A new document http://sun9.bex.syncsort.com/~raghu/apph.html is under construction with more detailed information. Please read it in conjunction with this document until they are merged (there is some overlap).
[edit]
Terminology

    appserver - The module that implements the backup/restore of an application. Even though snap supports VSS applications, it is separately mentioned as its interface is different from that of app servers. One example of an app server is oracleapp that implements ORACLE backup/restore. 

    VV - Virtual Volume. This refers to one application component like oracle database. In case of VSS applications such as EXCH and SQL, this refers to one writer component. 

    PV - Physical Volume. 

    CV - Covering Volume. This refers to the volumes where application data is hosted. This will usually be a PV but it can also be APPS. 

[edit]
APPH module details

    It facilitates OSSV/XRS backup and restores and is similar to EH in scope.
    It is written in jython and is distributed as APPH.jar.
    It creates log files in the format apph<num>.txt (main log) and std_apph_<pid>.txt (redirected stdout and stderr). The main log file can be uploaded to logutil for diagnostic purposes but should be rarely needed. The module logs all error message in job log and if any messages are not clear, they should be fixed.
    It is a reusable multi-threaded module.
    The module is started with apph.bat on windows and with apph on unix. 

[edit]
Coding Practices

    Only use spaces for indentation and no TABs. indentation of 4 should be used.
    Try to follow guidelines in http://www.python.org/dev/peps/pep-0008.
    Do not keep "$Log" directive in source files. This will clutter the source file with all the checkin comments. They can be obtained any time with razor_file_info.
    Use rim for all razor operations. Of course, this applies to only older versions of APPH as dev is now in Subversion. For Subversion check-ins, provide a detailed check-in comment. It need not be limited to one line.
    If an interface is added to apph.s but is not implemented, it must be added to list of unimplemented methods. See SvcImplementation_apph.py:listUnimplemented() 

[edit]
Build

To build "dev" APPH locally, run the following on sun9:

sun9:workdir/APPH$ OBJ_DIR=jpylib make

to build r310 and before, you need to use a different Makefile than what is used by the build system. It is available at sun9:~raghu/work/build/r310/Makefile.apph. This only works on sun9. Copy the make file to the directory where you have APPH files (missing files will be picked up from /bex/SBE/r310/src/APPH) and run the following command:

sun9:workdir/APPH$ make -f Makefile.apph

This generates APPH.jar in the directory jpylib.
[edit]
High Level Message Flow

This section describes a high level view of the messages that APPH sends and receives. For more details, corresponding project documents should be referred to. By default, apph implements apph service (apph.s). Similarly, app servers implement appserver service (appserver.s) and snap implements service vss (snap.s). Any exceptions will be explicitly mentioned. Also, the message flow assumes a non-cluster node for the most part. A separate section on clusters would detail the relevant changes.
[edit]
Browse

gui => apph: CONFIG_INFO
    -> master node, nibbler user name, password, and port number.

gui => apph: DISCOVER
    Create an xml document and add general information such as host name and
    IP.

    Obtain list of discover servers from ah_config.xml. Connect to each such
    server and send the following messages. Note that in case of cluster, apph
    connects to each discover server on each cluster node.

    apph => ssndmpc: QSERVER
    apph => ssndmpc: QCONFIG_FS
        <- returns list of file systems.

    Filter some file systems. For example, show only LVM file systems on
    Linux. Check the code for various filtering rules. Create one node in the
    xml document for each file system.

    apph => sssnap: DISCOVER (service vss)
        <- returns list of paths for each writer-id.

    Attach the paths to the xml document by creating required intermediate
    nodes. 

    apph => appservers: DISCOVER (service appserver)
        <- returns an xml string. 
    Attach the returned xml string to the document.

    <- Return the xml document

[edit]
Backup

svh => apph: CONFIG_INFO
    -> master node, nibbler user name, password, and port number.
    -> bex logical node name for this host
    -> bex configured host address 

svh => apph: PRE_BACKUP
    This message is sent only to apph on cluster virtual node. For more
    details, see the wiki document "ClusterAppsWithNonSharedDisks".
    The actions performed by apph for this message are a subset of those done
    for BACKUP and are explained there.

svh => apph: BACKUP
    -> jobdef. Note that the term jobdef as used in apph
    simply means list of paths to be backed up. e.g, /C:, /ORACLE/10g/db1
    etc. A null value implies whole node backup. 

    Do discover. This is exactly same as what happens during gui browse. The
    only exception is for cluster virtual node. In that case, svh sends list
    of cluster nodes which are used instead of the list discovered by apph at
    the time of startup. At the end of discover, an xml document called backup
    document is created with all the details about file systems and
    applications. This is the same document that is sent to gui on node
    browse. 

    Expand the jobdef using the information obtained during discover. The
    result of this step will be a list of complete paths to be backed up. For
    example, "/ORACLE" may be expanded to "/ORACLE/10g/db1, /ORACLE/10g/db2".

        The job will fail at this stage if any jobdef component can not
        be expanded. For example, if oracleapp does not return database info for
        whatever reason, "/ORACLE" can not be expanded. Similarly, if "/F:" is
        part of job def but F: is not returned by nibbler in discover, the job
        will fail.

    Convert the expanded jobdef into list of VVs and PVs. 

    For each non-VSS VV, do Prepare for Snap operation.
        apph => appservers: PFS_GENERIC
            -> backup document component
            -> list of paths 
            <- list of CVs (covering volumes)

    Combine all CVs and send BACKUP message to snap. 
        apph => sssnap: BACKUP
            -> List of volumes
            snap takes snapshot and does any required pre-snapshot VSS
            processing for apps.
            <- List of volumes. This list is union of the input CVs and any
            other CVs for VSS apps.  

    Create content files for each PV. 

    For all non-VSS VVs, do After Snap operation. 
        apph => appservers: AS_GENERIC
            -> snapshot status
            <- backup document component
            <- list of files to be backed up in APPS: task.

    Create ExpressDR files such volmap.txt.

    Create APPS content file. 

    <- list of PVs (including APPS:)
    <- list of VVs
    <- list of APPS: files.

svh => apph: UPDATE_BITMAP
    svh sends this message once for each PV.
    -> PV (eg, C:, D:)
    -> last successful jobid on secondary for this task

    apph => snap: UPDATE_BITMAP
        -> forward the mdata from svh.
        <- backup type: base or incr.

    <- forward snap's response

svh => apph: VOLUME_LUNINFO
    The message contains sufficient information for each PV such that IA map
    can be done.  

    apph => appservers: VOLUME_LUNINFO
        This message is sent to only those app servers that request
        it. Currently, both oracleapp and exprdrapp requests this message. 

svh => apph: GEN_APPS_CONTENT_FILE
    -> list of failed app disks

    This message is rarely sent. Its main purpose is when a task start failes
    for a given application (eg, due to license violation). We would like to
    remove the corresponding application's files from APPS task.

    for each application in the list, remove corresponding APPS files, if any
    and create the APPS: content file again.

svh => apph: GET_BACKUP_DOC
    -> list of disks for which data transfer failed. The list will also
    include application disks in that case. SVH would include *all*
    applications as it does not have information about application => PV
    mapping.  

    Perform application verification if enabled. For more details see the wiki
    document "VerificationDesign"

    apph => sssnap: GET_BACKUP_DOC
        -> list of failed disks
        <- For each writer, returns FH and a file where backup document is 
        saved.  

    apph reads the files where snap's backup documents are stored, coverts the
    binary data to base64 encoded text, and adds it to the backup document. 

    zip up all the ExpressDR files, encode the archive in base64 and add it to
    the backup document. This happens only on Windows at present. For more
    details, see the wiki document "VirtualImpl".

svh => apph: GET_APP_FH
    <- return list of FH.

    Please see the wiki document "VerificationDesign" for more details. This
    message has been introduced as part of that project.

svh => apph: BACKUP_STATUS
    -> status - 0 for success. If data transfer failed for any one PV, this
    field would be set to -1.  
    -> failed disk list as in GET_BACKUP_DOC. 

    apph => appservers: BACKUP_STATUS
        -> status

        oracleapp does lot of work during this message. If the status is -1,
        nothing is done however. Otherwise, archive logs are truncated (based
        on user option). Also, RMAN cataloging is performed. 

        EXCH2Kapp too truncates logs at this time (again based on 'status'
        value). 

    apph => sssnap: BACKUP_SESS_CLEANUP
        -> status
        -> failed disk list

        With the latest changes for app verification, sssnap doesn't do any
        work in this message.

[edit]
Restore

APPH comes into picture only in case of application restores. Its involvement in physical file restore is limited to two utility messages - GEN_RESTORE_CONTENT_FILE and GET_FS_HOSTING_NODE.

svh => apph: CONFIG_INFO
    -> master node, nibbler user name, password, and port number.
    -> bex logical node name for this host
    -> bex configured host address 

svh => apph: RESTORE
    -> backup document
    -> jobdef

    The processing for restore is very similar to backup. The jobdef is
    resolved to set of VVs. For each such VV, the corresponding app server or
    snap is obtained from backup document.

    For each VV, do Prepare for Restore operation.
        apph => appservers: PFR_GENERIC
            -> backup document component
            -> list of paths 

            <- A list of restore files (roughly a tuple of source and
            destination files).

        apph => snap: PREP_FOR_RESTORE
            -> backup document component for the writer
            -> list of paths 

            <- A list of restore files (roughly a tuple of source and
            destination files).

    <- list of VVs and for each VV, list of restore files.

svh => apph: GEN_RESTORE_CONTENT_FILE
    -> file contents
    -> file name

    apph merely creates the file with the given contents. 

svh => apph: GET_FS_HOSTING_NODE
    -> list of PVs (eg, R:, S:)

    This message is sent by svh only for cluster volumes. In that case, svh
    needs to know where a given cluster file system is hosted before
    initiating restore from that node.

    apph performs discover only using nibber. It will then check the node that
    returned given PVs.

    <- list of "vol" mdatas containing the hostname. 

svh => apph: VOLUME_LUNINFO
    See the description in Backup section.

svh => apph: NODE_INFO
    -> node mdata

    This message is sent for each secondary node. 

    apph => appservers: NODE_INFO
        -> forward the mdata received from svh.

svh => apph: POST_RESTORE
    -> restore status

    For each VV, do After Restore operation.
        apph => appservers: AR_GENERIC
            -> status

        apph => snap: AFTER_RESTORE
            -> status

svh => apph: GET_RESTORE_STATUS
    -> jobid

    This message is only used in app verification restore. Check the wiki
    document "VerificationDesign" for more details.

    <- list of VVs and for each VV, restore status.

[edit]
Cluster Support

A cluster is represented by N+1 nodes in Bex enterprise where there are N physical nodes and one cluster virtual node representing all the cluster resources (Both file systems and applications). The cluster virtual node is referred to as CV node while the cluster physical nodes are referred as CP nodes. For the sake of completeness, non-cluster nodes are called NC nodes.
[edit]
Browse

Check #Browse which describes non-cluster node browse. In case of cluster, the same logic applies but apph connects to all discover servers on all cluster nodes. This list of cluster nodes is obtained by apph at the time of coming up by sending Q_CLUST_NODES message to node browser.
[edit]
Backup

The backup follows the general outline described in #Backup with the following additional points.

    Each PV or VV has a hosting node. This is the node where PV or VV is hosted. The hosting node is determined at discover time and an attribute is set in XML backup document. Basically if a discover server on a node N returns a given VV or PV, N is the hosting node for that VV or PV. 

    All backup and restore operations for a given VV or PV are performed on the corresponding hosting nodes. For example, if R: is on N1 and S: is on N2 (where R: and S: are cluster volumes), R: will be sent to SNAP@N1 and S: will be sent to SNAP@N2 in BACKUP message for snapshotting. Same type of partitioning happens in other messages as well. 

    Snap creates its (binary) backup documents on respective hosting nodes as files and APPH@CV needs to read these files in order to add them to the backup document. It uses UNC paths to access the files on other nodes. For this to work, "file & print sharing" must be enabled on the Windows cluster. See bug 1383 for some details. 

    Some app servers (such as ExpressDR app server) create files that need to be backed up in APPS:. These files are created on respective nodes and the APPS backup is done from the CV node. In order for nibbler on CV node to be able to access these files, APPH converts these files to their UNC paths. 

Note that some changes have been made to support Exchange CCR clusters and the information about those changes is at ClusterAppsWithNonSharedDisks.
[edit]
Restore

Cluster restore follows the general outline described at #Restore with the following additional notes.

    A discover is performed in RESTORE message processing in order to find the current hosted node for each VV.
    And all further restore messages are sent to modules on respective hosting nodes.
    For PV restores, SVH invokes apph's interface GET_FS_HOSTING_NODE in order to find the current hosting node of the cluster volume. 

[edit]
General Concepts
[edit]
Backup Document Construction

At the end of the backup, an XML document is generated by apph with inputs from app servers. This document has all the information about backed up objects that will be required at the restore time. This is called backup document and is stored in DB (in db/APPS directory). SVH finds the corresponding document at restore time and sends it to apph.

    During backup, discover is done during which:
        all app servers respond with an xml string which gets attached to the backup doc.
        An element is created for each physical volume returned by nibbler.
        snap returns list of paths. There are converted to xml hierarchy with each path component corresponding to one xml element. 

    app servers will return their backup document in AfterSnap response (as xml string) which will replace the xml section sent in discover.
    snap sends one document for each writer and one global document. They are sent as file names and the files contain binary data. APPH reads the data and converts them to text (using base64 encoding) before saving them at /disc_resp/info/vss_writer and /disc_resp/info/vss_backup_doc.
    APPH would also save some information about the backup job at /disc_resp/docinfo. 

[edit]
Content Files

During backup, APPH creates Content Files for each physical volume (including APPS). The files are located in prodir/tmp and are named with the format <mangled_unique_id>.<jobname>.<jobid>, where <mangled_unique_id> is the mangled form of unique-id returned by snap. The name for APPS content file is APPS-<nodetype>.<jobname>, where <nodetype> can be one of CP (cluster physical), CV (cluster virtual), or NC (non-cluster). The <nodetype> is required to distinguish APPS content files when cluster virtual and cluster physical nodes are being backed up as part of the same job (which is typically the case). The format of these content files is described here. These files are only read by nibbler and is the only way for any kind of information to be passed to nibbler. The function AHSession.py:sssnap_sess.create_content_files() has the code to create these files.

Content files are created during restore as well but the actual contents are constructed by SVH. Note that the SVH uses the APPH interface gen_restore_content_file to actually create the physical file on the primary. Also, the name used <mangled_logical_name>.<jobname> is slightly different from that at backup time.
[edit]
Application versioning

In XRS/SNAPVAULT, we have a special format for application paths in the way they are displayed and cataloged. Each application path must have at least two components. The first one is the app name itself such as ORACLE or SYSTEMTABLE. The second component is version. For some application, the version is obvious. For example, ORACLE can have 10g, 9i, or 11g. For some other applications which are kind of pseudo-apps like ExpressDR and SYSTEMSTATE, we manufacture a version. For ExpressDR, it has been 1.0. There are places in APPH where this assumption of two components is there and all new applications must follow this convention.

More over, as part of VirtualImpl project, SVH will be storing this version in NDMPDATA which then will be used by GUI to make some judgments. For example, ExpressDR 2.0 backups can be virtualized while previous versions can't be. This is a general mechanism and all applications can use this framework to indicate enhancements in the backup.

Example application paths:

/ORACLE/10g/orcl
/ExpressDR/1.0
/SQL/2000/sqlinst/sqldb

[edit]
ah_config.xml

APPH reads this file (present in SSPRODIR/bin) at the beginning when it comes up. It is in XML format and contains information about "discover" servers and "appserver"s for each platform supported. A typical ah_config.xml looks like the following:

<Apphelper_Config version="1.0" >

	<OS name="Windows 2000">
		<disc_servers>

			<server name="SQLdisc" cluster_physical="0" 
				cluster_virtual="1" non_cluster="1" proxy="1"> 
			</server>

			<server name="EXCH2Kapp" cluster_physical="0" 
				cluster_virtual="1" non_cluster="1" proxy="1"> 
			</server>

			<server name="ssexprdr" cluster_physical="0" 
				cluster_virtual="0" non_cluster="1" proxy="0"> 
			</server>

			<server name="orclapp" cluster_physical="0" 
				cluster_virtual="0" non_cluster="1" proxy="0"> 
			</server>

		</disc_servers>
      
		<App name="/SQL/2000" app_server="SQL2Kapp" />
	</OS>

	<OS name="Windows XP">
		<disc_servers>

			<server name="sssnap" cluster_physical="1" 
				cluster_virtual="1" non_cluster="1" proxy="1"> 
			</server>


			<server name="ssexprdr" cluster_physical="0" 
				cluster_virtual="0" non_cluster="1" proxy="0"> 
			</server>

		</disc_servers>

		<App name="/SQL/2000" app_server="SQL2Kapp" />
	</OS>

	<OS name="Windows 2003">
		<disc_servers>

			<server name="sssnap" cluster_physical="1" 
				cluster_virtual="1" non_cluster="1" proxy="1"> 
			</server>

			<server name="ssexprdr" cluster_physical="1" 
				cluster_virtual="1" non_cluster="1" proxy="1"> 
			</server>

			<server name="orclapp" cluster_physical="0" 
				cluster_virtual="0" non_cluster="1" proxy="0"> 
			</server>

		</disc_servers>
	</OS>

	<OS name="Windows Vista">
		<disc_servers>

			<server name="sssnap" cluster_physical="1" 
				cluster_virtual="1" non_cluster="1" proxy="1"> 
			</server>

			<server name="ssexprdr" cluster_physical="1" 
				cluster_virtual="1" non_cluster="1" proxy="1"> 
			</server>

			<server name="orclapp" cluster_physical="0" 
				cluster_virtual="0" non_cluster="1" proxy="0"> 
			</server>

		</disc_servers>
	</OS>

	<OS name="Windows NT">
		<disc_servers>
		</disc_servers>
	</OS>

	<OS name="Linux">
        <disc_servers>

            <server name="orclapp" cluster_physical="0"
                cluster_virtual="0" non_cluster="1" proxy="0">
            </server>

            <server name="ssexprdr" cluster_physical="1" 
				cluster_virtual="1" non_cluster="1" proxy="1"> 
			</server>


        </disc_servers>
    </OS>

	<OS name="SunOS">
            <disc_servers>

                <server name="orclapp" cluster_physical="0"
                    cluster_virtual="0" non_cluster="1" proxy="0">
                </server>

				<server name="ssexprdr" cluster_physical="0" 
					cluster_virtual="0" non_cluster="1" proxy="0"> 
				</server>

            </disc_servers>
    </OS>

	<disc_servers>

		<server name="ssndmpc" cluster_physical="1" 
			cluster_virtual="1" non_cluster="1" proxy="1"> 
		</server>

	</disc_servers>

	<appservers servers="SQLdisc,SQL2Kapp,EXCH2Kapp,ssexprdr,orclapp" />

</Apphelper_Config>

Each "server" element has attributes to indicate whether the server should be contacted for browse on different type of nodes - cluster physical, cluster virtual, and non-cluster. More over, There is a section for each OS and there is a common section (containing ssndmpc) so it is possible to configure different discover servers for different Operating Systems.

The element "appservers" contains list of app servers that APPH needs to connect to in single-use mode.
[edit]
APPS Task

OSSV/XRS backup is mainly about backing up snapshotted file systems. How ever, the architecture also has the provision to backup individual files. There are various reasons why we need this feature:

    Some applications like Oracle generate log files after snapshot process.
    ExpressDR generates a bunch of files and APPH in turn creates some other files for ExpressDR.
    A hidden partition is backed up as a file (DRUtilityPartition). 

In order to support such file backup, a virtual task called "APPS:" is created by SVH during backup. Note that this task is not shown in restore screen and is only used internally.
[edit]
Implementation Details
[edit]
Cancel Handler

During various operations, APPH connects to various modules and send messages to them and then waits for their response. The thread waiting can indefinitely wait if the target module doesn't respond for whatever reason. In this condition, even if SVH cancels the job by closing APPH connection, APPH would still be stuck as it is waiting for response from another module. In order to support such cancel operation, APPH installs a cancel handler on all of its connections. Any time APPH waits for a response, the cancel handler is invoked every 2 minutes (by default) which then checks the state of SVH connection. If the connection is not closed, the processing returns to wait but if the connection is closed, the receive call is failed and the regular error processing kicks in. For more details, see I3010 and I3368.
[edit]
Features

This section describes some features implemented by APPH which do not strictly fall under backup and restore category.
[edit]
Script Support

APPH supports three types of scripts in OSSV/XRS backups (in addition to pre and post job scripts). They are

    pre-snap
    post-snap
    post-data transfer 

APPH checks for script information in the file <SSPRODIR>/bin/scrinfo.txt. The format of this file is as follows:

    blank lines and lines starting with "#" are ignored.
    all other lines are of the format: name1=val1<>name2=val2<>name3=val3...
    Recognized "names" are:
        cmd : actual command to be executed. This is sent to "syncrtm".
        node : node where syncrtm is brought up. default is current node.
        type : There are three types of scripts. "presnap", "postsnap", and "postxfer". These are executed before snapshots are taken, after snapshots are taken, and after data transfter is done respectively. Note that "postsnap" and "postxfer" scripts are executed even if snapshot and data transfer operations fail. To indicate the status of actual operation, "1" or "0" is added to the "postsnap" and "postxfer" commands. 1 indicates success and 0 indicates failure. It is upto the scripts to interpret this option and behave accordingly.
        jobname : If specified, this script is applicable only for this particular job. Otherwise, it will be applicable for all backup jobs.
        abort_on_fail : By default, the job continues even if a script fails (return status non-zero). If you want job to fail, set this to 1. Note that this option has no meaning for "postxfer" scripts. 
    "=" and "<>" are used as delimiters. As such, they can not be used in the actual fields. 

Example scrinfo.txt

# example script info file for Backup Express Snapvault backup jobs

# run "test1" before snapshotting for the job "3_J-scr"
cmd=test1<>type=presnap<>jobname=3_J-scr

# run "test2" after snapshots are taken for all backup jobs. fail the job
# if the script fails.
cmd=test2<>type=postsnap<>abort_on_fail=1

# run "test3" after data transfter is done for all backup jobs. 
cmd=test3<>type=postxfer

[edit]
Log and Temp file Cleanup

APPH removes old log files in SSPRODIR/logs and SSPRODIR/tmp. For cleaning up "logs" directly, APPH sends INIT message to EH in a separate (daemon) thread at the time of coming up. For details on "tmp" cleanup, please see TempFilesCleanup.
[edit]
ExpressDR

Even though ExpressDR app server behaves as any other app server such as oracleapp, APPH bestows a special status to ExpressDR application. ExpressDR app server creates some files such as bmr_info.txt and in addition, APPH creates a bunch of some other files when ExpressDR is part of the backup. For more details, please see ExpressDR. Also, if ExpressDR is selected, APPH automatically adds the applications SYSTEMSTATE and SYSTEMTABLE to the backup job on windows.
[edit]
Links 
