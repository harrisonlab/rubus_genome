Commands used to generate a collapsed assembly for raspberry cv. malling jewel:


```bash
WorkDir=/projects/rubus_genome
mkdir -p $WorkDir
cd $WorkDir
```

```bash
WorkDir=/projects/rubus_genome
cd $WorkDir

OutDir=raw_dna/minion/R.idaeus/malling_jewel
mkdir -p $OutDir

RawDatDir=$(ls -d /data/seq_data/minion/2018/Sam1MJ)
RunID=$(echo $RawDatDir | rev | cut -f1 -d '/' | rev)
cat $RawDatDir/*/*.fastq | gzip -cf > $OutDir/${RunID}.fq.gz

RawDatDir=$(ls -d /data/seq_data/minion/2018/SamMJ2)
RunID=$(echo $RawDatDir | rev | cut -f1 -d '/' | rev)
cat $RawDatDir/*/*.fastq | gzip -cf > $OutDir/${RunID}.fq.gz

RawDatDir=$(ls -d /data/seq_data/minion/2019/SamMJ3)
RunID=$(echo $RawDatDir | rev | cut -f1 -d '/' | rev)
cat $RawDatDir/*/*.fastq | gzip -cf > $OutDir/${RunID}.fq.gz
```


## Removal of adapters

Splitting reads and trimming adapters using porechop
```bash
for RawReads in $(ls raw_dna/minion/*/*/*.fq.gz | grep 'malling_jewel'); do
Organism=$(echo $RawReads| rev | cut -f3 -d '/' | rev)
Strain=$(echo $RawReads | rev | cut -f2 -d '/' | rev)
echo "$Organism - $Strain"
OutDir=qc_dna/minion/$Organism/$Strain
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/dna_qc
qsub $ProgDir/sub_porechop_high_mem.sh $RawReads $OutDir
done
```


## Identify sequencing coverage

For Minion data:

```bash
cd /home/groups/harrisonlab/project_files/rubus_genome
for RawData in $(ls ../../../../../projects/rubus_genome/qc_dna/minion/R.idaeus/malling_jewel/*.fastq.gz); do
echo $RawData;
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/dna_qc;
GenomeSz=293
# OutDir=$(dirname $RawData)
OutDir=$(echo $RawData | cut -f10,11,12,13 -d '/')
mkdir -p $OutDir
qsub $ProgDir/sub_count_nuc.sh $GenomeSz $RawData $OutDir
done
```


```bash
  for StrainDir in $(ls -d qc_dna/minion/*/*); do
    Strain=$(basename $StrainDir)
    printf "$Strain\t"
    for File in $(ls $StrainDir/*.txt); do
      echo $(basename $File);
      cat $File | tail -n1 | rev | cut -f2 -d ' ' | rev;
    done | grep -v '.txt' | awk '{ SUM += $1} END { print SUM }'
  done
```


MinION coverage

```
  autumn_bliss	64.35
  malling_jewel	48.93
```

### Read correction using Canu


This work was performed on the new cluster:

```bash
  screen -a
  ssh compute03
  cd /oldhpc/projects/rubus_genome
```

```bash
Organism="R.idaeus"
Strain="malling_jewel"

Fastq1=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n1 | tail -n1)
Fastq2=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n2 | tail -n1)
Fastq3=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n3 | tail -n1)
Size=293M
Prefix=${Strain}_v1
OutDir=assembly/canu-1.8/$Organism/"$Strain"
mkdir -p $OutDir

CurPath=$PWD
WorkDir=/tmp/canu/$Prefix

# ---------------
# Step 2
# Run Canu
# ---------------

mkdir -p $WorkDir
cd $WorkDir

canu \
  useGrid=false \
  -overlapper=mhap \
  -utgReAlign=true \
  -d $WorkDir/assembly \
  -p $Prefix genomeSize="$Size" \
  -nanopore-raw $CurPath/$Fastq1 \
  -nanopore-raw $CurPath/$Fastq2 \
  -nanopore-raw $CurPath/$Fastq3 \
  2>&1 | tee canu_run_log.txt

mkdir -p $CurPath/$OutDir
cp canu_run_log.txt $CurPath/$OutDir/.
cp $WorkDir/assembly/* $CurPath/$OutDir/.
```

Quast and busco were run to assess assembly quality:

```bash
cd /projects/rubus_genome
for Assembly in $(ls assembly/canu-1.8/*/*/*.contigs.fasta | grep 'malling_jewel'); do
  Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
  Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)  
  OutDir=$(dirname $Assembly)
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
  # qsub $ProgDir/sub_quast.sh $Assembly $OutDir
  BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/embryophyta_odb10)
  OutDir=gene_pred/busco/$Organism/$Strain/assembly
  ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
  qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```

```
Assembly                   malling_jewel_v1.contigs
# contigs (>= 0 bp)        2334                    
# contigs (>= 1000 bp)     2334                    
Total length (>= 0 bp)     389356978               
Total length (>= 1000 bp)  389356978               
# contigs                  2334                    
Largest contig             5728181                 
Total length               389356978               
GC (%)                     37.32                   
N50                        546040                  
N75                        202843                  
L50                        167                     
L75                        465                     
# N's per 100 kbp          0.00
```

### Assembly using SMARTdenovo


This work was performed on the new cluster:

```bash
  screen -a
  ssh compute03
  cd /projects/rubus_genome
```

```bash
Organism="R.idaeus"
Strain="malling_jewel"
FastaIn=$(ls assembly/canu-1.8/R.idaeus/malling_jewel/malling_jewel_v1.correctedReads.fasta.gz)
Prefix=${Strain}_v2
OutDir=assembly/SMARTdenovo/${Organism}/${Strain}
echo  "Running SMARTdenovo with the following inputs:"
echo "FastaIn - $FastaIn"
echo "Prefix - $Prefix"
echo "OutDir - $OutDir"

CurPath=$PWD
WorkDir=/tmp/SMRTdenovo/${Strain}

# ---------------
# Step 2
# Run SMARTdenovo
# ---------------

mkdir -p $WorkDir
cd $WorkDir
Fasta=$(basename $FastaIn)
cp $CurPath/$FastaIn $Fasta

cat $Fasta | gunzip -cf > reads.fa
smartdenovo.pl -t 40 reads.fa -p $Prefix > $Prefix.mak

make -f $Prefix.mak 2>&1 | tee "$Prefix"_run_log.txt

rm $Prefix.mak
rm $Fasta
rm reads.fa
mkdir -p $CurPath/$OutDir
for File in $(ls $WorkDir/wtasm*); do
  NewName=$(echo $File | sed "s/wtasm/$Prefix/g")
  mv $File $NewName
done
cp -r $WorkDir/* $CurPath/$OutDir/.
```


Quast and busco were run to assess the effects of racon on assembly quality:

```bash
for Assembly in $(ls assembly/SMARTdenovo/*/*/*.dmo.lay.utg); do
  Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
  Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)  
  OutDir=$(dirname $Assembly)
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
  qsub $ProgDir/sub_quast.sh $Assembly $OutDir
  BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/embryophyta_odb10)
  OutDir=gene_pred/busco/$Organism/$Strain/assembly
  ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
  qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```

```
Assembly                   malling_jewel_v2.dmo.lay.utg
# contigs (>= 0 bp)        1602                        
# contigs (>= 1000 bp)     1602                        
Total length (>= 0 bp)     265871838                   
Total length (>= 1000 bp)  265871838                   
# contigs                  1602                        
Largest contig             4243019                     
Total length               265871838                   
GC (%)                     37.15                       
N50                        329556                      
N75                        127024                      
L50                        191                         
L75                        519                         
# N's per 100 kbp          0.00  
```



# Genome assembly using Flye

This work was performed on the new cluster:

```bash
  screen -a
  srun --nodelist compute05 --partition unlimited --pty bash
  cd /projects/rubus_genome
```


```bash
conda activate flye

Organism="R.idaeus"
Strain="malling_jewel"

Fastq1=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n1 | tail -n1)
Fastq2=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n2 | tail -n1)
Fastq3=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n3 | tail -n1)
Size=293M
Prefix=malling_jewel_v3
OutDir=assembly/flye/$Organism/"$Strain"
mkdir -p $OutDir

CurPath=$PWD
WorkDir=/tmp/flye/$Prefix

# ---------------
# Step 2
# Run Canu
# ---------------

mkdir -p $WorkDir
cd $WorkDir

flye --nano-raw $CurPath/$Fastq1 $CurPath/$Fastq2  $CurPath/$Fastq3 --genome-size $Size --out-dir $Prefix --threads 40


ls $Prefix

conda deactivate
```

# Diploid genome assembly


## Diploid assembly using Canu


This work was performed on the new cluster:

```bash
  screen -a
  srun --nodelist compute04 --partition unlimited --pty bash
  cd /projects/rubus_genome
```

```bash
Organism="R.idaeus"
Strain="malling_jewel"

Fastq1=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n1 | tail -n1)
Fastq2=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n2 | tail -n1)
Fastq3=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n3 | tail -n1)
Size=586M
Prefix=${Strain}_v4
OutDir=assembly/canu-1.8/$Organism/"$Strain"/$Prefix
mkdir -p $OutDir

CurPath=$PWD
WorkDir=/tmp/canu/$Prefix

# ---------------
# Step 2
# Run Canu
# ---------------

mkdir -p $WorkDir
cd $WorkDir

canu \
  useGrid=false \
  -overlapper=mhap \
  -utgReAlign=true \
  -d $WorkDir/assembly \
  -p $Prefix genomeSize="$Size" \
  -nanopore-raw $CurPath/$Fastq1 \
  -nanopore-raw $CurPath/$Fastq2 \
  -nanopore-raw $CurPath/$Fastq3 \
  2>&1 | tee canu_run_log.txt

mkdir -p $CurPath/$OutDir
cp canu_run_log.txt $CurPath/$OutDir/.
cp $WorkDir/assembly/* $CurPath/$OutDir/.
```

Quast and busco were run to assess the effects of racon on assembly quality:

```bash
for Assembly in $(ls assembly/canu-1.8/R.idaeus/malling_jewel/malling_jewel_v4/malling_jewel_v4.contigs.fasta); do
  Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
  Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)  
  OutDir=$(dirname $Assembly)
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
  qsub $ProgDir/sub_quast.sh $Assembly $OutDir
  BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/embryophyta_odb10)
  OutDir=gene_pred/busco/$Organism/$Strain/assembly
  ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
  qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```

### Assembly using SMARTdenovo


This work was performed on the new cluster:

```bash
  screen -a
  ssh compute01
  cd /projects/rubus_genome
```

```bash
Organism="R.idaeus"
Strain="malling_jewel"
FastaIn=$(ls assembly/canu-1.8/R.idaeus/malling_jewel/malling_jewel_v4/malling_jewel_v4.correctedReads.fasta.gz)
Prefix=${Strain}_v4
OutDir=assembly/SMARTdenovo/${Organism}/${Strain}/$Prefix
echo  "Running SMARTdenovo with the following inputs:"
echo "FastaIn - $FastaIn"
echo "Prefix - $Prefix"
echo "OutDir - $OutDir"

CurPath=$PWD
WorkDir=/tmp/SMARTdenovo/${Strain}

# ---------------
# Step 2
# Run SMARTdenovo
# ---------------

mkdir -p $WorkDir
cd $WorkDir
Fasta=$(basename $FastaIn)
cp $CurPath/$FastaIn $Fasta

cat $Fasta | gunzip -cf > reads.fa
smartdenovo.pl -t 40 reads.fa -p $Prefix > $Prefix.mak

make -f $Prefix.mak 2>&1 | tee "$Prefix"_run_log.txt

rm $Prefix.mak
rm $Fasta
rm reads.fa
mkdir -p $CurPath/$OutDir
for File in $(ls $WorkDir/wtasm*); do
  NewName=$(echo $File | sed "s/wtasm/$Prefix/g")
  mv $File $NewName
done
cp -r $WorkDir/* $CurPath/$OutDir/.
```

## Diploid assembly using Flye


This work was performed on the new cluster:

```bash
  screen -a
  srun --nodelist compute02 --partition unlimited --pty bash
  cd /projects/rubus_genome
```


```bash
conda activate flye

Organism="R.idaeus"
Strain="malling_jewel"

Fastq1=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n1 | tail -n1)
Fastq2=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n2 | tail -n1)
Fastq3=$(ls qc_dna/minion/*/*/*q.gz | grep 'malling_jewel' | head -n3 | tail -n1)
Size=586M
Prefix=malling_jewel_v6
OutDir=assembly/flye/$Organism/"$Strain"
mkdir -p $OutDir

CurPath=$PWD
WorkDir=/tmp/flye/$Prefix

# ---------------
# Step 2
# Run Canu
# ---------------

mkdir -p $WorkDir
cd $WorkDir

flye --nano-raw $CurPath/$Fastq1 $CurPath/$Fastq2  $CurPath/$Fastq3 --genome-size $Size --out-dir $Prefix --threads 40


ls $Prefix

conda deactivate
```
