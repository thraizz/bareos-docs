API {#sec:api}
==============

General
-------

This document is intended mostly for developers who wish to develop a
new GUI interface to <span>**Bareos**</span>.

### Minimal Code in Console Program

All the Catalog code is in the Directory (with the
exception of ```dbcheck``` and ```bscan```).
Therefore also user level security and access is implemented in this central place.
If code would be spreaded
everywhere such as in a GUI this will be more difficult.
The other
advantage is that any code you add to the Director is automatically
available to all interface programs, like the tty console and other programs.

### GUI Interface is Difficult

Interfacing to an interactive program such as Bareos can be very
difficult because the interfacing program must interpret all the prompts
that may come. This can be next to impossible. There are are a number of
ways that Bareos is designed to facilitate this:

-   The Bareos network protocol is packet based, and thus pieces of
    information sent can be ASCII or binary.

-   The packet interface permits knowing where the end of a list is.

-   The packet interface permits special “signals” to be passed rather
    than data.

-   The Director has a number of commands that are non-interactive. They
    all begin with a period, and provide things such as the list of all
    Jobs, list of all Clients, list of all Pools, list of all Storage,
    ... Thus the GUI interface can get to virtually all information that
    the Director has in a deterministic way. See
    <https://github.com/bareos/bareos/blob/master/src/dird/ua_dotcmds.c>
    for more details on this.

-   Most console commands allow all the arguments to be specified on the
    command line: e.g. ```run job=NightlyBackup level=Full```

One of the first things to overcome is to be able to establish a
conversation with the Director. Although you can write all your own
code, it is probably easier to use the Bareos subroutines. The following
code is used by the Console program to begin a conversation.

    static BSOCK *UA_sock = NULL;
    static JCR *jcr;
    ...
      read-your-config-getting-address-and-pasword;
      UA_sock = bnet_connect(NULL, 5, 15, "Director daemon", dir->address,
                              NULL, dir->DIRport, 0);
       if (UA_sock == NULL) {
          terminate_console(0);
          return 1;
       }
       jcr.dir_bsock = UA_sock;
       if (!authenticate_director(\&jcr, dir)) {
          fprintf(stderr, "ERR=%s", UA_sock->msg);
          terminate_console(0);
          return 1;
       }
       read_and_process_input(stdin, UA_sock);
       if (UA_sock) {
          bnet_sig(UA_sock, BNET_TERMINATE); /* send EOF */
          bnet_close(UA_sock);
       }
       exit 0;

Then the read\_and\_process\_input routine looks like the following:

       get-input-to-send-to-the-Director;
       bnet_fsend(UA_sock, "%s", input);
       stat = bnet_recv(UA_sock);
       process-output-from-the-Director;

For a GUI program things will be a bit more complicated. Basically in
the very inner loop, you will need to check and see if any output is
available on the UA\_sock.

dot commands 
------------

Besides the normal commands (like list, status, run, mount, ...)
the Director offers a number of so called *dot commands*.
They all begin with a period, are all non-interactive, easily parseable and are indended to be used by other Bareos interface programs (GUIs).

See <https://github.com/bareos/bareos/blob/master/src/dird/ua_dotcmds.c> for more details.

* ```.actiononpurge```
* .```api [ 0 | 1 | 2 | off | on | json ]```
    * Switch between different [api modes](#sec:ApiMode)
* ```.clients```
    * List all client resources
* ```.catalogs```
    * List all catalog resources
* ```.defaults job=<job-name> | client=<client-name> | storage=<storage-name | pool=<pool-name>```
    * List default settings
* ```.filesets```
    * List all filesets
* ```.help [ all | item=cmd ]```
    * Print parsable information about a command
* ```.jobdefs```
    * List add JobDef resources
* ```.jobs```
    * List job resources
* ```.levels```
    * List all backup levels
* ```.locations```
* ```.messages```
* ```.media```
    * List all medias
* ```.mediatypes```
    * List all media types
* ```.msgs```
    * List all message resources
* ```.pools```
    * List all pool resources
* ```.profiles```
    * List all profile resources
* ```.quit```
    * Close connection
* ```.sql query=<sqlquery>```
    * Send an arbitary SQL command
* ```.schedule```
    * List all schedule resources
* ```.status```
* ```.storages```
    * List all storage resources
* ```.types```
    * List all job types
* ```.volstatus```
    * List all volume status
* ```.bvfs_lsdirs```
* ```.bvfs_lsfiles```
* ```.bvfs_update```
* ```.bvfs_get_jobids```
* ```.bvfs_versions```
* ```.bvfs_restore```
* ```.bvfs_cleanup```
* ```.bvfs_clear_cache```

API Modes {#sec:ApiMode}
---------

The ```.api``` command can be used to switch between the different API modes.
Besides the ```.api``` command, there is also the ```gui on | off``` command.
However, this command can be ignored, as it set to gui on in command execution anyway.

### API mode 0 (off)

```
.api 0
```

By default, a console connection to the Director is in interactive mode, meaning the api mode is off.
This is the normal mode you get when using the bconsole.
The output should be human readable.


### API mode 1 (on)

To get better parsable output, a console connection could be switched to API mode 1 (on).

```
.api 1
```

or (form times where they have only been one API flavour)

```
.api
```

This mode is intended to create output that is earlier parsable.
Internaly some commands vary there output for the API mode 1, but not all.

In API mode 1 some output is only delimted by the end of a packet, by not a new line.
bconsole does not display end of packets (for good reason, as some output (e.g. ```status```) is send in multiple packets).
If running in a bconsole, this leads not parsable output for human.

Example:
```
*.api 0
api: 0
*.defaults job=BackupClient1
job=BackupClient1
pool=Incremental
messages=Standard
client=client1.example.com-fd
storage=File
where=
level=Incremental
type=Backup
fileset=SelfTest
enabled=1
catalog=MyCatalog
*.api 1
api: 1
*.defaults job=BackupClient1
job=BackupClient1pool=Incrementalmessages=Standardclient=client1.example.com-fdstorage=Filewhere=level=Incrementaltype=Backupfileset=SelfTestenabled=1catalog=MyCatalog
```

This mode is used by BAT.

* [Signals](#sec:bnet_sig)

### API mode 2 (json)

The API mode 2 (or JSON mode) has been introduced in Bareos-15.2 and differs from API mode 1 in several aspects:

* JSON output
* The JSON output is in the format of JSON-RPC 2.0 responce objects (<http://www.jsonrpc.org/specification#response_object>). This should make it easier to implement a full JSON-RPC service later.
* No user interaction inside a command (meaning: if not all parameter are given to a `run` command, the command fails).
* Each command creates exaclty one responce object.

Currently a subset of the available commands return there result in JSON format, while others still write plain text output.
When finished, it should be safe to run all commands in JSON mode.

A successful responce should return 
```
"result": {
    "<type_of_the_results>": [
        {
            <result_object_1_key_1>: <result_object_1_value_1>,
            <result_object_1_key_2>: <result_object_1_value_2>,
            ...
        },
        {
            <result_object_2_key_1>: <result_object_2_value_1>,
            <result_object_2_key_2>: <result_object_2_value_2>,
            ...
        },
        ...
    ]
}
```

All keys are lower case.

#### Examples

* list
    * e.g.
```
*list jobs
{
  "jsonrpc": "2.0",
  "id": null,
  "result": {
    "jobs": [
      {
        "type": "B",
        "starttime": "2015-06-25 16:51:38",
        "jobfiles": "18",
        "jobid": "1",
        "name": "BackupClient1",
        "jobstatus": "T",
        "level": "F",
        "jobbytes": "4651943"
      },
      {
        "type": "B",
        "starttime": "2015-06-25 17:25:23",
        "jobfiles": "0",
        "jobid": "2",
        "name": "BackupClient1",
        "jobstatus": "T",
        "level": "I",
        "jobbytes": "0"
      },
      ...
    ]
  }
}
```
    * keys are the table names
* llist
    * e.g.
```
*llist jobs
{
  "jsonrpc": "2.0",
  "id": null,
  "result": {
    "jobs": [
      {
        "name": "BackupClient1",
        "realendtime": "2015-06-25 16:51:40",
        "Type": "B",
        "schedtime": "2015-06-25 16:51:33",
        "poolid": "1",
        "level": "F",
        "jobfiles": "18",
        "volsessionid": "1",
        "jobid": "1",
        "job": "BackupClient1.2015-06-25_16.51.35_04",
        "priorjobid": "0",
        "endtime": "2015-06-25 16:51:40",
        "jobtdate": "1435243900",
        "jobstatus": "T",
        "jobmissingfiles": "0",
        "joberrors": "0",
        "purgedfiles": "0",
        "starttime": "2015-06-25 16:51:38",
        "clientname": "ting.dass-it-fd",
        "clientid": "1",
        "volsessiontime": "1435243839",
        "filesetid": "1",
        "poolname": "Full",
        "fileset": "SelfTest"
      },
      {
        "name": "BackupClient1",
        "realendtime": "2015-06-25 17:25:24",
        "type": "B",
        "schedtime": "2015-06-25 17:25:10",
        "poolid": "3",
        "level": "I",
        "jobfiles": "0",
        "volsessionid": "2",
        "jobid": "2",
        "job": "BackupClient1.2015-06-25_17.25.20_04",
        "priorjobid": "0",
        "endtime": "2015-06-25 17:25:24",
        "jobtdate": "1435245924",
        "jobstatus": "T",
        "jobmissingfiles": "0",
        "JobErrors": "0",
        "purgedfiles": "0",
        "starttime": "2015-06-25 17:25:23",
        "clientname": "ting.dass-it-fd",
        "clientid": "1",
        "volsessiontime": "1435243839",
        "filesetid": "1",
        "poolname": "Incremental",
        "fileset": "SelfTest"
      },
      ...
    ]
  }
}
```
    * like the list ```command```, but more values
* .jobs
    * e.g.
```
*.jobs
{
  "jsonrpc": "2.0",
  "id": null,
  "result": {
    "jobs": [
      {
        "name": "BackupClient1"
      },
      {
        "name": "BackupCatalog"
      },
      {
        "name": "RestoreFiles"
      }
    ]
  }
}
```

##### Example of a JSON-RPC Error Response

Example of a JSON-RPC Error Response (<http://www.jsonrpc.org/specification#error_object>):
```
*gui
{
  "jsonrpc": "2.0",
  "id": null,
  "error": {
    "data": {
      "result": {},
      "messages": {
        "error": [
          "ON or OFF keyword missing.\n"
        ]
      }
    },
    "message": "failed",
    "code": 1
  }
}
```

  * an error response is emitted, if the command returns false or emitted an error message (```void UAContext::error_msg(const char *fmt, ...)```). Messages and the result so far will be part of the error response object.


Bvfs API {#sec:bvfs}
--------

The BVFS commands are do provide a API browsing the catalog, which is required for restoring.

Bat has now a bRestore panel that uses Bvfs to display files and
directories.

![image](\idir bat-brestore) [fig:batbrestore]

The Bvfs module works correctly with BaseJobs, Copy and Migration jobs.

The initial version in Bacula have be founded by Bacula Systems.

### General notes {#general-notes .unnumbered}

-   All fields are separated by a tab (api mode 0 and 1)

-   You can specify `limit=` and `offset=` to list smoothly records in
    very big directories

-   All operations (except cache creation) are designed to run instantly

-   The cache creation is dependent of the number of directories. As
    Bvfs shares information accross jobs, the first creation can be slow

-   Due to potential encoding problem, it’s advised to allways use
    pathid in queries.

### Get dependent jobs from a given JobId {#get-dependent-jobs-from-a-given-jobid .unnumbered}

Bvfs allows you to query the catalog against any combination of jobs.
You can combine all Jobs and all FileSet for a Client in a single
session.

To get all JobId needed to restore a particular job, you can use the
`.bvfs_get_jobids` command.

```
.bvfs_get_jobids jobid=num [all]
```

Example:
```
*.bvfs_get_jobids jobid=10
1,2,5,10
*.bvfs_get_jobids jobid=10 all
1,2,3,5,10
```

In this example, a normal restore will need to use JobIds 1,2,5,10 to
compute a complete restore of the system.

With the `all` option, the Director will use all defined FileSet for
this client.

### Generating Bvfs cache {#generating-bvfs-cache .unnumbered}

The `.bvfs_update` command computes the directory cache for jobs
specified in argument, or for all jobs if unspecified.

```
.bvfs_update [jobid=numlist]
```

Example:
```
*.bvfs_update jobid=1,2,3
```


You can run the cache update process in a RunScript after the catalog
backup.

### Get all versions of a specific file {#get-all-versions-of-a-specific-file .unnumbered}

Bvfs allows you to find all versions of a specific file for a given
Client with the `.bvfs_version` command. To avoid problems with
encoding, this function uses only PathId and FilenameId. The jobid
argument is mandatory but unused.

```
*.bvfs_versions client=filedaemon pathid=num filenameid=num jobid=1
PathId FilenameId FileId JobId LStat Md5 VolName Inchanger
PathId FilenameId FileId JobId LStat Md5 VolName Inchanger
...
```


Example:

```
*.bvfs_versions client=localhost-fd pathid=1 fnid=47 jobid=1
1  47  52  12  gD HRid IGk D Po Po A P BAA I A   /uPgWaxMgKZlnMti7LChyA  Vol1  1
```


### List directories {#list-directories .unnumbered}

Bvfs allows you to list directories in a specific path.

    *.bvfs_lsdirs pathid=num path=/apath jobid=numlist limit=num offset=num
    PathId  FilenameId  FileId  JobId  LStat  Path
    PathId  FilenameId  FileId  JobId  LStat  Path
    PathId  FilenameId  FileId  JobId  LStat  Path
    ...

You need to `pathid` or `path`. Using `path=` will list “/” on Unix and
all drives on Windows. If FilenameId is 0, the record listed is a
directory.

    *.bvfs_lsdirs pathid=4 jobid=1,11,12
    4       0       0       0       A A A A A A A A A A A A A A     .
    5       0       0       0       A A A A A A A A A A A A A A     ..
    3       0       0       0       A A A A A A A A A A A A A A     regress/

In this example, to list directories present in `regress/`, you can use

    *.bvfs_lsdirs pathid=3 jobid=1,11,12
    3       0       0       0       A A A A A A A A A A A A A A     .
    4       0       0       0       A A A A A A A A A A A A A A     ..
    2       0       0       0       A A A A A A A A A A A A A A     tmp/

### List files {#list-files .unnumbered}

#### API mode 0

Bvfs allows you to list files in a specific path.

    .bvfs_lsfiles pathid=num path=/apath jobid=numlist limit=num offset=num
    PathId  FilenameId  FileId  JobId  LStat  Path
    PathId  FilenameId  FileId  JobId  LStat  Path
    PathId  FilenameId  FileId  JobId  LStat  Path
    ...

You need to `pathid` or `path`. Using `path=` will list “/” on Unix and
all drives on Windows. If FilenameId is 0, the record listed is a
directory.

    *.bvfs_lsfiles pathid=4 jobid=1,11,12
    4       0       0       0       A A A A A A A A A A A A A A     .
    5       0       0       0       A A A A A A A A A A A A A A     ..
    1       0       0       0       A A A A A A A A A A A A A A     regress/

In this example, to list files present in `regress/`, you can use

    *.bvfs_lsfiles pathid=1 jobid=1,11,12
    1   47   52   12    gD HRid IGk BAA I BMqcPH BMqcPE BMqe+t A     titi
    1   49   53   12    gD HRid IGk BAA I BMqe/K BMqcPE BMqe+t B     toto
    1   48   54   12    gD HRie IGk BAA I BMqcPH BMqcPE BMqe+3 A     tutu
    1   45   55   12    gD HRid IGk BAA I BMqe/K BMqcPE BMqe+t B     ficheriro1.txt
    1   46   56   12    gD HRie IGk BAA I BMqe/K BMqcPE BMqe+3 D     ficheriro2.txt


#### API mode 1

```
*.api 1
*.bvfs_lsfiles jobid=1 pathid=1
1   11  7   1   gD OEE4 IHo B GHH GHH A G9S BAA 4 BVjBQG BVjBQG BVjBQG A A C    bpluginfo
1   12  4   1   gD OEE3 KH/ B GHH GHH A W BAA A BVjBQ7 BVjBQG BVjBQG A A C  bregex
...
```

#### API mode 2

```
*.api 2
*.bvfs_lsfiles jobid=1 pathid=1
{
  "jsonrpc": "2.0",
  "id": null,
  "result": {
    "files": [
      {
        "jobid": 1,
        "type": "F",
        "fileid": 7,
        "lstat": "gD OEE4 IHo B GHH GHH A G9S BAA 4 BVjBQG BVjBQG BVjBQG A A C",
        "pathid": 1,
        "stat": {
          "atime": 1435243526,
          "ino": 3686712,
          "dev": 2051,
          "mode": 33256,
          "gid": 25031,
          "nlink": 1,
          "uid": 25031,
          "ctime": 1435243526,
          "rdev": 0,
          "size": 28498,
          "mtime": 1435243526
        },
        "filenameid": 11,
        "name": "bpluginfo",
        "linkfileindex": 0
      },
      {
        "jobid": 1,
        "type": "F",
        "fileid": 4,
        "lstat": "gD OEE3 KH/ B GHH GHH A W BAA A BVjBQ7 BVjBQG BVjBQG A A C",
        "pathid": 1,
        "stat": {
          "atime": 1435243579,
          "ino": 3686711,
          "dev": 2051,
          "mode": 41471,
          "gid": 25031,
          "nlink": 1,
          "uid": 25031,
          "ctime": 1435243526,
          "rdev": 0,
          "size": 22,
          "mtime": 1435243526
        },
        "filenameid": 12,
        "name": "bregex",
        "linkfileindex": 0
      },
      ...
    ]
  }
}
```

API mode JSON contains all information also available in the other API modes, but displays them more verbose.

### Restore set of files {#restore-set-of-files .unnumbered}

Bvfs allows you to create a SQL table that contains files that you want
to restore. This table can be provided to a restore command with the
file option.

    *.bvfs_restore fileid=numlist dirid=numlist hardlink=numlist path=b2num
    OK
    *restore file=?b2num ...

To include a directory (with `dirid`), Bvfs needs to run a query to
select all files. This query could be time consuming.

`hardlink` list is always composed of a serie of two numbers (jobid,
fileindex). This information can be found in the LinkFileIndex (LinkFI) field of the
LStat packet.

The `path` argument represents the name of the table that Bvfs will
store results. The format of this table is `b2[0-9]+`. (Should start by
b2 and followed by digits).

Example:

    *.bvfs_restore fileid=1,2,3,4 hardlink=10,15,10,20 jobid=10 path=b20001
    OK

### Cleanup after Restore {#cleanup-after-restore .unnumbered}

To drop the table used by the restore command, you can use the
`.bvfs_cleanup` command.

    *.bvfs_cleanup path=b20001

### Clearing the BVFS Cache {#clearing-the-bvfs-cache .unnumbered}

To clear the BVFS cache, you can use the `.bvfs_clear_cache` command.

    *.bvfs_clear_cache yes
    OK