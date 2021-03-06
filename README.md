filesystem-reporting-tools
==========================

Tools to help system administors monitor very large file systems

### pwalk ###
pwalk (Parallel Walk). Walk a file system using many threads in parallel.
During the walk output inode information for every file. One line of output per file. 

### Build and install ###
pwalk is a single C program.  Once compiled pwalk runs on every flavor of 
Linux.  Default Linux builds do not have the libraries needed to build pwalk. 
Installing the 'lib32gcc1' package to Ubuntu provides all the libraries and
tools necessary to build pwalk. To build pwalk just compile pwalk.c. This one
gcc command is that is needed.

	gcc -pthread pwalk.c exclude.c -o pwalk

### Purpose ###
pwalk was written to solve the problem of reporting disk usage for large file 
systems.  Single name space file systems with tens of millions of files are 
time consuming to traverse and report on.  Traditional file systems like NFS 
store file meta data within the file system. The only way to discover and 
report on all meta data is to walk the whole file system tree.  pwalk walks 
a file system in parallel using many threads. 

### Why write pwalk?
Large file systems that support interactive users do not grow in a 
deterministic way.  As a system administrator I was not able to answer simple 
questions like: Who used the last 10TB of disk space last night?  There is 
no efficient method of discovering this information with very large file 
systems. The UNIX du (disk usage) command can take days to complete with 
large file systems.  This is a long time to wait while your file system is full
and users are calling you.  Pwalk was written to solve the problem of walking large single name space file system in a timely manor.  The typical usage of pwalk is to report on NFS mounted file systems. 

### Output ###
November 2013 - Two new fields have been added to the output. 
Parent Inode and directory level have been added to the
output.  parent Inode is output for every file.  Parent inode for the top
level directory is set to zero which identifies the top of the tree.  Directory 
level counter starts at zero relative to the directoy that is being reported on.The top level directory has a depth of -1.

The output of pwalk is designed to be loaded into a relational database. 
Large file systems are difficult to examine browse or examine by hand. A 
practical approach to examine the file system metadata is with SQL.  Pwalk 
performs a ‘stat’ on each file and every field of the inode is reported in 
CSV format. One file is reported per line of output.  pwalk complains about
file names that do not load into 'MYSQL' varchar fields.  File names are
POSIX compliant but MYSQL has many limits about what can be loaded into a 
varchar field. In my environement pwalk complains about a few hundred files per file
system which is a very very small percentage of the 52 million files. There
should be a flag to report all file names and not just DB safe file names.
Output can seem to be random since the program is threaded but every 
file does get reported.
ctime, mtime and atime are reported natively in UNIX epoch time 
(large integers). The file mode is reported as an octal string. Two additional 
fields are populated for directories: count of files in the directory and 
the sum of file sizes of all files in the directory.  File count is -1 if a 
file is not a directory.  All file and directory sizes are 
reported in bytes. Symbolic links are reported (they are a file) but symbolic
links are never followed.

April 27, 2017; st_dev has been added to the output. st_dev will allow tracking 
of file system volume.  This might lead to a new command line option
to prevent crossing of mounted file systems. Similar to find -x.

### Usage ###
Pwalk takes a single argument, which is the name of the file system to report
on.  In practice pwalk should be run as root or as setuid. Exposing NFS to 
the root user is not a good practice.  I run pwalk from a system only used by
administrative staff.  The NFS file systems to be reported on by pwalk are
exported read only to the admin machine.

### Options

    --NoSnap  Ignore directories that match the name .snapshot.

    --exclude filename

Exclude expects a single argument which is the name of a file.
  The exclude file contains paths of directories to skip, one path per line.
  pwalk will run with absolute or relative paths. The format of the pathnames
  in the exclude file should match the output of pwalk.

### Reporting ###
SQL allows you to look at file systems differently and more efficiently than 
just browsing a file system by hand.  As an example: How many files have been
created in the last seven days, sorted by size and grouped by user.  

	select uid, count(*), sum(size) from FSdata where ctime >  unix_timestamp( date_sub(now(), interval 7 day) ) group by uid order by 3;

Two additional data fields are reported by pwalk when a file is a directory;
sum in byes and number of files contained in each directory. If a file is not
a directory the ‘file count’ field is zero.  Collecting these additional fields
is almost free while walking the tree but very difficult to create with a SQL
query.  These fields allow you to easily report on the largest directives.  
Very large directories (greater than one million files) create performance
issues with applications.  Knowing the size in bytes for a given directory 
can help discover application issues, big data users etc.

### Performance ###
pwalk can be 10 to 100 times faster than the UNIX disk usage command ‘du’. The
performance of pwalk is based on many variables: performance of your storage
system, host system and the layout of your files.  Reporting on a directory is
the smallest division of work performed by a single tread.  Run time for a 
directory with ten million files will be no faster than the UNIX command ls.
What makes pwalk faster than du is that many directories are being scanned at
once.  On a small file system pwalk can be slower than du because of the 
extra time needed to create and manage threads. You should expect pwalk to 
perform about 8,000 to 30,000 stat commands per second.  

Example performance metric: 50,000,000 files at a rate of 20,000 stats per
second should take about 41 minutes to complete. 

### Author ###
Pwalk is written by John Dey.  I have been a UNIX/Linux administrator since
the days of VAXes and BSD 4.2.  I have been privileged to support scientific
researchers and their data since the mid 1990’s.  I hope you will find pwalk 
to be useful. 

### Fork commnet ###
Hi My name is Benor Or Linux administrator and I need the pwalk in order to know about storages..

Find who is the depth folder in folder tree?
	
	time awk -v max=0 'BEGIN { FS = "," } ;{if($3>max){want=$0; max=$3}}END{print max,want} ' report.csv
	
	awk 'BEGIN { FS = "," } ;(NR==1){Max=$3;line=""};(NR>=2){if(Max<$3) {Max=$3;line=$0}} END {print "The Max is:",Max,line}' reoprt.csv

Find date filed in report how to get information of date and time?
	
	time awk -v max=0 'BEGIN { FS = "," } ;{if($3>max){want=$0; max=$3}}END{print max,want} ' report.csv
	
	

	awk -v max=0 'BEGIN { FS = "," } ;{if ($15=strftime("%Y-%m-%d %H:%M:%S") > ("2019-12-25 16:25:49")){print $0}} ' report.csv
 	awk -v max=0 'BEGIN { FS = "," } ;{if ($15=strftime("%Y-%m-%d %") > ("2019-12-25")){print $0}} ' report.csv
  
	awk -F"," '{OFS=","; $15=strftime("%Y-%m-%d %H:%M:%S", $15); print $0}' report.csv
 	awk -F"," '{OFS=","; $15=strftime("%Y-%m-%d", $15); print $0}' report.csv

	sqlite> SELECT datetime(1552500253, 'unixepoch');
2019-03-13 18:04:13

sqlite> SELECT *  from itable  WHERE datetime(st_atime, 'unixepoch') BETWEEN  '2001-02-13 19:04:13' and '2019-03-13 18:04:13';
268525701|268435459|0|/gpfs/units/it//._.DS_Store|DS_Store|10575|10000|4096|51|1024|1|0100770|1552500253|1475060440|1564909551|-1|0
268667359|268435459|0|/gpfs/units/it//._Symantec Uninstaller.app|app|10575|10000|4096|51|1024|1|0100770|1550147199|1520952066|1564909552|-1|0
268556359|268435459|0|/gpfs/units/it//MediaID.bin|bin|10575|10000|528|51|1|1|0100775|1550147199|1485957440|1564908933|-1|0
268660875|268435459|0|/gpfs/units/it//._.TemporaryItems|TemporaryItems|10612|10000|4096|51|1024|1|0100770|1552500253|1519309323|1564909551|-1|0

SELECT inode,datetime (st_mtime, 'unixepoch') , datetime(st_ctime, 'unixepoch' ),st_dev   AS isodate from itable;


SELECT inode,datetime (st_mtime, 'unixepoch') , datetime(st_ctime, 'unixepoch' ),datetime(st_atime, 'unixepoch'),st_dev  AS isodate from itable WHERE datetime(st_mtime, 'unixepoch') BETWEEN  '2016-02-13 19:04:13' and '2017-12-13 18:04:13';
268525701|2016-09-28 11:00:40|2019-08-04 09:05:51|2019-03-13 18:04:13|51
268556359|2017-02-01 13:57:20|2019-08-04 08:55:33|2019-02-14 12:26:39|51

sqlite> SELECT filename, date(st_atime, 'unixepoch') AS isodate from itable ;
/gpfs/units/it//AcronisBackup_12.5_64-bit (1).x86_64|2019-12-01
/gpfs/units/it//._.DS_Store|2019-03-13
/gpfs/units/it//install_eav.bat|2019-08-08
/gpfs/units/it//a.txt|2019-07-24
/gpfs/units/it//MDCStoreUtils API Readme.doc|2019-12-01
/gpfs/units/it//._Symantec Uninstaller.app|2019-02-14
/gpfs/units/it//test_file_to_delete_itzik.txt|2019-09-19
/gpfs/units/it//MediaID.bin|2019-02-14
/gpfs/units/it//._.TemporaryItems|2019-03-13



sqlite> SELECT inode,date (st_mtime, 'unixepoch') , date(st_ctime, 'unixepoch' ),date(st_atime, 'unixepoch'),st_dev  AS isodate from itable WHERE datetime(st_mtime, 'unixepoch') BETWEEN  '2016-02-13' and '2017-12-13';
268525701|2016-09-28|2019-08-04|2019-03-13|51
268556359|2017-02-01|2019-08-04|2019-02-14|51

sqlite> SELECT inode,date (st_mtime, 'unixepoch') , date(st_ctime, 'unixepoch' ),date(st_atime, 'unixepoch'),st_dev  AS date from itable WHERE datetime(st_mtime, 'unixepoch') BETWEEN  '2016' and '2018';
268525701|2016-09-28|2019-08-04|2019-03-13|51
268556359|2017-02-01|2019-08-04|2019-02-14|51

1336  29/12/19 17:27:30 sqlite3 tt ".mode csv"
 1337  29/12/19 17:27:43 sqlite3 tt ".import csv"
 1338  29/12/19 17:28:26 sqlite3 tt  "it-report-root.csv itreport "
 1339  29/12/19 17:28:37 sqlite3 tt  "import it-report-root.csv itreport "
 1340  29/12/19 17:28:43 sqlite3 tt  ".import it-report-root.csv itreport "
 1341  29/12/19 17:31:30 sqlite3 tt  "select count(*) from itreport;"
 1342  29/12/19 17:31:50 time wc -l it-report-root.csv
 1343  29/12/19 17:31:57 time sqlite3 tt  "select count(*) from itreport;"
 
 
 	sqlite> SELECT strftime('%Y', st_mtime, 'unixepoch'), sum(st_size)/1024/1024/1024/1024 FROM bioinformatics30122019 GROUP by strftime('%Y', st_mtime, 'unixepoch' ); SELECT sum(st_size)/1024/1024/1024/1024," TB" from bioinformatics30122019 ;
	1970                            0
	1992                            0
	1993                            0
	1994                            0
	1995                            0
	1996                            0
	1997                            0
	1998                            0
	1999                            0
	2000                            0
	2001                            0
	2002                            0
	2003                            0
	2004                            0
	2005                            0
	2006                            0
	2007                            0
	2008                            1
	2009                            4
	2010                            9
	2011                            14
	2012                            27
	2013                            43
	2014                            29
		2015                            77
	2016                            65
	2017                            57
	2018                            39
	2019                            68
	438                              TB









