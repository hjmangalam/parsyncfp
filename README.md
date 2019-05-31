# parsyncfp
a parallel rsync wrapper in Perl. Released under GPL v3.

(Version changes moved to the bottom of this file)

## IMPORTANT NOTE (May 31, 2019)
Thanks to the long-suffering efforts of Jeff Dullnig, I've discovered that when parsyncfp goes thru 
multiple suspend/unsuspend cycles, it fails to correctly rsync all the src files to the target.

If the '--maxload' option is kept high enough to avoid any suspensions, it syncs correctly. 

If you're using parsyncfp now, please be aware that if forked rsyncs cycle thru suspend / unsuspends
you will probably not end up with a correct target.  I'll be working on this to determine if it can 
be fixed or if that 'feature' has to be removed.


## Background

parsyncfp (pfp) is the offspring of my [parsync](https://github.com/hjmangalam/parsync) 
 ([more info here](http://moo.nac.uci.edu/~hjm/parsync/))
 and Ganael LaPlanche's [fpart](http://goo.gl/K1WwtD), which
collects files based on size or number into chunkfiles which can be fed to rsync on a 
chunk by chunk basis.  This allows pfp to begin transferring files before the 
complete recursive descent of the source dir is complete.  This feature can save many 
hours of prep time on very large dir trees.

parsyncfp requires 'perfquery' and 'ibstat', Infiniband utilities both written 
by Hal Rosenstock < hal.rosenstock [at] gmail.com > if your use involves transit 
over IB networks.

pfp is primarily tested on Linux, but is being ported to MacOSX
as well. 

pfp needs to be installed only on the SOURCE end of the 
transfer and only works in local SOURCE -> remote TARGET mode 
(it won't allow remote local SOURCE <- remote TARGET, emitting an 
error and exiting if attempted). It requires that ssh shared keys be
set up prior to operation [see here](https://goo.gl/ghCazV).

It uses whatever rsync is available on the TARGET.  It uses a number 
of Linux-specific utilities so if you're transferring between Linux 
and a FreeBSD host, install parsyncfp on the Linux side. 

The only native rsync options that pfp uses is '-a' (archive) &
'-s' (respect bizarro characters in filenames).  If you need to define 
the rsync options differently, then it's up to you to define ALL of 
them via '--rsyncopts' (the default '-a -s' flags will NOT be provided 
automatically.

pfp checks to see if the current system load is too heavy and tries
to throttle the rsyncs during the run by monitoring and suspending / 
continuing them as needed.

It appropriates rsync's bandwidth throttle mechanism, using '--maxbw'
as a passthru to rsync's 'bwlimit' option, but divides it by NP so
as to keep the total bw the same as the stated limit.  It monitors and
shows network bandwidth, but can't change the bw allocation mid-job.
It can only suspend rsyncs until the load decreases below the cutoff.
(If you suspend parsyncfp (^Z), all rsync children will suspend as well,
regardless of current state.)

Unless changed by '--interface', it tried to figure out how to set the 
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
many parsyncfps can co-exist at the same time.

## Odd characters in names
parsyncfp will sometimes refuse to transfer some oddly named files, altho 
recent versions of rsync allow the '-s' flag (now a parsyncfp default) 
which tries to respect names with spaces and properly escaped shell 
characters.  Filenames with embedded newlines, DOS EOLs, and other 
odd chars will be recorded in the log files in the ~/.parsyncfp dir.

** Debugging **
To see where you (or I) have gone wrong, look at the 
rsync-logfile\* logs in the .parsyncfp dir.  If you're a masochist, use
the '--debug' flag which will spew lots of grotacious gratuitous info
to the screen.

## Options

[i] = integer number<br>
[f] = floating point number<br>
[s] = "quoted string"<br>
( ) = the default if any<br>
The syntax 'longarg|shortarg' means that either the long or short form may 
be used to denote that option.<br>

- --NP|np [i] (sqrt(#CPUs)) : number of rsync processes to start optimal NP depends on many vars.   Try the default and incr as needed
- --altcache|ac (~/.parsyncfp) : alternative cache dir for placing it on a another FS or for running multiple  parsyncfps simultaneously
- --startdir|sd [s] (`pwd`) : the directory it starts at(*)
- --maxbw [i] (unlimited) : in KB/s max bandwidth to use (--bwlimit  passthru to rsync).  maxbw is the total BW to be used, NOT per rsync.
- --maxload|ml [f] (NP+2)  :  max system load - if sysload > maxload, an rsync proc will sleep for 10s
- --chunksize|cs [s] (10G) : aggregate size of files allocated to one rsync  process.  Can specify in 'human' terms [100M, 50K, 1T] as well as integer bytes.
- --rsyncopts|ro [s] : options passed to rsync as quoted string (CAREFUL!) this opt triggers a pause before executing to verify the command(+)
- --fromlist|fl [s] ........... take explicit input file list from given file, 1 *fully qualified* path name per line.
- --interface|i [s] : network interface to monitor (not use; see above)
- --checkperiod|cp [i] (5) : sets the period in seconds between updates
- --verbose|v [0-3] (2) : sets chattiness. 3=debug; 2=normal; 1=less; 0=none.    This only affects verbosity post-start; warning & error messages will still be printed.
- --dispose|d [s] (l) : what to do with the cache files. (l)eave untouched, (c)ompress to a tarball, (d)elete.
- --email [s] : email address to send completion message
- --nowait  :  for scripting, sleep for a few s instead of pausing
- --version : dumps version string and exits
- --help : this help

## Examples

### Good Example 1
```
% parsyncfp  --maxload=5.5 --NP=4 --startdir='/home/hjm' dir1 dir2 dir3 hjm@remotehost:~/backups
```

where:

- "--startdir='/home/hjm'" sets the working dir of this operation to '/home/hjm' and dir1 dir2 dir3 are subdirs from '/home/hjm'
- the target "hjm\@remotehost:~/backups" is the same target rsync would use
- "--NP=4" forks 4 instances of rsync
- "--maxload=5.5" will start suspending rsync instances when the 5m system load gets to 5.5 and then unsuspending them when it goes below it.

It uses 4 instances to rsync dir1 dir2 dir3 to hjm@remotehost:~/backups


### Good Example 2

```
parsyncfp   --checkperiod 6  --NP 3 --interface eth0  --chunksize=87682352 
   --rsyncopts="--exclude='[abc]*'"  nacs/fabio   hjm@moo:~/backups
```

where

- --chunksize=87682352 - shows that the chunksize option can be used with explicit
integers as well as the human specifiers (TGMK).

- --rsyncopts="--exclude='[abc]*'" - shows the correct form for excluding files
based on regexes (note the quoting)

- nacs/fabio - shows that you can specify subdirs as well as top-level dirs (as
long as the shell is positioned in the dir above, or has been specified via
'--startdir'

### Good Example 3

```
parsyncfp -v 1 --nowait --ac pfpcache1 --NP 4 --cp=5 --cs=50M --ro '-az'  
linux-4.8.4 moo:~/test
```

where

- short version of several options (-v for --verbose, --cp for checkperiod, etc)
- shows use of --altcache (--ac pfpcache1), writing to relative dir pfpcache1
- again shows use of --rsyncopts (--ro '-az') indicating 'archive' & compression'.
- includes '--nowait' to allow unattended scripting of parsyncfp


###  Good example 4

parsyncfp-list --NP=8 --chunksize=500k --fromlist=/home/hjm/dl550 \
hjm@moo:/home/hjm/testparsync
                            
where

- if you use the '--fromlist' option, you cannot use explicit source dirs
  (all the files come from the file of files (which require full path names)
- that the '--chunksize' format can use human abbreviations (k or K for kilo).


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
```% parsyncfp --start-dir /home/hjm  mooslocal  hjm\@moo.boo.yoo.com:/usr/local```

Why this is an error:

- this command is trying to PULL data from a remote SOURCE to a 
    local TARGET.  parsyncfp doesn't support that kind of operation yet.
    
The correct version of the above command is:

```# ssh to hjm\@moo, install parsyncfp, then:
% parsyncfp  --startdir=/usr  local  hjm\@remote:/home/hjm/mooslocal```


## Changes

### 1.58
- added '--fromlist' to allow explicit lists of files to be pfp'ed.  Suggested by 
Bill Abbott to support GPFS's mmapplypolicy to generate lists so that pfp can immediately 
start moving data instead of iterating thru miles of already-synced files. Thanks, Bill.

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




