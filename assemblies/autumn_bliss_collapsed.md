Commands used to generate a collapsed assembly for raspberry cv. autumn bliss:


```bash
WorkDir=/data/scratch/armita/rubus
mkdir -p $WorkDir
cd $WorkDir
```

```bash
WorkDir=/data/scratch/armita/rubus
cd $WorkDir

OutDir=raw_dna/minion/R.idaeus/autumn_bliss
mkdir -p $OutDir

RawDatDir=$(ls -d /data/seq_data/minion/2018/20180616_AutumnBliss1)
RunID=$(echo $RawDatDir | rev | cut -f1 -d '/' | rev)
cat $RawDatDir/*/*/*.fastq | gzip -cf > $OutDir/${RunID}.fq.gz

RawDatDir=$(ls -d /data/seq_data/minion/2018/20180616_AutumnBliss1-2)
RunID=$(echo $RawDatDir | rev | cut -f1 -d '/' | rev)
cat $RawDatDir/*/*/*.fastq | gzip -cf > $OutDir/${RunID}.fq.gz

RawDatDir=$(ls -d /data/seq_data/minion/2018/20180616_AutumnBliss2)
RunID=$(echo $RawDatDir | rev | cut -f1 -d '/' | rev)
cat $RawDatDir/*/*/*.fastq | gzip -cf > $OutDir/${RunID}.fq.gz

RawDatDir=$(ls -d /data/seq_data/minion/2018/20180711_AutumnBliss2)
RunID=$(echo $RawDatDir | rev | cut -f1 -d '/' | rev)
cat $RawDatDir/*/*.fastq | gzip -cf > $OutDir/${RunID}.fq.gz

RawDatDir=$(ls -d /data/seq_data/minion/2018/AutumnBliss3)
RunID=$(echo $RawDatDir | rev | cut -f1 -d '/' | rev)
cat $RawDatDir/*/*.fastq | gzip -cf > $OutDir/${RunID}.fq.gz

RawDatDir=$(ls -d /data/seq_data/minion/2018/AutumnBliss-20180814)
RunID=$(echo $RawDatDir | rev | cut -f1 -d '/' | rev)
cat $RawDatDir/*/*/.fastq | gzip -cf > $OutDir/${RunID}.fq.gz
```


## Removal of adapters

Splitting reads and trimming adapters using porechop
```bash
for RawReads in $(ls raw_dna/minion/*/*/*.fq.gz | grep 'autumn_bliss'); do
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
for RawData in $(ls ../../../../../data/scratch/armita/rubus/qc_dna/minion/R.idaeus/autumn_bliss/*.fastq.gz); do
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

```
autumn_bliss	64.35
malling_jewel	48.93
```

### Read correction using Canu


This work was performed on the new cluster:

```bash
  screen -a
  srun --nodelist compute01 --partition unlimited --pty bash
  cd /data/scratch/armita/rubus

```

```bash

Organism="R.idaeus"
Strain="autumn_bliss"

Fastq1=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n1 | tail -n1)
Fastq2=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n2 | tail -n1)
Fastq3=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n3 | tail -n1)
Fastq4=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n4 | tail -n1)
Fastq5=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n5 | tail -n1)
Fastq6=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n6 | tail -n1)
Size=293M
Prefix=autumn_bliss_v1
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
  -nanopore-raw $CurPath/$Fastq4 \
  -nanopore-raw $CurPath/$Fastq5 \
  -nanopore-raw $CurPath/$Fastq6 \
  2>&1 | tee canu_run_log.txt

mkdir -p $CurPath/$OutDir
cp canu_run_log.txt $CurPath/$OutDir/.
cp $WorkDir/assembly/* $CurPath/$OutDir/.
```

Quast and busco were run to assess the effects of racon on assembly quality:

```bash
cd /data/scratch/armita/rubus
for Assembly in $(ls assembly/canu-1.8/*/*/autumn_bliss_v1/*.contigs.fasta); do
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
  # ssh compute03
  srun --nodelist compute01 --partition unlimited --pty bash
  cd /data/scratch/armita/rubus

```

```bash
Organism="R.idaeus"
Strain="autumn_bliss"
FastaIn=$(ls assembly/canu-1.8/R.idaeus/autumn_bliss/autumn_bliss_v1.correctedReads.fasta.gz)
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


Quast and busco were run to assess quality:

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

# Genome assembly using Flye

This work was performed on the new cluster:

```bash
  screen -a
  # ssh compute03
  srun --nodelist compute01 --partition unlimited --pty bash
  cd /data/scratch/armita/rubus
```


```bash
conda activate flye

Organism="R.idaeus"
Strain="autumn_bliss"

Fastq1=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n1 | tail -n1)
Fastq2=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n2 | tail -n1)
Fastq3=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n3 | tail -n1)
Fastq4=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n4 | tail -n1)
Fastq5=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n5 | tail -n1)
Fastq6=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n6 | tail -n1)
Size=293M
Prefix=autumn_bliss_v3
OutDir=assembly/flye/$Organism/"$Strain"/$Prefix
mkdir -p $OutDir

CurPath=$PWD
WorkDir=/tmp/flye/$Prefix

# ---------------
# Step 2
# Run Canu
# ---------------

mkdir -p $WorkDir
cd $WorkDir

flye --nano-raw $CurPath/$Fastq1 $CurPath/$Fastq2  $CurPath/$Fastq3 $CurPath/$Fastq4 $CurPath/$Fastq5 $CurPath/$Fastq6 --genome-size $Size --out-dir $Prefix --threads 40


ls $Prefix
cp $WorkDir/*/* $CurPath/$OutDir/.
rm -r $WorkDir

conda deactivate
```

```
Total length:   592682572
Contigs:        4471
Scaffolds:      4463
Scaffolds N50:  285668
Largest scf:    3113992
Mean coverage:  27
```


Quast and busco were run to assess assembly quality:

```bash
cd /data/scratch/armita/rubus
Prefix='autumn_bliss_v3'
for Assembly in $(ls assembly/flye/*/*/*/scaffolds.fasta | grep "$Prefix"); do
  Strain=$(echo $Assembly | rev | cut -f2 -d '/' | rev)
  Organism=$(echo $Assembly | rev | cut -f3 -d '/' | rev)  
  OutDir=$(dirname $Assembly)
  ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/assemblers/assembly_qc/quast
  qsub $ProgDir/sub_quast.sh $Assembly $OutDir
  BuscoDB=$(ls -d /home/groups/harrisonlab/dbBusco/embryophyta_odb10)
  # OutDir=gene_pred/busco/$Organism/$Strain/assembly
  ProgDir=/home/armita/git_repos/emr_repos/tools/gene_prediction/busco
  qsub $ProgDir/sub_busco3.sh $Assembly $BuscoDB $OutDir
done
```


## Diploid assembly using Flye


This work was performed on the new cluster:

```bash
  screen -a
  # ssh compute03
  srun --nodelist compute01 --partition unlimited --pty bash
  cd /data/scratch/armita/rubus
```


```bash
conda activate flye

Organism="R.idaeus"
Strain="autumn_bliss"

Fastq1=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n1 | tail -n1)
Fastq2=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n2 | tail -n1)
Fastq3=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n3 | tail -n1)
Fastq4=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n4 | tail -n1)
Fastq5=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n5 | tail -n1)
Fastq6=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n6 | tail -n1)
Size=586M
Prefix=autumn_bliss_v6
OutDir=assembly/flye/$Organism/"$Strain"/$Prefix
mkdir -p $OutDir

CurPath=$PWD
WorkDir=/tmp/flye/$Prefix

# ---------------
# Step 2
# Run Canu
# ---------------

mkdir -p $WorkDir
cd $WorkDir

flye --nano-raw $CurPath/$Fastq1 $CurPath/$Fastq2  $CurPath/$Fastq3 $CurPath/$Fastq4 $CurPath/$Fastq5 $CurPath/$Fastq6 --genome-size $Size --out-dir $Prefix --threads 40

cp $WorkDir/$Prefix/* $CurPath/$OutDir/.
ls $Prefix

conda deactivate
```


Quast and busco were run to assess quality:

```bash
for Assembly in $(ls assembly/flye/R.idaeus/autumn_bliss/autumn_bliss_v6/scaffolds.fasta); do
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
