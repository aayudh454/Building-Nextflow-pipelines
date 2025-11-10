**Nextflow Pipeline CRISPR ON-target: Trimming → Alignment → Sorting/Indexing**

Save as: main.nf
```
#!/usr/bin/env nextflow

nextflow.enable.dsl=2

process TRIMMING {
    tag "${sample_id}"
    publishDir "results/trimmed", mode: 'copy'

    input:
    tuple val(sample_id), path(reads)

    output:
    tuple val(sample_id), path("clipped_${reads[0].name}"), path("clipped_${reads[1].name}")

    script:
    """
    fastp --detect_adapter_for_pe \
          -W 4 \
          -M 20 \
          --cut_tail \
          --length_required 15 \
          -i ${reads[0]} \
          -I ${reads[1]} \
          -o clipped_${reads[0].name} \
          -O clipped_${reads[1].name}
    """
}

process ALIGNMENT {
    tag "${sample_id}"
    publishDir "results/aligned", mode: 'copy'

    input:
    tuple val(sample_id), path(read1), path(read2)
    path reference

    output:
    tuple val(sample_id), path("${sample_id}.bam")

    script:
    """
    bwa mem -t 8 \
        -R "@RG\\tID:${sample_id}\\tSM:${sample_id}\\tLB:lib1\\tPL:illumina" \
        -A 1 -B 4 -O 6,6 -E 1,1 -L 5,5 -U 17 \
        ${reference} ${read1} ${read2} > ${sample_id}.bam
    """
}

process SORT_INDEX {
    tag "${sample_id}"
    publishDir "results/sorted", mode: 'copy'

    input:
    tuple val(sample_id), path(bam_file)

    output:
    tuple val(sample_id), path("${sample_id}.sorted.bam"), path("${sample_id}.sorted.bam.bai")

    script:
    """
    samtools sort ${bam_file} -o ${sample_id}.sorted.bam
    samtools index ${sample_id}.sorted.bam
    """
}

workflow {
    samples = [
        tuple("edited", ["HC160014_S0_R1_001.fastq.gz", "HC160014_S0_R2_001.fastq.gz"]),
        tuple("control", ["HC160015_S1_R1_001.fastq.gz", "HC160015_S1_R2_001.fastq.gz"])
    ]

    reference = file("GRCh38.p12.fa")

    trimmed = TRIMMING(samples)
    aligned = ALIGNMENT(trimmed, reference)
    SORT_INDEX(aligned)
}

```
**Directory Structure Example**

```
project/
├── main.nf
├── GRCh38.p12.fa
├── HC160014_S0_R1_001.fastq.gz
├── HC160014_S0_R2_001.fastq.gz
├── HC160015_S1_R1_001.fastq.gz
├── HC160015_S1_R2_001.fastq.gz
└── results/
    ├── trimmed/
    ├── aligned/
    └── sorted/
```
**Run Command**

```
nextflow run main.nf -resume

```
