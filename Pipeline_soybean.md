# Pipeline for getting SMPs and SNPs for Soybean

1. Get data
```
/data/proj2/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/prefetch -X 9999999999999 --option-file  PRJNA432760_soybean_wgbs_SRR_Acc_List.txt  -O data
/data/proj2/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/prefetch -X 9999999999999 --option-file  PRJNA432760_soybean_wgs_SRR_Acc_List.txt  -O data_wgs
for file in *.sra; do /data/proj2/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/fastq-dump --split-3 --gzip  $file; done

/data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes
grep 'exon' GCF_000004515.6_Glycine_max_v4.0_genomic.gtf | cut -f 1,4,5 >gene_pos.bed
```

2. Trim data
```
for file in *_1.fastq.gz; do java -jar /data/proj2/popgen/a.ramesh/software/Trimmomatic-0.39/trimmomatic-0.39.jar PE -phred33 -threads 15 $file ${file/_1.fastq.gz/_2.fastq.gz} ${file/_1.fastq.gz/_1.paired.fq.gz} ${file/_1.fastq.gz/_1.unpaired.fq.gz} ${file/_1.fastq.gz/_2.paired.fq.gz} ${file/_1.fastq.gz/_2.unpaired.fq.gz} ILLUMINACLIP:TruSeq3-PE.fa:2:30:10:2:True LEADING:3 TRAILING:3 SLIDINGWINDOW:4:20 MINLEN:36; done
```

3. Index genome
```
samtools faidx GCF_000004515.6_Glycine_max_v4.0_genomic.fna
picard CreateSequenceDictionary -R GCF_000004515.6_Glycine_max_v4.0_genomic.fna -O GCF_000004515.6_Glycine_max_v4.0_genomic.dict
```

4. Map genomic data
```
for file in *_1.paired.fq.gz; do bwa mem -t 24 /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/GCF_000004515.6_Glycine_max_v4.0_genomic.fna  $file ${file/_1.paired.fq.gz/_2.paired.fq.gz} 2>paired_map_err | samtools view -Sb - > ${file/_1.paired.fq.gz/_mapped.bam} 2>paired_map_err; done
```

5. Add readgroups, filter reads, sort reads, markduplicates and index bams
```
for file in *_mapped.bam ; do picard AddOrReplaceReadGroups -I $file -O ${file/_mapped.bam/_readgroup.bam} -LB species -PL illumina -PU 1 -SM $file; done
for file in *_readgroup.bam; do samtools view -q 20 -f 0x0002 -F 0x0004 -F 0x0008 -b $file >${file/_readgroup.bam/_mq20.bam}; done
for file in *_mq20.bam; do samtools sort -@ 5 $file >${file/_mq20.bam/_sort.bam} ; done
for file in *_sort.bam ; do picard MarkDuplicates -I $file -O ${file/_sort.bam/_marked.bam} -M ${file/_sort.bam/_metrics.txt} ; done
for file in *_marked.bam ; do picard BuildBamIndex -I $file; done

ls *_marked.bam > lc_files
ls *_marked.bam | sed 's/*_marked.bam//' | paste - lc_files >lc_files2 ## for haplotyper caller script (file also called hc_files2)


```

6. Call variants in each sample
```
#!/usr/bin/perl -w


my $dir = "/data/proj2/popgen/a.ramesh/projects/methylomes/soybean/data_wgs";
my $input = "lc_files2_3";
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

cd /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/data_wgs

SLURM

  my $tmp_file = "lc.$sample";
  open (SLURM, ">$tmp_file") or die "Couldn't open temp file\n";
  $SLURM_header = $SLURM_header;
  print SLURM "$SLURM_header\n\n";
  print SLURM "gatk HaplotypeCaller -R /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/GCF_000004515.6_Glycine_max_v4.0_genomic.fna  -I $file -O  $sample".".g.vcf.gz -ERC GVCF\n";
  close SLURM;
  system("qsub $tmp_file");

}

            close(INPUT_FILE);

```

7. Combine gvcfs
```
gatk CombineGVCFs -R /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/GCF_000004515.6_Glycine_max_v4.0_genomic.fna --variant ACC-001.g.vcf.gz --variant ACC-051.g.vcf.gz --variant ACC-076.g.vcf.gz --variant ACC-082.g.vcf.gz --variant ACC-089.g.vcf.gz --variant ACC-100.g.vcf.gz --variant ACC-103.g.vcf.gz --variant ACC-108.g.vcf.gz --variant ACC-1279.g.vcf.gz --variant ACC-128.g.vcf.gz --variant ACC-1297.g.vcf.gz --variant ACC-130.g.vcf.gz --variant ACC-1379.g.vcf.gz --variant ACC-1399.g.vcf.gz --variant ACC-1412.g.vcf.gz --variant ACC-1433.g.vcf.gz --variant ACC-171.g.vcf.gz --variant ACC-172.g.vcf.gz --variant ACC-176.g.vcf.gz --variant ACC-1843.g.vcf.gz --variant ACC-1942.g.vcf.gz --variant ACC-212.g.vcf.gz --variant ACC-215.g.vcf.gz --variant ACC-2210.g.vcf.gz --variant ACC-2218.g.vcf.gz --variant ACC-2219.g.vcf.gz --variant ACC-2225.g.vcf.gz --variant ACC-2226.g.vcf.gz --variant ACC-2227.g.vcf.gz --variant ACC-2228.g.vcf.gz --variant ACC-2229.g.vcf.gz --variant ACC-2230.g.vcf.gz --variant ACC-244.g.vcf.gz --variant ACC-248.g.vcf.gz --variant ACC-250.g.vcf.gz --variant ACC-253.g.vcf.gz --variant ACC-262.g.vcf.gz --variant ACC-281.g.vcf.gz --variant ACC-319.g.vcf.gz --variant ACC-433.g.vcf.gz --variant ACC-438.g.vcf.gz --variant ACC-504.g.vcf.gz --variant ACC-546.g.vcf.gz --variant ACC-616.g.vcf.gz -O soybean.cohort.g.vcf.gz
```

8. genotype samples, select SNP variants and invariant sites
```
gatk --java-options "-Xmx4g" GenotypeGVCFs -R /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/GCF_000004515.6_Glycine_max_v4.0_genomic.fna -V soybean.cohort.g.vcf.gz -O soybean.output.vcf.gz

gatk --java-options "-Xmx4g" GenotypeGVCFs -R /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/GCF_000004515.6_Glycine_max_v4.0_genomic.fna -all-sites -V soybean.cohort.g.vcf.gz -O soybean.var_invar.vcf.gz
genotypegvcf.sh
```

9. select SNP variants and invariant sites
```
gatk SelectVariants -V soybean.output.vcf.gz -select-type SNP -O soybean.snps.vcf.gz
vcftools --gzvcf soybean.snps.vcf.gz --out soybean_snps_filtered --recode --recode-INFO-all --minDP 20 --minGQ 30  --minQ 30

gatk SelectVariants -V soybean.var_invar.vcf.gz -select-type NO_VARIATION -O soybean.invar.vcf.gz 
vcftools --gzvcf soybean.invar.vcf.gz --out soybean.invar --recode --recode-INFO-all --minDP 20  --minQ 30 --max-missing 0.5 --bed /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed
```

10. substitute SNPs into ref genome to create sample specific ref genomes
```
bgzip soybean_snps_filtered.recode.vcf
tabix soybean_snps_filtered.recode.vcf.gz
cat sample_chr | while read -r value1 value2 remainder ; do samtools faidx /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/GCF_000004515.6_Glycine_max_v4.0_genomic.fna $value2 | bcftools consensus -M N -s $value1 -p ${value1/$/_} -H 1pIu soybean_snps_filtered.recode.vcf.gz >>${value2/$/_soybean.fa}; done
```

11. index each new ref genome and move into seperate folders
```
cat chrlist | while read line; do /data/proj2/popgen/a.ramesh/software/faSplit byname $line ${line/^/chr_} ; done
cat samplenames | while read line ; do cat $line*.fa >$line.merged.fa  ; done
for file in *.merged.fa ; do samtools faidx $file; done
for file in *.merged.fa ; do picard CreateSequenceDictionary -R $file -O ${file/.fa/.dict}; done
cat samplenames | while read line ; do mkdir $line  ; done
cat samplenames | while read line ; do mv $line.merged.fa $line  ; done
cat samplenames | while read line ; do mv $line.merged.dict $line  ; done
cat samplenames | while read line ; do mv $line.merged.fa.fai $line  ; done
```

12. index genome using bismark for methylation mapping
```
/data/proj2/popgen/a.ramesh/software/Bismark-0.24.0/bismark_genome_preparation --hisat2 --verbose --path_to_aligner /data/proj2/popgen/a.ramesh/software/hisat2-2.2.1/ /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes
cat samplenames2 | while read line ; do /data/proj2/popgen/a.ramesh/software/Bismark-0.24.0/bismark_genome_preparation --hisat2 --verbose --path_to_aligner /data/proj2/popgen/a.ramesh/software/hisat2-2.2.1/ $line ; done
```

13. May methylomes to sample specific reference genomes
```
sed 's/^/\/data\/proj2\/popgen\/a.ramesh\/projects\/methylomes\/soybean\/pseudogenomes\//' samplenames | paste samplenames - >samplenames4
cat samplenames4 |  while read -r value1 value2 remainder ; do /data/proj2/popgen/a.ramesh/software/Bismark-0.24.0/bismark --multicore 4 --hisat2 --path_to_hisat2 /data/proj2/popgen/a.ramesh/software/hisat2-2.2.1/ --genome_folder $value2 -1 $value1.1.paired.fq.gz -2 $value1.2.paired.fq.gz  ; done
```

14. Deduplicate methyalation reads
```
for file in *_bismark_hisat2_pe.bam; do /data/proj2/popgen/a.ramesh/software/Bismark-0.24.0/deduplicate_bismark --bam $file ; done
```

15. Call methylation variants
```
cat samplenames4 |  while read -r value1 value2 remainder ; do /data/proj2/popgen/a.ramesh/software/Bismark-0.24.0/bismark_methylation_extractor --multicore 4 --gzip --bedGraph --buffer_size 10G --cytosine_report --genome_folder $value2 $value1.1.paired_bismark_hisat2_pe.deduplicated.bam ; done
```

16. Do binomial test
```
setwd("/data/proj2/popgen/a.ramesh/projects/methylomes/soybean/data")
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
cov_context$chromosome <- gsub("_mapped.bam","", gsub(gsub(".1.paired_bismark_hisat2_pe.deduplicated.bismark.cov","",covfile$V1)[1],"",cov_context$chromosome))
cov_context <- cov_context[cov_context$count.methylated + cov_context$count.unmethylated > 4, ]

cov_context$ID <- paste(cov_context$chromosome,cov_context$position,sep = "-")
cov_context$count.total <- cov_context$count.methylated + cov_context$count.unmethylated
cov_context$pval <- 0
noncoversionrate <- sum(cov_context[cov_context$chromosome %in% "NC_007942.1",]$count.methylated)/sum(cov_context[cov_context$chromosome %in% "NC_007942.1",]$count.total)

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
  cov_context$chromosome <- gsub("_mapped.bam","", gsub(gsub(".1.paired_bismark_hisat2_pe.deduplicated.bismark.cov","",covfile$V1)[i],"",cov_context$chromosome))
  cov_context <- cov_context[cov_context$count.methylated + cov_context$count.unmethylated > 4, ]
  
  cov_context$ID <- paste(cov_context$chromosome,cov_context$position,sep = "-")
  cov_context$count.total <- cov_context$count.methylated + cov_context$count.unmethylated
  cov_context$pval <- 0
  noncoversionrate <- sum(cov_context[cov_context$chromosome %in% "NC_007942.1",]$count.methylated)/sum(cov_context[cov_context$chromosome %in% "NC_007942.1",]$count.total)
  
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
write.table(cbind(meta,cov_context3),file="soybean_meth.vcf",quote = F, row.names = F,sep="\t")

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
write.table(cbind(meta2,cov_context4),file="soybean_meth_var_invar.vcf",quote = F, row.names = F,sep="\t")
```

17. Some general stats
```
vcftools --gzvcf soybean_snps_filtered.recode.vcf.gz --out soybean_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --freq --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed
vcftools --gzvcf soybean_snps_filtered.recode.vcf.gz --out soybean_snp --min-alleles 2  --max-missing 0.5 --window-pi  10000 --bed /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed
vcftools --vcf soybean.invar.recode.vcf --out soybean_invar  --max-missing 0.5 --site-pi --bed /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed
vcftools --gzvcf soybean_snps_filtered.recode.vcf.gz --out soybean_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --het --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed
vcftools --gzvcf soybean_snps_filtered.recode.vcf.gz --out soybean_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --geno-r2 --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed --ld-window-bp 100

vcftools --gzvcf soybean_snps_filtered.recode.vcf.gz --out soybean_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --recode --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed

#vcftools --gzvcf soybean_snps_filtered.recode.vcf.gz --chr NC_016088.4 --chr NC_016089.4 --chr NC_016090.4 --chr NC_016091.4 --chr NC_038241.2 --chr NC_038242.2 --chr NC_038243.2 --chr NC_038244.2 --chr NC_038245.2 --chr NC_038246.2 --chr NC_038247.2 --chr NC_038248.2 --chr NC_038249.2 --chr NC_038250.2 --chr NC_038251.2 --chr NC_038252.2 --chr NC_038253.2 --chr NC_038254.2 --chr NC_038255.2 --chr NC_038256.2 --recode --out filtered_plink --max-missing 0.5
#/data/proj2/popgen/a.ramesh/software/plink --vcf filtered_plink.recode.vcf --double-id  --allow-extra-chr --set-missing-var-ids @:# --maf 0.01  -r2 gz --ld-window 100 --ld-window-kb 1000 --ld-window-r2 0  --out soybean_ld --keep wild_plink
#python2.7 /data/proj2/popgen/a.ramesh/software/ld_decay_calc.py  -i soybean_ld.ld.gz -o soybean_ld_mean

cd /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/data/
vcftools --vcf soybean_meth_all.vcf  --out soybean_meth --min-alleles 2 --max-alleles 2 --max-missing 0.5 --freq --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed
vcftools --vcf soybean_meth_all.vcf  --out soybean_meth --min-alleles 2 --max-alleles 2 --max-missing 0.5 --site-pi --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed
vcftools --vcf soybean_meth_all.vcf  --out soybean_meth --min-alleles 2 --max-alleles 2 --max-missing 0.5 --geno-r2 --bed  /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.bed --ld-window-bp 100

#vcftools --vcf soybean_meth_all.vcf --chr NC_016088.4 --chr NC_016089.4 --chr NC_016090.4 --chr NC_016091.4 --chr NC_038241.2 --chr NC_038242.2 --chr NC_038243.2 --chr NC_038244.2 --chr NC_038245.2 --chr NC_038246.2 --chr NC_038247.2 --chr NC_038248.2 --chr NC_038249.2 --chr NC_038250.2 --chr NC_038251.2 --chr NC_038252.2 --chr NC_038253.2 --chr NC_038254.2 --chr NC_038255.2 --chr NC_038256.2 --recode --out filtered_meth_plink --max-missing 0.5
/data/proj2/popgen/a.ramesh/software/plink --vcf filtered_meth_plink.recode.vcf --double-id --mac 2 --max-mac 2 --allow-extra-chr  --maf 0.01 --out sortedfile --keep wild_plink --make-bed
/data/proj2/popgen/a.ramesh/software/plink --bfile sortedfile --set-missing-var-ids @:#  -r2 gz --ld-window 100 --ld-window-kb 1000 --ld-window-r2 0 --out soybean_meth_ld --allow-extra-chr
python2.7 /data/proj2/popgen/a.ramesh/software/ld_decay_calc.py -i soybean_meth_ld.ld.gz -o soybean_meth_ld_mean
```

18. LD decay plotting script 
```
## LD decay plotting script 

library(tidyverse)

## soybean
my_bins_snps_soybean <- "soybean_ld_mean.ld_decay_bins"
ld_bins_snps_soybean  <- read_tsv(my_bins_snps_soybean )
my_bins_meth_soybean  <- "soybean_meth_ld_mean.ld_decay_bins"
ld_bins_meth_soybean  <- read_tsv(my_bins_meth_soybean)
ld_bins_snps_soybean$type <- "SNP"
ld_bins_meth_soybean$type <- "SMP"
ld_bins_all_soybean <- rbind(ld_bins_snps_soybean ,ld_bins_meth_soybean )
ggplot(ld_bins_all_soybean, aes(x=distance, y=avg_R2,color=type)) + 
  geom_point() +
  geom_smooth(se = F) +
  xlab("Distance (bp)") + ylab(expression(italic(r)^2)) +
  xlim(c(0,1000000))
ggplot(ld_bins_all_soybean, aes(x=distance, y=avg_R2,color=type)) + 
  geom_point() +
  geom_smooth(se = F) +
  xlab("Distance (bp)") + ylab(expression(italic(r)^2)) +
  xlim(c(0,50000))
```

19. R plots for other metrics. Per site or across genome, not per gene.
```
# SFS
soybean_snp.frq <- read.table(file="soybean_snp.frq",row.names = NULL)
soybean_snp.frq$raf <- as.numeric(gsub(".*:","",soybean_snp.frq$N_CHR))
soybean_snp.frq$aaf <- as.numeric(gsub(".*:","",soybean_snp.frq$X.ALLELE.FREQ.))
soybean_snp.frq$maf <- apply(soybean_snp.frq[7:8],1,min)
soybean_snp.frq <- soybean_snp.frq[soybean_snp.frq$maf > 0,]
soybean_snp.frq <- soybean_snp.frq[soybean_snp.frq$maf < 1,]
soybean_snp.frq <- soybean_snp.frq[soybean_snp.frq$N_ALLELES > 87,]
soybean_snp.frq$type <- 'SNP'

soybean_meth.frq <- read.table(file="soybean_meth.frq",row.names = NULL)
soybean_meth.frq$raf <- as.numeric(gsub(".*:","",soybean_meth.frq$N_CHR))
soybean_meth.frq$aaf <- as.numeric(gsub(".*:","",soybean_meth.frq$X.ALLELE.FREQ.))
soybean_meth.frq$maf <- apply(soybean_meth.frq[7:8],1,min)
soybean_meth.frq <- soybean_meth.frq[soybean_meth.frq$maf > 0,]
soybean_meth.frq <- soybean_meth.frq[soybean_meth.frq$maf < 1,]
soybean_meth.frq <- soybean_meth.frq[soybean_meth.frq$N_ALLELES > 81,]
soybean_meth.frq$type <- 'SMP'

snp_smp_soybean <- rbind(soybean_snp.frq,soybean_meth.frq)

soybean_sfs <- ggplot(snp_smp_soybean,aes(fill=type,x=maf)) +
  geom_histogram(aes(y=0.05*..density..),binwidth=0.05, alpha=0.4, position='identity') +
  ylab("Proportion of sites")

nrow(soybean_meth.frq)/nrow(soybean_snp.frq)

## pi
soybean_snp.sites.pi <- read.table(file="soybean_snp.sites.pi",row.names = NULL, header = T)
soybean_invar.sites.pi <- read.table(file="soybean_invar.sites.pi",row.names = NULL, header = T)
soybean_snp.sites.pi <- rbind(soybean_snp.sites.pi,soybean_invar.sites.pi)
soybean_snp.sites.pi$type <- 'SNP'

soybean_meth.sites.pi <- read.table(file="soybean_meth.sites.pi",row.names = NULL, header = T)
soybean_meth.sites.pi$type <- 'SMP'

snp_smp_soybean <- rbind(soybean_snp.sites.pi,soybean_meth.sites.pi)

df_soybean_pi <- dplyr::count(snp_smp_soybean, type)
df_soybean_pi$n <- paste("n=",df_soybean_pi$n,sep="")

soybean_pi <- ggplot(snp_smp_soybean,aes(x=type,y=PI)) +
  geom_boxplot() +
  ylab("Per site π") +
  xlab("")+
  ggtitle("Soybean")+
  geom_text(data = df_soybean_pi, aes(y = 0.6, label = n))

soybean_pi
mean(soybean_snp.sites.pi$PI,na.rm=T)
median(soybean_snp.sites.pi$PI,na.rm=T)

nrow(soybean_meth.sites.pi)/nrow(soybean_snp.sites.pi)

soybean_prop_seg_smp <- 54832/(54832+167691)
soybean_prop_seg_snp <- 214773/(214773+221498)

## LD
soybean_snp.geno.ld <- read.table(file="soybean_snp.geno.ld",row.names = NULL, header = T)
soybean_snp.geno.ld$type <- 'SNP'

soybean_meth.geno.ld <- read.table(file="soybean_meth.geno.ld",row.names = NULL, header = T)
soybean_meth.geno.ld$type <- 'SMP'

snp_smp_soybean <- rbind(soybean_snp.geno.ld,soybean_meth.geno.ld)

soybean_ld <- ggplot(snp_smp_soybean,aes(x=type,y=R.2)) +
  geom_violin() +
  ylab("Pairwise LD (r2)") +
  xlab("") +
  ggtitle("Soybean")

soybean_ld <- ggplot(snp_smp_soybean,aes(fill=type,x=R.2)) +
  geom_histogram(aes(y=0.1*..density..),binwidth=0.1, alpha=0.3, position='identity') +
  ylab("Proportion of sites") +
  xlab("Pairwise LD (r2)") +
  ggtitle("Soybean") +
  scale_fill_manual(values=c("red","green"))

nrow(soybean_meth.geno.ld)/nrow(soybean_snp.geno.ld)


```

20. Split reference genome into genes
```
cd  /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes
sed -e 's/\t/:/' -e  's/\t/-/' gene_pos.bed >gene_pos.list
cd /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/data_wgs
cat /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/gene_pos.list | while read -r line ; do samtools faidx /data/proj2/popgen/a.ramesh/projects/methylomes/soybean/genomes/GCF_000004515.6_Glycine_max_v4.0_genomic.fna $line >>genes.fasta; done
mkdir genes_fasta/
/data/proj2/popgen/a.ramesh/software/faSplit byname genes.fasta genes_fasta/
cd genes_fasta/
ls *fa >filenames
```

21. Rscript to count number of cytosines

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
