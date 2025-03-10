# Quailty control of illumina reads with fastp (v 0.23.1)
fastp (default parameter)

# Genome survey using kmer distribution generated by the Illumina reads with SOAPec (v 2.01) and GenomeScope (v 2.0)
SOAPec -k 17 -s 10 -u 0.2 -t 20 -m 300 -f Pah_illumina_R1_clean.fq Pah_illumina_R2_clean.fq
Rscript genomescope.R -i out_17merFrq.tsv -k 17 -o kmer17

# Genome Assembly Steps, Software Types and Software Parameters
Step 1: Assemble contig sequence
The pacbio platform was used for sequencing, and the amount of third generation data was measured to be 341.51Gb, 
with a coverage depth of 108× (calculated according to the genome size of 3127.74 Mb predicted by SURVEY). 
Using all third generations of data, preliminary assembly was performed using falcon software to obtain the contigs sequence of the genome.

Software: wtdbg
Version: 2.5
Parameters: ND0.20_NL2304_NM200_WS0.05_WE3_TAIL9216
max_depth = 50
node-drop=0.25,0.20
node-len=1024,1536,2048
node-max=200
brute_force=1
kmer=21

Step 2 : Perform three-generation error correction on the genome
Using all the third generations of data, use arrow software to perform third generations of error correction on the contigs sequence of the genome, and get the genome sequence after error correction.
   
Software: ARROW
Version: Smrtlink_8.0
Parameters: Default parameters

Step 3: Second-generation error correction of the genome
Pair end sequencing was performed by Illumina Hiseq sequencing platform, and the total sequencing volume obtained was 140.56 Gb with a coverage depth of 72.44×. Using pilon software, the genome was subjected to second generation error correction.

Software: pilon
Version number: pilon-1.22
Parameters: Default parameters

Step 4: Mount the chromosome
According to the 364.32 Gb Hi-C data obtained from sequencing, the contigs/scaffolds sequences obtained from assembly were uplifted to the chromosome level using all-hic software to obtain the chromosome genome.

Software : all.hic
Parameters :
break=N
CLUSTER=73
filter=yes
MaxLinkDensity=3
longest_=800
enz=mboI
format_=Sanger
NonInformativeRatio=0
shortest_=150
minREs=50
fill=yes

# Genome assessment
## Use BUSCO (v 3.0.2) to check the genome completeness
busco -c 10 --augustus -m genome --offline --download_path busco_downloads -i P_h_homarus_genome.fasta -l arthropoda_odb9 -o busco_augustus
## detect gene orthologs using tblastn, Augustus, and HMMER
tblastn -query protein_query.fasta -db P_h_homarus_genome.fasta -evalue 1e-5 -outfmt 6 -out tblastn_results.txt
augustus --species=species_model --genome=P_h_homarus_genome.fasta --gff3=on > augustus_predictions.gff
hmmer3 -o hmmer_results.txt --tblout hmmer_tblout.txt --cpu 4 --evalue 1e-5 --domtblout hmmer_domtblout.txt protein_domain_model.hmm P_h_homarus_genome.fasta
## Use Merqury to check the QV and genome completeness
meryl k=21 memory=500G count output reads.meryl ../clean_R1.fq.gz ../clean_R2.fq.gz
merqury.sh reads.meryl/ ../P_h_homarus_genome.fasta merqury_output
## Map short DNA reads to genome with bwa (0.7.8)
bwa index Pah.asm.v1.fa
bwa mem -t 10 -R '@RG\tID:Pah_illumina\tSM:Pah_illumina\tPL:illumina' ../P_h_homarus_genome.fasta Pah_illumina_R1_clean.fq.gz Pah_illumina_R2_clean.fq.gz 2>Pah_illumina.bwa.log
samtools sort -@ 10 -m 10G -o Pah_illumina.sort.bam
samtools flagstat -@ 10  Pah_illumina.sort.bam > Pah.bam.flagstat

# Repeat annotation
## Build de novo repeat library 
RepeatScout -genome P_h_homarus_genome.fasta -output RepeatScout_output
cat RepeatScout_output/RepeatScout.fa Repbase.fa > Integrated_repeats.fa
RepeatModeler -database P_h_homarus_genome.fasta -output RepeatModeler_output
## Use RepeatModeler (v2.0.1) to predict transposable element families
RepeatModeler -database P_h_homarus_genome.fasta -output RepeatModeler_output
## Use Piler (1.0) to identify transposable elements
Piler -genome P_h_homarus_genome.fasta -output Piler_output
## Use LTR-FINDER (1.0.6) to predict LTR sequences
LTR-FINDER -seq P_h_homarus_genome.fasta -output LTRFinder_output
## Use RepeatMasker (v4.1.0) to classify repetitive sequences in the genome
RepeatMasker -lib Integrated_repeats.fa P_h_homarus_genome.fasta -gff -dir RepeatMasker_output
## Use RepeatProteinMask (v4.1.0) to process protein-coding repetitive sequences
RepeatProteinMask -lib Integrated_repeats.fa P_h_homarus_genome.fasta -gff -dir RepeatProteinMask_output
## Use TRF (v4.0.9) to identify tandem repeat sequences
TRF P_h_homarus_genome.fasta 2 7 7 80 10 50 500 -d

# Non-coding RNA prediction
## Predict tRNA by tRNAscan-SE (v1.4)
tRNAscan-SE -m stat -o tRNA -f structure --thread 20 P_h_homarus_genome.fasta
## Predict rRNA, miRNA and snRNA by infernal (v 1.0) from Rfam (14.5)
cmscan -Z 1060.323626 --cut_ga --rfam --nohmmonly --tblout my-genome.tblout --fmt 2 --cpu 40 --clanin Rfam.clanin Rfam.cm P_h_homarus_genome.fasta > my-genome.cmscan
grep -v '=' genome.tblout > genome.deoverlapped.tblout
awk 'BEGIN{OFS="\t";}{if(FNR==1) print "target_name\taccession\tquery_name\tquery_start\tquery_end\tstrand\tscore\tEvalue"; if(FNR>2 && $20!="=" && $0!~/^#/) print $2,$3,$4,$10,$11,$12,$17,$18; }' genome.tblout > genome.tblout.final.xls
less rfam_anno.txt | awk 'BEGIN {FS=OFS="\t"}{split($3,x,";");class=x[2];print $1,$2,$3,class}' > rfam_anno.txt
awk 'BEGIN{OFS=FS="\t"} ARGIND==1{a[$2]=$4;} ARGIND==2{type=a[$1];if(type=="") type="Others";count[type]+=1;} END{for(type in count) print type, count[type];}' rfam_anno.txt ./genome.tblout.final.xls
awk 'BEGIN{OFS=FS="\t"}ARGIND==1{a[$2]=$4;}ARGIND==2{type=a[$1]; if(type=="") type="Others"; if($6=="-")len[type]+=$4-$5+1;if($6=="+")len[type]+=$5-$4+1}END{for(type in len) print type, len[type];}' rfam_anno.txt ./genome.tblout.final.xls

# Gene structure anntation
## De novo transcriptome assembly by Trinity (v 2.11.0)
Trinity --no_version_check --seqType fq --max_memory 10G --CPU 10 --left eye_clean_R1.fq.gz,gill_clean_R1.fq.gz,stalk_clean_R1.fq.gz,hemocyte_clean_R1.fq.gz,liver_clean_R1.fq.gz,muscle_clean_R1.fq.gz,intestine_clean_R1.fq.gz  --right eye_clean_R2.fq.gz,gill_clean_R2.fq.gz,stalk_clean_R2.fq.gz,hemocyte_clean_R2.fq.gz,liver_clean_R2.fq.gz,muscle_clean_R2.fq.gz,intestine_clean_R2.fq.gz
## Genome guided transcriptome assembly by Hisat (v 2.2.1), samtools (v 1.12), and Trinity (v 2.12.0)
hisat2-build P_h_homarus_genome.fasta P_h_homarus_genome.fasta -p 20
hisat2 -p 40 --dta -x P_h_homarus_genome.fasta -1 eye_clean_R1.fq.gz,gill_clean_R1.fq.gz,stalk_clean_R1.fq.gz,hemocyte_clean_R1.fq.gz,liver_clean_R1.fq.gz,muscle_clean_R1.fq.gz,intestine_clean_R1.fq.gz -2 eye_clean_R2.fq.gz,gill_clean_R2.fq.gz,stalk_clean_R2.fq.gz,hemocyte_clean_R2.fq.gz,liver_clean_R2.fq.gz,muscle_clean_R2.fq.gz,intestine_clean_R2.fq.gz
samtools view -bhS hisat2_GG.sam -@ 20 -o hisat2_GG.bam | samtools sort - -@ 20 -m 10G -o hisat2_GG.sorted.bam && samtools index hisat2_GG.sorted.bam
Trinity --no_version_check --genome_guided_bam hisat2_GG.sorted.bam --max_memory 10G --CPU 10 --genome_guided_max_intron 20000 --output Trinity_GG
## Build a comprehensive transcriptome database by PASA (v 2.1.0) , combining the denovo assembly and genome guided assembly
cat Trinity.fasta Trinity-GG.fasta > Trinity_all_new.fasta
seqclean Trinity.fasta -v UniVec -c 10
seqclean Trinity_all_new.fasta -v UniVec -c 10
accession_extractor.pl < Trinity.fasta.clean >tdn.accs
Launch_PASA_pipeline.pl -c alignAssembly.config -C -R -g Pha.asm.v1.softmask.fa -t Trinity_all_new.fasta.clean -T -u Trinity_all_new.fasta --TDN tdn.accs --ALIGNERS gmap,blat --CPU 2 --stringent_alignment_overlap 30.0 --gene_overlap 50.0
build_comprehensive_transcriptome.dbi -c alignAssembly.config -t Trinity_all_new.fasta.clean --min_per_ID 95 --min_per_aligned 30

# Functional annotation
## Diamond Blast (v0.88.2) the whole NCBI NR database (Release 2021_9_29)
diamond makedb --in nr.fa.gz -d nr
diamond blastp -e 1e-5 -k 1 -f 6 -p 15 -d nr -q Pah.longest.gene.pep.fasta -o Pah.nr.diamond.result
## Blast the SWISS-PROT database (Release 2022_01)
makeblastdb -in uniprot_sprot.fasta -dbtype prot -out uniprot_sprot.db -parse_seqids
blastp -query Pah.longest.gene.pep.fasta -db uniprot_sprot.db -out Pah.swissprot.blastp.result -outfmt "6 qseqid sseqid evalue bitscore stitle" -evalue 1e-5 -num_threads 30
## Protein domain annotation by interproscan (v 5.52-86.0) and pfam_scan.pl
interproscan.sh -i Pah.longest.gene.pep.fasta -cpu 20 -goterms -iprlookup -pa
pfam_scan.pl -e_seq 0.05 -e_dom 0.05 -cpu 20 -fasta Pah.longest.gene.pep.fasta -dir Pfam35 -outfile Pah.pfam
