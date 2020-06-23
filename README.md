# GDC Import details

Utility to download and when appropriate index data downloaded from GDC.
This utility takes as input Catalog files which describe data available at
GDC and produces BamMap files which provide path and other metadata for
downstream processing

GDC Import was developed for use with the [CPTAC3
project](https://github.com/ding-lab/CPTAC3.catalog) but can be used for any
GDC data.

## Installation

Uses `start_docker.sh` from [WUDocker](https://github.com/ding-lab/WUDocker.git)
Obtain with,
```
git clone https://github.com/ding-lab/WUDocker.git
```

## TODO:

update documentation to discuss LSF use.  It is back, see README.7fba5f4.md for past wording

* Do not run 00 (confirm)


## Preliminaries

Importing here relies on data file `CPTAC3.Catalog.dat` generated by [CPTAC3 Case Discover](https://github.com/ding-lab/CPTAC3.case.discover)
and available [here](https://github.com/ding-lab/CPTAC3.catalog/blob/master/CPTAC3.Catalog.dat)

GDC User Token is obtained from GDC and copied to `./token`.
Note that this token is valid for one month

### `gdc-import.config.sh`
A number of locale-specific variables are defined in `gdc-import.config.sh`:

* `BATCH` - arbitrary name of this import project
* `GDC_TOKEN`
* The following are system-specific
    * `CATALOGD` - path to location of [catalog file](https://github.com/ding-lab/CPTAC3.catalog)
    * `DATA_ROOT` - location where download data will be stored.
    * `START_DOCKERD` - path to directory of `start_docker.sh` described above
    * `FILE_SYSTEM` - Currently one of `MGI`, `compute1`, `katmai`
        * Used in BamMap to identify system where data resides
    * `DOCKER_SYSTEM` - One of `MGI`, `compute1`, or `docker`
        * `docker` is any generic docker system
    * `LSF` - 1 for MGI and compute1, 0 otherwise

## Execution chain

Downloading takes place through a series of command line steps, each starting with a number (`XX_step.sh`),
which are executed in numerical order and which do not take arguments.  Project-specific details are kept
in `README.project.md`.

### Technical summary

Downloading is performed by [GDC Data Transfer Tool](https://gdc.cancer.gov/access-data/gdc-data-transfer-tool).
BAM files are indexed and output of `samtools flagstat` is written to provide an overview of read statistics.
The tool is wrapped in a docker image, `mwyczalkowski/importgdc`, and a wrapper shell script iterates over all
UUIDs to invoke the dockerized tool in a system-dependent way.  Parallelization is implemented in the wrapper script
using `GNU parallel`.

### LSF Groups

*Specific to MGI*

Using LSF groups to limit download bandwidth; doing max 5 running jobs seems to do the trick.
* Background: https://confluence.gsc.wustl.edu/pages/viewpage.action?pageId=27592450
* Submission script (`start_batch_import.sh`) uses LSF groups if LSF_GROUP environment variable is defined.  Suggested use:
    export LSF_GROUP="/mwyczalk/gdc-download"
* first time doing this, need to create group as,
    * bgadd -L 5 /mwyczalk/gdc-download
* To limit to 5 running jobs: `bgadd -L 5 /mwyczalk/gdc-download`  (this should be a part of a setup script?)
* To examine: `bjgroup -s /mwyczalk/gdc-download`
* To modify, `bgmod -L 2 /mwyczalk/gdc-download`


### Initialize

* `00_start_docker.sh` - necessary on compute1 to start image `mwyczalkowski/cromwell-runner` and make available `parallel` utility
    * NO LONGER USED.  See Start download on compute1 section below
* `10_get_UUID.sh` - This will typically change with every download project.  Its goal is to parse existing Catalog and BamMap files
   to create a list of UUIDs, saved to `dat/UUID-download.dat`, which define the data to be downloaded
* `15_summarize_download.sh` - a convenience utility which calculates the disk space required for this download.  Generates
   an ad hoc catalog file which can be used to examine the planned download

### Start download

NOTE: see below for downloads on compute1

`cat dat/UUID-download.dat | bash 20_start_download.sh -` will start download of all UUIDs. There are a number of flags to review and modify this download
* `-d` will perform a dry run, to examine commands without running them
* `-1` stops execution after one UUID is processed, can be combined with `-d`
* `-J N` will perform N downloads in parallel, and can significantly speed up downloads
  * Note, do not use -J on MGI.  Rather, number of downloads will be governed by LSF system
* By default, this step will download all UUIDs in `dat/UUID-download.dat`.  Alternatively, UUIDs can be
  specified as command line arguments, or read from stdin if the argument is `-`.
* `-h` will list complete set of options

#### Start download on compute1 (new)

In order to run for >24 hrs compute1, the download job (step 20, src/start_downloads.sh) must be started
in a non-interactive session.  This is performed as step 25, which wraps the call to `src/start_downloads.sh` 
within a non-interactive bsub call.  Then, `start_downloads` proceeds as normal, looping over all UUIDs
and launching a bsub download command for each.

One complication with this approach is that it is no longer possible to pass UUIDs on command line to step 25.
Instead, list of UUIDs to download is generated prior to this step and defined in step 25.

To summarize, on compute1 run `25_start_download_docker.sh` instead of `20_start_download.sh`.


### Evaluate progress

`30_evaluate_download_status.sh` will list download status of all UUIDs.  

#### Starting all downloads

The evaluate and start scrips can be combined to start only those with a certain status.  In 
the example below, all UUIDs which are ready to download are passed to `20_start_download.sh`, to
be processed 5 at a time:
```
bash 30_evaluate_download_status.sh -f import:ready -u | bash 20_start_download.sh -J5 -
```

### Create BamMap

* `40_make_BamMap.sh` will create a BamMap file which lists the path and other metadata associated with
a given download.  BamMap files are described in more detail in the [CPTAC3.Catalog project](https://github.com/ding-lab/CPTAC3.catalog), 
and examples are [here](https://github.com/ding-lab/CPTAC3.catalog/tree/master/BamMap).

# Contact 

   Matthew Wyczalkowski <m.wyczalkowski@wustl.edu>
   [Ding Lab](http://dinglab.wustl.edu)
   Washinton University School of Medicine
