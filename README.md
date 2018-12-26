# Rotation-project-CNV_BP-analysis

## 1 prostatic数据准备

学会建立前列腺癌wes数据软链接：
```
tail -n +2 /public/home/liuxs/biodata/gdc/wes/gdc_manifest.2018-10-17.txt |\
head -n 5| \
awk 'BEGIN{OFS="\t";}{info=$2; gsub(/\.bam/,"",info); \
system("ln -s /public/home/liuxs/biodata/gdc/wes/"$1"/"info".bam /public/home/liuxs/biodata/gdc/test-link/"info".bam && \
ln -s /public/home/liuxs/biodata/gdc/wes/"$1"/"info".bai /public/home/liuxs/biodata/gdc/test-link/"info".bai")}'
```

### 1.1 表观数据
Roadmap 数据库暂无前列腺组织表观数据，用前列腺癌组织的Chip-Seq替代，包括：H3K27ac 92个样本，H3K4me3 56个样本， 82个样本
- liftOver
Chip-Seq数据参考基因组为hg19，我们所有的分析是建立在hg38参考基因组上的，因此需要利用liftOver工具对bed文件进行转换。
安装地址：`http://hgdownload.cse.ucsc.edu/admin/exe/linux.x86_64/`\
转换需要一个坐标注释文件:`http://hgdownload.cse.ucsc.edu/goldenpath/hg19/liftOver/hg19ToHg38.over.chain.gz`\
Command：`liftOver <your.hg19.bed> hg19ToHg38.over.chain <out.hg38.bed> <out.hg38.unmap>`

### 1.2 CPG Island 数据
UCSC genome browser ---> table browser 下载hg19的CPG岛数据（暂无hg38），输出为bed格式。\
同样需要liftOver工具转换坐标。

### 1.3 GC含量数据
尚未下载，需要将原始数据（没5bpGC含量）重新计算为每1M bp。

## 2 Matchclips 
Matchclips2：基于long soft clips的CNV断点计算方法 ,安装及说明：`https://github.com/yhwu/matchclips2`\
运行命令：
```
for sample in /$PATH/*.bam;do matchclips -t $cpu -f $ref -b $sample -o /public/home/zhangjing1/Documents/prostatic_CNV_BP/`basename $sample`".mc";done
```
- 结果的简单处理：
  bam文件建立索引：
  ```$cat bamf.txt

  1.bam
  2.bam
  ...
  1000.bam
  ```
  cnv断点结果文件建立索引：
  ```
  cat bamf.txt | awk '{print $1".mc"}' > cnvf.txt
  ```
  计算CNV断点样本的overlap：
  ```
  cnvtable -L 10000 -cnvf cnvf.txt -O 0.5 -o overlap.txt
  ```

## 3 ANNOVAR
利用ANNOVAR脚本可以对断点文件进行注释

数据库导入：
```
Perl annotate_variation.pl -buildver hg38 -downdb -webfrom annovar refGene humandb/
# -buildver 表示version
# -downdb 下载数据库的指令
# -webfrom annovar 从annovar提供的镜像下载，不加此参数将寻找数据库本身的源
# humandb/ 存放于humandb/目录下
```
所有数据下载见:`http://annovar.openbioinformatics.org/en/latest/user-guide/download/`\
ANNOVAR输入格式
```
ANNOVAR使用.avinput格式，如以上代码所示，该格式每列以tab分割，最重要的地方为前5列，分别是:

1. 染色体(Chromosome)

2. 起始位置(Start)

3. 结束位置(End)

4. 参考等位基因(Reference Allele)

5. 替代等位基因(Alternative Allele)

6. 剩下为注释部分（可选）。

ANNOVAR主要也是依靠这5处信息对数据库进行比对，进而注释变异。
```
脚本运行示例如下：
```
for CNV in /public/home/zhangjing1/Documents/prostatic_CNV_BP/*.mc;\
do awk '{OFS="\t"; print $1,$2,$3,0,0,$0}' $CNV > $CNV.anno_input;\
perl $ANNOVAR/table_annovar.pl $CNV.anno_input $ANNOVAR/humandb/ -buildver hg38 -out $CNV.anno -remove -protocol refGene,phastConsElements100way,genomicSuperDups,esp6500siv2_all,1000g2015aug_all,avsnp150,ljb26_all -operation g,r,r,f,f,f,f -nastring NA -csvout;\
perl /public/home/zhangjing1/software/matchclips2/src/append_anno.pl $CNV $CNV.anno.hg38_multianno.csv > /public/home/zhangjing1/Documents/prostatic_CNV_BP/annotation/`basename $CNV`".anno";\
rm $CNV.anno_input $CNV.anno.hg38*;done
```
输出信息格式：`https://brb.nci.nih.gov/seqtools/colexpanno.html`
- 其中CNV断点所处的基因座位是比较重要的信息
## 4 prostatic CNV breakpoint 相关性分析
### 4.1 数据预处理
- 将所有结果合并为一个文件，并在最后加入ID列。
### 4.2 RStudio
VSHunter:::cnv_getLengthFraction 计算CNV长度与所在臂的比例
- 计算比例绘成直方图\
`> hist(subset(result,fraction<=0.001)$fraction,breaks = 200)`

- 表观数据与断点的关联分析\
  已知断点（5'，3'），统计其在表观注释区间出现频数统计\
  已知表观修饰区间，统计其内出现断点（5', 3'）的频数统计\
  计算每1M区间内断点（5',3'）出现频数与表观修饰的相关性
- CPG岛与断点的关联分析\
  计算每1M区间内断点（5',3'）出现频数与CPG岛数目的相关性
    
  

