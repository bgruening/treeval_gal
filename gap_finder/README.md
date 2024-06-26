## [#4 gap_finder](https://github.com/sanger-tol/treeval/blob/dev/subworkflows/local/gap_finder.nf)

### Galaxy prototype solution

A [prototype gap_finder workflow](https://github.com/fubar2/treeval_gal/blob/main/gap_finder/Galaxy-Workflow-gap_finder_vgp_0.ga) was developed as a subworkflow but is now incorporated into the main [treevalgal](treevalgal) workflow.

A Galaxy user builds a workflow's tools, data flows and default parameter settings entirely through the GUI, compared to writing the 50+ lines of DDL needed for the NF implementation. 
A preconfigured, shareable JBrowse viewer shows the Galaxy version outputs immediately to the user, without the need to manually shuffle files around.

Could this be usefully shown to NF users?

It could be created quickly because it did not require any new Galaxy tools, unlike many of the other remaining TreeVal subworkflows.

![image](https://github.com/fubar2/treeval_gal/blob/main/gap_finder/treevalgal_gap_finder_wf.png)

#### Tutorial: steps for testing the prototype

The server must be using a recently updated seqtk package. Please get the admin to make sure the very latest seqtk from the IUC is installed before you try to use this prototype.
Older versions will fail with a complaint about `abs(c3-c2)` in the compute step.

In a Galaxy session, [import this link](https://github.com/fubar2/treeval_gal/raw/main/gap_finder/Galaxy-Workflow-gap_finder_vgp_0.ga) to your Galaxy user workflows. 

In a new history, upload an ecoli sample reference sequence for Jbrowse [gapsjbrowseref.fa from this link](https://github.com/fubar2/treeval_gal/raw/main/gap_finder/gapsjbrowseref.fa) duplicated from the GTN [Jbrowse tutorial](https://training.galaxyproject.org/training-material/topics/visualisation/tutorials/jbrowse/tutorial.html). Or if you prefer, download it to your own machine for a Jbrowse session. 

Gaps have been overwritten, more or less haphazardly, in a copy of that reference, and saved as a faked *gappy* sample input fasta for testing the workflow. 

Upload [gapsjbrowsegaps.fa from this link](https://github.com/fubar2/treeval_gal/raw/main/gap_finder/gapsjbrowsegaps.fa) to the history as input for the workflow.

Execute the workflow. Select that *gappy* file as the input and select the option to *expand the full workflow* because for this test to work,
the **Minimum size of N tract** must be set to a small value for this test run - use 1 for example. Otherwise the small fake gaps will be missed.
![image](https://github.com/fubar2/treeval_gal/assets/6016266/3a633765-c8b1-46c9-8278-789ddf6cb9bc)


Run the workflow. A new output JBrowse file with all the gaps found as a track, will appear in the history.

Using that input data and a setting for **Minimum size of N tract** of 1, the expected output is this bed [gapsjbrowse.bed](https://github.com/fubar2/treeval_gal/raw/main/gap_finder/gapsjbrowse.bed) output. 

View that JBrowse html history file. Note that the JBrowse tool must be whitelisted for HTML on the Galaxy server for this to work.
It should look like:

![jbrowse_gap_finder_bed](https://github.com/fubar2/treeval_gal/assets/6016266/41b3675d-9634-4087-bfc1-97e076cae409)

VGP and other interested researchers are asked to try it out on their own data to see if it is helpful. 
Please discuss, suggest improvements or raise issues here.

### Treeval NF DDL subworkflow deconstruction and explanation

This writes a tabix file with gap lengths added as absolute values to the end of each row of the bed file from seqtk-cutn.

Galaxy's Jbrowse automatically indexes with tabix, and will work using a bed file, such as this workflow produces, or other feature track on the appropriate reference.

![Flow chart](https://raw.githubusercontent.com/sanger-tol/treeval/dev/docs/images/v1-1-0/treeval_1_1_0_gap_finder.png)

```
Output files

    treeval_upload/
        *.bed.gz: A bgzipped file containing gap locations
        *.bed.gz.tbi: A tabix index file for the above file.
    hic_files/
        *.bed: The raw bed file needed for ingestion into Pretext
```

Tools for this subworkflow are available already so it is ideal for quick implementation and testing.

This DDL has function calls explained below.
Most of the rest of the DDL is not going to be needed other than to
figure out exactly how each function gets parameters supplied to the actual command lines.

```
#!/usr/bin/env nextflow

//
// MODULE IMPORT BLOCK
//
include { SEQTK_CUTN        } from '../../modules/nf-core/seqtk/cutn/main'
include { GAP_LENGTH        } from '../../modules/local/gap_length'
include { TABIX_BGZIPTABIX  } from '../../modules/nf-core/tabix/bgziptabix/main'

workflow GAP_FINDER {
    take:
    reference_tuple     // Channel: tuple [ val(meta), path(fasta) ]
    max_scaff_size      // Channel: val(size of largest scaffold in bp)

    main:
    ch_versions     = Channel.empty()

    //
    // MODULE: GENERATES A GAP SUMMARY FILE
    //
    SEQTK_CUTN (
        reference_tuple
    )
    ch_versions     = ch_versions.mix( SEQTK_CUTN.out.versions )

    //
    // MODULE: ADD THE LENGTH OF GAP TO BED FILE - INPUT FOR PRETEXT MODULE
    //
    GAP_LENGTH (
        SEQTK_CUTN.out.bed
    )
    ch_versions     = ch_versions.mix( GAP_LENGTH.out.versions )

    //
    // LOGIC: Adding the largest scaffold size to the meta data so it can be used in the modules.config
    //
    SEQTK_CUTN.out.bed
        .combine(max_scaff_size)
        .map {meta, row, scaff ->
            tuple([ id          : meta.id,
                    max_scaff   : scaff >= 500000000 ? 'csi': ''
                ],
                file(row)
            )}
        .set { modified_bed_ch }

    //
    // MODULE: BGZIP AND TABIX THE GAP FILE
    //
    TABIX_BGZIPTABIX (
        modified_bed_ch
    )
    ch_versions     = ch_versions.mix( TABIX_BGZIPTABIX.out.versions )

    emit:
    gap_file        = GAP_LENGTH.out.bedgraph
    gap_tabix       = TABIX_BGZIPTABIX.out.gz_csi
    versions        = ch_versions.ifEmpty(null)
}
```


[seqtk is available](https://toolshed.g2.bx.psu.edu/view/iuc/seqtk/3da72230c066) in the Toolshed.

[SEQTK_CUTN](https://github.com/sanger-tol/treeval/blob/dev/modules/nf-core/seqtk/cutn/main.nf) is a nf-core module and it just does this:
Uses *"bioconda::seqtk=1.4"*

```
    seqtk \\
            cutN \\
            $args \\
            -g $fasta \\
            > ${prefix}.bed
```


[GAP_LENGTH](https://github.com/sanger-tol/treeval/blob/dev/modules/local/gap_length.nf) is a local nf module that runs awk. If _a = $3-$2_ the math seems to return sqrt(a*a) or abs(a) - awk does not have an abs function, to enforce positive values, on the output from seqtk. This takes the seqtk cutn output bed file and appends the absolute value of the gap length to the end of each bed row

```
    $/
    cat "${file}" \
    | awk '{print $0"\t"sqrt(($3-$2)*($3-$2))}' > ${prefix}_gap.bedgraph
```

Should not be hard to implement.

There is [a generic tool available](https://usegalaxy.eu/root?tool_id=toolshed.g2.bx.psu.edu/repos/devteam/column_maker/Add_a_column1/2.0) - requires the workflow builder to set up the conversion function as a text string. Unfortunately, it insists on converting the input to tabular first which then requires a different syntax for column names because of a header now added. Expecting the generic tool to be correctly configured by inexperienced builders is perhaps expecting too much of them. Fortunately, there is an [alternate specialised version](https://toolshed.g2.bx.psu.edu/view/fubar2/abslen_bed/551c076a635c) that does not require any configuration - just drop into the workflow and it runs.

There is a always going to be a trade off between
* the cost of maintaining a single generic tool
  * Requires well informed configuration like the one above *by every workflow builder*.
* and the cost of maintaining multiple highly specialised tools
  * require zero configuration for use in a workflow so no user error
  * require one more tool. Fortunately it is so trivial that the need for maintenance is effectively zero.

**Would our VGP colleagues prefer to configure generic tools or use a plethora of highly specialised tools?**

[TABIX_BGZIPTABIX](https://github.com/sanger-tol/treeval/blob/dev/subworkflows/local/gap_finder.nf) uses *bioconda::tabix=1.11* preceded by
some interesting DDL that seems to flag the need for csi indexes if largest scaffold is too big to use tabix:

```
 SEQTK_CUTN.out.bed
        .combine(max_scaff_size)
        .map {meta, row, scaff ->
            tuple([ id          : meta.id,
                    max_scaff   : scaff >= 500000000 ? 'csi': ''
                ],
                file(row)
            )}
        .set { modified_bed_ch }
    //
```
The actual call is just:

```
 bgzip  --threads ${task.cpus} -c $args $input > ${prefix}.${input.getExtension()}.gz
 tabix $args2 ${prefix}.${input.getExtension()}.gz
```
Tabix is told to use csi indexes if max_scaffold exceeds 500M bases with
```
def args2   = meta.max_scaff == 'csi' ? "--csi" : ''
```

The 500MB max_scaff limit is a tabix limit according to the documentation:

>The tabix (.tbi) and BAI index formats can handle individual chromosomes up to 512 Mbp (2^29 bases) in length. If your input file might contain data lines with begin or end positions greater than that, you will need to use a CSI index.

Tabix bgzip [is available from the IUC](https://toolshed.g2.bx.psu.edu/repository/browse_repositories?f-free-text-search=tabix&sort=name&operation=view_or_manage_repository&id=84a670226cfe30f4) in the Toolshed. It needs a separate compression step, but if tabix is only being run for Jbrowse, Galaxy does not need this step.
