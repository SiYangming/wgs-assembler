# Celera Assembler 8.3rc2 安装说明

本仓库归档 Celera Assembler（wgs-assembler，又称 CABOG）8.3rc2 的发行包内容。

仓库根目录保留上游 tar 包解压后的 `wgs-8.3rc2/` 一层目录，**未做扁平化**——上游 `README` 中的编译与运行命令写死了这一层路径。

`wgs-8.3rc2/README` 与 `wgs-8.3rc2/LICENSE.txt` 为上游原件，未作任何修改。

## 1. 目录结构

```
.
├── INSTALL.md
├── Example.md
└── wgs-8.3rc2/
    ├── LICENSE.txt
    ├── README
    ├── kmer/      # kmer 包（r1994）子集，编译后提供 meryl 等
    └── src/       # Celera Assembler 源码
```

## 2. 获取方式

本仓库 Release `wgs-8.3rc2` 提供以下附件：

| 附件 | 说明 | 大小（字节） | SHA256 |
| ---- | ---- | ---- | ---- |
| `wgs-8.3rc2.tar.bz2` | 官方源码包（源码 revision 4627，含 Makefile 与脚本） | 24603412 | `6ba1711ebe56629b670be87ae040ac948057474b3959666a88f0862c6a40f27b` |
| `wgs-8.3rc2-Linux_amd64.tar.bz2` | 官方 Linux x86_64 预编译包（解压即用，无需编译） | 34608895 | `234150f9948d1d279605bc82e32ab65b1fe9c7b5b1e2fccbcf333ecbe5a16180` |
| `PacBioToCA_sampleData.tar.gz` | PBcR 示例数据（λ 噬菌体） | 2596604 | — |

> 两种安装包解压后都会得到名为 `wgs-8.3rc2/` 的目录，请分别解压到不同位置，避免相互覆盖。

也可直接自上游 SourceForge 文件区获取：
<https://sourceforge.net/projects/wgs-assembler/files/wgs-assembler/wgs-8.3/>

## 3. 从源码编译

与上游 `README` 给出的步骤一致：

```bash
bzip2 -dc wgs-8.3rc2.tar.bz2 | tar -xf -
cd wgs-8.3rc2
cd kmer && make install && cd ..
cd src  && make            && cd ..
cd ..
```

若需安装到指定位置：

```bash
mkdir -p /path/to/install
tar xjf wgs-8.3rc2.tar.bz2 -C /path/to/install/
cd /path/to/install/wgs-8.3rc2/kmer && make install && cd ../src && make && cd ..
```

编译产物位于 `wgs-8.3rc2/Linux-amd64/bin/`。

## 4. 使用预编译包

```bash
mkdir -p /path/to/install
tar xjf wgs-8.3rc2-Linux_amd64.tar.bz2 -C /path/to/install/
```

解压后即可直接使用，无需编译；可执行文件位于 `/path/to/install/wgs-8.3rc2/Linux-amd64/bin/`。

## 5. 配置 PATH

源码编译与预编译包的可执行文件目录相同：

```bash
export PATH=/path/to/install/wgs-8.3rc2/Linux-amd64/bin:$PATH
```

按上游 `README` 的说明，组装入口为：

```bash
wgs-8.3rc2/*/bin/runCA
```

主要可执行文件包括 `runCA`、`fastqToCA`、`PBcR`、`meryl` 等。

## 6. 依赖

- 源码编译需要 C / C++ 编译器与 `make`。
- PBcR 等 Perl 驱动脚本依赖 `Statistics::Descriptive` 模块：

```bash
sudo cpan -i Statistics::Descriptive
# Debian / Ubuntu 亦可：sudo apt-get install libstatistics-descriptive-perl
```

- 本发行包**自带**上游 README 列出的以下组件，无需另行安装：SAMtools、Jellyfish 2.0、PBUTGCNS、PBDAGCON、BLASR，以及部分 FALCON；源码包还包含 kmer 包（版本 r1994）的一个子集。

## 7. 许可与引用

- 许可：GNU General Public License, version 2（GPLv2）。Copyright 1999–2004 Applera Corporation；Copyright 2005–2013 J. Craig Venter Institute。详见 `wgs-8.3rc2/LICENSE.txt`。
- 引用：简述算法或使用其输出时，请引用 Myers et al. (2000) *A Whole-Genome Assembly of Drosophila*, Science 287:2196–2204；PacBio 相关流程另见 Berlin et al. (2014)。完整引用列表见 `wgs-8.3rc2/README`。
- 上游项目主页：<http://wgs-assembler.sourceforge.net/>

## 8. 使用示例

PacBioToCA / PBcR 的纠错示例（含上游示例数据的两条流程）见 [`Example.md`](./Example.md)。

相关项目：

- 独立的 BLASR 用法见 <https://github.com/SiYangming/blasr>
- FALCON / pb-assembly 见 <https://github.com/SiYangming/pb-assembly>
