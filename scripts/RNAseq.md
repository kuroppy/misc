# RNA-seq解析計算例

本資料では、テキストで省略されていた解析部分の実行例を紹介する。

1. データ取得
2. Quality Control (QC)
3. リードマッピング
4. 発現量カウント

# 0. 仮想環境の準備

### conda環境の作成

``` bash
conda create -n rnaseq
```

-   `-n rnaseq` : 環境名を `rnaseq` に設定

### 環境の有効化

``` bash
conda activate rnaseq
```

### 必要なソフトウェアのインストール

``` bash
conda install \
  -c conda-forge \
  -c bioconda \
  sra-tools fastqc seqkit trimmomatic hisat2 samtools subread
```

-   `-c` : 使用するチャンネルを指定


# 1. データ取得

### ゲノム配列（FASTA）

```bash
wget -P GCF_000146045.2 \
    https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/146/045/GCF_000146045.2_R64/
    GCF_000146045.2_R64_genomic.fna.gz
```

オプション解説

| option | 意味 |
|---|---|
| `-P` | 保存先ディレクトリ |

### アノテーションファイル（GFF）

```bash
wget -P GCF_000146045.2 \
    https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/146/045/GCF_000146045.2_R64/
    GCF_000146045.2_R64_genomic.gff.gz

gunzip GCF_000146045.2/GCF_000146045.2_R64_genomic.gff.gz
```

### RNA-seq FASTQデータ取得

``` bash
fasterq-dump \
  SRR453566 \
  -O fastq \
  -p
```

-   `-O` : 出力先ディレクトリ
-   `-p` : 進捗表示


> ※ENAから直接ダウンロードする方が高速です。
> 
> ``` bash
> wget -P fastq \
>   ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR453/SRR453566/SRR453566_1.fastq.gz
> 
> wget -P fastq \
>   ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR453/SRR453566/SRR453566_2.fastq.gz
> ```

### FASTQ統計情報確認

```bash
seqkit stats fastq/SRR453566_*
```

確認ポイント

- read数
- read長
- GC含量
- ファイルサイズ


# 2. Quality Control (QC)

### FastQCによる品質確認

```bash
fastqc \
    -t 8 \
    -o fastqc/raw \
    fastq/SRR453566_1.fastq.gz \
    fastq/SRR453566_2.fastq.gz
```

オプション解説

| option | 意味 |
|---|---|
| `-t 8` | 使用スレッド数 |
| `-o` | 出力ディレクトリ |

### Trimmomatic による quality filtering

```bash
trimmomatic PE \
    -threads 8 \
    -phred33 \
    fastq/SRR453566_1.fastq.gz \
    fastq/SRR453566_2.fastq.gz \
    trimmed/SRR453566_1_trimmed.fastq.gz \
    trimmed/SRR453566_1_unpaired.fastq.gz \
    trimmed/SRR453566_2_trimmed.fastq.gz \
    trimmed/SRR453566_2_unpaired.fastq.gz \
    ILLUMINACLIP:TruSeq3-PE-2.fa:2:30:10:1:keepBothReads \
    LEADING:3 \
    TRAILING:3 \
    SLIDINGWINDOW:4:20 \
    MINLEN:36
```

オプション解説

| option | 意味 |
|---|---|
| `PE` | paired-end mode |
| `-threads 8` | 使用スレッド数 |
| `-phred33` | quality score形式 |
| `ILLUMINACLIP` | adapter除去 |
| `LEADING:3` | 先頭Q<3除去 |
| `TRAILING:3` | 末尾Q<3除去 |
| `SLIDINGWINDOW:4:20` | 4bp窓で平均Q<20なら切断 |
| `MINLEN:36` | 36bp未満除去 |

### trimming後 FastQC

```bash
fastqc \
    -t 8 \
    -o fastqc/trimmed \
    trimmed/SRR453566_1_trimmed.fastq.gz \
    trimmed/SRR453566_2_trimmed.fastq.gz
```


# 3. Mapping

### HISAT2 index作成

```bash
hisat2-build \
    -p 8 \
    GCF_000146045.2/GCF_000146045.2_R64_genomic.fna \
    hisat2/index/genome
```
オプション解説

| option | 意味 |
|---|---|
| `-p 8` | 使用スレッド数 |

### HISAT2 mapping

```bash
hisat2 \
    -p 8 \
    -x hisat2/index/genome \
    -1 trimmed/SRR453566_1_trimmed.fastq.gz \
    -2 trimmed/SRR453566_2_trimmed.fastq.gz \
    -S hisat2/SRR453566.sam
```

オプション解説

| option | 意味 |
|---|---|
| `-p 8` | 使用スレッド数 |
| `-x` | HISAT2 index prefix |
| `-1` | Read1 FASTQ |
| `-2` | Read2 FASTQ |
| `-S` | 出力SAMファイル |

### SAM → sorted BAM

```bash
samtools sort \
    hisat2/SRR453566.sam \
    -@ 8 \
    -o hisat2/SRR453566.sorted.bam
```

オプション解説

| option | 意味 |
|---|---|
| `-@ 8` | 使用スレッド数 |
| `-o` | 出力BAM |

### BAM indexing

```bash
samtools index \
    hisat2/SRR453566.sorted.bam
```


# 4. 発現量カウント

### featureCounts

```bash
featureCounts \
    -T 8 \
    -p \
    --countReadPairs \
    -t exon \
    -g locus_tag \
    -a GCF_000146045.2/GCF_000146045.2_R64_genomic.gff \
    -F GFF \
    -o featureCounts/counts.txt \
    hisat2/SRR453566.sorted.bam
```

オプション解説

| option | 意味 |
|---|---|
| `-T 8` | 使用スレッド数 |
| `-p` | paired-endとして扱う |
| `--countReadPairs` | read pair単位でカウント |
| `-t exon` | exon featureを集計 |
| `-g locus_tag` | 遺伝子IDとしてlocus_tag使用 |
| `-a` | annotation file |
| `-F GFF` | annotation形式 |
| `-o` | 出力ファイル |

