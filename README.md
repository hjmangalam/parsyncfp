# parsyncfp
a parallel rsync wrapper in Perl. Released under GPL v3.

(Version changes moved to the bottom of this file)

## See Changes at bottom.  

## PARSYNCFP2 ANNOUNCEMENT 
Feb 24, 2022
Please check out the latest in the parsync family: **parsyncfp2** (aka pfp2 - note the '2'). Version 2, besides 
benefiting from a number of bug-fixes and improvements vs parsyncfp (aka pfp), now allows MultiHost sends.
It's still a single executable, still pretty simple, but now can spread itself over multiple hosts to 
send to single or multiple hosts with identical or multiple endpoints.  
[Please check it out](https://github.com/hjmangalam/parsyncfp2).

## Background

NB: If you don't want to transfer at least 10s of GB across a network, this is probably not the 
the tool you want.  Use rsync alone if a sync, 'tar and netcat' if a fresh copy, or scp if the
copy needs to be encrypted.


parsyncfp is the offspring of my [parsync](https://github.com/hjmangalam/parsync) 
 ([more info here](http://moo.nac.uci.edu/~hjm/parsync/))
 and Ganael LaPlanche's fpart [1](https://github.com/martymac/fpart) [2](https://goo.gl/K1WwtD), which
collects files based on size or number into chunkfiles which can be fed to rsync on a 
chunk by chunk basis.  This allows pfp to begin transferring files before the 
complete recursive descent of the source dir is complete.  This feature can save many 
hours of prep time on very large dir trees.

If your use involves transit over IB networks, parsyncfp requires 'perfquery' and 'ibstat', 
Infiniband utilities written by Hal Rosenstock < hal.rosenstock [at] gmail.com > 

pfp is tested on Linux.  The MacOSX port is in hibernation.

pfp needs to be installed only on the SOURCE end of the 
transfer and only works in local SOURCE -> remote TARGET mode 
(it won't allow remote local SOURCE <- remote TARGET, emitting an 
error and exiting if attempted). It requires that ssh shared keys be
set up prior to operation [see here](https://goo.gl/ghCazV).  If it detects 
that ssh keys are NOT set up correctly, it will ask for permission to try
to remedy that situation.  Check your local and remote ssh keys to make
sure that it has done so correctly.  Typically, they're in your ~/.ssh dir.

It uses whatever rsync is available on the TARGET.  It uses a number 
of Linux-specific utilities so if you're transferring between Linux 
and a FreeBSD host, install parsyncfp on the Linux side. 

The only native rsync options that pfp uses is '-a' (archive) &
'-s' (respect odd characters in filenames).  If you need to define 
the rsync options differently, then it's up to you to define ALL of 
them via '--rsyncopts' (the default '-a -s' flags will NOT be provided 
automatically.

pfp checks to see if the current system load is too heavy and tries
to throttle the rsyncs during the run by monitoring and suspending / 
continuing them as needed. There is a 'suspend.log' file in the 
~/.parsyncfp dir that tracks these transitions.

It appropriates rsync's bandwidth throttle mechanism, using '--maxbw'
as a passthru to rsync's 'bwlimit' option, but divides it by NP so
as to keep the total bw the same as the stated limit.  It monitors and
shows network bandwidth, but can't change the bw allocation mid-job.
It can only suspend rsyncs until the load decreases below the cutoff.
(In the single host version of pfp, if you suspend parsyncfp (^Z), all 
rsync children will suspend as well, regardless of current state.) In the 
Multihost version, the SEND hosts are independent so ^C & ^Z will have no 
effect on them.  A socket-based control system is next.

Unless changed by '--interface', it tries to figure out how to set the 
interface to monitor.  The transfer will use whatever interface routing 
provides, normally set by the name of the target.  It can also be used for 
non-host-based transfers (between mounted filesystems) but the network 
bandwidth continues to be (usually pointlessly) shown.

NB: Between mounted filesystems, parsyncfp sometimes works very poorly for
reasons still mysterious.  In such cases (monitor with 'ifstat'), use 'cp'
or [tnc](https://goo.gl/5FiSxR) for the initial data movement and a single
rsync to finalize.  I believe the multiple rsync chatter is interfering with 
the transfer.

It only works on dirs and files that originate from the current dir (or
specified via "--startdir").  You cannot include dirs and files from
discontinuous or higher-level dirs.

## the ~/.parsyncfp files 
The cache dir (~/.parsyncfp by default) contains the fpcache dir which holds the 
fpart log and all the PID files as well as the chunk files (f*).   parsyncfp no 
longer provides cache reuse because the fpart chunking is so fast.  The log files are
datestamped and are NOT overwritten.  A new option allows you to specify alternative
locations for the cache as well as specifying locations for multiple instances so that
many parsyncfps can run at the same time, altho they will detect each others' 
fparts running and question this situation at startup.  The multihost version 
will test for rsyncs running on the SEND hosts and alert you.

## Odd characters in names
parsyncfp will sometimes refuse to transfer some oddly named files, altho 
recent versions of rsync allow the '-s' flag (now a parsyncfp default) 
which tries to respect names with spaces and properly escaped shell 
characters.  Filenames with embedded newlines, DOS EOLs, and other 
odd chars will be recorded in the log files in the ~/.parsyncfp dir.

##  Debugging
To see where you (or I) have gone wrong, look at the 
rsync-logfile\* logs in the ~/.parsyncfp dir.  If you're a masochist, use
the '--debug' flag which will spew lots of gratuitous info
to the screen and pause multiple times to allow checking of intermediate 
results.

## Options

[i] = integer number<br>
[f] = floating point number<br>
[s] = "quoted string"<br>
( ) = the default if any<br>
The syntax 'longarg|shortarg' means that either the long or short form may 
be used to denote that option.<br>

~~~

 --NP|np [i] (sqrt(#CPUs)) : number of rsync processes to start optimal NP depends on many vars.   Try the default and incr as needed
 --altcache|ac (~/.parsyncfp) : alternative cache dir for placing it on a another FS or for running multiple  parsyncfps simultaneously
 --startdir|sd [s] (`pwd`) : the directory it starts at(*)
 --maxbw [i] (unlimited) : in KB/s max bandwidth to use (--bwlimit  passthru to rsync).  maxbw is the total BW to be used, NOT per rsync.
 --maxload|ml [f] (NP+2)  :  max system load - if sysload > maxload, an rsync proc will sleep for 10s
 --chunksize|cs [s] (10G) : aggregate size of files allocated to one rsync  process.  Can specify in 'human' terms [100M, 50K, 1T] as well as integer bytes.
 --rsyncopts|ro [s] : options passed to rsync as quoted string (CAREFUL!) this opt triggers a pause before executing to verify the command
 --fromlist|fl [s]  +
 --trimpath|tp [s]  +-- see "Options for using filelists" below
 --trustme|tm       +
 --interface|i [s] : network interface to monitor (not use; see above)
 --checkperiod|cp [i] (5) : sets the period in seconds between updates
 --verbose|v [0-3] (2) : sets chattiness. 3=debug; 2=normal; 1=less; 0=none.    This only affects verbosity post-start; warning & error messages will still be printed.
 --dispose|d [s] (l) : what to do with the cache files. (l)eave untouched, (c)ompress to a tarball, (d)elete.
 --email [s] : email address to send completion message
 --nowait  :  for scripting, sleep for a few seconds instead of pausing & requiring user input.
 --udr : wrap rsync in the UDT lib for UDP transmissions. UDP is more filtered than TCP and may fail depending on whose networks the packets have to pass.
 --hosts [s]     +
 --hostcheck     +-- see "Options for MultiHost transfers" below
 --commondir [s] +
 --rpathpfx [s]  +
 --version : dumps version string and exits
 --help : this help

~~~

###  Options for using filelists

(thanks to Bill Abbott for the inspiration/guidance).

The following 3 options provide a means of explicitly naming the files
you  wish to transfer by means of filelists, whether by 'find' or other
means. Typically, you will provide a list of files, for example generated
by a DB lookup (GPFS or Robinhood) with full path names.  If you use
this list directly with rsync, it will remove the leading '/' but then
place the file with that otherwise full path inside the target dir. So
'/home/hjm/DL/hello.c' would be placed in '/target/home/hjm/DL/hello.c'.  
If this result is OK, then simply use the '--fromlist' option to specify 
the file of files.

If the list of files are NOT fully qualified then you should make sure
that the command is run from the correct dir so that the rsyncs can find
the designated dirs & files.

If you want the file 'hello.c' to end up as '/target/DL/hello.c' (ie
remove the original '/home/hjm'), you would use the --trimpath option
as follows: '--trimpath=/home/hjm'.  This will remove the given path
before transferring it and assure that the file ends up in the right
place.  This should work even if the command is executed away from the
directory where the files are rooted. If you have already modified the
file list to remove the leading dir path, then of course you don't need
to use '--trimpath' option.

~~~

- --fromlist|fl [s] : take explicit input file list from given file, 
                        1 path name per line.
- --trimpath|tp [s] : path to trim from front of full path name if 
                        '--fromlist' file contains full path names and 
                        you want to trim them.
- --trustme|tm : with '--fromlist' above allows the use of file lists of the form:
                        
                        size in bytes<tab>/fully/qualified/filename/path  
                        825692            /home/hjm/nacs/hpc/movedata.txt
                        87456826          /home/hjm/Downloads/xme.tar.gz
                        etc
~~~

This allows lists to be produced elsewhere to be fed directly to pfp without 
a file stat() or complete recursion of the dir tree.  So if you're using an 
SQL DB to track your filesystem usage like Robinhood or a filesystem like GPFS
that can emit such data, it can save some startup time on gigantic file trees.

### Options for MultiHost transfers

~~~

--hosts [s] ....... specify the Send and Receive hosts, optionally supplying
                    R hosts with individually alternate paths to store data.
                    If you specify /different/ REC paths, the SEND data will 
                   be split over those host:/path combinations, so they will
                                    have to be manually combined afterwards.
                     This is to allow different remote filesystems to accept
                        (hopefully) very high bandwidth transmission without
                                                  impacting other processes)
                         The required last element in a multihost command is
                          'POD::/path' which is the default path for any REC
                    hosts that haven't been defined in the '--hosts' option.
                    (see Good Example 4 below)
--hostcheck  ............. pre-check to make sure that the SEND & REC hosts
                    specified with '--hosts' do not have any rsyncs running.
                If they do, the number of them is reported. Those rsyncs may 
                 be valid and independent of pfp but it may be evidence of a 
                       failed pfp which may interfere with a new pfp launch.
                               In order to prevent mistakes, I rec using the 
                              '--ro' flag rather than the full '--rsyncopts'
                   This option also pushes the required utilities to the REC 
                           slave hosts to make sure that they have the exe's
                           necessary to run with full functionality.
--commondir [s] ............ the shared, common dir in which all chunk files 
                        and rsync logs will stored.  Similar to '--altcache'
                                     but MUST be readable by all SEND hosts.
--rpath [s] .............. the remote PATH prefix on the SLAVES to check for 
                         the bits needed to run this. Prefixed to the remote 
                             ssh cmd as 'export PATH=<rpath string>:\$PATH;' 
                    The rpath string can contain as many paths as you'd like
                separated by ':', tho vars have to be escaped appropriately.
                                               --rpath="~/bin:\$HOME/pfp/bin"
                 (default is ~/bin:~/.pfp, and ':\$PATH' is also appended so
                            --rpath="~/bin:\$HOME/pfp/bin"
                                        is transmitted as:
                            --rpath="~/bin:\$HOME/pfp/bin:\$PATH"
                              (This option may be removed as the --hostcheck 
                            option obviates most of the utility of --rpaths)

~~~

The host string format is a comma-delimited set of 'Send=Receive' hosts.
example: "s1=r1:/path1,s2=r2:/path2,s3=r3:/path3,s4=r4,s5=r5"
where each 's#' and 'r#' imply a full "login\@host" string, altho if the 
user on the master server is the same as on the other hosts, the 'user@' is 
not required (ssh rules).
Also, each 'r#' can have a storage path appended (r2:/path2).  If the REC 
path is not given, the path from the final 'POD::/path' target is appended.
ie pfp [option option option..] POD::/common/default/receive/target.

The multihost command will exit once the 'fpart' chunking process is finished 
and leave the SEND hosts running independently on the remote hosts.  They will
continue to send output back to the originating terminal prefixed/suffixed
with the SEND hostname so you can decipher which SEND host is saying what.
However, unlike the single-host version, killing or suspending the originating 
program will have no effect on the SEND hosts. The remote rsyncs will have to be 
killed manually.  The following command executed on the originating node will 
kill all the rsyncs on the SEND and REC hosts defined:

~~~
   hosts2hit="send1 send2 send3 rec1 rec2"
   for host in $hosts2hit; do 
     echo [ processing $host ] ; 
     ssh $host "kill -9 \` ps ux | grep rsyn[c] | awk '{print \$2}'\` " ; 
   done

~~~

(The above bash loop has been edited to render correctly in rendered markdown.)
(I think.)
This SEND host independence should be addressed shortly via socket-based controls. 

See 'Good Example 5', below.
                        

## Examples

### Good Example 1
```
% parsyncfp  --maxload=5.5 --NP=4 \
--chunksize=\$((1024 * 1024 * 4)) \
--startdir='/home/hjm' dir[123]  \
hjm\@remotehost:~/backups

```

where:

- "--maxload=5.5" will start suspending rsync instances when the 5m system load gets to 5.5 and then unsuspending them when it goes below it.
- "--NP=4" forks 4 instances of rsync
- "chunksize=$((1024 * 1024 * 4))" defines the chunksize as the product of numbers (equalling 4MB)
- "--startdir='/home/hjm'" sets the working dir of this operation to '/home/hjm' and dir1 dir2 dir3 are subdirs from '/home/hjm'
- the target "hjm\@remotehost:~/backups" is the same target rsync would use

It uses 4 instances to rsync dir1 dir2 dir3 to hjm@remotehost:~/backups


### Good Example 2

```% parsyncfp   --checkperiod 6  --NP 3 \
--interface eth0  --chunksize=87682352 \
--rsyncopts="--exclude='[abc]*'"  nacs/fabio   
hjm\@moo:~/backups
```

where

- "--checkperiod 6" - defines a 6s cycle time between updates
- "--NP 3" - requests 3 instances of rsync 
- "--interface eth0" - requests that the eth0 interface be monitored explicitly
- "--chunksize=87682352" - shows that the chunksize option can be used with explicit
integers as well as the human specifiers (TGMK).
- --rsyncopts="--exclude='[abc]*'" - shows the correct form for excluding files
based on regexes (note the quoting)
- nacs/fabio - shows that you can specify subdirs as well as top-level dirs (as
long as the shell is positioned in the dir above, or has been specified via
'--startdir'

### Good Example 3

```
% parsyncfp -v 1 --nowait --ac pfpcache1 --NP 4 \
--cp=5 --cs=50M --ro '-az' linux-4.8.4 moo:~/test
```

where

- short version of several options (-v for --verbose, --cp for checkperiod, etc)
- shows use of --altcache (--ac pfpcache1), writing to relative dir pfpcache1
- again shows use of --rsyncopts (--ro '-az') indicating 'archive' & compression'.
- includes '--nowait' to allow unattended scripting of parsyncfp


###  Good example 4
```
parsyncfp --NP=8 --chunksize=500M --fromlist=/home/hjm/dl550 \
hjm@moo:/home/hjm/test
```

where

- if you use the '--fromlist' option, you cannot use explicit source dirs
  (all the files come from the file of files (which require full path names)
- that the '--chunksize' format can use human abbreviations (M or m for Mega).


###  Good example 5

```
parsyncfp --verbose=2 --ro='-aslz' \
--hosts="bigben=bridgit.ure.edu:/d1/in, \
          pooki=bridgit.ure.edu:/d2/in, \
        stunted=bridgit.ure.edu:/d3/in" \
--hostcheck  --ro='-aslz' --NP 4 --chunk 15G --check 5 --dispo=c \
--interface=wlp3s0 --commondir=/home/hjm/pfp --startdir /home/hjm/pfp \
zeba motif-2.3.6 tree-1.8.0 singularity-2.3.1 dmtcp-2.5.2  POD::/
```


The above multihost command shows 3 SEND hosts (bigben, pooki, stunted) all sending 
data to a single REC host bridgit.ure.edu altho the data is being split among 3
filesystems (mounted on /d1, /d2, /d3).  You could also define 3 REC hosts, writing 
data to the SAME PATH if that had a better performance profile.
It shows the preferred way of defining the rsyncopts with "--ro='-aslz'" and the 
'--dispo=c' option requests that the cachefiles be compressed and tar'ed afterwards.
The 'POD::/' terminal element is the (required) default path for any undefined 
REC hosts (of which there aren't any in this example)

### Error Example 1

```
% pwd
/home/hjm  # executing parsyncfp from here

% parsyncfp --NP4 --compress /usr/local  /media/backupdisk
```

Why this is an error:

- '--NP4' is not an option (parsyncfp will say "Unknown option: np4")
    It should be '--NP=4'
- if you were trying to rsync '/usr/local' to '/media/backupdisk', 
    it will fail since there is no /home/hjm/usr/local dir to use as 
    a source. This will be shown in the log files in 
    ~/.parsyncfp/rsync-logfile-<datestamp>_#
    as a spew of "No such file or directory (2)" errors
- the '--compress' is a native rsync option, not a native parsyncfp option.
    You have to pass it to rsync with "--rsyncopts='--compress'"

The correct version of the above command is:

```% parsyncfp --NP=4  --rsyncopts='--compress' --startdir=/usr  local  /media/backupdisk```

### Error Example 2
```% parsyncfp  hjm\@moo.boo.yoo.com:/usr/local --start-dir /home/hjm mooslocal
```

Why this is an error:

- this command is trying to PULL data from a remote SOURCE to a 
    local TARGET.  parsyncfp doesn't support that kind of operation yet.
    
The correct version of the above command is:

``` # ssh to hjm\@moo, install parsyncfp, then:
% parsyncfp  --startdir=/usr  local  hjm\@remote:/home/hjm/mooslocal 
```

### Error Example 3

```% parsyncfp --NP=4 --chunksize=500M -startdir=/usr/local/bin hjm\@remote.host.edu:/home/backups
```

Why this is an error:

- you've specified a 'startdir' but haven't specified the dirs or files 
to be transferred.

The correct version of the above command is:
```% parsyncfp --NP=4 --chunksize=500M -startdir=/usr/local bin hjm\@remote.host.edu:/home/backups
```



## Changes

### 2.00 (as pfppod)
(Multihost ITERation) Apr 22, 2021. Lots of changes...

- added multihost capability.  Can send via multiple SEND hosts to multiple REC hosts, 
with multiple REC paths if wanted.
    - with --hostcheck, multihost version pushes all required scripts to targets for better 
    reliability, checks rsync status on remote hosts.
    - stat remote ~/.ssh/config file to warn about conflicting ssh-defined hostnames
    - added --rpath to set remote PATHs (tho may deprecate)
    - the multiple hosts store their logs in /hostname/ subdirs in the cache dir so they're 
    more easily separable
- removed prohibition of using '~' to denote remote HOME targets.  I think this has no more use.
- incorporated Chris Rapier's code to remove the long ls output, leading to crashes with processing 
zillions of fpart chunk files (but left the warnings about too many of them).
- considerable code cleanup, tho you'd be hard-pressed to tell.
- mod debug() to be more useful.
- added --udr to support UDP-transport supposedly good for increasing large file transport.
works on local nets, but not much better than normal.  Need much longer rtt networks to test on.
udr is not in repositories, so users have to add it for now.
- multihost version still works as single host version
- side effect of multihost version: SEND hosts are completely independent; need to add socket 
control to suspend/kill/cleanup/pass info


### 1.72 
(California Lockdown) Dec 6, 2020, No option changes.  Intercepted rsync options to forbid 
those that increase verbosity to avoid collision with pfp's IO handling. 
Including: -v/-verbose, --version, -h/--help 

### 1.71
(GoVote) Nov 2, 2020.  No option changes.  Changed how checking for external utilities 
   works.  Separates the required from recommended utilities and now continues with 
   a WARN if it doesn't find the recommended utils.

### 1.70 
(Silverado Fire), Oct 27, 2020.  No option changes.  Fixed bug about setting up the 
   fpart command (not a problem with fpart, just coercing names with spaces to be represented 
   correctly).

### 1.69 
(Covid Synchronicity), Aug 17, 2020. No option changes, but included a significant change in the way 
    that pfp reads the chunk files that fpart provides.  Before this versio, pfp 
    checked only for the existence of the chunk files and
    could therefore launch an rsync instance on a filelist that had not been completed.  If rsync overran the files,
    This might happen when a fileset that had already been mostly transferred, 
    and so it could theoretically exit before 
    fpart finished the writing to the file, leaving some files unsync'ed.
    
pfp now uses fpart's '-W' option to run a post-file-close script to move the finished 
    and closed file to the processing directory,
    assuring that the chunk files are not read before fpart is finished writing
    to them.  Thanks again to Ganael Laplanche (fpart author) for discussion and 
    suggesting the simplest way to address the problem.
    
Also some better checking for nonsense or non-existent files/dirs.

### 1.65
- tidied, changed a lot of output routines to consolidate subs and tested on HPC on some TB sized sets
   to verify clean ending.  Also tested mostly complete rsyncs to verify the fpart keep-ahead routines.
   Seems to be good.
   
### 1.64
- fixed bug that allowed for infinite waiting for fpart to produce another chunkfile.

### 1.63
- added check for underflow in fpart chunks; warning emitted if # of chunks is < # of rsyncs requested
- updated / replaced all 'route' and 'ifconfig' bits with 'ip' equivalents.  Also re-did the multihoming 
    bits to probe, and present all routed interfaces and IP #s so user can choose the right one on a multihomed
    host. Thanks to Ryan Novosielski for suggesting / pushing on this ;).
- usual bits of code cleanup and messup in equal amounts, tho overall things should progress more smoothly.
- detected a weird output format bug(?) inconsistency in CentOS 6.9 'ps' vs recent 'ps'.  Not going to fix.
  reveals itself if exceed MAXLOAD on CentOS 6.9 - the load balancing continues to work OK afaict but the suspended PIDs aren't shown correctly.  The workaround is to set MAXLOAD high enough that it won't be invoked.
- added more utilities to check for, before startup and corrected links to them.

### 1.61
- added total raw bytes transferred as well as autoscale to MB, GB, TB in signoff stanza.

### 1.60
- figured out why the suspend/unsuspends noted below weren't tracking correctly.  Thanks to Ben Dullnig 
for his perseverance and Mikael Barfred for his code suggestions. Lots of cosmetic and interface tweaks.  
Added Bill Abbott's suggestion to allow explicit sizing to the --fromlist option. Added a warning if the
number of chunkfiles exceed 2000 (hard-coded).  Also caught a rare condition where the run ends with suspended jobs. 
Now it should continue until the suspended PIDs get unsuspended and complete.

### 1.58
- added '--fromlist' to allow explicit lists of files to be pfp'ed.  Suggested by 
Bill Abbott to support GPFS's mmapplypolicy to generate lists so that pfp can immediately 
start moving data instead of iterating thru miles of already-synced files. Thanks, Bill.

### IMPORTANT NOTE (May 31, 2019)
Thanks to the long-suffering efforts of Jeff Dullnig, I've discovered that when parsyncfp goes thru 
multiple suspend/unsuspend cycles, it fails to correctly rsync all the src files to the target.

If the '--maxload' option is kept high enough to avoid any suspensions, it syncs correctly. 

If you're using parsyncfp now, please be aware that if forked rsyncs cycle thru suspend / unsuspends
you will probably not end up with a correct target.  I'll be working on this to determine if it can 
be fixed or if that 'feature' has to be removed.

### 1.57 
- added explicit GPL v3 licence 

### 1.56
- added a better measurement of TCP bytes sent (via /proc/net/dev) 
- added attempt at measuring RDMA bytes sent by using 'perfquery'; looks like it's
doing what it's supposed to.

### 1.54
- Bungled a commit.  This one should straighten is out.

### 1.53 
- removed internal space handling for target names since this interferes with multiple
dir targets.  Have to re-think this.

### 1.50
- fixed Ken Bass' bug where trimming dir name was not constrained to front
   of the string and could lead to problems if dir was names something like
   '/data/rna/hjm/rnaseq/data/something/version/data/smith'
   if the leading name was '/data/' the condensed, trimmed dir was 
   changed to
   'rna/hjm/rnaseq/something/version/smith' ie removal of ALL 'data/'
- fixed finding top level targets with embedded spaces - had to trim spaces
   and escape filenames going into fpart.
- many verbosity fixes
- some changes to ending text to reference both rsync and fpart logs 


### 1.47
- changed format of output to add elapsed time, changed date format
- code cleanup
- fixed bandwidth speed calculation
- updated help and fixed some inaccuracies for latest version.
- added (declinable) scripted mod to ~/.ssh/config to reduce ssh warnings

### 1.46
- made it variably verbose (--verbose)
- adjusted ending conditions to be accurate
- some code cleanup.  getting there.
- added checks for multihomed devices.




