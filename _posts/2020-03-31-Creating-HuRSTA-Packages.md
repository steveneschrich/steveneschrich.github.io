layout: page
title: "Creating HuRSTA Packages"
date: 2020-03-31 13:45:00 -0000
categories: R Programming

# Creating HuRSTA Packages

## Links
- [https://github.com/steveneschrich/hursta2a520709cdf](steveneschrich/hursta2a520709cdf (GPL15048))
- [https://github.com/steveneschrich/hursta2a520709.db](steveneschrich/hursta2a520709.db (GPL15048))

- [https://github.com/steveneschrich/hursta2a520709customcdf](steveneschrich/hursta2a520709customcdf (GPL10379))
- [https://github.com/steveneschrich/hursta2a520709custom.db](steveneschrich/hursta2a520709custom.db (GPL10379))

## Overview
I have been wanting to incorporate a custom array into R for quite some time. Thanks to recent events, I've had a chance to tackle this
while working on a specific project. I thought I would document some of the details of the chip and the annotations, in case I need to
remember them again at some point.

The chip itself is called `HuRSTA`, which is short for "Human" RSTA chip. It is a custom Affymetrix chip that was designed by Rosetta Inpharmatics / Merck Pharmaceuticals pre-2010 timeframe. One of the reasons that I am working on it is that Moffitt and Merck formed a partnership to investigate the molecular characteristics of cancer [e.g., News Article](https://www.biospace.com/article/releases/h-lee-moffitt-cancer-center-and-research-institute-announces-new-research-initiative-and-collaboration-with-merck-and-co-inc-/). The effort resulted in 15k tumors profiled on the custom HuRSTA chip and available to Moffitt and Merck researchers. Some of this data has made it's way into GEO and is available, both in raw (CEL file) form and processed form.

Fortunately, the information on the custom Affymetrix array is available on GEO. There are actually two platforms based on the specific chip (`HuRSTA_2a520709`). 

- [GPL15048](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GPL15048) is the original design. It consists of 60,607 probesets.
- [GPL10379](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GPL10379) is the updated design consisting of fewer probesets (52,378) and sometimes fewer probes per probeset. This is likely due to experience with the array and dropping noisy or poor performing probes.

## Making R packages
Both GEO GPL descriptions include the corresponding CDF (chip definition file) for the platform. Therefore, using tools such as RMAExpress or libaffy, it is possible to normalize HuRSTA data using rma/mas5. However, most array-based bioinformatics seems to involved R/Bioconductor these days so it is important to have access to these platforms within R. 

On looking into the topic, there are actually two different packages should be created within R for each platform. There is a cdf definition (as an R package, rather than a file) and a chip annotation package. Both topics have very good documentation and tooling available.

First, there was the issue of naming. Each CEL file has embedded in it (in the DATHeader), a description of the chip type. The most common chip type is the HG-U133 Plus 2.0 array, which is labeled as 'HG-U133_Plus_2.1sq'. Likewise, the HuRSTA chip has embdedded the name 'HuRSTA-2a520709.1sq'. R, or at least the affy package in Bioconductor, reads this DatHeader and selects the corresponding R package. It does this by stripping off the .1sq (not sure what that stands for) and cleaning the remaining name (lower-case, strip -,_,space) and tacking on a 'cdf'. So, R wants to look for a package called "hursta2a520709cdf". But with two definitions, which package gets that name and which gets an alternate name?

I chose to have the original design (GPL15048) as "hursta2a520709cdf" and the re-designed array (GPL10379) as "hursta2a520709customcdf". This was somewhat arbitrary. 

### CDF
With the naming set, it was a simple matter of using the package `makecdfenv` via
```r
library("makecdfenv")
```

and then using the function `make.cdf.package`. There is good [http://bioconductor.org/packages/release/bioc/vignettes/makecdfenv/inst/doc/makecdfenv.pdf](documentation). The only caveat was to rename the original CDF to allow normalizing the name into hursta2a520709. I used "HuRSTA_2a520709.CDF". 

So now there are two github projects for CDF environments for this chip type:

- [https://github.com/steveneschrich/hursta2a520709cdf](steveneschrich/hursta2a520709cdf (GPL15048))
- [https://github.com/steveneschrich/hursta2a520709customcdf](steveneschrich/hursta2a520709customcdf (GPL10379))

### Annotation
When creating an annotation package, there are excellent resources available on Bioconductor related to [https://www.bioconductor.org/packages/release/bioc/vignettes/AnnotationForge/inst/doc/SQLForge.pdf](AnnotationForge). The GPL records include a SOFT file with the probeset annotation, which I directly used for GPL10379. At Moffitt, we internally update the annotation based on sequence searching for the original definition so updated data was used for GPL15048.

- [https://github.com/steveneschrich/hursta2a520709.db](steveneschrich/hursta2a520709.db (GPL15048))
- [https://github.com/steveneschrich/hursta2a520709custom.db](steveneschrich/hursta2a520709custom.db (GPL10379))

## Installation
At the moment, I'm leaving them on github so I can access them (via `devtools::install_github()`). It looks like you can contribute these to Bioconductor, but it's not quite as straightforward (I think) since they are somewhat non-standard packages. 

## Parting Thoughts
The experience turned out to be much easier than I had built it up in my mind (as most things inevitably are). As a result of this effort, I would like to investigate re-annotation of the arrays as well as look at oligo-based annotation (not sure if it is different).

### Re-annotation
There are many papers on re-annotating Affymetrix arrays (brainarray being one of the more popular). It seems that frequently better, more reliable signal is attained using cleaned up definitions. This would be great to do for these arrays, and as indicated, I suspect that GPL10379 represents this as of 2010. Most papers I found described their process but did not include code to run. This makes sense, as re-annotation ends up being a bit of an art form vs. an exact science. How many base mismatches are allowed, how many transcript matches are tolerated, etc. However, there is benefit from working out the process which I'd like to tackle.

### Oligo-based annotation
Obviously there is a whole environment around [https://www.bioconductor.org/packages/release/bioc/vignettes/oligo/inst/doc/oug.pdf](oligo) from Bioconductor. I have no idea what is required to make these same packages for that environment but it looks like [https://www.bioconductor.org/packages/release/bioc/vignettes/pdInfoBuilder/inst/doc/BuildingPDInfoPkgs.pdf](pdInfoBuilder) is the tool to use for this job. Hopefully I can figure this out and create corresponding packages.

