# GDC Import details

Utility to download and when appropriate index data downloaded from GDC.
This utility takes as input Catalog files which describe data available at
GDC and produces BamMap files which provide path and other metadata for
downstream processing

GDC Import was developed for use with the [CPTAC3
project](https://github.com/ding-lab/CPTAC3.catalog) but can be used for any
GDC data.

## Installation
Each download has its own clone of the importGDC.CPTAC3 github repository with a directory
named after the batch name, for instance, `20220217.TestDownload`.  Then, clone the repository with,
```
git clone --recurse-submodules https://github.com/ding-lab/importGDC.CPTAC3.git 20220217.TestDownload
cd 20220217.TestDownload
```

## CPTAC3 Catalog
Importing here relies on data file `CPTAC3.Catalog.dat` generated by [CPTAC3 Case Discover](https://github.com/ding-lab/CPTAC3.case.discover)
and available [here](https://github.com/ding-lab/CPTAC3.catalog/blob/master/CPTAC3.Catalog.dat)

Default locations for Catalog are provided for MGI, katmai, and compute1.

## LSF Groups
For LSF systems (compute1 and MGI), the number of simultaneous downloads is controlled by [Job Groups](https://docs.ris.wustl.edu/doc/compute/recipes/job-execution-examples.html?highlight=bjgroup#id9).

The group name is typically `/USER/gdc-download`, where USER is the login name.  Please substitute your own name 
for this value below.

First time users will need to create a job group with,
```
bgadd -L 5 /USER/gdc-download
```
This will create the named job group with a limit of 5 simultaneous downloads.  To see the number of jobs queued, running and completed use the command,
```
bjgroup -s /USER/gdc-download
```
and to change the number of jobs running to 8 do,
```
bgmod -L 8 /USER/gdc-download
```

Don't make the number of jobs too high: this will saturate the network and
reduce system performance.  Suggest 5 jobs to start, and consult system
administrators if you have questions.

Note that the Katmai system does not use LSF groups.  Instead, the utility
`parallel` controls the number of simultaneous jobs and is determined at start
time as described below.

### Destination directory
Genomics files are large, with WGS often larger than 100Gb, so the choice of storage location is important.
This is determined by the `DATA_ROOT` defined below.  Make sure this allocation has adequate free space (`df -h DATA_ROOT`)
and that you can write to it

The individual files will be written to,
```
   $DATA_ROOT/GDC_import/data/<UUID>/<FILENAME>
```
where UUID is associated with the data file and provided by GDC.  BAM files
will have an index file `<FILENAME>.bai` and summary file `<FILENAME>.flagstat`
generated as well.

## Configuration

### Obtain GDC Token

GDC User Token is obtained from [GDC Cancer
Portal](https://portal.gdc.cancer.gov/) and has a filename which looks like,
`gdc-user-token.2022-01-05T22_45_39.319Z.txt`.  Note that this token is valid
for one month.  If a new one is downloaded old tokens are invalidated.

### Tracking download details

It is suggested that you track all batch-specific details in the README.project.md file for future reference.
Also, all imports associated with CPTAC3 Y3 should be tracked [here](https://docs.google.com/spreadsheets/d/1fbBZRPgyM21E1Eq1Se4qzHRWPII34CIUWUutuKRRoCs/edit#gid=662389878).

### Edit `gdc-import.config.sh`
A number of locale-specific variables are defined in `gdc-import.config.sh`:

* `SYSTEM` - the name of this system: MGI, katmai, or compute1.  This provides settings of a lot of other variables below.
* `GDC_TOKEN` - path to GDC token
* `LSF_GROUP` - LSF Group name (e.g. `/USER/gdc-download`) as described above.  Ignored on Katmai.
* The following are system-specific.  Default values are provided based on SYSTEM, and these may not have to be modified
    * `CATALOGD` - path to location of [catalog file](https://github.com/ding-lab/CPTAC3.catalog)
    * `DATA_ROOT` - location where download data will be stored.  
    * `START_DOCKERD` - path to directory of `start_docker.sh` described above
    * `FILE_SYSTEM` - Currently one of `MGI`, `compute1`, `katmai`
        * Used in BamMap to identify system where data resides
    * `DOCKER_SYSTEM` - One of `MGI`, `compute1`, or `docker`
        * `docker` is any generic docker system
    * `LSF` - 1 for MGI and compute1, 0 otherwise
    * `DL_ARGS`  - optional compute group arguments
    * `LSF_ARGS` - optional LSF arguments


### Initialize

* Place UUIDs to be downloaded in file `dat/download_UUID.dat`
* `10_summarize_download.sh` - calculates the disk space required for this download.  Generates
   an ad hoc catalog file which can be used to examine the planned download
    * Suggest placing output of this in `README.project.md`

## Start download

### Suggested procedure
First, dry run of one UUID:
```
cat dat/download_UUID.dat | bash 20_start_download.sh -1d -
```

If looks good, run one UUID:
```
cat dat/download_UUID.dat | bash 20_start_download.sh -1 -
```

If download starts OK (check logs directory), download remainder (skipping the first UUID). 

#### Katmai
```
tail -n +2 dat/download_UUID.dat | bash 20_start_download.sh -J 5 -
```

#### MGI and compute1
```
tail -n +2 dat/download_UUID.dat | bash 20_start_download.sh -
```

#### Additional downloader options
`cat dat/UUID-download.dat | bash 20_start_download.sh -` will start download of all UUIDs. There are a number of flags to review and modify this download
* `-d` will perform a dry run, to examine commands without running them
* `-1` stops execution after one UUID is processed, can be combined with `-d`
* `-J N` will perform N downloads in parallel on katmai, and can significantly speed up downloads
  * Note, do not use -J on MGI or compute1.  Rather, number of downloads will be governed by LSF system
* `-h` will list complete set of options

A number of other options exist. Run with `-h` to view


### Evaluate progress

`30_evaluate_download_status.sh` will list download status of all UUIDs.  

### Create BamMap

 `40_make_BamMap.sh` will create a BamMap file which lists the path and other metadata associated with
a given download.  BamMap files are described in more detail in the [CPTAC3.Catalog project](https://github.com/ding-lab/CPTAC3.catalog), 
and examples are [here](https://github.com/ding-lab/CPTAC3.catalog/tree/master/BamMap).

### Download details

Downloading is performed by [GDC Data Transfer
Tool](https://gdc.cancer.gov/access-data/gdc-data-transfer-tool).  BAM files
are indexed and output of `samtools flagstat` is written to provide an overview
of read statistics.  The tool is wrapped in a docker image,
`mwyczalkowski/importgdc`, and a wrapper shell script iterates over all UUIDs
to invoke the dockerized tool in a system-dependent way.  Parallelization for
katmai is implemented in the wrapper script using `GNU parallel`.


# Contact 

   Matthew Wyczalkowski <m.wyczalkowski@wustl.edu>
   [Ding Lab](http://dinglab.wustl.edu)
   Washinton University School of Medicine
