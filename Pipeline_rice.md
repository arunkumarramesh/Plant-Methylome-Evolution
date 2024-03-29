# Pipeline for getting SMPs and SNPs for rice

1. Download data from NCBI
```
## Genomes usually donwloaded from Ensembl, need to ensure it includes the choloplast contig. Index with samtools faidx.

cd /proj/popgen/a.ramesh/projects/methylomes/rice
/proj/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/prefetch --option-file PRJNA597475_rice_Acc_List.txt  -O data
/proj/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/prefetch --option-file SRR_Acc_List_rice_rnaseq.txt  -O data_rna

cd /proj/popgen/a.ramesh/projects/methylomes/rice/data
for file in *.sra; do /proj/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/fastq-dump --split-3 --gzip  $file; done

cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna
for file in *.sra; do /proj/popgen/a.ramesh/software/sratoolkit.3.0.0-centos_linux64/bin/fastq-dump --split-3 --gzip  $file; done

/proj/popgen/a.ramesh/projects/methylomes/rice/genomes
grep 'exon' Oryza_sativa.IRGSP-1.0.55.gtf | cut -f 1,4,5 >gene_pos.bed
```

2. Trim data
```
cd /proj/popgen/a.ramesh/projects/methylomes/lyrata/data
for file in *_1.fastq.gz; do java -jar /proj/popgen/a.ramesh/software/Trimmomatic-0.39/trimmomatic-0.39.jar PE -phred33 -threads 20 $file ${file/_1.fastq.gz/_2.fastq.gz} ${file/_1.fastq.gz/_1.paired.fq.gz} ${file/_1.fastq.gz/_1.unpaired.fq.gz} ${file/_1.fastq.gz/_2.paired.fq.gz} ${file/_1.fastq.gz/_2.unpaired.fq.gz} ILLUMINACLIP:TruSeq3-PE.fa:2:30:10:2:True LEADING:3 TRAILING:3 SLIDINGWINDOW:4:20 MINLEN:36; done

cd /proj/popgen/a.ramesh/projects/methylomes/lyrata/data_rna
for file in *_1.fastq.gz; do java -jar /proj/popgen/a.ramesh/software/Trimmomatic-0.39/trimmomatic-0.39.jar PE -phred33 -threads 20 $file ${file/_1.fastq.gz/_2.fastq.gz} ${file/_1.fastq.gz/_1.paired.fq.gz} ${file/_1.fastq.gz/_1.unpaired.fq.gz} ${file/_1.fastq.gz/_2.paired.fq.gz} ${file/_1.fastq.gz/_2.unpaired.fq.gz} ILLUMINACLIP:TruSeq3-PE.fa:2:30:10:2:True LEADING:3 TRAILING:3 SLIDINGWINDOW:4:20 MINLEN:36; done
```

3. Index reference genome and map RNA-seq data with with STAR 2-pass mode

```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/genomes
/proj/popgen/a.ramesh/software/STAR-2.7.10b/source/STAR --runMode genomeGenerate --genomeDir STARindex/ --genomeFastaFiles /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/Oryza_sativa.IRGSP-1.0.dna.toplevel.fa --sjdbGTFfile Oryza_sativa.IRGSP-1.0.55.gtf --sjdbOverhang 149 --runThreadN 20

cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/
ulimit -n 10000
for file in *_1.paired.fq.gz; do /proj/popgen/a.ramesh/software/STAR-2.7.10b/source/STAR --runMode alignReads --genomeDir /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/STARindex/ --readFilesIn $file ${file/_1.paired.fq.gz/_2.paired.fq.gz} --readFilesCommand zcat --outFileNamePrefix ${file/_1.paired.fq.gz//}  --outReadsUnmapped Fastx --outSAMtype BAM SortedByCoordinate --twopassMode Basic --runThreadN 20  ; done

find . -type f -name "Aligned.sortedByCoord.out.bam" -printf "/%P\n" | while read FILE ; do DIR=$(dirname "$FILE" ); mv ."$FILE" ."$DIR""$DIR".sort.bam;done
find . -name '*.sort.bam' -exec mv {} . \;

```

4. Add read groups
```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/

for file in SRR*.sort.bam ; do java -jar /proj/popgen/a.ramesh/software/picard.jar AddOrReplaceReadGroups -I $file -O ${file/.sort.bam/_readgroup.bam} -LB species -PL illumina -PU 1 -SM $file; done

```

5. Mark duplicate reads and index
```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/

for file in SRR*.sort.bam ; do java -jar /proj/popgen/a.ramesh/software/picard.jar AddOrReplaceReadGroups -I $file -O ${file/.sort.bam/_readgroup.bam} -LB species -PL illumina -PU 1 -SM $file; done

java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751892_readgroup.bam -I SRR10751893_readgroup.bam -O Minghui63_marked.bam -M Minghui63_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751900_readgroup.bam -I SRR10751901_readgroup.bam -O ZhenShan97_marked.bam -M ZhenShan97_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751908_readgroup.bam -I SRR10751909_readgroup.bam -O Nipponare_marked.bam -M Nipponare_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751916_readgroup.bam -I SRR10751917_readgroup.bam -O HKG98_marked.bam -M HKG98_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751918_readgroup.bam -I SRR10751919_readgroup.bam -O Garia_marked.bam -M Garia_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751920_readgroup.bam -I SRR10751921_readgroup.bam -O Aijiaonante_marked.bam -M Aijiaonante_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751922_readgroup.bam -I SRR10751923_readgroup.bam -O Xibaizhan_marked.bam -M Xibaizhan_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751924_readgroup.bam -I SRR10751925_readgroup.bam -O WH139_marked.bam -M WH139_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751926_readgroup.bam -I SRR10751927_readgroup.bam -O Nanjing11_marked.bam -M Nanjing11_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751928_readgroup.bam -I SRR10751929_readgroup.bam -O 9311_marked.bam -M 9311_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751930_readgroup.bam -I SRR10751931_readgroup.bam -O Y134_marked.bam -M Y134_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751932_readgroup.bam -I SRR10751933_readgroup.bam -O IR72_marked.bam -M IR72_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751934_readgroup.bam -I SRR10751935_readgroup.bam -O PeiC122_marked.bam -M PeiC122_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751936_readgroup.bam -I SRR10751937_readgroup.bam -O YongChalByo_marked.bam -M YongChalByo_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751938_readgroup.bam -I SRR10751939_readgroup.bam -O TAINO38_marked.bam -M TAINO38_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751940_readgroup.bam -I SRR10751941_readgroup.bam -O Ginga_marked.bam -M Ginga_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751942_readgroup.bam -I SRR10751943_readgroup.bam -O LABELLE_marked.bam -M LABELLE_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751944_readgroup.bam -I SRR10751945_readgroup.bam -O Latisai1_marked.bam -M Latisai1_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751946_readgroup.bam -I SRR10751947_readgroup.bam -O GasymHany_marked.bam -M GasymHany_metrics.txt
java -jar /proj/popgen/a.ramesh/software/picard.jar MarkDuplicates -I SRR10751948_readgroup.bam -I SRR10751949_readgroup.bam -O BASMATI385_marked.bam -M BASMATI385_metrics.txt

for file in *_marked.bam ; do java -jar /proj/popgen/a.ramesh/software/picard.jar BuildBamIndex -I $file ; done

```

6. SplitNCigarReads and change read groups to represent accessions.
```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/
for file in *_marked.bam ; do /proj/popgen/a.ramesh/software/gatk-4.3.0.0/gatk SplitNCigarReads -R /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/Oryza_sativa.IRGSP-1.0.dna.toplevel.fa  -I $file -O  ${file/_marked.bam/_split.bam}  ; done
for file in *_split.bam ; do picard BuildBamIndex -I $file; done

for file in *_split.bam ; do java -jar /proj/popgen/a.ramesh/software/picard.jar AddOrReplaceReadGroups -I $file -O ${file/_split.bam/_readgroup.bam} -LB species -PL illumina -PU 1 -SM $file; done
for file in *_readgroup.bam ; do java -jar /proj/popgen/a.ramesh/software/picard.jar BuildBamIndex -I $file; done

ls *_readgroup.bam > lc_files
ls *_readgroup.bam | sed 's/*_readgroup.bam//' | paste - lc_files >lc_files2 ## for haplotyper caller script (file also called hc_files2)
```

7. Run haplotype caller using perl script to parallelize
```
#!/usr/bin/perl -w


my $dir = "/proj/popgen/a.ramesh/projects/methylomes/rice/data_rna";
my $input = "lc_files2";
chdir("$dir") or die "couldn't move to input directory";

open (INPUT_FILE, "$input") || die "couldn't open the input file $input!";
                    while (my $line = <INPUT_FILE>) {
                        chomp $line;
my @array = split(/\t/,$line);
my $sample = $array[0];
my $file = $array[1];

my $SLURM_header = <<"SLURM";
#!/bin/bash
#SBATCH -c 2
#SBATCH --mem=100000
#SBATCH -J lc
#SBATCH -o lc.out
#SBATCH -e lc.err

cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna

SLURM

  my $tmp_file = "hc.$sample";
  open (SLURM, ">$tmp_file") or die "Couldn't open temp file\n";
  $SLURM_header = $SLURM_header;
  print SLURM "$SLURM_header\n\n";
  print SLURM "/proj/popgen/a.ramesh/software/gatk-4.3.0.0/gatk HaplotypeCaller -R /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/Oryza_sativa.IRGSP-1.0.dna.toplevel.fa  -I $file -O  $sample".".g.vcf.gz  -ERC GVCF\n";
  close SLURM;
  system("sbatch $tmp_file");

}

            close(INPUT_FILE);

```


8. Combine GVCFs
```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/
/proj/popgen/a.ramesh/software/gatk-4.3.0.0/gatk CombineGVCFs -R /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/Oryza_sativa.IRGSP-1.0.dna.toplevel.fa --variant C019.g.vcf.gz --variant C051.g.vcf.gz --variant C135.g.vcf.gz --variant C139.g.vcf.gz --variant C148.g.vcf.gz --variant C151.g.vcf.gz --variant MH63.g.vcf.gz --variant NIP.g.vcf.gz --variant W081.g.vcf.gz --variant W105.g.vcf.gz --variant W125.g.vcf.gz --variant W128.g.vcf.gz --variant W161.g.vcf.gz --variant W169.g.vcf.gz --variant W257.g.vcf.gz --variant W261.g.vcf.gz --variant W286.g.vcf.gz --variant W294.g.vcf.gz --variant W306.g.vcf.gz --variant ZS97.g.vcf.gz -O rice.cohort.g.vcf.gz
```

9. Genotype combined GVCFs

```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/
/proj/popgen/a.ramesh/software/gatk-4.3.0.0/gatk --java-options "-Xmx4g" GenotypeGVCFs -R /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/Oryza_sativa.IRGSP-1.0.dna.toplevel.fa  -V rice.cohort.g.vcf.gz -O rice.output.vcf.gz

/proj/popgen/a.ramesh/software/gatk-4.3.0.0/gatk --java-options "-Xmx4g" GenotypeGVCFs -R /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/Oryza_sativa.IRGSP-1.0.dna.toplevel.fa -all-sites -V rice.cohort.g.vcf.gz -O rice.var_invar.vcf.gz
```

10. Filter vcf, only keep high quality SNPs
```
#/proj/popgen/a.ramesh/software/gatk-4.3.0.0/gatk SelectVariants -V rice.output.vcf.gz -select-type SNP -O rice.snps.vcf.gz
#/proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --gzvcf rice.snps.vcf.gz --out rice_snps_filtered --recode --recode-INFO-all --minDP 20 --minGQ 30 --minQ 30
#sed -i -e 's/9311_split.bam/C148/' -e 's/Aijiaonante_split.bam/C019/' -e 's/BASMATI385_split.bam/W306/' -e 's/Garia_split.bam/W286/' -e 's/GasymHany_split.bam/W081/' -e 's/Ginga_split.bam/W261/' -e 's/HKG98_split.bam/W105/' -e 's/IR72_split.bam/W169/' -e 's/LABELLE_split.bam/W294/' -e 's/Latisai1_split.bam/W257/' -e 's/Minghui63_split.bam/MH63/' -e 's/Nanjing11_split.bam/C151/' -e 's/Nipponare_split.bam/NIP/' -e 's/PeiC122_split.bam/C051/' -e 's/TAINO38_split.bam/W128/' -e 's/WH139_split.bam/C139/' -e 's/Xibaizhan_split.bam/C135/' -e 's/Y134_split.bam/W161/' -e 's/YongChalByo_split.bam/W125/' -e 's/ZhenShan97_split.bam/ZS97/' rice_snps_filtered.recode.vcf


#/proj/popgen/a.ramesh/software/gatk-4.3.0.0/gatk SelectVariants -V rice.var_invar.vcf.gz -select-type NO_VARIATION -O rice.invar.vcf.gz
#/proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --gzvcf rice.invar.vcf.gz --out rice.invar --recode --recode-INFO-all --minDP 20 --minGQ 30 --minQ 30 --max-missing 0.5 --bed /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/gene_pos.bed
#sed -i -e 's/9311_split.bam/C148/' -e 's/Aijiaonante_split.bam/C019/' -e 's/BASMATI385_split.bam/W306/' -e 's/Garia_split.bam/W286/' -e 's/GasymHany_split.bam/W081/' -e 's/Ginga_split.bam/W261/' -e 's/HKG98_split.bam/W105/' -e 's/IR72_split.bam/W169/' -e 's/LABELLE_split.bam/W294/' -e 's/Latisai1_split.bam/W257/' -e 's/Minghui63_split.bam/MH63/' -e 's/Nanjing11_split.bam/C151/' -e 's/Nipponare_split.bam/NIP/' -e 's/PeiC122_split.bam/C051/' -e 's/TAINO38_split.bam/W128/' -e 's/WH139_split.bam/C139/' -e 's/Xibaizhan_split.bam/C135/' -e 's/Y134_split.bam/W161/' -e 's/YongChalByo_split.bam/W125/' -e 's/ZhenShan97_split.bam/ZS97/' rice.invar.recode.vcf

/proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --gzvcf rice.var_invar.vcf.gz --out rice.allsites --recode --recode-INFO-all --minDP 20 --minQ 30 --max-missing 1 --bed /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/gene_pos.bed
sed -i -e 's/9311_split.bam/C148/' -e 's/Aijiaonante_split.bam/C019/' -e 's/BASMATI385_split.bam/W306/' -e 's/Garia_split.bam/W286/' -e 's/GasymHany_split.bam/W081/' -e 's/Ginga_split.bam/W261/' -e 's/HKG98_split.bam/W105/' -e 's/IR72_split.bam/W169/' -e 's/LABELLE_split.bam/W294/' -e 's/Latisai1_split.bam/W257/' -e 's/Minghui63_split.bam/MH63/' -e 's/Nanjing11_split.bam/C151/' -e 's/Nipponare_split.bam/NIP/' -e 's/PeiC122_split.bam/C051/' -e 's/TAINO38_split.bam/W128/' -e 's/WH139_split.bam/C139/' -e 's/Xibaizhan_split.bam/C135/' -e 's/Y134_split.bam/W161/' -e 's/YongChalByo_split.bam/W125/' -e 's/ZhenShan97_split.bam/ZS97/' rice.allsites.recode.vcf
```

11. Get chromosome and sample names, can be done many ways
```
cut -f 1 ../genomes/Oryza_sativa.IRGSP-1.0.dna.toplevel.fa.fai >chrlist
ls *_split.bam | sed 's/_split.bam//' >samplenames
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
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/
/proj/popgen/a.ramesh/software/htslib-1.16/bgzip rice_snps_filtered.recode.vcf
/proj/popgen/a.ramesh/software/htslib-1.16/tabix  rice_snps_filtered.recode.vcf.gz
cat sample_chr | while read -r value1 value2 remainder ; do samtools faidx /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/Oryza_sativa.IRGSP-1.0.dna.toplevel.fa $value2 | /proj/popgen/a.ramesh/software/bcftools-1.16/bcftools consensus -M N -s $value1 -p ${value1/$/_} -H 1pIu rice_snps_filtered.recode.vcf.gz >>${value2/$/_rice.fa}; done
cat chrlist | while read line; do /data/proj2/popgen/a.ramesh/software/faSplit byname $line ${line/^/chr_} ; done
mkdir /proj/popgen/a.ramesh/projects/methylomes/rice/pseudogenomes
mv *fa /proj/popgen/a.ramesh/projects/methylomes/rice/pseudogenomes
cd /proj/popgen/a.ramesh/projects/methylomes/rice/pseudogenomes
cat samplenames | while read line ; do cat $line*.fa >$line.merged.fa  ; done
for file in *.merged.fa ; do samtools faidx $file; done
for file in *.merged.fa ; do java -jar /proj/popgen/a.ramesh/software/picard.jar CreateSequenceDictionary -R $file -O ${file/.fa/.dict}; done
cat samplenames | while read line ; do mkdir $line  ; done
cat samplenames | while read line ; do mv $line.merged.fa $line  ; done
cat samplenames | while read line ; do mv $line.merged.dict $line  ; done
cat samplenames | while read line ; do mv $line.merged.fa.fai $line  ; done
cat samplenames2 | while read line ; do /proj/popgen/a.ramesh/software/Bismark-0.24.0/bismark_genome_preparation --hisat2 --verbose --path_to_aligner /proj/popgen/a.ramesh/software/hisat2-2.2.1/ $line ; done

```
14. Now map methylation reads with Bismark
```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data

sed 's/^/\/proj\/popgen\/a.ramesh\/projects\/methylomes\/rice\/pseudogenomes\//' samplenames | paste samplenames >samplenames2

cat samplenames2 |  while read -r value1 value2 remainder ; do /proj/popgen/a.ramesh/software/Bismark-0.24.0/bismark --multicore 4 --hisat2 --path_to_hisat2 /proj/popgen/a.ramesh/software/hisat2-2.2.1/ --genome_folder $value2 -1 $value1.1.paired.fq.gz -2 $value1.2.paired.fq.gz  ; done

```

15. Deduplicate methylation reads with Bismark
```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data
for file in *.1.paired_bismark_hisat2_pe.bam ; do /proj/popgen/a.ramesh/software/Bismark-0.24.0/deduplicate_bismark --bam $file ; done
```

16. Call methylation variants
```
d /proj/popgen/a.ramesh/projects/methylomes/rice/data
cat samplenames2 |  while read -r value1 value2 remainder ; do /proj/popgen/a.ramesh/software/Bismark-0.24.0/bismark_methylation_extractor --multicore 4 --gzip --bedGraph --buffer_size 10G --cytosine_report --genome_folder $value2 $value1.1.paired_bismark_hisat2_pe.deduplicated.bam ; done
```

17. This R script is to genotype mC using a binomial test
```
setwd("/data/proj2/popgen/a.ramesh/projects/methylomes/rice/data")
library(dplyr)
library(ggplot2)

covfile <- read.table(file="covfiles")
contextfiles <- read.table(file="contextfiles")

cov <- read.table(file=covfile[1,], header=F)
colnames(cov) <- c("chromosome", "position", "end.position", "methylation.percentage", "count.methylated", "count.unmethylated" )
cov$ID <- paste(cov$chromosome,cov$position,sep = "_")

context <- read.table(file=contextfiles[1,],header=F)
colnames(context) <- c("chromosome", "position", "strand", "count.methylated", "count.unmethylated", "C-context", "trinucleotide context")
context$ID <- paste(context$chromosome,context$position,sep = "_")

cov_context <- inner_join(cov,context[c(3,6:8)],by="ID")
dim(cov_context)
cov_context$chromosome <- gsub(gsub(".1.paired_bismark_hisat2_pe.deduplicated.bismark.cov","",covfile$V1)[1],"",cov_context$chromosome)
cov_context <- cov_context[cov_context$count.methylated + cov_context$count.unmethylated > 4, ]

cov_context$ID <- paste(cov_context$chromosome,cov_context$position,sep = "_")
cov_context$count.total <- cov_context$count.methylated + cov_context$count.unmethylated
cov_context$pval <- 0
noncoversionrate <- sum(cov_context[cov_context$chromosome %in% "Pt",]$count.methylated)/sum(cov_context[cov_context$chromosome %in% "Pt",]$count.total)
if (is.na(noncoversionrate)) {
  noncoversionrate <- 0
}
samplename <- gsub(".1.paired_bismark_hisat2_pe.deduplicated.bismark.cov","",covfile[1,])

b <- apply(cov_context[c(5,11)],1,binom.test,p = noncoversionrate, alternative = c("greater"))
cov_context$pval <- do.call(rbind,lapply(b,function(v){v$p.value}))

cov_context <- cov_context[c(1,2,7,8,9,12)]
cov_context$fdr <- p.adjust(cov_context$pval,method = "fdr")
cov_context$call <- "U"
cov_context[cov_context$fdr < 0.01,]$call <- "M"
cov_context <- cov_context[-c(6,7)]

colnames(cov_context)[ncol(cov_context)] <- samplename
cov_context2 <- cov_context

for (i in 2:nrow(covfile)){
  covfile <- read.table(file="covfiles")
  contextfiles <- read.table(file="contextfiles")
  
  print(i)
  
  cov <- read.table(file=covfile[i,], header=F)
  colnames(cov) <- c("chromosome", "position", "end.position", "methylation.percentage", "count.methylated", "count.unmethylated" )
  cov$ID <- paste(cov$chromosome,cov$position,sep = "_")
  
  context <- read.table(file=contextfiles[i,],header=F)
  colnames(context) <- c("chromosome", "position", "strand", "count.methylated", "count.unmethylated", "C-context", "trinucleotide context")
  context$ID <- paste(context$chromosome,context$position,sep = "_")
  
  cov_context <- inner_join(cov,context[c(3,6:8)],by="ID")
  dim(cov_context)
  cov_context$chromosome <- gsub(gsub(".1.paired_bismark_hisat2_pe.deduplicated.bismark.cov","",covfile$V1)[i],"",cov_context$chromosome)
  cov_context <- cov_context[cov_context$count.methylated + cov_context$count.unmethylated > 4, ]
  
  cov_context$ID <- paste(cov_context$chromosome,cov_context$position,sep = "_")
  cov_context$count.total <- cov_context$count.methylated + cov_context$count.unmethylated
  cov_context$pval <- 0
  noncoversionrate <- sum(cov_context[cov_context$chromosome %in% "Pt",]$count.methylated)/sum(cov_context[cov_context$chromosome %in% "Pt",]$count.total)
  if (is.na(noncoversionrate)) {
    noncoversionrate <- 0
  }
  samplename <- gsub(".1.paired_bismark_hisat2_pe.deduplicated.bismark.cov","",covfile[i,])
    
  b <- apply(cov_context[c(5,11)],1,binom.test,p = noncoversionrate, alternative = c("greater"))
  cov_context$pval <- do.call(rbind,lapply(b,function(v){v$p.value}))

  cov_context <- cov_context[c(1,2,7,8,9,12)]
  cov_context$fdr <- p.adjust(cov_context$pval,method = "fdr")
  cov_context$call <- "U"
  cov_context[cov_context$fdr < 0.01,]$call <- "M"
  cov_context <- cov_context[-c(6,7)]
  

  colnames(cov_context)[ncol(cov_context)] <- samplename
  cov_context <- cov_context[c(3,6)]
  cov_context2 <- full_join(cov_context2,cov_context,by="ID")
}
cov_context2$chromosome <- gsub("_.*","",cov_context2$ID)
cov_context2$position   <- as.numeric(gsub(".*_","",cov_context2$ID))

write.table(cov_context2,file="cov_context3.txt",row.names = F)
```

18. This R script is to generate methylation vcfs 

```
setwd("/data/proj2/popgen/a.ramesh/projects/methylomes/rice/data")

#cov_context3 <- read.table(file="cov_context3.txt",header=T)
#na_count <- apply(cov_context3[6:ncol(cov_context3)], 1, function(x) sum(is.na(x)))
#na_count <- na_count/ncol(cov_context3[6:ncol(cov_context3)])
#cov_context3 <- cov_context3[na_count < 0.5,] # change to appropriate number
#cov_context4 <- cov_context3
#ploymorphic <- apply(cov_context3[6:ncol(cov_context3)], 1, table)
#ploymorphic <- sapply(ploymorphic,length)
#cov_context3 <- cov_context3[ploymorphic > 1,]

#meta <- cov_context3[1:3]
#colnames(meta) <- c("#CHROM","POS","ID")
#meta$REF <- "A"
#meta$ALT <- "T"
#meta$QUAL <- 4000
#meta$FILTER <- "PASS"
#meta$INFO <- "DP=1000"
#meta$FORMAT <- "GT"

#cov_context3 <- cov_context3[-c(1:5)]
#cov_context3[cov_context3 == "U"] <- "0/0"
#cov_context3[cov_context3 == "M"] <- "1/1"
#cov_context3[is.na(cov_context3)] <- "./."

#meta <- cbind(meta,cov_context3)
#meta <- meta[meta$`#CHROM` %in% c("1","2","3","4","5","6","7","8","9","10","11","12"),]
#meta$`#CHROM` <- as.numeric(meta$`#CHROM`)
#meta$POS <- as.numeric(meta$POS)
#meta <- meta[order(meta$`#CHROM`,meta$POS),]

#write.table(meta,file="rice_meth.vcf",quote = F, row.names = F,sep="\t")

#meta2 <- cov_context4[1:3]
#colnames(meta2) <- c("#CHROM","POS","ID")
#meta2$REF <- "A"
#meta2$ALT <- "T"
#meta2$QUAL <- 4000
#meta2$FILTER <- "PASS"
#meta2$INFO <- "DP=1000"
#meta2$FORMAT <- "GT"
#cov_context4 <- cov_context4[-c(1:5)]
#cov_context4[cov_context4 == "U"] <- "0/0"
#cov_context4[cov_context4 == "M"] <- "1/1"
#cov_context4[is.na(cov_context4)] <- "./."
#meta2 <- cbind(meta2,cov_context4)
#meta2 <- meta2[meta2$`#CHROM` %in% c("1","2","3","4","5","6","7","8","9","10","11","12"),]
#meta2$`#CHROM` <- as.numeric(meta2$`#CHROM`)
#meta2$POS <- as.numeric(meta2$POS)
#meta2 <- meta2[order(meta2$`#CHROM`,meta2$POS),]
#write.table(meta2,file="rice_meth_var_invar.vcf",quote = F, row.names = F,sep="\t")
```

19. Get vcf header
```
cat rice_snps_filtered.recode.vcf | grep '##' > vcfheader
cat vcfheader lyrata_meth.vcf >lyrata_meth_all.vcf
cat vcfheader rice_meth_var_invar.vcf >rice_meth_var_invar_all.vcf
```
20. Get methylation vcf file compatable for multihetsep
```
setwd("/data/proj2/popgen/a.ramesh/projects/methylomes/rice/data")

cov_context3 <- read.table(file="rice_meth_var_invar_all.vcf",header=T)
na_count <- apply(cov_context3[6:ncol(cov_context3)], 1, function(x) sum(is.na(x)))
na_count <- na_count/ncol(cov_context3[6:ncol(cov_context3)])
cov_context3 <- cov_context3[na_count < 0.5,] # change to appropriate number
cov_context4 <- cov_context3

meta <- cov_context3[1:3]
colnames(meta) <- c("#CHROM","POS","ID")
meta$REF <- "D"
meta$ALT <- "M"
meta$QUAL <- 4000
meta$FILTER <- "PASS"
meta$INFO <- "DP=1000"
meta$FORMAT <- "GT"

meta <- meta[order(meta$`#CHROM`,meta$POS),]

cov_context3 <- cov_context3[-c(1:5)]
cov_context3[cov_context3 == "U"] <- "0/0"
cov_context3[cov_context3 == "M"] <- "1/1"
cov_context3[is.na(cov_context3)] <- "2/2"

meta <- cbind(meta,cov_context3)

meta[which(apply(meta[10:ncol(meta)], 1, function(r) any(r %in% c("2/2")))),]$ALT <- "M,C"
meta[which(apply(meta[10:ncol(meta)], 1, function(r) all(r %in% c("0/0")))),]$ALT <- "." 

write.table(meta,file="rice_meth_SMCm.vcf",quote = F, row.names = F,sep="\t")
```
21. Generate multihetsep file for SMCm for SMPs. Do for indica1 and indica2
```
cd  /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data

cat vcfheader rice_meth_SMCm.vcf >rice_meth_allsites.vcf
bgzip -f rice_meth_allsites.vcf
tabix -f rice_meth_allsites.vcf.gz
cat samplenames |  while read -r  sample remainder; do bcftools view -c1 -O v -s $sample -o $sample.filtered.vcf rice_meth_allsites.vcf.gz ; done

for file in *.filtered.vcf ; do bcftools annotate -x INFO,^FORMAT/GT -O v -o ${file/.filtered/.annotated} $file ; done

for file in *.annotated.vcf  ; do bgzip -f $file; done
for file in *.annotated.vcf.gz  ; do tabix -f $file; done

cat sample_chr2 |  while read -r value1 value2 remainder ;  do vcftools --gzvcf $value1.annotated.vcf.gz --out $value1.$value2.snps --min-alleles 2 --max-alleles 3 --recode --recode-INFO-all  --chr $value2 ; done

for file in *.snps.recode.vcf  ; do bgzip -f $file; done
for file in *.snps.recode.vcf.gz  ; do tabix -f $file; done

cat chrlist2 | while read line; do /data/proj2/popgen/a.ramesh/software/msmc-tools/generate_multihetsep.py --chr $line  C019.$line.snps.recode.vcf.gz C135.$line.snps.recode.vcf.gz C139.$line.snps.recode.vcf.gz C151.$line.snps.recode.vcf.gz ZS97.$line.snps.recode.vcf.gz >indica1_multihetsep_meth_$line ; done
cat chrlist2 | while read line; do /data/proj2/popgen/a.ramesh/software/msmc-tools/generate_multihetsep.py --chr $line  C148.$line.snps.recode.vcf.gz W161.$line.snps.recode.vcf.gz W169.$line.snps.recode.vcf.gz MH63.$line.snps.recode.vcf.gz >indica2_multihetsep_meth_$line  ; done

```
22. Generate multihetsep file for SMCm for SNPs. Do for indica1 and indica2

```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/

cat samplenames |  while read -r  sample remainder; do /proj/popgen/a.ramesh/software/bcftools-1.16/bcftools view  -O v -s $sample -o $sample.filtered.vcf  --regions 1,2,3,4,5,6,7,8,9,10,11,12 rice_snps_filtered.recode.vcf.gz; done
for file in *.filtered.vcf ; do /proj/popgen/a.ramesh/software/bcftools-1.16/bcftools annotate -x INFO,^FORMAT/GT -O v -o ${file/.filtered/.annotated} $file ; done
for file in *.annotated.vcf; do sed -i '/0\/1/d' $file ; done
cat sample_chr2 |  while read -r value1 value2 remainder ; do  /proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --max-alleles 2 --vcf  $value1.annotated.vcf --out $value1.$value2.snps --recode --recode-INFO-all  --remove-indels --max-missing 1 --chr $value2 ; done
for file in *.snps.recode.vcf; do bgzip -f $file; done
for file in *.snps.recode.vcf.gz; do tabix -f $file; done

cat chrlist2 | while read line; do /proj/popgen/a.ramesh/software/msmc-tools/generate_multihetsep.py --chr $line  C019.$line.snps.recode.vcf.gz C135.$line.snps.recode.vcf.gz C139.$line.snps.recode.vcf.gz C151.$line.snps.recode.vcf.gz ZS97.$line.snps.recode.vcf.gz >indica1_multihetsep_$line ; done
cat chrlist2 | while read line; do /proj/popgen/a.ramesh/software/msmc-tools/generate_multihetsep.py --chr $line  C148.$line.snps.recode.vcf.gz W161.$line.snps.recode.vcf.gz W169.$line.snps.recode.vcf.gz MH63.$line.snps.recode.vcf.gz >indica2_multihetsep_$line  ; done
```

23. Combine multihetsep files for SNPs and SMPs
```
for (i in 1:12){
  snps <- read.table(paste("indica1_multihetsep_",i,sep=""))
  smps <- read.table(paste("indica1_multihetsep_meth_",i,sep=""))
  both <- rbind(snps,smps)
  both <- both[order(both$V1,both$V2),]
  both$V3 <- c(both$V2[1]-1,diff(both$V2))
  write.table(both,paste("indica1_multihetsep_snp_meth_",i,sep=""),sep=" ",col.names = F, row.names = F, quote = F)
}

for (i in 1:12){
  snps <- read.table(paste("indica2_multihetsep_",i,sep=""))
  smps <- read.table(paste("indica2_multihetsep_meth_",i,sep=""))
  both <- rbind(snps,smps)
  both <- both[order(both$V1,both$V2),]
  both$V3 <- c(both$V2[1]-1,diff(both$V2))
  write.table(both,paste("indica2_multihetsep_snp_meth_",i,sep=""),sep=" ",col.names = F, row.names = F, quote = F)
}
```

24. SMCm for indica2
```
###################
# Source function #
###################
library("eSMC2",lib.loc="/data/home/users/a.ramesh/R/x86_64-redhat-linux-gnu-library/4.1/")
################################

########
#Script#
########

CHR=c(1:12)#
mu <- 6.95*10^-9
r=3.6*10^-8

rate_m=3.48*10^(-4)
rate_d=1.47*10^(-3)
rate_m_reg=1.6*10^(-4)
rate_d_reg=9.5*10^(-4)

results_eSMC2=list()
results_SMCm_site=list()
results_SMCm_region=list()
results_SMCm_site_and_region=list()

for(chr in CHR){
  M=8
  O=as.matrix(Get_real_data("/data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/",M=M,paste("indica2_multihetsep_snp_meth_",chr,sep ="" )))
  rho=r/mu
  
  
  results_SMCm_site[[chr]]=SMCm(n=40,rho= rho,methylation=c(abs(log10(rate_m)),abs(log10(rate_d))),BoxM=c(1,1),BoxU=c(1,1),O,mu_r=(mu),maxit =20,BoxB=c(0.05,1),BoxP=c(3,3),Boxr=c(1,1),Boxs=c(0,0.99),pop=F,SB=F,SF=T,ER=F,NC=1,mu_b=1,sigma=0.9,beta=1,Region=F,region_methylation = c(abs(log10(rate_m_reg)),abs(log10(rate_d_reg))),pop_vect = c(4,4,rep(2,16)))
  
  results_SMCm_region[[chr]]=SMCm(n=40,rho= rho,methylation=c(abs(log10(rate_m)),abs(log10(rate_d))),BoxM=c(1,1),BoxU=c(1,1),O,mu_r=(mu),maxit =20,BoxB=c(0.05,1),BoxP=c(3,3),Boxr=c(1,1),nb_site_min=4,window_min=100,Boxs=c(0,0.99),pop=F,SB=F,SF=T,ER=F,NC=1,mu_b=1,sigma=0.9,beta=1,Region=2,region_methylation = c(abs(log10(rate_m_reg)),abs(log10(rate_d_reg))),pop_vect = c(4,4,rep(2,16)))
  
  results_SMCm_site_and_region[[chr]]=SMCm(n=40,rho= rho,methylation=c(abs(log10(rate_m)),abs(log10(rate_d))),BoxM=c(1,1),BoxU=c(1,1),O,mu_r=(mu),maxit =20,BoxB=c(0.05,1),BoxP=c(3,3),Boxr=c(1,1),nb_site_min=4,window_min=100,Boxs=c(0,0.99),pop=F,SB=F,SF=T,ER=F,NC=1,mu_b=1,sigma=0.9,beta=1,Region=T,region_methylation = c(abs(log10(rate_m_reg)),abs(log10(rate_d_reg))),pop_vect = c(4,4,rep(2,16)))
  
  
  rm_pos=c()
  M=dim(O)[1]-2
  for(m in 1:M){
    
    pos=as.numeric(which(O[m,]%in%c("M","D","C")))
    O[m,pos]="C"
  }
  
  
  results_eSMC2[[chr]]=eSMC2(n=40,rho=rho,O,maxit =20,BoxB=c(0.05,1),BoxP=c(3,3),Boxr=c(1,1),Boxs=c(0,0.99),pop=F,SB=F,SF=T,Rho=F,NC=1,mu_b=1,sigma=0.9,beta=1,LH_opt = F,pop_vect = c(4,4,rep(2,16)))
}

if(T){
  gen <- 1
  
  save(list = ls(), file = "Methylome_indica2_super_clean.RData") 
  name_sc=c("Saw-tooth "," Bottleneck"," Expansion"," Decrease")
  col_u=c("red","orange","green","blue","purple")
  mat_save_p=matrix(NA,nrow=(4*4),ncol=40)
  mat_save_t=matrix(NA,nrow=(4*4),ncol=40)
  mat_save_s=matrix(NA,nrow=(4*4),ncol=1)
  count_p=0
  count_t=0
  count_s=0
  #eSMC
  plot(c(100,2*10^6),c(1,1), log=c("x"), ylim =c(3,7) ,
       type="n", xlab= paste("Generations ago",sep=" "), ylab="population size (log10)",main = "")
  for(x in CHR){
    
    
    
    lines((results_SMCm_site[[x]]$Tc*results_SMCm_site[[x]]$Ne), log10(results_SMCm_site[[x]]$Xi*0.5*results_SMCm_site[[x]]$Ne), type="s", col=col_u[1])
    count_p=count_p+1
    count_t=count_t+1
    count_s=count_s+1
    mat_save_p[count_p,]=log10(results_SMCm_site[[x]]$Xi*0.5*results_SMCm_site[[x]]$Ne)
    mat_save_t[count_t,]=(results_SMCm_site[[x]]$Tc*results_SMCm_site[[x]]$Ne)
    mat_save_s[count_s,]=results_SMCm_site[[x]]$sigma
    
    
    lines((results_SMCm_region[[x]]$Tc*results_SMCm_region[[x]]$Ne), log10(results_SMCm_region[[x]]$Xi*0.5*results_SMCm_region[[x]]$Ne), type="s", col=col_u[2])
    count_p=count_p+1
    count_t=count_t+1
    count_s=count_s+1
    mat_save_p[count_p,]=log10(results_SMCm_region[[x]]$Xi*0.5*results_SMCm_region[[x]]$Ne)
    mat_save_t[count_t,]=(results_SMCm_region[[x]]$Tc*results_SMCm_region[[x]]$Ne)
    mat_save_s[count_s,]=results_SMCm_region[[x]]$sigma
    
    lines((results_SMCm_site_and_region[[x]]$Tc*results_SMCm_site_and_region[[x]]$Ne), log10(results_SMCm_site_and_region[[x]]$Xi*0.5*results_SMCm_site_and_region[[x]]$Ne), type="s", col=col_u[3])
    count_p=count_p+1
    count_t=count_t+1
    count_s=count_s+1
    mat_save_p[count_p,]=log10(results_SMCm_site_and_region[[x]]$Xi*0.5*results_SMCm_site_and_region[[x]]$Ne)
    mat_save_t[count_t,]=(results_SMCm_site_and_region[[x]]$Tc*results_SMCm_site_and_region[[x]]$Ne)
    mat_save_s[count_s,]=results_SMCm_site_and_region[[x]]$sigma
    
    
    Pop=results_eSMC2[[x]]$mu/mu
    lines((results_eSMC2[[x]]$Tc*Pop), log10(results_eSMC2[[x]]$Xi*0.5*Pop), type="s", col=col_u[4])
    count_p=count_p+1
    count_t=count_t+1
    count_s=count_s+1
    mat_save_p[count_p,]=log10(results_eSMC2[[x]]$Xi*0.5*Pop)
    mat_save_t[count_t,]=(results_eSMC2[[x]]$Tc*Pop)
    mat_save_s[count_s,]=results_eSMC2[[x]]$sigma
    
  }
  legend("topright",legend=c("SMCm:Site","SMCm:Region","SMCm:site&region","eSMC2"), col=col_u[1:4], lty=c(1,1),cex=0.75,x.intersp=0.5,y.intersp=0.8)
}
write.csv(mat_save_p,file = paste("Methylome_SMCm_Region_indica2_super_clean_n40_mat_save_p_real.csv",sep="_"))
write.csv(mat_save_t,file = paste("Methylome_SMCm_Region_indica2_super_clean_n40_mat_save_t_real.csv",sep="_"))
write.csv(mat_save_s,file = paste("Methylome_SMCm_Region_indica2_super_clean_n40_mat_save_s_real.csv",sep="_"))

```

25. SMCm for indica1
```
###################
# Source function #
###################
library("eSMC2",lib.loc="/data/home/users/a.ramesh/R/x86_64-redhat-linux-gnu-library/4.1/")
################################

########
#Script#
########

CHR=c(1:12)#
mu <- 6.95*10^-9
r=3.6*10^-8

rate_m=3.48*10^(-4)
rate_d=1.47*10^(-3)
rate_m_reg=1.6*10^(-4)
rate_d_reg=9.5*10^(-4)

results_eSMC2=list()
results_SMCm_site=list()
results_SMCm_region=list()
results_SMCm_site_and_region=list()

for(chr in CHR){
  M=10
  O=as.matrix(Get_real_data("/data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/",M=M,paste("indica1_multihetsep_snp_meth_",chr,sep ="" )))
  rho=r/mu
  
  
  results_SMCm_site[[chr]]=SMCm(n=40,rho= rho,methylation=c(abs(log10(rate_m)),abs(log10(rate_d))),BoxM=c(1,1),BoxU=c(1,1),O,mu_r=(mu),maxit =20,BoxB=c(0.05,1),BoxP=c(3,3),Boxr=c(1,1),Boxs=c(0,0.99),pop=F,SB=F,SF=T,ER=F,NC=1,mu_b=1,sigma=0.9,beta=1,Region=F,region_methylation = c(abs(log10(rate_m_reg)),abs(log10(rate_d_reg))),pop_vect = c(4,4,rep(2,16)))
  
  results_SMCm_region[[chr]]=SMCm(n=40,rho= rho,methylation=c(abs(log10(rate_m)),abs(log10(rate_d))),BoxM=c(1,1),BoxU=c(1,1),O,mu_r=(mu),maxit =20,BoxB=c(0.05,1),BoxP=c(3,3),Boxr=c(1,1),nb_site_min=4,window_min=100,Boxs=c(0,0.99),pop=F,SB=F,SF=T,ER=F,NC=1,mu_b=1,sigma=0.9,beta=1,Region=2,region_methylation = c(abs(log10(rate_m_reg)),abs(log10(rate_d_reg))),pop_vect = c(4,4,rep(2,16)))
  
  results_SMCm_site_and_region[[chr]]=SMCm(n=40,rho= rho,methylation=c(abs(log10(rate_m)),abs(log10(rate_d))),BoxM=c(1,1),BoxU=c(1,1),O,mu_r=(mu),maxit =20,BoxB=c(0.05,1),BoxP=c(3,3),Boxr=c(1,1),nb_site_min=4,window_min=100,Boxs=c(0,0.99),pop=F,SB=F,SF=T,ER=F,NC=1,mu_b=1,sigma=0.9,beta=1,Region=T,region_methylation = c(abs(log10(rate_m_reg)),abs(log10(rate_d_reg))),pop_vect = c(4,4,rep(2,16)))
  
  
  rm_pos=c()
  M=dim(O)[1]-2
  for(m in 1:M){
    
    pos=as.numeric(which(O[m,]%in%c("M","D","C")))
    O[m,pos]="C"
  }
  
  
  results_eSMC2[[chr]]=eSMC2(n=40,rho=rho,O,maxit =20,BoxB=c(0.05,1),BoxP=c(3,3),Boxr=c(1,1),Boxs=c(0,0.99),pop=F,SB=F,SF=T,Rho=F,NC=1,mu_b=1,sigma=0.9,beta=1,LH_opt = F,pop_vect = c(4,4,rep(2,16)))
}

if(T){
  gen <- 1
  
  save(list = ls(), file = "Methylome_indica1_super_clean.RData") 
  name_sc=c("Saw-tooth "," Bottleneck"," Expansion"," Decrease")
  col_u=c("red","orange","green","blue","purple")
mat_save_p=matrix(NA,nrow=(4*5),ncol=40)
  mat_save_t=matrix(NA,nrow=(4*5),ncol=40)
  mat_save_s=matrix(NA,nrow=(4*5),ncol=1)
  count_p=0
  count_t=0
  count_s=0
  #eSMC
  plot(c(100,2*10^6),c(1,1), log=c("x"), ylim =c(3,7) ,
       type="n", xlab= paste("Generations ago",sep=" "), ylab="population size (log10)",main = "")
  for(x in CHR){
    
    
    
    lines((results_SMCm_site[[x]]$Tc*results_SMCm_site[[x]]$Ne), log10(results_SMCm_site[[x]]$Xi*0.5*results_SMCm_site[[x]]$Ne), type="s", col=col_u[1])
    count_p=count_p+1
    count_t=count_t+1
    count_s=count_s+1
    mat_save_p[count_p,]=log10(results_SMCm_site[[x]]$Xi*0.5*results_SMCm_site[[x]]$Ne)
    mat_save_t[count_t,]=(results_SMCm_site[[x]]$Tc*results_SMCm_site[[x]]$Ne)
    mat_save_s[count_s,]=results_SMCm_site[[x]]$sigma
    
    
    lines((results_SMCm_region[[x]]$Tc*results_SMCm_region[[x]]$Ne), log10(results_SMCm_region[[x]]$Xi*0.5*results_SMCm_region[[x]]$Ne), type="s", col=col_u[2])
    count_p=count_p+1
    count_t=count_t+1
    count_s=count_s+1
    mat_save_p[count_p,]=log10(results_SMCm_region[[x]]$Xi*0.5*results_SMCm_region[[x]]$Ne)
    mat_save_t[count_t,]=(results_SMCm_region[[x]]$Tc*results_SMCm_region[[x]]$Ne)
    mat_save_s[count_s,]=results_SMCm_region[[x]]$sigma
    
    lines((results_SMCm_site_and_region[[x]]$Tc*results_SMCm_site_and_region[[x]]$Ne), log10(results_SMCm_site_and_region[[x]]$Xi*0.5*results_SMCm_site_and_region[[x]]$Ne), type="s", col=col_u[3])
    count_p=count_p+1
    count_t=count_t+1
    count_s=count_s+1
    mat_save_p[count_p,]=log10(results_SMCm_site_and_region[[x]]$Xi*0.5*results_SMCm_site_and_region[[x]]$Ne)
    mat_save_t[count_t,]=(results_SMCm_site_and_region[[x]]$Tc*results_SMCm_site_and_region[[x]]$Ne)
    mat_save_s[count_s,]=results_SMCm_site_and_region[[x]]$sigma
    
    
    Pop=results_eSMC2[[x]]$mu/mu
    lines((results_eSMC2[[x]]$Tc*Pop), log10(results_eSMC2[[x]]$Xi*0.5*Pop), type="s", col=col_u[4])
    count_p=count_p+1
    count_t=count_t+1
    count_s=count_s+1
    mat_save_p[count_p,]=log10(results_eSMC2[[x]]$Xi*0.5*Pop)
    mat_save_t[count_t,]=(results_eSMC2[[x]]$Tc*Pop)
    mat_save_s[count_s,]=results_eSMC2[[x]]$sigma
    
  }
  legend("topright",legend=c("SMCm:Site","SMCm:Region","SMCm:site&region","eSMC2"), col=col_u[1:4], lty=c(1,1),cex=0.75,x.intersp=0.5,y.intersp=0.8)
}
write.csv(mat_save_p,file = paste("Methylome_SMCm_Region_indica1_super_clean_n40_mat_save_p_real.csv",sep="_"))
write.csv(mat_save_t,file = paste("Methylome_SMCm_Region_indica1_super_clean_n40_mat_save_t_real.csv",sep="_"))
write.csv(mat_save_s,file = paste("Methylome_SMCm_Region_indica1_super_clean_n40_mat_save_s_real.csv",sep="_"))

```

26. Get summary statistics for methylation and genomic variants. Done on biallelic SNPs. Further NA filtering in R. Only keep gene body variants.
```
cd  /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data
vcftools --gzvcf rice_snps_filtered.recode.vcf.gz --out rice_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --freq --bed  gene_pos.bed
vcftools --gzvcf rice_snps_filtered.recode.vcf.gz --out rice_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --site-pi --bed  gene_pos.bed
vcftools --gzvcf rice_snps_filtered.recode.vcf.gz --out rice_snp --min-alleles 2 --max-alleles 2 --max-missing 0.5 --het --bed  gene_pos.bed

vcftools --vcf rice_meth_all.vcf  --out rice_meth --min-alleles 2 --max-alleles 2 --max-missing 0.5 --freq --bed  gene_pos.bed
vcftools --vcf rice_meth_all.vcf  --out rice_meth --min-alleles 2 --max-alleles 2 --max-missing 0.5 --site-pi --bed  gene_pos.bed

vcftools --vcf rice_meth_var_invar_all.vcf  --out rice_meth_var_invar --max-missing 0.5  --recode --bed  gene_pos.bed

```

27. R plots for other metrics. Per site or across genome, not per gene.
```
#SFS 
rice_snp.frq <- read.table(file="rice_snp.frq",row.names = NULL)
rice_snp.frq$raf <- as.numeric(gsub(".*:","",rice_snp.frq$N_CHR))
rice_snp.frq$aaf <- as.numeric(gsub(".*:","",rice_snp.frq$X.ALLELE.FREQ.))
rice_snp.frq$maf <- apply(rice_snp.frq[7:8],1,min)
rice_snp.frq <- rice_snp.frq[rice_snp.frq$maf > 0,]
rice_snp.frq <- rice_snp.frq[rice_snp.frq$maf < 1,]
rice_snp.frq <- rice_snp.frq[rice_snp.frq$N_ALLELES > 39,]
rice_snp.frq$type <- 'SNP'

rice_meth.frq <- read.table(file="rice_meth.frq",row.names = NULL)
rice_meth.frq$raf <- as.numeric(gsub(".*:","",rice_meth.frq$N_CHR))
rice_meth.frq$aaf <- as.numeric(gsub(".*:","",rice_meth.frq$X.ALLELE.FREQ.))
rice_meth.frq$maf <- apply(rice_meth.frq[7:8],1,min)
rice_meth.frq <- rice_meth.frq[rice_meth.frq$maf > 0,]
rice_meth.frq <- rice_meth.frq[rice_meth.frq$maf < 1,]
rice_meth.frq <- rice_meth.frq[rice_meth.frq$N_ALLELES > 39,]
rice_meth.frq$type <- 'SMP'

snp_smp_rice <- rbind(rice_snp.frq,rice_meth.frq)

rice_sfs <- ggplot(snp_smp_rice,aes(fill=type,x=maf)) +
  geom_histogram(aes(y=0.05*..density..),binwidth=0.05, alpha=0.4, position='identity') +
  ylab("Proportion of sites")

nrow(rice_meth.frq)/nrow(rice_snp.frq)

## PI
rice_snp.sites.pi <- read.table(file="rice_snp.sites.pi",row.names = NULL, header = T)
rice_snp.sites.pi$type <- 'SNP'

rice_meth.sites.pi <- read.table(file="rice_meth.sites.pi",row.names = NULL, header = T)
rice_meth.sites.pi$type <- 'SMP'

snp_smp_rice <- rbind(rice_snp.sites.pi,rice_meth.sites.pi)

df_rice_pi <- dplyr::count(snp_smp_rice, type)
df_rice_pi$n <- paste("n=",df_rice_pi$n,sep="")

rice_pi <- ggplot(snp_smp_rice,aes(x=type,y=PI)) +
  geom_boxplot() +
  ylab("Per site π") +
  xlab("")+
  ggtitle("O. sativa")+
  geom_text(data = df_rice_pi, aes(y = 0.6, label = n))

nrow(rice_meth.sites.pi)/nrow(rice_snp.sites.pi)

rice_prop_seg_smp <- 5853/86378
rice_prop_seg_snp <- 171399/845585

## LD

rice_snp.geno.ld <- read.table(file="rice_snp.geno.ld",row.names = NULL, header = T)
rice_snp.geno.ld$type <- 'SNP'

rice_meth.geno.ld <- read.table(file="rice_meth.geno.ld",row.names = NULL, header = T)
rice_meth.geno.ld$type <- 'SMP'

snp_smp_rice <- rbind(rice_snp.geno.ld,rice_meth.geno.ld)

rice_ld <- ggplot(snp_smp_rice,aes(x=type,y=R.2)) +
  geom_violin() +
  ylab("Pairwise LD (r2)") +
  xlab("") +
  ggtitle("O. sativa")

rice_ld <- ggplot(snp_smp_rice,aes(fill=type,x=R.2)) +
  geom_histogram(aes(y=0.1*..density..),binwidth=0.1, alpha=0.3, position='identity') +
  ylab("Proportion of sites") +
  xlab("Pairwise LD (r2)") +
  ggtitle("O. sativa") +
  scale_fill_manual(values=c("red","green"))

nrow(rice_meth.geno.ld)/nrow(rice_snp.geno.ld)

## Tajima's D

rice_snp.Tajima.D <- read.table(file="rice_snp.Tajima.D",row.names = NULL, header = T)
rice_snp.Tajima.D$type <- 'SNP'

rice_meth.Tajima.D <- read.table(file="rice_meth.Tajima.D",row.names = NULL, header = T)
rice_meth.Tajima.D$type <- 'SMP'

snp_smp_rice <- rbind(rice_snp.Tajima.D,rice_meth.Tajima.D)

rice_env_tajd <- ggplot(snp_smp_rice,aes(x=type,y=TajimaD)) +
  geom_boxplot()  +
  ylab("Tajima's D") +
  xlab("") +
  ggtitle("O. sativa")

nrow(rice_meth.Tajima.D)/nrow(rice_snp.Tajima.D)

```

28. Split SNP vcf by gene
```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna
 
/proj/popgen/a.ramesh/software/htslib-1.16/bgzip -f rice.allsites.recode.vcf
/proj/popgen/a.ramesh/software/htslib-1.16/tabix -f rice.allsites.recode.vcf.gz

cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/genes_fasta
cat /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/gene_pos.list | while read -r line ; do /proj/popgen/a.ramesh/software/htslib-1.16/tabix ../rice.allsites.recode.vcf.gz  $line >$line.var_invar.vcf; done
wc -l *vcf >vcflengths_var_invar
zcat ../rice.allsites.recode.vcf.gz | grep '##' >vcfheader
for file in *.var_invar.vcf ; do cat vcfheader $file >${file/var_invar.vcf/all.vcf}; done


/proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --gzvcf rice.allsites.recode.vcf.gz  --out rice_var_invar_indica2 --max-missing 0.5  --recode --bed  /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/gene_pos.bed --keep  indica2
/proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --gzvcf rice.allsites.recode.vcf.gz  --out rice_var_invar_indica1 --max-missing 0.5  --recode --bed  /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/gene_pos.bed --keep  indica1

/proj/popgen/a.ramesh/software/htslib-1.16/bgzip -f rice_var_invar_indica1.recode.vcf
/proj/popgen/a.ramesh/software/htslib-1.16/tabix -f rice_var_invar_indica1.recode.vcf.gz

/proj/popgen/a.ramesh/software/htslib-1.16/bgzip -f rice_var_invar_indica2.recode.vcf
/proj/popgen/a.ramesh/software/htslib-1.16/tabix -f rice_var_invar_indica2.recode.vcf.gz

mkdir genes_indica1
cd genes_indica1
cat /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/gene_pos.list | while read -r line ; do /proj/popgen/a.ramesh/software/htslib-1.16/tabix ../rice_var_invar_indica1.recode.vcf.gz  $line >$line.var_invar.vcf; done
wc -l *vcf >vcflengths_var_invar

mkdir ../genes_indica2
cd ../genes_indica2
cat /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/gene_pos.list | while read -r line ; do /proj/popgen/a.ramesh/software/htslib-1.16/tabix ../rice_var_invar_indica2.recode.vcf.gz  $line >$line.var_invar.vcf; done
wc -l *vcf >vcflengths_var_invar
```

29. Get per gene theta and tajima's D for SNPs. some parts differ for each group
```
vcflengths_var_invar <- read.table(file="vcflengths_var_invar")
vcflengths_var_invar <- vcflengths_var_invar[-c(nrow(vcflengths_var_invar)),]
vcflengths_var_invar$V2 <- gsub(".var_invar.vcf","",vcflengths_var_invar$V2)
colnames(vcflengths_var_invar) <- c("numvar","interval")
vcflengths_var_invar$length <- as.numeric(gsub(".*-","", gsub(".*:","",vcflengths_var_invar$interval))) - as.numeric(gsub("-.*","", gsub(".*:","",vcflengths_var_invar$interval)))
vcflengths_var_invar$prop <- vcflengths_var_invar$numvar/vcflengths_var_invar$length
vcflengths_var_invar <- vcflengths_var_invar[vcflengths_var_invar$numvar < 11,]
vcflengths_var_invar <- vcflengths_var_invar[vcflengths_var_invar$prop < 0.05,]

write.table(paste("rm ",vcflengths_var_invar$interval,".var_invar.vcf",sep=""),file="bad_intervals.sh",sep="\t",quote=F,row.names = F, col.names = F)

vcflengths_var_invar <- read.table(file="vcflengths_var_invar")
vcflengths_var_invar <- vcflengths_var_invar[-c(nrow(vcflengths_var_invar)),]
vcflengths_var_invar$V2 <- gsub(".var_invar.vcf","",vcflengths_var_invar$V2)
colnames(vcflengths_var_invar) <- c("numvar","interval")
vcflengths_var_invar$length <- as.numeric(gsub(".*-","", gsub(".*:","",vcflengths_var_invar$interval))) - as.numeric(gsub("-.*","", gsub(".*:","",vcflengths_var_invar$interval)))
vcflengths_var_invar$prop <- vcflengths_var_invar$numvar/vcflengths_var_invar$length
vcflengths_var_invar <- vcflengths_var_invar[vcflengths_var_invar$numvar > 10,]
vcflengths_var_invar <- vcflengths_var_invar[vcflengths_var_invar$prop > 0.05,]

write.table(cbind(paste(vcflengths_var_invar$interval,".all.vcf",sep=""),vcflengths_var_invar$length),file="goodfiles",sep="\t",quote=F,row.names = F, col.names = F)

## run these in linux before continuing. this part differs for each group (e.g. indica 1, indica 2)
#./bad_intervals.sh
# zcat ../rice_var_invar_indica1.recode.vcf.gz | grep '#' >vcfheader
# for file in *.var_invar.vcf ; do cat vcfheader $file > ${file/.var_invar.vcf/.all.vcf} ; done

goodfiles <- read.table(file="goodfiles")
for(f in 1:nrow(goodfiles)){
  i=0
  while(i<200){
    print(i)
    i <- i + 1
    system(paste("/proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --vcf ",goodfiles$V1[f]," --out tmp --TajimaD ",goodfiles$V2[f]+i,sep=""))
    tmpfile <- read.table(file="tmp.Tajima.D",header = T)
    print(nrow(tmpfile))
    if (nrow(tmpfile) == 1) {
      system(paste("/proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --vcf ",goodfiles$V1[f]," --out ",goodfiles$V1[f]," --TajimaD ",goodfiles$V2[f]+i,sep=""))
      break
    }
  }
}

for(f in 1:nrow(goodfiles)){
  i=0
  while(i<200){
    print(i)
    i <- i + 1
    system(paste("/proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --vcf ",goodfiles$V1[f]," --out tmp --window-pi ",goodfiles$V2[f]+i,sep=""))
    tmpfile <- read.table(file="tmp.windowed.pi",header = T)
    print(nrow(tmpfile))
    if (nrow(tmpfile) == 1) {
      system(paste("/proj/popgen/a.ramesh/software/vcftools-vcftools-581c231/bin/vcftools --vcf ",goodfiles$V1[f]," --out ",goodfiles$V1[f]," --window-pi ",goodfiles$V2[f]+i,sep=""))
      break
    }
  }
}
```
30. Run above R script for each group
```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/genes_fasta
Rscript vcfstats.R

cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/genes_indica1
Rscript vcfstats.R

cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/genes_indica2
Rscript vcfstats.R
```

31. Split reference fasta file by gene
```
cd /proj/popgen/a.ramesh/projects/methylomes/rice/data_rna/
#cat /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/gene_pos.list | while read -r line ; do samtools faidx /proj/popgen/a.ramesh/projects/methylomes/rice/genomes/Oryza_sativa.IRGSP-1.0.dna.toplevel.fa $line >>genes.fasta; done
/proj/popgen/a.ramesh/software/faSplit byname genes.fasta genes_fasta/
cd genes_fasta/
ls *fa >filenames
```

32. Rscript to count number of cytosines
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
33. Split methylation vcf by gene intervals
```
#cd /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/genes_fasta

#cat ../gene_pos.list | while read -r line ; do tabix ../rice_meth_var_invar.recode.vcf.gz  $line >$line.var_invar.vcf; done
#wc -l *vcf >vcflengths_var_invar

cd  /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data

#vcftools --vcf rice_meth_var_invar_all.vcf  --out rice_meth_var_invar_indica1 --max-missing 0.5  --recode --bed  gene_pos.bed --keep  indica1
#vcftools --vcf rice_meth_var_invar_all.vcf  --out rice_meth_var_invar_indica2 --max-missing 0.5  --recode --bed  gene_pos.bed --keep  indica2

#bgzip rice_meth_var_invar_indica1.recode.vcf
#tabix rice_meth_var_invar_indica1.recode.vcf.gz

#bgzip rice_meth_var_invar_indica2.recode.vcf
#tabix rice_meth_var_invar_indica2.recode.vcf.gz

#mkdir genes_indica1
cd genes_indica1
#cat ../gene_pos.list | while read -r line ; do tabix ../rice_meth_var_invar_indica1.recode.vcf.gz  $line >$line.var_invar.vcf; done
#wc -l *vcf >vcflengths_var_invar


mkdir ../genes_indica2
cd ../genes_indica2
cat ../gene_pos.list | while read -r line ; do tabix ../rice_meth_var_invar_indica2.recode.vcf.gz  $line >$line.var_invar.vcf; done
wc -l *vcf >vcflengths_var_invar
```

34. Get good intervals for Dm alpha. file saved as good_intervals.R in each genes folder.
```
vcflengths_var_invar <- read.table(file="vcflengths_var_invar")
vcflengths_var_invar <- vcflengths_var_invar[-c(nrow(vcflengths_var_invar)),]
vcflengths_var_invar$V2 <- gsub(".var_invar.vcf","",vcflengths_var_invar$V2)
colnames(vcflengths_var_invar) <- c("numvar","interval")
cytsosine_count <- read.table(file="../cytsosine_count.txt")
cytsosine_count$V1 <- gsub(".fa","",cytsosine_count$V1)
colnames(cytsosine_count) <- c("interval","numc")

library(dplyr)

merged <- inner_join(cytsosine_count,vcflengths_var_invar,by="interval")
merged$prop <- merged$numvar/merged$numc
merged <- merged[merged$numvar > 10,]
merged <- merged[merged$prop > 0.05,]

write.table(merged,file="good_intervals",sep="\t",quote=F,row.names = F, col.names = F)
```

35. Theta and Tajima's D for methylation, first get alpha
```
cd  /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/genes_fasta
## Dm header looks like this: #chr    position        C019    C051    C135    C139    C148    C151    MH63    NIP     W081    W105    W125    W128    W161    W169    W257    W261    W286    W294    W306    ZS97
## Dm header for indica1: #chr    position        C019    C135    C139    C151    ZS97
## Dm header for indica2: #chr    position        C148    MH63    W161    W169

cd  /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/genes_fasta
#for file in *.var_invar.vcf; do sed '/##/d' $file | cut -f 1,2,10- | sed 's/\/.//g' | cat Dm_header -  >${file/.var_invar.vcf/.input.txt} ; done 
#cut -f 1-2 good_intervals | sed 's/\t/.input.txt\t/' >length_list
#perl /data/proj2/popgen/a.ramesh/software/alpha_estimation.pl -dir input -output  alpha_Dm_rice -length_list length_list

cd /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/genes_indica1
for file in *.var_invar.vcf; do sed '/##/d' $file | cut -f 1,2,10- | sed 's/\/.//g' | cat Dm_header -  >${file/.var_invar.vcf/.input.txt} ; done 
cut -f 1-2 good_intervals | sed 's/\t/.input.txt\t/' >length_list
perl /data/proj2/popgen/a.ramesh/software/alpha_estimation.pl -dir input -output  alpha_Dm_rice -length_list length_list

cd /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/genes_indica2
for file in *.var_invar.vcf; do sed '/##/d' $file | cut -f 1,2,10- | sed 's/\/.//g' | cat Dm_header -  >${file/.var_invar.vcf/.input.txt} ; done 
cut -f 1-2 good_intervals | sed 's/\t/.input.txt\t/' >length_list
perl /data/proj2/popgen/a.ramesh/software/alpha_estimation.pl -dir input -output  alpha_Dm_rice -length_list length_list

```

36. Get good intervals for Dm. run this for each group. Script continues from before in good_intervals.R
```
good_intervals <- read.table(file="good_intervals")
alpha_dm <- read.table(file="alpha_Dm_rice")
alpha_dm$V1 <- gsub(".input.txt","",alpha_dm$V1)
alpha_dm <- alpha_dm[1:2]
good_intervals <- inner_join(good_intervals,alpha_dm,by="V1")

write.table(merged,file="good_intervals_alpha",sep="\t",quote=F,row.names = F, col.names = F

```
37. Now get Dm estimates
```
cd  /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/genes_fasta
cat good_intervals_alpha |  while read -r value1 value2 value3 value4 value5 remainder ;  do perl /data/proj2/popgen/a.ramesh/software/Dm_test_new.pl -input $value1.input.txt -output $value1.Dm_rice.txt -length $value2 -alpha $value5  ; done
#cp good_intervals_alpha input/
cd input/
#cat good_intervals_alpha |  while read -r value1 value2 value3 value4 value5 remainder ;  do perl /data/proj2/popgen/a.ramesh/software/Dm_test_new.pl -input $value1.input.txt -output $value1.Dm_rice.txt -length $value2 -alpha $value5  ; done

cd /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/genes_indica1
#cp good_intervals_alpha input/
cd input/
cat good_intervals_alpha |  while read -r value1 value2 value3 value4 value5 remainder ;  do perl /data/proj2/popgen/a.ramesh/software/Dm_test_new.pl -input $value1.input.txt -output $value1.Dm_indica1.txt -length $value2 -alpha $value5  ; done

cd /data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/genes_indica2
cp good_intervals_alpha input/
cd input/
cat good_intervals_alpha |  while read -r value1 value2 value3 value4 value5 remainder ;  do perl /data/proj2/popgen/a.ramesh/software/Dm_test_new.pl -input $value1.input.txt -output $value1.Dm_indica2.txt -length $value2 -alpha $value5  ; done

```
38. compare per gene theta and tajima's D estimates
```
library(dplyr)
library(ggplot2)
library(Cairo)
rice.Dm_rice.txt <- read.table(file="rice.Dm_rice.txt",header= T)
rice.Dm_rice.txt <- rice.Dm_rice.txt[!rice.Dm_rice.txt$chr %in% "chr",]
rice.Dm_rice.txt[2:ncol(rice.Dm_rice.txt)] <- apply(rice.Dm_rice.txt[2:ncol(rice.Dm_rice.txt)],2,as.numeric)
interval <- read.table(file="Dm_filenames")
interval <- gsub(".Dm_rice.txt","",interval$V1)
interval <- interval[-c(length(interval))]
rice.Dm_rice.txt$interval <- interval
rice.Dm_rice_tojoin <- rice.Dm_rice.txt[c(6,7,4)]
colnames(rice.Dm_rice_tojoin) <- c("theta_pi","theta_s","tajimasD")
rice.Dm_rice_tojoin$type <- " SMP"

rice.windowed.pi <- read.table(file="rice.windowed.pi",header= T)
rice.windowed.pi <- rice.windowed.pi[!rice.windowed.pi$BIN_START %in% "BIN_START",]
rice.windowed.pi[2:ncol(rice.windowed.pi)] <- apply(rice.windowed.pi[2:ncol(rice.windowed.pi)],2,as.numeric)
rice.windowed.pi$interval <- gsub(".all.vcf.*","",rice.windowed.pi$X10.12983013.12983507.all.vcf.windowed.pi.CHROM)
rice.windowed.pi$CHROM <- gsub(".*:","",rice.windowed.pi$X10.12983013.12983507.all.vcf.windowed.pi.CHROM)
rice.windowed.pi <- rice.windowed.pi[-c(1)]

rice.Tajima.D <- read.table(file="rice.Tajima.D",header= T)
rice.Tajima.D <- rice.Tajima.D[!rice.Tajima.D$N_SNPS %in% "N_SNPS",]
rice.Tajima.D[2:ncol(rice.Tajima.D)] <- apply(rice.Tajima.D[2:ncol(rice.Tajima.D)],2,as.numeric)
rice.Tajima.D$interval <- gsub(".all.vcf.*","",rice.Tajima.D$X10.12983013.12983507.all.vcf.Tajima.D.CHROM)
rice.Tajima.D$CHROM <- gsub(".*:","",rice.Tajima.D$X10.12983013.12983507.all.vcf.Tajima.D.CHROM)
rice.Tajima.D <- rice.Tajima.D[-c(1)]

snp_metrics <- inner_join(rice.windowed.pi,rice.Tajima.D,by="interval")
snp_metrics <- snp_metrics[c(3,4,5,8,9)]
snp_metrics$chrom <- gsub(":.*","",snp_metrics$interval)
snp_metrics$length <- as.numeric(gsub(".*-","", gsub(".*:","",snp_metrics$interval))) - as.numeric(gsub("-.*","", gsub(".*:","",snp_metrics$interval)))

harmonicNumber = 0
numChromosomes = 40
for (i in 1:(numChromosomes - 1)) {
  harmonicNumber = harmonicNumber + 1.0/i
}
print(harmonicNumber)
snp_metrics$watterson <- (snp_metrics$N_VARIANTS/harmonicNumber)/snp_metrics$length
plot(snp_metrics$PI,snp_metrics$watterson)
abline(0,1)
plot((snp_metrics$PI/snp_metrics$watterson),snp_metrics$TajimaD)
abline(0,1)

snp_metrics_tojoin <- snp_metrics[c(2,8,5)]
colnames(snp_metrics_tojoin) <- c("theta_pi","theta_s","tajimasD")
snp_metrics_tojoin$type <- " SNP"

rice.Dm_rice_tojoin$type <- "mC"
concatenated <- rbind(snp_metrics_tojoin,rice.Dm_rice_tojoin)
cairo_pdf("theta_pi_rice.pdf",height=2.5,width=2)
ggplot(concatenated,aes(x=type,y=theta_pi)) +
  geom_boxplot() +
  ylab("Nucleotide diversity (π)") +
  xlab("")
dev.off()
ggplot(concatenated,aes(x=type,y=theta_s)) +
  geom_boxplot()
ggplot(concatenated,aes(x=type,y=tajimasD)) +
  geom_boxplot()

#snp_smp_metrics <- inner_join(rice.Dm_rice.txt,snp_metrics,by="interval")
```

39. Process reference genome file and bismark methylation call files for DMR calling
```
cut -f 1-2 Oryza_sativa.IRGSP-1.0.dna.toplevel.fa.fai >rice_chr

sed 's/C019//' C019.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >C019_context.txt
sed 's/C051//' C051.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >C051_context.txt
sed 's/C135//' C135.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >C135_context.txt
sed 's/C139//' C139.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >C139_context.txt
sed 's/C148//' C148.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >C148_context.txt
sed 's/C151//' C151.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >C151_context.txt
sed 's/MH63//' MH63.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >MH63_context.txt
sed 's/NIP//' NIP.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >NIP_context.txt
sed 's/W081//' W081.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W081_context.txt
sed 's/W105//' W105.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W105_context.txt
sed 's/W125//' W125.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W125_context.txt
sed 's/W128//' W128.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W128_context.txt
sed 's/W161//' W161.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W161_context.txt
sed 's/W169//' W169.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W169_context.txt
sed 's/W257//' W257.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W257_context.txt
sed 's/W261//' W261.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W261_context.txt
sed 's/W286//' W286.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W286_context.txt
sed 's/W294//' W294.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W294_context.txt
sed 's/W306//' W306.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >W306_context.txt
sed 's/ZS97//' ZS97.1.paired_bismark_hisat2_pe.deduplicated.CpG_report.txt >ZS97_context.txt

for file in *context.txt; do sed -i -e '/Syn/d' -e '/AP/d' -e '/AC/d' -e '/Mt/d' -e '/Pt/d' $file; done

sed 's/ .*//' Oryza_sativa.IRGSP-1.0.dna.toplevel.fa >newref.fa
samtools faidx newref.fa
samtools faidx newref.fa 1 >newref_1.fa
samtools faidx newref.fa 2 >newref_2.fa
samtools faidx newref.fa 3 >newref_3.fa
samtools faidx newref.fa 4 >newref_4.fa
samtools faidx newref.fa 5 >newref_5.fa
samtools faidx newref.fa 6 >newref_6.fa
samtools faidx newref.fa 7 >newref_7.fa
samtools faidx newref.fa 8 >newref_8.fa
samtools faidx newref.fa 9 >newref_9.fa
samtools faidx newref.fa 10 >newref_10.fa
samtools faidx newref.fa 11 >newref_11.fa
samtools faidx newref.fa 12 >newref_12.fa
cat newref_*.fa > newref_all.fa
```

40. Do DMR calling
```
library("methimpute",lib.loc="/data/home/users/a.ramesh/R/x86_64-redhat-linux-gnu-library/4.1/")
library("jDMR",lib.loc="/data/home/users/a.ramesh/R/x86_64-redhat-linux-gnu-library/4.1/")
setwd("/data/proj2/popgen/a.ramesh/projects/methylomes/rice/data")

rice_chr <- read.table(file="rice_chr")
colnames(rice_chr) <- c("chromosome","length")
cytosine.positions <- extractCytosinesFromFASTA("Oryza_sativa.IRGSP-1.0.dna.toplevel.fa",contexts = c('CG','CHG','CHH'))
cytosine.positions@seqnames <- gsub(" .*","",cytosine.positions@seqnames)
cytosine.positions@seqinfo@seqnames <- gsub(" .*","",cytosine.positions@seqinfo@seqnames)
contextfiles <- read.table(file="contextfiles3")

#contextfiles3 looks like this, tab seperated
#W294_context.txt
#W306_context.txt
#ZS97_context.txt

for (i in 1:nrow(contextfiles)){
  bismark.data <- importBismark(file=as.character(contextfiles[i,]), chrom.lengths=rice_chr)
  methylome <- inflateMethylome(bismark.data, cytosine.positions)
  distcor <- distanceCorrelation(methylome, separate.contexts = TRUE)
  fit <- estimateTransDist(distcor)
  model <- callMethylationSeparate(data = methylome, transDist = fit$transDist,verbosity = 0)
  exportMethylome(model, filename = paste(c(gsub("_context.txt","", contextfiles[i,])), "_methimpute",sep=""))
}

out.dir <- "/data/proj2/popgen/a.ramesh/projects/methylomes/rice/data/jdmr"

#filenames looks like this, tab seperated
#file    sample
#C019_methimpute C019
#C051_methimpute C051

runjDMRgrid(out.dir=out.dir,fasta.file="newref_all.fa",samplefiles="filenames",genome="rice",contexts = c("CG"),min.C = 10)
```
