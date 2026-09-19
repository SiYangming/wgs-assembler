# PacBioToCA / PBcR 使用示例

Celera Assembler 提供 PacBioToCA 流程（命令行入口为 `PBcR`），用于：

- **hybrid 纠错**：以高保真短读（Illumina）为 trusted 序列，对 PacBio 长读做校正；
- **自纠错**：在没有短读时，用高覆盖 PacBio 长读自身做校正（需 PacBio C2 及以上测序化学、约 50X 覆盖度）。

校正产物为 `.fasta` / `.qual` / `.fastq` 与 `.frg`，其中 `.frg` 可直接交给 `runCA` 做后续组装。

本文命令中的 `path/to/` 与 `/path/to/install/` 均为占位路径，请按实际环境替换。

## 1. 环境准备

将可执行文件目录加入 `PATH`（源码编译或预编译包解压后的路径见 `INSTALL.md`）：

```bash
export PATH=/path/to/install/wgs-8.3rc2/Linux-amd64/bin:$PATH
```

PBcR 的 Perl 驱动脚本依赖 `Statistics::Descriptive` 模块：

```bash
sudo cpan -i Statistics::Descriptive
```

## 2. hybrid 纠错（以 Illumina 短读校正 PacBio 长读）

以约 4.6 Mb 基因组为例：

```bash
mkdir -p path/to/pacbiotoca
cd path/to/pacbiotoca

# 仅做纠错、不做组装
echo "assemble=0" > pacbio.spec

# Illumina 双端 reads → Celera .frg 文库
#   -insertsize <均值> <标准差>  插入片段长度期望与标准差（此处 177 bp ± 25 bp）
#   -libraryname                 文库 UID 名
#   -mates read1,read2           双端 reads（逗号分隔，不可有空格）
fastqToCA -insertsize 177 25 -libraryname pe150 \
    -mates illumina.1.fastq,illumina.2.fastq > illumina.frg

# PBcR hybrid 纠错
#   -libraryname   本次纠错的文库 UID 名
#   -s             规格文件（spec），其中 assemble=0 表示仅纠错不组装
#   -fastq         待校正的 PacBio 长读
#   -genomeSize    期望基因组大小（bp）
#   -maxCoverage   截断覆盖度
#   末尾的 illumina.frg 为位置参数（trusted 高保真序列）
PBcR -libraryname E_coli_pacbio -s pacbio.spec \
    -fastq path/to/pacbio_subreads.fastq \
    -genomeSize 4600000 -maxCoverage 40 illumina.frg &> pacbiotoca.log
```

实测耗时（约 4.6 Mb 基因组）：`real 141m26.718s` / `user 489m7.042s` / `sys 24m10.249s`。

产物：

| 文件 | 说明 |
| ---- | ---- |
| `<libraryname>.frg` | 单 LIB 消息，可直接喂给 `runCA` |
| `<libraryname>.fasta` / `.qual` / `.fastq` | 校正后的 reads |
| `<libraryname>.log` | 每条校正序列的来源与坐标 |

## 3. 上游示例数据（自纠错 + Illumina 纠错两条链）

本仓库 Release 附件 `PacBioToCA_sampleData.tar.gz` 即上游提供的 PBcR 示例数据包（λ 噬菌体）。解压：

```bash
tar zxf PacBioToCA_sampleData.tar.gz
cd sampleData
```

### 3.1 自纠错（仅使用 PacBio 数据）

示例数据中的 PacBio reads 为 `.fasta`（含质量信息），先用附带工具转为 FASTQ：

```bash
echo "assemble=0" > pacbio.spec

java -jar convertFastaAndQualToFastq.jar pacbio.filtered_subreads.fasta > pacbio.filtered_subreads.fastq

PBcR -length 500 -partitions 200 -l lambda -s pacbio.spec \
    -fastq pacbio.filtered_subreads.fastq genomeSize=50000 &> lambda_pacbiotoca.log
```

- `-length 500`：保留的最小 PacBio 片段长度
- `-partitions 200`：consensus 分区数
- `genomeSize=50000`：λ 噬菌体基因组约 50 kb（此处以 `key=value` 形式写在命令行）

实测耗时：`real 1m46.910s` / `user 4m9.361s` / `sys 0m38.952s`。

### 3.2 使用 Illumina 数据进行纠错

```bash
# Illumina 单端 reads → .frg
#   -technology illumina  测序平台
#   -type sanger          读长类型（示例数据为较长读长）
#   -innie                内向配对
#   -reads                单端 reads
fastqToCA -libraryname illumina -technology illumina -type sanger -innie \
    -reads illumina.fastq > illumina.frg

java -jar convertFastaAndQualToFastq.jar pacbio.filtered_subreads.fasta > pacbio.filtered_subreads.fastq

PBcR -length 500 -partitions 200 -l lambdaIll -s pacbio.spec \
    -fastq pacbio.filtered_subreads.fastq genomeSize=50000 illumina.frg > lambdaIll_pacbiotoca.log 2>&1
```

实测耗时：`real 2m13.707s` / `user 5m11.338s` / `sys 0m20.958s`。

## 4. 常用参数说明

| 参数 | 说明 |
| ---- | ---- |
| `-libraryname`（或 `-l`） | 文库 UID 名，不可含空格 / 逗号，必填 |
| `-s <spec>` | Celera 规格文件，必填；`assemble=0` 表示仅纠错不组装 |
| `-fastq <reads>` | 待校正的 PacBio 长读（FASTA / FASTQ） |
| `-genomeSize` | 期望基因组大小（bp），也可写入 spec |
| `-maxCoverage` | 截断覆盖度（默认校正最长 40X），也可写入 spec |
| `-length` | 保留的最小 PacBio 片段长度 |
| `-partitions` | consensus 分区数 |
| 位置参数 `.frg` | 高保真（Illumina / 454 / CCS）frg；省略则进入自纠错模式 |

## 5. 注意事项

- **命令行形态随版本变化**：本文采用 `-libraryname / -fastq / -genomeSize / -maxCoverage` 加位置参数 `.frg` 的写法；官方 wiki 中 8.3 的用法为 `PBcR [options] -s spec.file fastqFile=... [frg]`。复现时请与所用文档版本保持一致。
- **自纠错条件**：需要 PacBio C2 及以上测序化学，且覆盖度约 50X。
- **后续组装**：纠错产出的 `<libraryname>.frg` 可直接作为 `runCA` 的输入。

## 6. 相关项目

- 独立的 BLASR 用法见 <https://github.com/SiYangming/blasr>
- FALCON / pb-assembly 见 <https://github.com/SiYangming/pb-assembly>

> 本发行包内自带 SAMtools、Jellyfish 2.0、PBUTGCNS、PBDAGCON、BLASR 及部分 FALCON（详见 `wgs-8.3rc2/README`）。
