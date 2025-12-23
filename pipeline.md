# Pipeline ATAC-Seq

+ 00 FastQ Screen 
```bash
$ LD_LIBRARY_PATH="" ./00_fastqscreen.sh ../directorio_raw_data
```
Detecta contaminación mapeando contra múltiples genomas. En ATAC-seq es importante verificar que no haya exceso de DNA mitocondrial o contaminación bacteriana.

+ 01 FastQC
```bash
$ LD_LIBRARY_PATH="" ./01_fastqc.sh ../directorio_raw_data
```
Control de calidad inicial. En ATAC-seq se espera ver un patrón periódico en la distribución de tamaños de inserto (~200bp) correspondiente al espaciado nucleosomal.

+ 02 Trim Galore
```bash
$ LD_LIBRARY_PATH="" ./02_trimgalore_parallel.sh ../directorio_raw_data
```
Elimina adaptadores y bases de baja calidad. Crítico en ATAC-seq porque los fragmentos nucleosome-free son cortos y pueden contener secuencia de adaptador.

+ 03 Bowtie2
```bash
$conda activate samtools
$nohup bowtie2 -X 2000 --very-sensitive --no-discordant --no-mixed \
  -q --phred33 -p 30 \
  -x /home/resources/genomes/Genomes/hg38_ENCODEv3/bowtie2_index/hg38 \
  -1 1hCFC_n1_R1_val_1.fq.gz -2 1hCFC_n1_R2_val_2.fq.gz \
  -S 1hCFC_n1_1.sam > bowtie2_1.log &
```
Alinea las lecturas con parámetros específicos para ATAC-seq: -X 2000 permite fragmentos grandes (dinucleosomas), --very-sensitive maximiza la precisión, y --no-discordant --no-mixed asegura solo pares concordantes para mejor estimación de tamaño de fragmento.

+ 04 Samtools
```bash
$ conda activate samtools
$ ./04_samtools_filter_chrM_grep.sh ../outputs/03_bowtie2
```
Convierte SAM a BAM, elimina los chromosomas y solo mantiene los cromosomas principales, eliminando los mitocondriales (chrM que suelen representar 30-80% en ATAC-seq), ordena por coordenadas e indexa. Prepara los archivos para el filtrado posterior.

+ 05 Mark duplicates
```bash
$ ./05_markdups_parallel.sh ../outputs/05_markdups 
```
Marca duplicados de PCR. En ATAC-seq la tasa de duplicados puede ser alta debido a la amplificación del DNA tagmentado; es importante monitorear este porcentaje como métrica de calidad.

+ 06 Filter Bam
```bash
$ ../06_filterbam_parallel.sh ../outputs/05_markdups
```
Filtra lecturas no deseadas: duplicados, no mapeadas y regiones artefactuales.

+ 07 MACS2 Peaks calling
```bash
$ ./07_macs2_parallel.sh ../outputs/06_filterbam
```
Detecta regiones de cromatina accesible (picos). A diferencia de ChIP-seq, no usa input/control ya que ATAC-seq mide accesibilidad directamente. Opcionalmente usar --nomodel --shift -100 --extsize 200 para centrar los picos en el sitio de corte de la transposasa Tn5.

+ Generación de Tracks (BigWig)
```bash
#Obtener número de reads para factor de escala -S
$ samtools flagstat ../outputs/06_filterbam/muestra.bam -@ 10

# Normalizar señal (sin control en ATAC-seq, usar fold enrichment o señal directa)
macs2 bdgcmp -t sample_treat_pileup.bdg -c sample_control_lambda.bdg \
  -m ppois -p 1.0 -S 30.0 -o sample_ppois.bedGraph

# Ordenar bedGraph
$ LC_COLLATE=C sort -k1,1 -k2,2n sample_ppois.bedGraph > sample_ppois.sorted.bedGraph

# Convertir a BigWig
$ bedGraphToBigWig sample_ppois.sorted.bedGraph \
  /home/resources/genomes/Genomes/chrom_sizes/hg38.chrom.sizes sample_ppois.bw
```
Genera tracks normalizados para visualización. El factor -S escala por profundidad de secuenciación para comparar entre muestras. Los tracks de ATAC-seq muestran picos en promotores, enhancers y otras regiones regulatorias accesibles.




  
