

## Count PAMs on 22 chromosomes

We used the GRCh37 assembly of the human genome, which can be downloaded [here](https://gatk.broadinstitute.org/hc/en-us/articles/360035890711-GRCh37-hg19-b37-humanG1Kv37-Human-Reference-Discrepancies#humanG1Kv37). 
Briefly, run `fecth.py` can get positions of PAM sequence. 
Taking **NGG** for example, run
``` 
python fecth.py human_g1k_v37_bk.fasta GG > GG_hg19_pos.tsv
```
can get **NGG** positions into **GG_hg19_pos.tsv** file.

Considering the reverse strand, we also run 
```
python fecth.py human_g1k_v37_bk.fasta CC > CC_hg19_pos.tsv
``` 
to get reversing **NGG** positions.

Then, run `make_bed_PAM.R` with argument `GG` to get PAM positions on both strands.



After getting positions of PAM, we count the number of each PAM on each cut of 20,000 segments (We simply think the 22 chromosomes as one). For each Cas enzyme, we sum the number of **all PAMs that it recognises** on each segments, and get top10% (2,000)segments that contain most PAM sequences.
```
load('/opt/ShaoYa/Users/ranran/PAM/GRC37_genome/CHRs_length.RData') ## get total length of each chromosome

N <- 20000
M <- dd[1,23]/N
M <- round(M) ## length of each segment

dir.create(paste0('/opt/working/projects/prj_026_CRISPR/Cas/', Cas))
PAM <- cas_info[which(cas_info$Cas == Cas), 'PAM']
aa <- read.table(paste0('/opt/working/projects/prj_026_CRISPR/PAM/', PAM[1], '_PAM_count_both_strand.txt'), header = T, stringsAsFactors = F)
aa <- aa[,1:3]
for (pam in PAM) {
	tmp <- read.table(paste0('/opt/working/projects/prj_026_CRISPR/PAM/', pam, '_PAM_count_both_strand.txt'), header = T, stringsAsFactors = F)
	aa <- cbind(aa, tmp[,4])
}
colnames(aa)[-c(1:3)] <- PAM
if (ncol(aa) > 4) {
	aa$PAM_sum <- rowSums(aa[,-c(1:3)])
} else {
	aa$PAM_sum <- aa[,4]
}
yy <- aa[order(aa[,'PAM_sum'], decreasing = T),]
top10 <- yy[1:2000,] ## get Top10%

idx <- which(top10[,3]-top10[,2] < M-1)

if (length(idx) > 0) {
	for (i in 1:length(idx)) {
		ccc <- top10[idx[i],3]-top10[idx[i],2]
		tmp <- c(0,1, (M-ccc), (top10[idx[i],4]+1))
		top10 <- rbind(top10, tmp)
	}
}

options(scipen = 200)
Top10 <- top10[,c(1,2,3)]
Top10$Chr <- paste0('chr', Top10$Chr)

write.table(Top10, file = paste0('/opt/working/projects/prj_026_CRISPR/Cas/', Cas, '/Top10.bed'), col.names = F, row.names = F, quote = F, sep = '\t')
```

## Heritability enrichment analysis

With the `.bed` file, we can **compute annotation-specific LD scores** by running
```
bash make_annot_Cas.sh SpCas9 ## make annotation file

bash compute_l2_Cas.sh SpCas9 ## compute LD scores
```
which is modified from the [LDSC turorial](https://github.com/bulik/ldsc/wiki/LD-Score-Estimation-Tutorial#partitioned-ld-scores).


**Partitioning heritability** of 28 human complex traits also followed the instructions from [Partitioned Heritability](https://github.com/bulik/ldsc/wiki/Partitioned-Heritability) of the LDSC software.
```
bash Partition_heritability_Cas.sh SpCas9
```

## Annotation of the X chromosome
The s-LDSC does not include the X chromosome for heritability enriichment analysis, or provide the functional annotations (e.g., coding, enhancer, etc.) for the X chromosome. To determine whether the pattern we found in the autosomal regions is similar on the X, we used the SNP annotation tool, SNPnexus, to annotate the HapMap3 SNPs on the X chromosome. 

The functional annotations we used for the X chromosome includes gene annotations from UCSC, epigenomic markers from Roadmap, and regulatory elements from Ensembl. 

To be comparable with the autosomal analysis, the X chromosome was also devided into 1078 cuts of the length 144 kb. Then the top 1078 cuts were selected as the Cas-featured regions.



## References
1. Bulik-Sullivan, B., Loh, PR., Finucane, H. et al. LD Score regression distinguishes confounding from polygenicity in genome-wide association studies. Nat Genet 47, 291–295 (2015). https://doi.org/10.1038/ng.3211
2. Finucane, H., Bulik-Sullivan, B., Gusev, A. et al. Partitioning heritability by functional annotation using genome-wide association summary statistics. Nat Genet 47, 1228–1235 (2015). https://doi.org/10.1038/ng.3404
3. Jorge Oscanoa, Lavanya Sivapalan, Emanuela Gadaleta, Abu Z Dayem Ullah, Nicholas R Lemoine, Claude Chelala, SNPnexus: a web server for functional annotation of human genome sequence variation (2020 update), Nucleic Acids Research, Volume 48, Issue W1, 02 July 2020, Pages W185–W192, https://doi.org/10.1093/nar/gkaa420







