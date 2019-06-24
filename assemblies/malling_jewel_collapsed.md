Commands used to generate a collapsed assembly for raspberry cv. malling jewel:


```bash
WorkDir=/data/scratch/armita/rubus
mkdir -p $WorkDir
cd $WorkDir
```

```bash
WorkDir=/data/scratch/armita/rubus
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
for RawData in $(ls ../../../../../data/scratch/armita/rubus/qc_dna/minion/R.idaeus/malling_jewel/*.fastq.gz); do
echo $RawData;
ProgDir=/home/armita/git_repos/emr_repos/tools/seq_tools/dna_qc;
GenomeSz=293
# OutDir=$(dirname $RawData)
OutDir=$(echo $RawData | cut -f10,11,12,13 -d '/')
mkdir -p $OutDir
qsub $ProgDir/sub_count_nuc.sh $GenomeSz $RawData $OutDir
done
```

<!--
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
-->

MinION coverage

```

```

### Read correction using Canu


This work was performed on the new cluster:

```bash
  screen -a
  ssh compute03
  cd /oldhpc/data/scratch/armita/rubus

```

```bash
Organism="R.idaeus"
Strain="malling_jewel"

Fastq1=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n1 | tail -n1)
Fastq2=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n2 | tail -n1)
Fastq3=$(ls qc_dna/minion/*/*/*q.gz | grep 'autumn_bliss' | head -n3 | tail -n1)
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
