# Pipeline for getting SMPs and SNPs for Arabidopsis lyrata

1. Download data from NCBI
```
## Genomes usually donwloaded from Ensembl, need to ensure it includes the choloplast contig. Index with samtools faidx.

cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata
/data/proj2/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/prefetch --option-file  acc_list_bs  -O data
/data/proj2/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/prefetch --option-file  acc_list_rna  -O data_rna

cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data
for file in *.sra; do /data/proj2/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/fastq-dump --split-3 --gzip  $file; done

cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna
for file in *.sra; do /data/proj2/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/fastq-dump --split-3 --gzip  $file; done

/data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes
grep 'exon' Arabidopsis_lyrata.v.1.0.55.gtf | cut -f 1,4,5 >gene_pos.bed

```

2. Trim data
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data
for file in *_1.fastq.gz; do java -jar /data/proj2/popgen/a.ramesh/software/Trimmomatic-0.39/trimmomatic-0.39.jar PE -phred33 -threads 20 $file ${file/_1.fastq.gz/_2.fastq.gz} ${file/_1.fastq.gz/_1.paired.fq.gz} ${file/_1.fastq.gz/_1.unpaired.fq.gz} ${file/_1.fastq.gz/_2.paired.fq.gz} ${file/_1.fastq.gz/_2.unpaired.fq.gz} ILLUMINACLIP:TruSeq3-PE.fa:2:30:10:2:True LEADING:3 TRAILING:3 SLIDINGWINDOW:4:20 MINLEN:36; done

cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna
for file in *_1.fastq.gz; do java -jar /data/proj2/popgen/a.ramesh/software/Trimmomatic-0.39/trimmomatic-0.39.jar PE -phred33 -threads 20 $file ${file/_1.fastq.gz/_2.fastq.gz} ${file/_1.fastq.gz/_1.paired.fq.gz} ${file/_1.fastq.gz/_1.unpaired.fq.gz} ${file/_1.fastq.gz/_2.paired.fq.gz} ${file/_1.fastq.gz/_2.unpaired.fq.gz} ILLUMINACLIP:TruSeq3-PE.fa:2:30:10:2:True LEADING:3 TRAILING:3 SLIDINGWINDOW:4:20 MINLEN:36; done
```

3. Index reference genome and map RNA-seq data with with STAR 2-pass mode

```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes
/data/proj2/popgen/a.ramesh/software/STAR-2.7.10b/bin/Linux_x86_64/STAR --runMode genomeGenerate --genomeDir STARindex/ --genomeFastaFiles Arabidopsis_lyrata.v.1.0.dna.toplevel_chrloroplast.fa --sjdbGTFfile Arabidopsis_lyrata.v.1.0.55.gtf --sjdbOverhang 201 --runThreadN 20

cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna
ulimit -n 10000
for file in *_1.paired.fq.gz; do /data/proj2/popgen/a.ramesh/software/STAR-2.7.10b/source/STAR --runMode alignReads --genomeDir /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/STARindex/ --readFilesIn $file ${file/_1.paired.fq.gz/_2.paired.fq.gz} --readFilesCommand zcat --outFileNamePrefix ${file/_1.paired.fq.gz//}  --outReadsUnmapped Fastx --outSAMtype BAM SortedByCoordinate --twopassMode Basic --runThreadN 20  ; done

find . -type f -name "Aligned.sortedByCoord.out.bam" -printf "/%P\n" | while read FILE ; do DIR=$(dirname "$FILE" ); mv ."$FILE" ."$DIR""$DIR".sort.bam;done
find . -name '*.sort.bam' -exec mv {} . \;

```

4. Mark duplicate reads and index
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/
for file in *.sort.bam ; do picard MarkDuplicates -I $file -O ${file/.sort.bam/_marked.bam} -M ${file/.sort.bam/_metrics.txt} 2>mark_err; done
for file in *_marked.bam ; do picard BuildBamIndex -I $file 2>index_err; done
```

5. SplitNCigarReads
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/
for file in *_marked.bam ; do gatk SplitNCigarReads -R /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/Arabidopsis_lyrata.v.1.0.dna.toplevel_chrloroplast.fa -I $file -O  ${file/_marked.bam/_split.bam}  ; done
```

6. Add read groups and index
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/
for file in *_split.bam ; do picard AddOrReplaceReadGroups -I $file -O ${file/_split.bam/_readgroup.bam} -LB species -PL illumina -PU 1 -SM $file; done
for file in *_readgroup.bam ; do picard BuildBamIndex -I $file; done
ls *_readgroup.bam > lc_files
ls *_readgroup.bam | sed 's/*_readgroup.bam//' | paste - lc_files >lc_files2 ## for haplotyper caller script (file also called hc_files2)
```

7. Run haplotype caller using perl script to parallelize
```
#!/usr/bin/perl -w


my $dir = "/data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/";
my $input = "hc_files2";
chdir("$dir") or die "couldn't move to input directory";

open (INPUT_FILE, "$input") || die "couldn't open the input file $input!";
                    while (my $line = <INPUT_FILE>) {
                        chomp $line;
my @array = split(/\t/,$line);
my $sample = $array[0];
my $file = $array[1];

my $SLURM_header = <<"SLURM";
#\$-cwd
# This option tells gridware to change to the current directory before executing the job
# (default is the home of the user).

#\$-pe serial 2
# Specify this option only for multithreaded jobs that use more than one cpu core.
# IMPORTANT! Don't use more than 4 cores to keep free space for other students!

#\$-l vf=2g
# This option declares the amount of memory the job will use. Specify this value as accurate as possible.
# IMPORTANT! memory request is multiplied by the number of requested CPU cores specified with the -pe.
# Thus, you should divide the overall memory consumption of your job by the number of parallel threads.

#\$-N lc


source ~/.bashrc

cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/

SLURM

  my $tmp_file = "lc.$sample";
  open (SLURM, ">$tmp_file") or die "Couldn't open temp file\n";
  $SLURM_header = $SLURM_header;
  print SLURM "$SLURM_header\n\n";
  print SLURM "gatk HaplotypeCaller -R /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/Arabidopsis_lyrata.v.1.0.dna.toplevel_chrloroplast.fa  -I $file -O  $sample".".g.lc.vcf.gz -ERC GVCF\n";
  close SLURM;
  system("qsub $tmp_file");

}

            close(INPUT_FILE);
```


8. Combine GVCFs
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/
gatk CombineGVCFs -R /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/Arabidopsis_lyrata.v.1.0.dna.toplevel_chrloroplast.fa --variant HighSite_GER_2274.g.lc.vcf.gz  --variant HighSite_J1_182.g.lc.vcf.gz   --variant LowSite_GER_2324.g.lc.vcf.gz  --variant LowSite_J3_1601.g.lc.vcf.gz --variant HighSite_GER_2285.g.lc.vcf.gz  --variant HighSite_J1_2023.g.lc.vcf.gz  --variant LowSite_J1_163.g.lc.vcf.gz    --variant LowSite_J3_1651.g.lc.vcf.gz --variant HighSite_GER_2456.g.lc.vcf.gz  --variant LowSite_GER_2187.g.lc.vcf.gz  --variant LowSite_J1_383.g.lc.vcf.gz    --variant LowSite_J3_238.g.lc.vcf.gz --variant HighSite_J1_1490.g.lc.vcf.gz   --variant LowSite_GER_2214.g.lc.vcf.gz  --variant LowSite_J1_761.g.lc.vcf.gz    --variant LowSite_J3_750.g.lc.vcf.gz --variant HighSite_J1_1658.g.lc.vcf.gz   --variant LowSite_GER_2304.g.lc.vcf.gz  --variant LowSite_J1_985.g.lc.vcf.gz -O lyrata.cohort.g.vcf.gz
```

9. Genotype combined GVCFs

```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/
gatk --java-options "-Xmx4g" GenotypeGVCFs -R /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/Arabidopsis_lyrata.v.1.0.dna.toplevel_chrloroplast.fa -V lyrata.cohort.g.vcf.gz -O lyrata.output.vcf.gz
gatk --java-options "-Xmx4g" GenotypeGVCFs -R /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/Arabidopsis_lyrata.v.1.0.dna.toplevel_chrloroplast.fa -all-sites -V lyrata.cohort.g.vcf.gz -O lyrata.var_invar.vcf.gz
```

10. Filter vcf, only keep high quality SNPs
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/
gatk SelectVariants -V lyrata.output.vcf.gz -select-type SNP -O lyrata.snps.vcf.gz
vcftools --gzvcf lyrata.snps.vcf.gz --out lyrata_snps_filtered --recode --recode-INFO-all --minDP 20 --minGQ 30  --minQ 30

gatk SelectVariants -V lyrata.var_invar.vcf.gz -select-type NO_VARIATION -O lyrata.invar.vcf.gz 
vcftools --gzvcf lyrata.invar.vcf.gz --out lyrata.invar --recode --recode-INFO-all --minDP 20 --minGQ 30 --minQ 30 --max-missing 0.5 --bed /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/gene_pos.bed
```

11. Get chromosome and sample names, can be done many ways
```
cut -f 1 ../genomes/PL_genomeandchloroplast_assemblies_S119.fa.fai >chrlist
ls *_marked.bam | sed 's/_marked.bam//' >samplenames
```

12. Create combined list containing every pair of chromosome and sample names. In R. 
```
file1 <- read.table(file="samplenames")
file2 <- read.table(file="chrlist")

fileall <- expand.grid(file1$V1, file2$V1)  
write.table(fileall, file = "sample_chr", quote = F, sep = "\t", row.names = F, col.names = F)
```

13. Create pseudogenomes by substituting SNPs for each sample from VCF into the reference genome
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/
bgzip lyrata_snps_filtered.recode.vcf
tabix lyrata_snps_filtered.recode.vcf.gz
cat sample_chr | while read -r value1 value2 remainder ; do samtools faidx /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/Arabidopsis_lyrata.v.1.0.dna.toplevel_chrloroplast.fa $value2 | bcftools consensus -M N -s $value1 -p ${value1/$/_} -H 1pIu lyrata_snps_filtered.recode.vcf.gz >>${value2/$/_lyrata.fa}; done
cat chrlist | while read line; do /data/proj2/popgen/a.ramesh/software/faSplit byname $line ${line/^/chr_} ; done
mkdir /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/pseudogenomes
mv *fa /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/pseudogenomes
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/pseudogenomes
cat samplenames | while read line ; do cat $line*.fa >$line.merged.fa  ; done
for file in *.merged.fa ; do samtools faidx $file; done
for file in *.merged.fa ; do picard CreateSequenceDictionary -R $file -O ${file/.fa/.dict}; done
cat samplenames | while read line ; do mkdir $line  ; done
cat samplenames | while read line ; do mv $line.merged.fa $line  ; done
cat samplenames | while read line ; do mv $line.merged.dict $line  ; done
cat samplenames | while read line ; do mv $line.merged.fa.fai $line  ; done
sed 's/^/\/data\/proj2\/popgen\/a.ramesh\/projects\/methylomes\/lyrata\/pseudogenomes\//' samplenames | paste samplenames - >samplenames2 ## not exactly the same, variants exist
```

14. Now map methylation reads with Bismark
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data/
cat samplenames2 |  while read -r value1 value2 remainder ; do /data/proj2/popgen/a.ramesh/software/Bismark-0.24.0/bismark --multicore 4 --hisat2 --path_to_hisat2 /data/proj2/popgen/a.ramesh/software/hisat2-2.2.1/ --genome_folder $value2 -1 $value1.1.paired.fq.gz -2 $value1.2.paired.fq.gz  ; done
```

15. Deduplicate methylation reads with Bismark
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data
for file in *_bismark_hisat2_pe.bam; do /data/proj2/popgen/a.ramesh/software/Bismark-0.24.0/deduplicate_bismark --bam $file ; done
```

16. Call methylation variants
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data
cat samplenames2 |  while read -r value1 value2 remainder ; do /data/proj2/popgen/a.ramesh/software/Bismark-0.24.0/bismark_methylation_extractor --multicore 4 --gzip --bedGraph --buffer_size 10G --cytosine_report --genome_folder $value2 $value1.1.paired_bismark_hisat2_pe.deduplicated.bam ; done
```

17. This script is to create methylation vcfs. I need add this after annotating it properly. Lots done here. In R.
```
setwd("/data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data")
library(dplyr)
library(ggplot2)

covfile <- read.table(file="covfiles")
contextfiles <- read.table(file="contextfiles")

cov <- read.table(file=covfile[1,], header=F)
colnames(cov) <- c("chromosome", "position", "end.position", "methylation.percentage", "count.methylated", "count.unmethylated" )
cov$ID <- paste(cov$chromosome,cov$position,sep = "-")

context <- read.table(file=contextfiles[1,],header=F)
colnames(context) <- c("chromosome", "position", "strand", "count.methylated", "count.unmethylated", "C-context", "trinucleotide context")
context$ID <- paste(context$chromosome,context$position,sep = "-")

cov_context <- inner_join(cov,context[c(3,6:8)],by="ID")
dim(cov_context)
cov_context$chromosome <- gsub(".*_split.bam","",cov_context$chromosome)
cov_context <- cov_context[cov_context$count.methylated + cov_context$count.unmethylated > 4, ]

cov_context$ID <- paste(cov_context$chromosome,cov_context$position,sep = "-")
cov_context$count.total <- cov_context$count.methylated + cov_context$count.unmethylated
cov_context$pval <- 0
noncoversionrate <- sum(cov_context[cov_context$chromosome %in% "NC_034365.1",]$count.methylated)/sum(cov_context[cov_context$chromosome %in% "NC_034365.1",]$count.total)

#cov_context <- cov_context[cov_context$count.total > 9,]

b <- apply(cov_context[c(5,11)],1,binom.test,p = noncoversionrate, alternative = c("greater"))
cov_context$pval <- do.call(rbind,lapply(b,function(v){v$p.value}))

cov_context <- cov_context[c(1,2,7,8,9,12)]
cov_context$fdr <- p.adjust(cov_context$pval,method = "fdr")
cov_context$call <- "U"
cov_context[cov_context$fdr < 0.01,]$call <- "M"
cov_context <- cov_context[-c(6,7)]

samplename <- gsub(".1.paired_bismark_hisat2_pe.deduplicated.bismark.cov","",covfile[1,])
colnames(cov_context)[ncol(cov_context)] <- samplename
cov_context2 <- cov_context

for (i in 2:nrow(covfile)){
  covfile <- read.table(file="covfiles")
  contextfiles <- read.table(file="contextfiles")
  
  cov <- read.table(file=covfile[i,], header=F)
  colnames(cov) <- c("chromosome", "position", "end.position", "methylation.percentage", "count.methylated", "count.unmethylated" )
  cov$ID <- paste(cov$chromosome,cov$position,sep = "-")
  
  context <- read.table(file=contextfiles[i,],header=F)
  colnames(context) <- c("chromosome", "position", "strand", "count.methylated", "count.unmethylated", "C-context", "trinucleotide context")
  context$ID <- paste(context$chromosome,context$position,sep = "-")
  
  cov_context <- inner_join(cov,context[c(3,6:8)],by="ID")
  dim(cov_context)
  cov_context$chromosome <- gsub(".*_split.bam","",cov_context$chromosome)
  cov_context <- cov_context[cov_context$count.methylated + cov_context$count.unmethylated > 4, ]
  
  cov_context$ID <- paste(cov_context$chromosome,cov_context$position,sep = "-")
  cov_context$count.total <- cov_context$count.methylated + cov_context$count.unmethylated
  cov_context$pval <- 0
  noncoversionrate <- sum(cov_context[cov_context$chromosome %in% "NC_034365.1",]$count.methylated)/sum(cov_context[cov_context$chromosome %in% "NC_034365.1",]$count.total)
  
  #cov_context <- cov_context[cov_context$count.total > 9,]
  
  b <- apply(cov_context[c(5,11)],1,binom.test,p = noncoversionrate, alternative = c("greater"))
  cov_context$pval <- do.call(rbind,lapply(b,function(v){v$p.value}))

  cov_context <- cov_context[c(1,2,7,8,9,12)]
  cov_context$fdr <- p.adjust(cov_context$pval,method = "fdr")
  cov_context$call <- "U"
  cov_context[cov_context$fdr < 0.01,]$call <- "M"
  cov_context <- cov_context[-c(6,7)]
  
  samplename <- gsub(".1.paired_bismark_hisat2_pe.deduplicated.bismark.cov","",covfile[i,])
  colnames(cov_context)[ncol(cov_context)] <- samplename
  cov_context <- cov_context[c(3,6)]
  cov_context2 <- full_join(cov_context2,cov_context,by="ID")
}

cov_context2$chromosome <- gsub("-.*","",cov_context2$ID)
cov_context2$position   <- as.numeric(gsub(".*-","",cov_context2$ID))
write.table(cov_context2,file="cov_context3.txt",row.names = F)

cov_context3 <- read.table(file="cov_context3.txt",header=T)
na_count <- apply(cov_context3[6:ncol(cov_context3)], 1, function(x) sum(is.na(x)))
na_count <- na_count/ncol(cov_context3[6:ncol(cov_context3)])
cov_context3 <- cov_context3[na_count < 0.5,] # change to appropriate number
cov_context4 <- cov_context3
ploymorphic <- apply(cov_context3[6:ncol(cov_context3)], 1, table)
ploymorphic <- sapply(ploymorphic,length)
cov_context3 <- cov_context3[ploymorphic > 1,]

meta <- cov_context3[1:3]
colnames(meta) <- c("#CHROM","POS","ID")
meta$REF <- "A"
meta$ALT <- "T"
meta$QUAL <- 4000
meta$FILTER <- "PASS"
meta$INFO <- "DP=1000"
meta$FORMAT <- "GT"
cov_context3 <- cov_context3[-c(1:5)]
cov_context3[cov_context3 == "U"] <- "0/0"
cov_context3[cov_context3 == "M"] <- "1/1"
cov_context3[is.na(cov_context3)] <- "./."
write.table(cbind(meta,cov_context3),file="lyrata_meth.vcf",quote = F, row.names = F,sep="\t")

meta2 <- cov_context4[1:3]
colnames(meta2) <- c("#CHROM","POS","ID")
meta2$REF <- "A"
meta2$ALT <- "T"
meta2$QUAL <- 4000
meta2$FILTER <- "PASS"
meta2$INFO <- "DP=1000"
meta2$FORMAT <- "GT"
cov_context4 <- cov_context4[-c(1:5)]
cov_context4[cov_context4 == "U"] <- "0/0"
cov_context4[cov_context4 == "M"] <- "1/1"
cov_context4[is.na(cov_context4)] <- "./."
write.table(cbind(meta2,cov_context4),file="lyrata_meth_var_invar.vcf",quote = F, row.names = F,sep="\t")

```

18. Get vcf header
```
zcat lyrata_snps_filtered.recode.vcf.gz | grep '##' > vcfheader
cat vcfheader lyrata_meth.vcf >lyrata_meth_all.vcf
```

19. Get summary statistics for methylation and genomic variants. Done on biallelic SNPs. Further NA filtering in R. Only keep gene body variants.
```
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/
awk '$3 ~ /^gene$/' Arabidopsis_lyrata.v.1.0.55.chr.gff3 | cut -f 1,4,5 >gene_pos.bed
 
cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna/
vcftools --gzvcf lyrata.snps_filtered.lc.recode.vcf.gz --out lyrata_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --freq --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/gene_pos.bed
vcftools --gzvcf lyrata.snps_filtered.lc.recode.vcf.gz --out lyrata_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --site-pi --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/gene_pos.bed
vcftools --gzvcf lyrata.snps_filtered.lc.recode.vcf.gz --out lyrata_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --het --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/gene_pos.bed

cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data/
vcftools --vcf lyrata_meth_all.vcf  --out lyrata_meth --min-alleles 2 --max-alleles 2 --max-missing 0.5 --freq --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/gene_pos.bed
vcftools --vcf lyrata_meth_all.vcf  --out lyrata_meth --min-alleles 2 --max-alleles 2 --max-missing 0.5 --site-pi --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/gene_pos.bed

vcftools --vcf lyrata_meth_var_invar_all.vcf --out lyrata_meth_var_invar --recode --min-alleles 2 --max-alleles 2 --max-missing 0.5 --bed /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/gene_pos.bed

```

20. R plot for other metrics. Per site or across genome, not per gene.
```
#SFS

lyrata_snp.frq <- read.table(file="lyrata_snp.frq",row.names = NULL)
lyrata_snp.frq$raf <- as.numeric(gsub(".*:","",lyrata_snp.frq$N_CHR))
lyrata_snp.frq$aaf <- as.numeric(gsub(".*:","",lyrata_snp.frq$X.ALLELE.FREQ.))
lyrata_snp.frq$maf <- apply(lyrata_snp.frq[7:8],1,min)
lyrata_snp.frq <- lyrata_snp.frq[lyrata_snp.frq$maf > 0,]
lyrata_snp.frq <- lyrata_snp.frq[lyrata_snp.frq$maf < 1,]
lyrata_snp.frq <- lyrata_snp.frq[lyrata_snp.frq$N_ALLELES > 37,]
lyrata_snp.frq$type <- 'SNP'

lyrata_meth.frq <- read.table(file="lyrata_meth.frq",row.names = NULL)
lyrata_meth.frq$raf <- as.numeric(gsub(".*:","",lyrata_meth.frq$N_CHR))
lyrata_meth.frq$aaf <- as.numeric(gsub(".*:","",lyrata_meth.frq$X.ALLELE.FREQ.))
lyrata_meth.frq$maf <- apply(lyrata_meth.frq[7:8],1,min)
lyrata_meth.frq <- lyrata_meth.frq[lyrata_meth.frq$maf > 0,]
lyrata_meth.frq <- lyrata_meth.frq[lyrata_meth.frq$maf < 1,]
lyrata_meth.frq <- lyrata_meth.frq[lyrata_meth.frq$N_ALLELES < 37,]
lyrata_meth.frq$type <- 'SMP'

snp_smp_lyrata <- rbind(lyrata_snp.frq,lyrata_meth.frq)

lyrata_sfs <- ggplot(snp_smp_lyrata,aes(fill=type,x=maf)) +
  geom_histogram(aes(y=0.05*..density..),binwidth=0.05, alpha=0.4, position='identity') +
  ylab("Proportion of sites")

nrow(lyrata_meth.frq)/nrow(lyrata_snp.frq)

## PI
lyrata_snp.sites.pi <- read.table(file="lyrata_snp.sites.pi",row.names = NULL, header = T)
lyrata_snp.sites.pi$type <- 'SNP'

lyrata_meth.sites.pi <- read.table(file="lyrata_meth.sites.pi",row.names = NULL, header = T)
lyrata_meth.sites.pi$type <- 'SMP'

snp_smp_lyrata <- rbind(lyrata_snp.sites.pi,lyrata_meth.sites.pi)

df_lyrata_pi <- dplyr::count(snp_smp_lyrata, type)
df_lyrata_pi$n <- paste("n=",df_lyrata_pi$n,sep="")
lyrata_pi <- ggplot(snp_smp_lyrata,aes(x=type,y=PI)) +
  geom_boxplot() +
  ylab("Per site π") +
  xlab("")+
  ggtitle("A. lyrata")  +
  geom_text(data = df_lyrata_pi, aes(y = 0.6, label = n))
lyrata_pi

nrow(lyrata_meth.sites.pi)/nrow(lyrata_snp.sites.pi)

lyrata_prop_seg_smp <- 475/1346
lyrata_prop_seg_snp <- 404842/(443781+443781)


## LD
lyrata_snp.geno.ld <- read.table(file="lyrata_snp.geno.ld",row.names = NULL, header = T)
lyrata_snp.geno.ld$type <- 'SNP'

lyrata_meth.geno.ld <- read.table(file="lyrata_meth.geno.ld",row.names = NULL, header = T)
lyrata_meth.geno.ld$type <- 'SMP'

snp_smp_lyrata <- rbind(lyrata_snp.geno.ld,lyrata_meth.geno.ld)

lyrata_ld <- ggplot(snp_smp_lyrata,aes(x=type,y=R.2)) +
  geom_boxplot() +
  ylab("Pairwise LD (r2)") +
  xlab("") +
  ggtitle("A. lyrata")

lyrata_ld <- ggplot(snp_smp_lyrata,aes(fill=type,x=R.2)) +
  geom_histogram(aes(y=0.1*..density..),binwidth=0.1, alpha=0.3, position='identity') +
  ylab("Proportion of sites") +
  xlab("Pairwise LD (r2)") +
  ggtitle("A. lyrata") +
  scale_fill_manual(values=c("red","green"))

nrow(lyrata_meth.geno.ld)/nrow(lyrata_snp.geno.ld)

## tajima's D
lyrata_snp.Tajima.D <- read.table(file="lyrata_snp.Tajima.D",row.names = NULL, header = T)
lyrata_snp.Tajima.D$type <- 'SNP'

lyrata_meth.Tajima.D <- read.table(file="lyrata_meth.Tajima.D",row.names = NULL, header = T)
lyrata_meth.Tajima.D$type <- 'SMP'

snp_smp_lyrata <- rbind(lyrata_snp.Tajima.D,lyrata_meth.Tajima.D)

lyrata_tajd <- ggplot(snp_smp_lyrata,aes(x=type,y=TajimaD)) +
  geom_boxplot()  +
  ylab("Tajima's D") +
  xlab("") +
  ggtitle("A. lyrata")

nrow(lyrata_meth.Tajima.D)/nrow(lyrata_snp.Tajima.D)

```

21. Split reference genome into genes
```
cd  /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes
sed -e 's/\t/:/' -e  's/\t/-/' gene_pos.bed >gene_pos.list


cd /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/data_rna
cat /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/gene_pos.list | while read -r line ; do samtools faidx /data/proj2/popgen/a.ramesh/projects/methylomes/lyrata/genomes/Arabidopsis_lyrata.v.1.0.dna.toplevel_chrloroplast.fa $line >>genes.fasta; done
mkdir genes_fasta/
/data/proj2/popgen/a.ramesh/software/faSplit byname genes.fasta genes_fasta/
cd genes_fasta/
ls *fa >filenames
```

22. Rscript to count number of cytosines

```
library("methimpute",lib.loc="/data/home/users/a.ramesh/R/x86_64-redhat-linux-gnu-library/4.1/")

## Only CG context
files <- read.table(file="filenames")
files <- as.character(files$V1)

cytsosine_count <- ""
for (f in 1:length(files)){
  print(f)
  cytosines <- c()
  try(cytosines <- extractCytosinesFromFASTA(files[f], contexts = 'CG'),silent=T)
  if (length(cytosines) > 0){
    cytsosine_count <- rbind(cytsosine_count,(c(files[f],table(cytosines$context))))
  } else {
    cytsosine_count <- rbind(cytsosine_count,(c(files[f],0)))
  }
  #print(cytsosine_count)
}
cytsosine_count <- cytsosine_count[-c(1),]
write.table(cytsosine_count,file="cytsosine_count.txt",row.names=F, col.names=F,quote=F,sep="\t")
```
