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

✅ Common Nextflow Problems & How to Troubleshoot Them
1. A process fails with an error (script crashes, missing tool, bad input)
Symptoms

Red failed step in execution

.command.err shows error like “command not found” or tool not installed

.command.exit has non-zero exit code

How to troubleshoot

✅ Check logs in the process working directory:
```
work/<hash>/.command.log
work/<hash>/.command.err
work/<hash>/.command.out
```

✅ Verify required binaries or container contain the tool
✅ Check input format and required parameters
✅ Test the command manually inside container:

```
nextflow run ... -with-docker
docker run <container> bash
```

2. Pipeline does not resume even when using -resume
Why it happens

Input files changed

Process script changed

Container or environment version changed

Work directory deleted or modified

Fix

✅ Make sure work directory and .nextflow/ folder are intact
✅ Confirm container / tool versions are the same
✅ Avoid editing outputs manually
✅ If re-running from scratch:

```
nextflow run main.nf -resume
```

3. Channels do not emit data / pipeline stops early
Symptoms

“No data was emitted by channel …”

Processes never trigger

Causes

Wrong file pattern: *.fq.gz instead of *.fastq.gz

Empty directory

Typo in file path

Using Channel.fromPath() without checkIfExists: true

Fix

✅ Print channel contents:

Channel.fromPath('*.fastq.gz').view()


✅ Check path and file extensions
✅ Use absolute paths to avoid relative path issues

4. Processes stuck waiting forever (deadlock or no inputs)
Why

A downstream process waits on two channels, but one never gets data

join, zip, combine used incorrectly

Mismatched tuple structure

Fix

✅ Print channel output shapes:

ch_samples.view()


✅ Use trace or .debug() to inspect flow
✅ Ensure all channels emit same number of items when pairing

5. Output files missing or not published
Symptoms

Job runs but results folder is empty

Work directory has outputs but publishDir didn’t trigger

Causes

Output pattern mismatch:

output: file '*.bam'


but pipeline produces .sam

Using publishDir without mode 'copy'

Fix

✅ Check .command.out to confirm file created
✅ Fix output glob pattern
✅ Add:

publishDir 'results', mode: 'copy'

6. Container errors (Docker/Singularity not working)
Symptoms

“container not found”

Tool not installed inside container

Permissions / mount issues

Fix

✅ Test container manually:

docker run -it <image> bash


✅ Ensure container has required tools
✅ Check user permissions on HPC (Singularity often needed)
✅ Add correct profile:

-profile docker
-profile singularity

7. Resource problems (job killed, oom, walltime exceeded)
Symptoms

SLURM or Kubernetes kills job

“Out of memory”

“Killed” in .command.err

Fix

✅ Increase resources in process:

cpus 16
memory '64 GB'
time '24h'


✅ For HPC: update executor.queueSize or cluster config
✅ Monitor files in .command.log

8. File path issues on HPC vs local
Typical cause

Relative paths behave differently on remote nodes

Missing permissions

Fix

✅ Use absolute paths
✅ Use stageInMode = 'copy' if shared filesystem is unstable
✅ Ensure user has permissions for dirs

9. Groovy syntax problems (common in DSL2)
Examples

Missing commas or parentheses

Wrong indentation

Wrong variable scope

Fix

✅ Run syntax only:

nextflow run main.nf -stub-run


✅ Start with minimal script, add complexity step-by-step
✅ Use println debugging inside workflow

10. Large pipelines run slow
Reasons

Small executor queue

Too many serial processes

Channel combining causing bottlenecks

Fix

✅ Increase executor.queueSize
✅ Parallelize input with Channel.fromPath().splitCsv() etc.
✅ Use cloud executors for scale (AWS Batch, Google Life Sciences)

✅ Summary Table
Problem	How to Troubleshoot
Process fails	Check .command.err, .command.out, test container manually
Resume doesn’t work	Ensure work folder intact, inputs unchanged
Empty channels	Check file patterns, use .view()
Pipeline hangs	Verify all channels emit data needed by downstream processes
Output not published	Fix globs, check publishDir
Container issues	Test image manually, correct profile
Job killed	Increase cpus/memory/time
Path/permission errors	Use absolute paths, check HPC config
Syntax errors	-stub-run, Groovy debug prints
Pipeline slow	Increase queue size, parallelize, use cloud
