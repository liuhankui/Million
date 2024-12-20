# fq2bam
```
fastp -q 20 -u 10 -n 1 --in1 fq.gz -o clean.fq.gz
bwa aln GRCh38.fa clean.fq.gz|samblaster --excludeDups --ignoreUnmated --maxSplitCount 2 --minNonOverlap 20|samtools view -Sb -F 2820 -q 1 -|sambamba sort -t 4 -m 2G --tmpdir=./tmp -o sample.bam /dev/stdin
mosdepth qc sample.bam -f GRCh38.fa --fast-mode --no-per-base --by 1000000 --thresholds 1,2,4
java -Xmx3g -jar gatk.jar BaseRecalibrator -R GRCh38.fa -I sample.bam -O bqsr.file --known-sites dbsnp_146.hg38.vcf.gz --known-sites 1000G_omni2.5.hg38.vcf.gz --known-sites hapmap_3.3.hg38.vcf.gz
java -Xmx3g -jar gatk.jar ApplyBQSR -R GRCh38.fa -I sample.bam -O sample.bqsr.bam --bqsr-recal-file bqsr.file
```

# bam2SNV
```
ls path2bam/*.bqsr.bam > bam.list
BaseVar basetype -t 10 -b 100 -i bam.list -r GRCh38.fa -o sample
zcat sample.vcf.gz|cut -f '1-8'|bcftools norm - -m -|bcftools view - -i "QUAL > 100 && FORMAT/CM_DP > 100 && FORMAT/CM_AF > 0.01 && FORMAT/CM_AF < 0.99" -T 35bp.bed -O z -o sample.filter.vcf.gz
java -Xmx30g -jar gatk.jar VariantRecalibrator --tmp-dir ./tmp -R GRCh38.fa -V sample.filter.vcf.gz \
             -resource:hapmap,known=false,training=true,truth=true,prior=15.0 hapmap_3.3.hg38.vcf.gz \
             -resource:omini,known=false,training=true,truth=false,prior=12.0 1000G_omni2.5.hg38.vcf.gz \
             -resource:1000G,known=false,training=true,truth=false,prior=10.0 1000G_phase1.snps.high_confidence.hg38.vcf.gz \
             -resource:dbsnp,known=true,training=false,truth=false,prior=2.0 dbsnp_146.hg38.vcf.gz \
             --max-attempts 2 --maximum-training-variants 5000000 \
             -an BaseQRankSum -an ReadPosRankSum -an FS -an QD -an SOR -an MQRankSum -an CM_DP \
             -mode SNP \
             -tranche 99.9 -tranche 99.5 -tranche 99.0 -tranche 95.0 -tranche 90.0 -tranche 89.0 -tranche 88.0 -tranche 87.0 -tranche 86.0 -tranche 85.0 -tranche 80.0 \
             --rscript-file vqsr.R \
             --tranches-file vqsr.tranches \
            -O vqsr.recal
java -Xmx30g -jar gatk.jar ApplyVQSR --tmp-dir ./tmp -R GRCh38.fa -V sample.filter.vcf.gz \
            -ts-filter-level 99.0 \
            --tranches-file vqsr.tranches \
            --recal-file vqsr.recal \
            -mode SNP \
            -O sample.filter.vqsr.vcf.gz
zcat sample.filter.vqsr.vcf.gz|vawk --header '{if($7=="PASS")print}'|bgzip -f > sample.pass.vcf.gz
bcftools stats sample.pass.vcf.gz | grep "TSTV"
```

# SNP imputation
```
#reference panel
cat bam.list|awk -F '/' '{print $NF}'|awk -F '.' '{print $1}' > sample.id
tabix Han.vcf.gz chr1:1-10000000 -h|bcftools convert - --haplegendsample ref
tabix sample.pass.vcf.gz chr1:1-10000000|cut -f '1,2,4,5' > pos.txt
#QUILT for sample size < 10000
Rscript QUILT.R  --outputdir=.  --tempdir=./tmp  --sampleNames_file=sample.id  --chr=chr1  --regionStart=1  --regionEnd=10000000  --buffer=250000  --bamlist=bam.list  --posfile=pos.txt  --reference_haplotype_file=ref.hap.gz  --reference_legend_file=ref.legend.gz  --nGen=100  --nCores=20
zcat quilt.chr1.1.10000000.vcf.gz|vawk '{if(I$INFO_SCORE > 0.8 && I$EAF > 0.025 && I$EAF < 0.975 && I$HWE > 0.0005){print}}' --header|bgzip -f > chr1.1.10000000.SNP.vcf.gz
#STITCH for sample size > 10000
Rscript STITCH.R  --outputdir=.  --tempdir=./tmp  --sampleNames_file=sample.id  --chr=chr1  --regionStart=1  --regionEnd=10000000  --buffer=250000  --bamlist=bam.list  --posfile=pos.txt  --reference_haplotype_file=ref.hap.gz  --reference_legend_file=ref.legend.gz  --nGen=100  --nCores=20
zcat stitch.chr1.1.10000000.vcf.gz|vawk '{if(I$INFO_SCORE > 0.8 && I$EAF > 0.025 && I$EAF < 0.975 && I$HWE > 0.0005 && I$AC/I$AN > 0.8){print}}' --header|bgzip -f > chr1.1.10000000.SNP.vcf.gz
```

# VCF concat
```
ls path2vcf/*.SNP.vcf.gz|awk -F '/' '{print $NF}'|tr '.' ' '|sort -k1,1 -k2,2n|tr ' ' '.'|awk '{print "path2vcf/"$0}' > vcf.list
bcftools concat -f vcf.list -O z -o SNP.vcf.gz
tabix -s1 -b2 -e2 -f SNP.vcf.gz
```
