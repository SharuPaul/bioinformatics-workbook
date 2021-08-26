# How to iterate multiple rounds of Pilon polishing/gap-filling on a genome assembly

After assembling a genome, polishing is usually done to correct small assembly errors. Typically, multiple rounds of polishing is not recommended, due to declining BUSCO scores. However, in certain situations, multiple rounds of polishing may be warranted.  In this case, my reads were obtained from thousands of individuals (population assembly). Exacerbating the problem, soybean cyst nematodes do not like homozygosity, so these individuals were fairly diverse.  

### Prerequisite software

* Hisat2 --  a quick aligner
* Samtools -- who doesnt need to convert sam to bam
* Pilon -- my polisher of choice due to its overwhelmingly informative output.  


### Setting up your directory
```
#my working directory
/work/gif/remkv6/USDA/03_LoopingPolisher

#softlink your genome
ln -s /work/gif/remkv6/Baum/04_DovetailSCNGenome/49_RenameChromosomes/01_Transfer2Box/SCNgenome.fasta

#softlink the reads
ln -s /work/gif/archiveNova/Baum/08_GlandEndoreduplicationDNA-seq/lane2/Hg2S-1_S1_L002_R1_001.fastq.gz
ln -s /work/gif/archiveNova/Baum/08_GlandEndoreduplicationDNA-seq/lane2/Hg2S-1_S1_L002_R2_001.fastq.gz
```

### Create a run script for pilon
```
#runPilon.sh
###############################################################################
#!/bin/bash

#You must provide the following. Note variable DBDIR does not need a "/" at the end.
# sh runPilon.sh  /work/GIF/remkv6/files genome.fa ShortReadsR1.fq ShortReadsR2.fq

#create unix variables
DIR="$1"
GENOME="$2"
R1_FQ="$3"
R2_FQ="$4"

# Index the genome and perform short read mapping using Hisat2
module load hisat2
hisat2-build ${GENOME} ${GENOME%.*}
hisat2 -p 36 -x ${GENOME%.*} -1 $R1_FQ -2 $R2_FQ -S ${GENOME%.*}.${R1_FQ%.*}.sam

# Convert your sam to bam, sort, and index.
# Note that I use only a part of my available threads to ensure I do not run into RAM usage issues.  This is the safest approach when iterating.
module load samtools
samtools view --threads 12 -b -o ${GENOME%.*}.${R1_FQ%.*}.bam ${GENOME%.*}.${R1_FQ%.*}.sam
mkdir Samtemp
samtools sort  -o ${GENOME%.*}.${R1_FQ%.*}_sorted.bam -T Samtemp --threads 16 ${GENOME%.*}.${R1_FQ%.*}.bam
samtools index ${GENOME%.*}.${R1_FQ%.*}_sorted.bam


module load pilon/1.22-s7zrot6
# I am running pilon here using the max memory available to me, using a temp folder I created above. Pilon can also have issues with RAM usage, so in this case I cut my thread usage to half of those available. My chunk size is also smaller than default, again catering to potential ram issues that would affect iteration
java -Xmx200g -Djava.io.tmpdir=Samtemp -jar /opt/rit/spack-app/linux-rhel7-x86_64/gcc-4.8.5/pilon-1.22-s7zrot6o5yqjh6oxpdxsxcdiswpjioyy/bin/pilon-1.22.jar  --genome ${GENOME} --frags  ${GENOME%.*}.${R1_FQ%.*}_sorted.bam --output ${GENOME%.*}.pilon --outdir ${DIR} --changes --fix all --threads 18  --chunksize 30000


# The mapping files can get out of hand, so when I have the loop working, I delete the sam and bam files after each round.
rm *sam
rm *bam
```

### How to iterate the runPilon.sh script
```
Typically I would run one round of polishing  like this
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz

However, to iterate we need to put this into a loop. Lets run 20 rounds of polishing
for f in {01..20}; do echo "sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome"${f}".fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome"${f}".pilon.fasta" ;done |paste - <(for f in {02..21}; do echo $line" SCNgenome"${f}".fasta";done) |sed 's/\t/ /g' >PolishLoop.sh


```

###### PolishLoop.sh
<details>
  <summary>PolishLoop.sh content</summary>
  <pre>
```
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome01.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome01.pilon.fasta  SCNgenome02.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome02.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome02.pilon.fasta  SCNgenome03.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome03.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome03.pilon.fasta  SCNgenome04.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome04.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome04.pilon.fasta  SCNgenome05.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome05.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome05.pilon.fasta  SCNgenome06.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome06.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome06.pilon.fasta  SCNgenome07.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome07.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome07.pilon.fasta  SCNgenome08.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome08.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome08.pilon.fasta  SCNgenome09.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome09.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome09.pilon.fasta  SCNgenome10.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome10.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome10.pilon.fasta  SCNgenome11.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome11.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome11.pilon.fasta  SCNgenome12.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome12.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome12.pilon.fasta  SCNgenome13.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome13.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome13.pilon.fasta  SCNgenome14.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome14.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome14.pilon.fasta  SCNgenome15.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome15.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome15.pilon.fasta  SCNgenome16.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome16.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome16.pilon.fasta  SCNgenome17.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome17.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome17.pilon.fasta  SCNgenome18.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome18.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome18.pilon.fasta  SCNgenome19.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome19.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome19.pilon.fasta  SCNgenome20.fasta
sh runPilon.sh /work/gif/remkv6/USDA/03_LoopingPolisher SCNgenome20.fasta Hg2S-1_S1_L002_R1_001.fastq.gz Hg2S-1_S1_L002_R2_001.fastq.gz; mv SCNgenome20.pilon.fasta  SCNgenome21.fasta
```
</pre>
</details>

### Evaluate the results of polishing the genome in a loop
```
Since we set the --changes flag in our Pilon runs, we know the number of changes made in each round
wc -l *changes
  43577 SCNgenome01.pilon.changes
   4396 SCNgenome02.pilon.changes
   1241 SCNgenome03.pilon.changes
    496 SCNgenome04.pilon.changes
    308 SCNgenome05.pilon.changes
    186 SCNgenome06.pilon.changes
    105 SCNgenome07.pilon.changes

```