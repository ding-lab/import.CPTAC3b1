# Download CPTAC3 Batch 1 data

This project has (or will have) branches for both DC2 and MGI environment downloads.

## Installation

Scripts here rely on [importGDC.git](/gscuser/mwyczalk/src/importGDC). This is installed as a submodule with command,
```
git clone --recursive TODO-update
```

Importing here relies on data file `SR.CPTAC3.b1.dat` generated by [queryGDC.git](https://github.com/ding-lab/queryGDC).  This file is generated at and copied
from `epazote:/Users/mwyczalk/Data/CPTAC3/discover.CPTAC3.b1` 
[Google link](https://drive.google.com/open?id=1-GBKph16nUPtJ0LIMXQgfHqulMEcaA01)

GDC User Token is obtained from GDC and copied to `./token`.
Note that this token expires after some time (one month?) so this process needs to be repeated.  

### `0_init.sh`

A number of locale-specific variables are defined in `0_init.sh`:

* `IMPORTGDC_HOME` defines the location of [importGDC](https://github.com/ding-lab/importGDC) project.
* `DATA_DIR` is where data will be stored; in particular, tokens will be written to $DATA_DIR/token and read data will be written to `$DATA_DIR/GDC_import` 
* `GDCTOKEN` is path to token file, e.g., `token/gdc-user-token.2017-11-04T01-21-42.215Z.txt`.

Set IMPORTGDC_HOME variable to where importGDC.git project is installed.  Default is /usr/local/importGDC.  
```
    export IMPORTGDC_HOME="/gscuser/mwyczalk/src/importGDC"
```

## LSF Groups

Using LSF groups to limit download bandwidth; doing max 5 running jobs seems to do the trick.
* Background: https://confluence.gsc.wustl.edu/pages/viewpage.action?pageId=27592450
* Submission script (start_import.c3b1.sh) uses LSF groups if LSF_GROUP environment variable is defined.  Suggested use:
    export LSF_GROUP="/mwyczalk/gdc-download"
* To limit to 5 running jobs: `bgadd -L 5 /mwyczalk/gdc-download`  (this should be a part of a setup script?)
* To examine: `bjgroup -s /mwyczalk/gdc-download`
* To modify, `bgmod -L 2 /mwyczalk/gdc-download`

## Batches

Collections of SR (Submitted Reads, i.e., BAM or FASTQ files) to be processed together.  Here, CPTAC3.b1 is split
into WGS, WXS, and RNA-Seq batches.

## Workflow

Importing in practice tends to be a nonlinear workflow where it may be necessary to track, diagnose, and restart SR import jobs.
To aid in this, we have two tools to track job status and start jobs:
* evaluate_status.sh : check status of download for each SR in batch
* start_step.sh : Start a processing step (import, typically) for given SR UUIDs

These tools can be strung together; the following command will start import of all WXS samples which have a status of "ready":
```
    bash evaluate_status.sh -u -f import:ready -O ... WXS.batch.dat | bash start_step.sh -S SR/SR_merged.dat -g /mwyczalk/gdc-download import -
```

Scripts evaluate_status.c3b1.sh and start_step.c3b1.sh are wrappers around importGDC.git scripts which are specific to CPTAC3 Batch 1 work at MGI.
Using these the above script becomes,
```
    bash evaluate_status.c3b1.sh -u -f import:ready WXS.batch.dat | bash start_import.c3b1.sh -
```
Note that these scripts need to be edited for specific paths, if token changes, etc.

## BAM Map

Validation of downloading and indexing, as well as providing summaries of downloaded data, is done with summarize_import.c3b1.sh
Create summaries of all completed RNA-Seq downloads with,
```
    ./evaluate_status.c3b1.sh -u -f import:completed RNA-Seq.batch.dat | ./summarize_import.c3b1.sh -H -

```
