As part of an ongoing project using Affymetrix HuEx arrays, we are using brainarray redefined annotations (http://brainarray.mbni.med.umich.edu/Brainarray/Database/CustomCDF/genomic_curated_CDF.asp) of the gene array. How can we annotate a gencode cdf with gene symbols, etc?

My solution is to retrieve the gene annotation file (gtf) from gencode corresponding to the brainarray annotation, load this data into R and combine with the gene expression results.

## Brainarray annotation
Each brainarray release consists of a number of alternate cdf's for popular Affymetrix platforms (http://brainarray.mbni.med.umich.edu/Brainarray/Database/CustomCDF/CDF_download.asp). Here you can choose the brainarray version (latest is v25) and annotation (we are using GENCODEG, or genecode gene annotation). For our purposes, the platforms (v25, GENCODEG) is at (http://brainarray.mbni.med.umich.edu/Brainarray/Database/CustomCDF/25.0.0/gencodeg.asp). This is where you can download the annotations.


To find out what release of gencode was used to assemble the v25.0.0 cdf, we can refer to (http://brainarray.mbni.med.umich.edu/Brainarray/Database/CustomCDF/25.0.0/version.html). Here we see that the Homo_sapiens GENCODEGTF is v36 and is linked directly (http://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_36/gencode.v36.annotation.gtf.gz). This is what we are looking for.

## Using the GTF
It is easy to load the gtf into R with the rtracklayer package. We can use the following R commands. Note that since our annotation is gene-level GENCODE, we want features at the level of "gene".

```r
download.file(
    "http://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_36/gencode.v36.annotation.gtf.gz",
    destfile="gencode.v36.annotation.gtf.gz"
)
v36<-rtracklayer::import(
    "gencode.v36.annotation.gtf.gz", 
    format="gtf", 
    feature.type = "gene"
)
```

This is a bioconductor data type (object), but can be converted to a plain data frame for additional processing:
```r
as.data.frame(v36)
```
 
This can be added to a `Biobase::ExpressionSet` as below (with my bias for tidyverse).
```r
fdata <- tibble::enframe(
    name=NULL, 
    value="probe",
    rownames(exprset)
) |>
    dplyr::left_join(
        as.data.frame(v36),
        by=c("probe"="gene_id")
    )
Biobase::featureDat(exprset) <- Biobase::AnnotatedDataFrame(fdata)
```

