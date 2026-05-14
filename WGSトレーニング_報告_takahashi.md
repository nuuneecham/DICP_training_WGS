# WGSトレーニング

## 実施環境

手元のPCで解析を実行する。


|OS|MacOS Tahoe 26.3.1 (a)|
|:---:|:---|
|メモリ|16GB|
|CPU|apple M4|

## 環境準備

MacなのでHomebrewを使ってインストールした。

Homebrewでできないものは、各ツールの指示に従ってpipなどでインストールした。

gatk が Best Practices を実行するために必要なツールと指示している下記のツールをインストールした。

1. BWA
2. SAMtools
3. Picard
4. Genome Analysis Toolkit (GATK)
5. IGV
6. RStudio IDE and R libraries ggplot2 and gsalib

ただし、情報が古いためバージョンなどは適宜動作するものに変更した。

### ツールバージョン

|ツール|バージョン|備考|
|:---:|:---|:---|
|java|17.0.18 2026-01-20||
|GATK|4.6.2.0|解凍のみ|
|BWA|0.7.19-r1273||
|samtools|1.23.1||
|bcftools|1.23.1||
|Picard|2.27.1||
|R|4.5.3||
|RStudio|2026.01.2+418 (2026.01.2+418)||
|ggplot2|4.0.2|Rのパッケージ|
|gsalib|2.2.1|Rのパッケージ|
|bbtools|39.81||
|MultiQC|1.33||
|Python|3.14.3|インストール済み|
|IGV|2.19.5|インストール済み|

### ゲノムリファレンス、Known site

Broad Institute

https://console.cloud.google.com/storage/browser/gcp-public-data--broad-references

**リファレンス**

- Homo_sapiens_assembly38.fasta
- Homo_sapiens_assembly38.fasta.fai
- Homo_sapiens_assembly38.fasta.disc

**Known site**

- Homo_sapiens_assembly38.dbsnp138.vcf

## fastq準備

Sample : NA18939

PCR-free high coverage のfastqをダウンロードした。

fsatqのサマリ

```seqkit stas
file    format  type    num_seqs        sum_len min_len avg_len max_len
SRR1295537_1.fastq.gz    FASTQ   DNA     201525186       50381296500     250     250.0   250
SRR1295537_2.fastq.gz    FASTQ   DNA     201525186       50381296500     250     250.0   250
```

PCR-free high coverage のデータを約1/3にサンプリングした。

```reformat.sh
reformat.sh in=SRR1295537_1.fastq.gz in2=SRR1295537_2.fastq.gz \
    out=SRR1295537_1_33p.fastq.gz out2=SRR1295537_2_33p.fastq.gz \
    samplerate=0.333 \
 > log_sampling_p33 2> log_sampling_err_p33
```

ログの一部

```
~ 略 ~
Input is being processed as paired
Input:                          403050372 reads                 100762593000 bases
Processed:                      134228986 reads                 33557246500 bases
Output:                         134228986 reads (33.30%)        33557246500 bases (33.30%)
~ 略 ~
```

## QC

```fasted
fastqc -t 4 SRR1295537_1_33p.fastq.gz -o ./fastQC
fastqc -t 4 SRR1295537_2_33p.fastq.gz -o ./fastQC
```

特に後ろのポジションのクオリティが低く、PolyA、PolyGが若干検出されていた。

## トリミング

```fastp
fastp \
 -I SRR1295537_1_33p.fastq.gz \
 -I SRR1295537_2_33p.fastq.gz \
 -o SRR1295537_1_33p_trim.fastq \
 -O SRR1295537_2_33p_trim.fastq \
 -3 -p -w 4 --detect_adapter_for_pe --trim_poly_g \
 -h fastp_report.html
 -j fastp_report.json
```

Q30が66%程度だったものが、85%程度まで上がった。

FastQCでPolyA、PolyGも確認し、概ね取れていた。

## マッピング

1回目は NCBIから取得したリファレンスを使用したが、HLA配列が含まれていなかったためBroad Instituteが提供しているリファレンスでやり直した。

```bwa+samtools
bwa mem \
 -R @RGtID:NA18939_SRR1295537tPU:unknowntSM:NA18939_SRR1295537tPL:ILLUMINAtLB:NA18939_SRR1295537_LB \
 ./reference/hg38_gatk/Homo_sapiens_assembly38.fasta \
 SRR1295537_1_33p_trim.fastq.gz \
 SRR1295537_2_33p_trim.fastq.gz | \
 samtools view -@4 -bS -o NA18939_SRR1295537_hg38.bam
```

## BAMソート, BAM index作成

```
samtools sort -O BAM -o NA18939_SRR1295537_hg38_sorted.bam NA18939_SRR1295537_hg38.bam

bwa index NA18939_SRR1295537_hg38_sorted.bam
```

## 重複マーク：Mark Duplicates, BAMソート

```MarkDuplicates
java -jar picard.jar MarkDuplicates \
 -I NA18939_SRR1295537_hg38_sorted.bam \
 -O NA18939_SRR1295537_hg38_sorted_MarkDup.bam \
 -M NA18939_SRR1295537_hg38_sorted_MarkDup_metrics.txt

java -jar picard.jar SortSam \
 -I NA18939_SRR1295537_hg38_sorted_MarkDup.bam \
 -O NA18939_SRR1295537_hg38_sorted_MarkDup_resorted.bam \
 --SORT_ORDER coordinate
```

## BQSR : BaseRecalibrator

```
gatk BaseRecalibrator -I NA18939_SRR1295537_hg38_sorted_MarkDup_resorted.bam \
 -R ./reference/hg38_gatk/Homo_sapiens_assembly38.fasta \
 -O ./BQSR/NA18939_SRR1295537_hg38_BQSR.table \
 --known-sites ./reference/hg38_resources/Homo_sapiens_assembly38.dbsnp138.vcf
```

## ApplyBQSR

```
gatk ApplyBQSR -R ./reference/hg38_gatk/Homo_sapiens_assembly38.fasta \
 -I NA18939_SRR1295537_hg38_sorted_MarkDup_resorted.bam \
 --bqsr-recal-file NA18939_SRR1295537_hg38_BQSR.table \
 -O ./BQSR/NA18939_SRR1295537_hg38_ApplyBQSR.bam
```

## BAM サマリ

ApplyBQSR後のBAMのサマリ。

```
samtools stats ./BQSR/NA18939_SRR1295537_hg38_ApplyBQSR.bam > ./BQSR/stats/samtools_stats.txt
```

一部

```
~
# Summary Numbers. Use `grep ^SN | cut -f 2-` to extract this part.
SN	raw total sequences:	109968824	# excluding supplementary and secondary reads
SN	filtered sequences:	0
SN	sequences:	109968824
SN	is sorted:	1	# sorted by coordinate
SN	1st fragments:	54984412
SN	last fragments:	54984412
SN	reads mapped:	109251271
SN	reads mapped and paired:	108645592	# paired-end technology bit set + both mates mapped
SN	reads unmapped:	717553
SN	reads properly paired:	105925244	# proper-pair bit set
SN	reads paired:	109968824	# paired-end technology bit set
SN	reads duplicated:	313567	# PCR or optical duplicate bit set
SN	reads MQ0:	6378901	# mapped and MQ=0
SN	reads QC failed:	0
SN	non-primary alignments:	0
SN	supplementary alignments:	721557
SN	total length:	24928928915	# ignores clipping
SN	total first fragment length:	12805508548	# ignores clipping
SN	total last fragment length:	12123420367	# ignores clipping
SN	bases mapped:	24913011279	# ignores clipping
SN	bases mapped (cigar):	24891186395	# more accurate
SN	bases trimmed:	0
SN	bases duplicated:	68993209
SN	mismatches:	127014673	# from NM fields
SN	error rate:	5.102797e-03	# mismatches / bases mapped (cigar)
SN	average length:	227
SN	average first fragment length:	233
SN	average last fragment length:	220
SN	maximum length:	250
SN	maximum first fragment length:	250
SN	maximum last fragment length:	250
SN	average quality:	30.1
SN	insert size average:	537.6
SN	insert size standard deviation:	893.1
SN	inward oriented pairs:	53350140
SN	outward oriented pairs:	466811
SN	pairs with other orientation:	116614
SN	pairs on different chromosomes:	389231
SN	percentage of properly paired reads (%):	96.3
~
```

## バリアント検出

### HaplotypeCaller

バリアント検出後、GVCFのindex作成

```
gatk --java-options -Xmx4g HaplotypeCaller \
 -R ./reference/hg38_gatk/Homo_sapiens_assembly38.fasta \
 -I ./BQSR/NA18939_SRR1295537_hg38_ApplyBQSR.bam \
 -O ./VCF/NA18939_SRR1295537_hg38.g.vcf.gz \
 -ERC GVCF

gatk IndexFeatureFile -I ./VCF/NA18939_SRR1295537_hg38.g.vcf.gz
```

### Genotyping

```
gatk --java-options -Xmx4g GenotypeGVCFs \
 -R ./reference/hg38/hg38_reference.fa \
 -V ./VCF/NA18939_SRR1295537_hg38.g.vcf.gz \
 -O ./VCF/NA18939_SRR1295537_hg38_Genotype.vcf.gz
```

## ハードフィルタリング

SNP, indelを分けてフィルタリング

### SNP

```
gatk SelectVariants \
 -V .VCF/NA18939_SRR1295537_hg38_Genotype.vcf.gz \
 -select-type SNP \
 -O ./VCF/NA18939_SRR1295537_hg38_Genotype_snp.vcf.gz

gatk VariantFiltration \
 -V ./VCF/NA18939_SRR1295537_hg38_Genotype_snp.vcf.gz \
 -R ./reference/hg38_gatk/Homo_sapiens_assembly38.fasta \
 -O ./VCF/NA18939_SRR1295537_hg38_Genotype_snp_filtered.vcf.gz \
 -filter "QD < 2.0" --filter-name "QD2" \
 -filter "QUAL < 30.0" --filter-name "QUAL30" \
 -filter "SOR > 3.0" --filter-name "SOR4" \
 -filter "FS > 60.0" --filter-name "FS60" \
 -filter "MQ < 40.0" --filter-name "MQ40" \
 -filter "MQRankSum < -12.5" --filter-name "MQRankSum-12.5" \
 -filter "ReadPosRankSum < -8.0" --filter-name "ReadPosRankSum-8"
```

### indel

```
gatk SelectVariants \
 -V ./VCF/NA18939_SRR1295537_hg38_Genotype.vcf.gz \
 -select-type INDEL \
 -O ./VCF/NA18939_SRR1295537_hg38_Genotype_indel.vcf.gz

gatk VariantFiltration \
 -V ./VCF/NA18939_SRR1295537_hg38_Genotype_indel.vcf.gz \
 -R ./reference/hg38_gatk/Homo_sapiens_assembly38.fasta \
 -O ./VCF/NA18939_SRR1295537_hg38_Genotype_indel_filtered.vcf.gz \
 -filter "QD < 2.0" --filter-name "QD2" \
 -filter "QUAL < 30.0" --filter-name "QUAL30" \
 -filter "FS > 200.0" --filter-name "FS200" \
 -filter "ReadPosRankSum < -20.0" --filter-name "ReadPosRankSum-20"
```

### PASSのみ取得

```
bcftools view --with-header -f PASS -Oz \
 -o ./VCF/NA18939_SRR1295537_hg38_Genotype_snp_filtered_PASS.vcf.gz \
 ./VCF/NA18939_SRR1295537_hg38_Genotype_snp_filtered.vcf.gz

bcftools view --with-header -f PASS -Oz \
 -o ./VCF/NA18939_SRR1295537_hg38_Genotype_indel_filtered_PASS.vcf.gz \
 ./VCF/NA18939_SRR1295537_hg38_Genotype_indel_filtered.vcf.gz
```

## サマリ

```
bcftools stats \
 ./VCF/NA18939_SRR1295537_hg38_Genotype_snp_filtered_PASS.vcf.gz \
 > ./VCF/vcf-stats/NA18939_SRR1295537_hg38_Genotype_snp_filtered_PASS_stats.vchk

plot-vcfstats \
 -p ./data/VCF/vcf-stats/vcf-stats \
 ./VCF/vcf-stats/NA18939_SRR1295537_hg38_Genotype_snp_filtered_PASS_stats.vchk
```

結果一部

```
~
# Definition of sets:
# ID	[2]id	[3]tab-separated file names
ID	0	/Users/chitosetakahashi/Analysis/NA18939_SAME123041/data/VCF/NA18939_SRR1295537_hg38_Genotype_snp_filtered_PASS.vcf.gz
# SN, Summary numbers:
#   number of records   .. number of data rows in the VCF
#   number of no-ALTs   .. reference-only sites, ALT is either "." or identical to REF
#   number of SNPs      .. number of rows with a SNP
#   number of MNPs      .. number of rows with a MNP, such as CC>TT
#   number of indels    .. number of rows with an indel
#   number of others    .. number of rows with other type, for example a symbolic allele or
#                          a complex substitution, such as ACT>TCGA
#   number of multiallelic sites     .. number of rows with multiple alternate alleles
#   number of multiallelic SNP sites .. number of rows with multiple alternate alleles, all SNPs
# 
#   Note that rows containing multiple types will be counted multiple times, in each
#   counter. For example, a row with a SNP and an indel increments both the SNP and
#   the indel counter.
# 
# SN	[2]id	[3]key	[4]value
SN	0	number of samples:	1
SN	0	number of records:	3222321
SN	0	number of no-ALTs:	0
SN	0	number of SNPs:	3222321
SN	0	number of MNPs:	0
SN	0	number of indels:	0
SN	0	number of others:	0
SN	0	number of multiallelic sites:	3618
SN	0	number of multiallelic SNP sites:	1098
# TSTV, transitions/transversions
#   - transitions, see https://en.wikipedia.org/wiki/Transition_(genetics)
#   - transversions, see https://en.wikipedia.org/wiki/Transversion
# TSTV	[2]id	[3]ts	[4]tv	[5]ts/tv	[6]ts (1st ALT)	[7]tv (1st ALT)	[8]ts/tv (1st ALT)
TSTV	0	2139982	1083437	1.98	2138462	1081527	1.98
# SiS, Singleton stats:
#   - allele count, i.e. the number of singleton genotypes (AC=1)
#   - number of transitions, see above
#   - number of transversions, see above
#   - repeat-consistent, inconsistent and n/a: experimental and useless stats [DEPRECATED]
# SiS	[2]id	[3]allele count	[4]number of SNPs	[5]number of transitions	[6]number of transversions	[7]number of indels	[8]repeat-consistent	[9]repeat-inconsistent	[10]not applicable
SiS	0	1	1639464	1079422	560042	0	0	0	0
# AF, Stats by non-reference allele frequency:
# AF	[2]id	[3]allele frequency	[4]number of SNPs	[5]number of transitions	[6]number of transversions	[7]number of indels	[8]repeat-consistent	[9]repeat-inconsistent	[10]not applicable
AF	0	0.000000	1639464	1079422	560042	0	0	0	0
AF	0	0.990000	1583955	1060560	523395	0	0	0	0
~
```

indelのサマリは出していません