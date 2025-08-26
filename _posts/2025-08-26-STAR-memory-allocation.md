I was running a RNASeq job and trying to use STAR for alignments. Which works well, however in a HPC environment one has to specify the amount of memory the job requires prior to starting the job. One question that I had was: what is the memory estimate for using STAR?

It turns out this question has been asked and answered. The github issue I found was at (https://github.com/alexdobin/STAR/issues/1063). Specifically, the answer (paraphrased from this site):

Size of index files: 
Genome, SA and SAi plus the `Nthreads * --limitIObufferSize` plus 1GB.

Based on the genome I was using, the totals were 

```
 4GB (Genome) + 27GB (SA) + 1.7GB (SAi) + 32 (Nthreads) * 300MB 
```
So about 42/43GB. I had run out of memory (oom job) using ~45GB, so that seems fairly reasonable. I rounded up to 64GB (I had plenty of RAM here) and had not problems after that.