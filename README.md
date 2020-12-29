LAVA TUTORIAL
================
Josefin Werme, CTG Lab, VU Amsterdam
2020-12-29

This tutorial shows you how to read in and analyse data with LAVA
(**L**ocal **A**nalysis of \[co\]**V**ariant **A**ssociation): A tool
developed for local genetic correlation (*r*<sub>g</sub>) analysis.

LAVA can analyse the local *r*<sub>g</sub> between two or more
phenotypes, analyse both binary and continuous phenotypes, and account
for known or estimated sample overlap. It can also test the univariate
local genetic signal for all phenotypes of interest (i.e. the local
*h*<sup>2</sup>), which may be used to filter out non-associated loci,
and model the local genetic relations between multiple phenotypes using
conditional models.

The tutorial will show you how to install and run LAVA using some
example input data. If you wish, you can inspect the data in the
‘vignettes/data’ folder.

------------------------------------------------------------------------

## Installing LAVA

To read in the genotype data, LAVA uses functions from the snpStats
packge. As this package needs to be installed using BiocManager, you
should do so before installing LAVA

``` r
install.packages("BiocManager")
BiocManager::install("snpStats")
```

You will also need devtools for the installation

``` r
install.packages("devtools")
```

LAVA can then be installed directly from github

``` r
devtools::install_github("https://github.com/josefin-werme/lava.git")
```

Or by downloading the code and installing it from a local directory

``` r
devtools::install("~/Programs/R/lava")      # adapt as necessary
```

``` r
library(lava)
```

## Input format

*NOTE: Example input files can be found in the ‘vignettes/data’
directory*

#### As input, LAVA needs the following data:

-   **Reference genotype data** in plink format (.bim, .bed, .fam), used
    for the estimation of LD

    -   e.g. [1000 genomes](https://www.internationalgenome.org/data/),
        pre-processed input files can be found
        [here](%5Bhttps://ctg.cncr.nl/software/magma)\](<https://ctg.cncr.nl/software/magma>))

-   **Input info file**, used for convenient processing of multiple
    phenotypes. Requires the columns:

    -   ***phenotype***: phenotype IDs

    -   ***cases***: number of cases

        -   set to 1 for continuous phenotypes

    -   ***controls:*** number of controls

        -   set to 0 for continuous phenotypes (important!); the case /
            control ratio is used to distinguish binary and continuous
            phenotypes, and is required for appropriate processing of
            binary phenotypes.

    -   ***prevalence*** (optional): the population prevalence of binary
        phenotypes

        -   this is only relevant if you want an estimate of the local
            population *h*<sup>2</sup> for binary phenotypes. Estimates
            of the observed local sample *h*<sup>2</sup> are still
            provided

    -   ***filename***: paths and file names to the relevant summary
        statistics

-   **Summary statistics** for all phenotypes of interest. To
    accommodate common summary statistics formats, a number of different
    column names are possible for the same column type (but **only ONE**
    header should be provided for each category; i.e do not provide a
    file that contains multiple valid SNP ID columns):

    -   ***SNP / ID / SNPID\_UKB/ SNPID / MarkerName / RSID***: SNP IDs

    -   ***A1 / ALT***: effect allele

    -   ***A2 / REF***: reference allele

    -   ***N / NMISS / N\_analyzed***: number of samples

    -   ***Z / T / STAT / Zscore***: if provided, no p-values or
        coefficients are needed; otherwise, please provide:

        -   ***B / BETA / OR / logOdds***: effect size coefficients

            -   please provide beta coefficients for continuous
                phenotypes, and odds ratios / log odds for binary
                phenotypes

        -   ***P***: p-values

-   **Locus definition file**: File that defines loci either based on
    genomic coordinates or a list of SNPs. Required headers:

    -   ***LOC***: locus ID

    -   ***CHR, START, STOP***: coordinates (i.e. basepair positions)

    -   ***SNPS***: list of SNPS. ‘;’ separated, no white-space
        (optional)

        -   note: if a SNP list is provided, coordinates will be ignored
            and the reference data will be subsetted based on SNP IDs
            instead. This can be convenient if the locus definition file
            is based on a different GRChX version than the reference
            data. If no SNP list is provided, the versions of reference
            and the locus file MUST match!

-   **Sample overlap file** (optional)

    -   This can be obtained using the results from cross-trait LDSC
        (see the ‘vignettes/sample\_overlap.Rmd’ file for a walk through
        on how to do this)

## Process input

Run the script below to process the sample input files. If you wish, you
can examine the input object to get an idea of what your data looks like
after processing. You can also check out the original data in the
‘vignettes/data’ folder.

``` r
### Read in summarystatistics and related info
input = process.input(input.info.file="vignettes/data/input.info.txt",           # input info file
                      sample.overlap.file="vignettes/data/sample.overlap.txt",   # sample overlap file (can be set to NULL if there is no overlap)
                      ref.prefix="vignettes/data/g1000_test",                    # reference genotype data prefix
                      phenos=c("depression","neuro","bmi"))       # subset of phenotypes listed in the input info file that we want to process

# inspect the processed input data
ls(input)                     # this is actually an environment; hence ls() rather than str()
ls(input$sum.stats)   # processed summary statistics
input$info            # processed input info file, with additional variables N, prop_cases, binary computed by process.input()
input$sample.overlap  # sample overlap file
head(input$ref$bim)   # bim file from reference data
  
# read more about this function
?process.input()
```

**&gt;&gt; \[ comment on the use of case control ratio to distinguish
binary and continuous phentoypes \] &lt;&lt;**

## Prepare a locus for analysis

Before analysing the genetic correlation at a locus, we need to convert
the marginal GWAS SNP effects within the locus to their corresponding
joint effects (in order to account for the LD between SNPs), and
estimate certain key parameter that are used for the analysis.

``` r
### Read in locus info file
loci = read.loci("vignettes/data/test.loci")
head(loci)                          # inspect the locus file
#>   LOC CHR     START      STOP
#> 1 100   1 113418038 114664387
#> 2 230   2  26894103  28819510
#> 3 266   2  57952946  59251996
#> 4 374   2 191051955 193033982
#> 5 464   3  47588462  50387742
#> 6 950   6  25684630  26396200
?read.loci()                        # read more about the function and possible data formats

### Create a locus object for the first locus to prepare it for analysis
locus = process.locus(loci[1,], input)
```

If we inspect the locus object, we can see that it contains a lot of
info about the locus, such as:

-   Locus coordinates (‘chr’,‘start’,‘stop’)

-   The number of SNPs (‘n.snps’) / PCs (‘K’) within the locus

-   The (estimated) PC projected joint SNP effects, *δ* (‘delta’)

-   The sampling covariance *σ*<sup>2</sup> (‘sigma’)

-   The genetic covariance matrix
    *Ω* = *t*(*δ*)′*δ*/*K* − *σ*<sup>2</sup> (‘omega’) with
    corresponding correlation matrix *Ω*<sup>\*</sup> (‘omega.cor’)

``` r
ls(locus)                               # inspect locus
#>  [1] "binary"    "chr"       "delta"     "h2.latent" "h2.obs"    "id"       
#>  [7] "K"         "N"         "n.snps"    "omega"     "omega.cor" "phenos"   
#> [13] "sigma"     "snps"      "start"     "stop"
c(locus$chr, locus$start, locus$stop)   # locus coordinates
#> [1]         1 113418038 114664387
str(locus$snps)                         # locus snps
#>  chr [1:2231] "rs2360008" "rs1237670" "rs1235629" "rs1216539" "rs61819971" ...
locus$n.snps                            # N snps
#> [1] 2231
locus$omega                             # genetic covariance matrix
#>               depression         neuro           bmi
#> depression  1.391995e-06 -2.074891e-07 -4.335870e-07
#> neuro      -2.074891e-07  4.477136e-07 -2.674627e-07
#> bmi        -4.335870e-07 -2.674627e-07  7.443673e-07
locus$omega.cor                         # standardised genetic covariance matrix
#>            depression      neuro        bmi
#> depression  1.0000000 -0.2628309 -0.4259550
#> neuro      -0.2628309  1.0000000 -0.4633077
#> bmi        -0.4259550 -0.4633077  1.0000000
locus$phenos                            # locus phenotypes
#> [1] "depression" "neuro"      "bmi"

?process.locus()  # find our more details in the function documentation
```

## Perform the univariate test

To determine the amount of local genetic signal for all phenotypes of
interest, and filter out non-associated loci, we use the univariate
test.

After obtaining the locus object, we pass it directly to the run.univ()
function in order to test the local heritability for all phenotypes (or
a subset)

``` r
# for all phenotypes in the locus
run.univ(locus)
#>         phen      h2.obs          p
#> 1 depression 8.05125e-05 0.04240950
#> 2      neuro 1.16406e-04 0.03154340
#> 3        bmi 1.93535e-04 0.00146622

# or just a subset
run.univ(locus, phenos=c("depression","bmi"))
#>         phen      h2.obs          p
#> 1 depression 8.05125e-05 0.04240950
#> 2        bmi 1.93535e-04 0.00146622
```

As there was little evidence of any genetic signal for either
phenotypes, there will be no point in testing the local *r*<sub>g</sub>
(and if we did, the estimates would be unreliable and uninterpretable)..

If we check another locus however:

``` r
locus = process.locus(loci[3,], input)
run.univ(locus)
#>         phen      h2.obs           p
#> 1 depression 0.000260236 4.13399e-07
#> 2      neuro 0.000573255 1.25993e-14
#> 3        bmi 0.000975242 6.60429e-32
```

we see that there is plenty of signal for all phenotypes, and we can
proceed with the bivariate test!

## Analyse the bivariate local *r*<sub>g</sub> between two phenotypes

To perform a bivariane *r*<sub>g</sub> analysis, testing the local
*r*<sub>g</sub> between pairs of phenotypes, we simply pass the locus
object the run.bivar() function

``` r
run.bivar(locus)
#>        phen1 phen2       rho rho.lower rho.upper         r2 r2.lower r2.upper
#> 1 depression   bmi 0.0723039  -0.18708   0.32633 0.00522785        0  0.10877
#> 2      neuro   bmi 0.1069490  -0.09698   0.30797 0.01143810        0  0.09495
#>          p
#> 1 0.566506
#> 2 0.301328
```

When multiple phenotypes are entered simultaneously, the last phenotype
will be treated as the target phenotype, and separate bivariate tests
will be performed between this phenotype and all others. (Note that this
test is symmetric, the order of the phenotypes doesn’t matter!).

You can subset and change the order of the phenotypes using the ‘phenos’
argument:

``` r
run.bivar(locus, phenos=c("neuro","bmi","depression"))
#>   phen1      phen2       rho rho.lower rho.upper         r2 r2.lower r2.upper
#> 1 neuro depression 0.5278610   0.26767   0.77906 0.27863700  0.07165  0.60694
#> 2   bmi depression 0.0723039  -0.17562   0.32728 0.00522785  0.00000  0.10887
#>             p
#> 1 0.000493229
#> 2 0.567830000
```

To filter automatically based on the univariate signal, you can also use
the run.univ.bivar() function to perform both tests, with the condition
that the bivariate test is only performed for the phenotypes that reach
the univariate significance:

``` r
# with the default p-value threshold of .05
run.univ.bivar(locus)
#> $univ
#>         phen      h2.obs           p
#> 1 depression 0.000260236 4.13399e-07
#> 2      neuro 0.000573255 1.25993e-14
#> 3        bmi 0.000975242 6.60429e-32
#> 
#> $bivar
#>        phen1 phen2       rho rho.lower rho.upper         r2 r2.lower r2.upper
#> 1 depression   bmi 0.0723039  -0.17828   0.32766 0.00522785        0  0.10977
#> 2      neuro   bmi 0.1069490  -0.10010   0.31571 0.01143810        0  0.09967
#>          p
#> 1 0.566549
#> 2 0.300910

# or with a custom p-value threshold
run.univ.bivar(locus, univ.thresh = 1e-8)
#> $univ
#>         phen      h2.obs           p
#> 1 depression 0.000260236 4.13399e-07
#> 2      neuro 0.000573255 1.25993e-14
#> 3        bmi 0.000975242 6.60429e-32
#> 
#> $bivar
#>   phen1 phen2      rho rho.lower rho.upper        r2 r2.lower r2.upper       p
#> 1 neuro   bmi 0.106949  -0.09424   0.31401 0.0114381        0   0.0986 0.30216
```

If you would still like to print the bivariate output for unanalysed
phentoypes, you can set the ‘return.unanalysed’ argument to TRUE.

``` r
run.univ.bivar(locus, univ.thresh=1e-8, return.unanalysed=T)
#> $univ
#>         phen      h2.obs           p
#> 1 depression 0.000260236 4.13399e-07
#> 2      neuro 0.000573255 1.25993e-14
#> 3        bmi 0.000975242 6.60429e-32
#> 
#> $bivar
#>        phen1 phen2      rho rho.lower rho.upper        r2 r2.lower r2.upper
#> 2 depression   bmi       NA        NA        NA        NA       NA       NA
#> 1      neuro   bmi 0.106949  -0.09914   0.31233 0.0114381        0  0.09757
#>         p
#> 2      NA
#> 1 0.30045
```

## Apply the multivariate approaches to more than two phenotypes

For loci in which you find significant correlations between more than
two phenotypes, you might want to apply one of two possible multivariate
approaches in order to compute conditional local genetic correlations
within the locus

### Multiple regression

With the multiple regression option, you can model the genetic signal
for an outcome phenotype using several other predictor phenotypes
simultaneously. When doing so, any potential local *r*<sub>g</sub>
between the predictor phenotypes will be accounted for, and their
relation with the outcome will be conditional on all the other
predictors.

For this analysis we will select a locus and a phenotype subset which
has a bit more signal:

``` r
input = process.input(input.info.file="vignettes/data/input.info.txt",
                      sample.overlap.file="vignettes/data/sample.overlap.txt",
                      ref.prefix="vignettes/data/g1000_test",
                      phenos=c("asthma","rheuma","diabetes","hypothyroidism"))
#> [1] "...Reading in sumstats"
#> [1] "...Reading in SNP info from reference data"
#> [1] "...Extracting common SNPs"
#> [1] "~ 90342 SNPs shared across data sets"
#> [1] "...Aligning effect alleles to reference data set"
#> [1] "...Removing 12980 SNPs which could not be aligned, 77362 remaining"

locus = process.locus(loci[19,], input)
```

Say that we are interested in modelling hypothyroidism as the outcome.
Lets first check that there is actually some local genetic signal for
all phenotypes, and that all predictors are genetically correlated with
the outcome within this locus

``` r
run.univ.bivar(locus)
#> $univ
#>             phen      h2.obs            p
#> 1         asthma 0.001778430 4.66411e-148
#> 2         rheuma 0.054520700  0.00000e+00
#> 3       diabetes 0.000661558  1.54348e-50
#> 4 hypothyroidism 0.001717760  6.33045e-84
#> 
#> $bivar
#>      phen1          phen2      rho rho.lower rho.upper       r2 r2.lower
#> 1   asthma hypothyroidism 0.847161   0.77963   0.90896 0.717682  0.60782
#> 2   rheuma hypothyroidism 0.525462   0.44003   0.61055 0.276111  0.19363
#> 3 diabetes hypothyroidism 0.859903   0.77235   0.93955 0.739434  0.59652
#>   r2.upper           p
#> 1  0.82622 4.52800e-43
#> 2  0.37278 1.31253e-24
#> 3  0.88275 3.78709e-26
```

They do! We can then run the multiple regression for all predictors.

Here, the last phenotype will again be treated as the outcome

``` r
run.multireg(locus)
#> [1] "~ Running multiple regression for outcome 'hypothyroidism', with predictors 'asthma', 'rheuma', 'diabetes'"
#> [[1]]
#> [[1]][[1]]
#>   predictors        outcome     gamma gamma.lower gamma.upper       r2 r2.lower
#> 1     asthma hypothyroidism 0.8184750     0.70623     0.93617 0.719291  0.61112
#> 2     rheuma hypothyroidism 0.0493099    -0.10402     0.18675 0.719291  0.61112
#>   r2.upper           p
#> 1  0.83195 9.31021e-24
#> 2  0.83195 5.09806e-01
#> 
#> [[1]][[2]]
#>   predictors        outcome    gamma gamma.lower gamma.upper       r2 r2.lower
#> 1     asthma hypothyroidism 0.405003    -0.06887     0.76640 0.778212  0.67956
#> 2   diabetes hypothyroidism 0.505998     0.12651     0.97871 0.778212  0.67956
#>   r2.upper         p
#> 1   0.8958 0.1033330
#> 2   0.8958 0.0346502
#> 
#> [[1]][[3]]
#>   predictors        outcome     gamma gamma.lower gamma.upper       r2 r2.lower
#> 1     rheuma hypothyroidism -0.195568    -0.50169     0.02574 0.757829  0.61422
#> 2   diabetes hypothyroidism  1.000800     0.80859     1.26661 0.757829  0.61422
#>   r2.upper           p
#> 1  0.92185 1.11112e-01
#> 2  0.92185 8.49675e-10
#> 
#> 
#> [[2]]
#> [[2]][[1]]
#>   predictors        outcome     gamma gamma.lower gamma.upper       r2 r2.lower
#> 1     asthma hypothyroidism  0.372955    -0.30357     0.75497 0.790052  0.69224
#> 2     rheuma hypothyroidism -0.158500    -0.50281     0.04413 0.790052  0.69224
#> 3   diabetes hypothyroidism  0.648194     0.16434     1.50002 0.790052  0.69224
#>   r2.upper         p
#> 1  0.93542 0.1884220
#> 2  0.93542 0.1872890
#> 3  0.93542 0.0546288
```

Here you see that, by default, this function does not only return the
full model with all predictors, but also all intermediate models. The
reason for this is that when running a multiple regression it is
important to be aware of how each predictor affects the fit of the other
predictors in order to interpret the results correctly.

For example, in this case we can see that both asthma and diabetes are
quite significant in the two-predictor models with rheuma, but not in
the full model, or the two-predictor model with asthma and diabetes.
This indicates that there is some collinearity between these predictors,
which explains why neither appear very significant in the full model.

Had we not looked at the intermediate models beforehand, this pattern
would have not been obvious. If we also didn’t already know the
bivariate correlations between the phenotypes, and only looked at the
multivariate model, we might have made the erroneous conclusion that
neither are relevant.

**&gt;&gt;&gt; \[ comment on model r2 \] &lt;&lt;&lt;**

If for some reason you only want to return the full model, however, you
can speed up the analysis by setting the ‘only.full.model’ argument to
TRUE

``` r
run.multireg(locus, only.full.model=T)
#> [1] "~ Running multiple regression for outcome 'hypothyroidism', with predictors 'asthma', 'rheuma', 'diabetes'"
#> [[1]]
#> [[1]][[1]]
#>   predictors        outcome     gamma gamma.lower gamma.upper       r2 r2.lower
#> 1     asthma hypothyroidism  0.372955    -0.28383     0.76211 0.790052  0.69352
#> 2     rheuma hypothyroidism -0.158500    -0.49385     0.04508 0.790052  0.69352
#> 3   diabetes hypothyroidism  0.648194     0.16242     1.47409 0.790052  0.69352
#>   r2.upper         p
#> 1  0.93306 0.1858700
#> 2  0.93306 0.1866880
#> 3  0.93306 0.0563036
```

### Partial correlation

In contrast with the multiple regression, the partial correlation allows
you to compute the local r<sub>*g*</sub> between two phenotypes of
interest, conditioned on one or more other phenotypes.

For this example, we will use the same locus and set of phenotypes as
before. Lets say we are now interested in the partial correlation
between hypothyroidism and diabetes. Recall that there was also some
collinearity between asthma and diabetes. We can therefore expect that
if we compute the partial correlation between hypothyroidism and
diabetes conditioned on asthma, that asthma might account for a notable
proportion of the r<sub>*g*</sub> between hypothyroidism and asthma
(which we determined to be .86, from the bivariate test earlier)

``` r
run.partial.cor(locus, phenos=c("hypothyroidism","diabetes","asthma"))
#> [1] "~ Running partial correlation for 'hypothyroidism' and 'diabetes', conditioned on 'asthma'"
#>            phen1    phen2      z r2.phen1_z r2.phen2_z     pcor ci.lower
#> 1 hypothyroidism diabetes asthma   0.717682   0.763587 0.463037   0.1133
#>   ci.upper         p
#> 1  0.76241 0.0143186
```

Indeed, here you see that the partial correlation has been almost
halved, and is no longer significant. The columns r2.phen1\_z and
r2.phen2\_z confirm that asthma explains a substantial proportion of
genetic variance for both hypothyroidism and diabetes.

If we instead condition on rheuma

``` r
run.partial.cor(locus, phenos=c("hypothyroidism","diabetes","rheuma"))
#> [1] "~ Running partial correlation for 'hypothyroidism' and 'diabetes', conditioned on 'rheuma'"
#>            phen1    phen2      z r2.phen1_z r2.phen2_z     pcor ci.lower
#> 1 hypothyroidism diabetes rheuma   0.276111   0.519053 0.815756  0.68132
#>   ci.upper           p
#> 1  0.94573 6.57496e-13
```

The partial correlation (.82) is now only slightly lower than the
bivariate correlation (.86).

As seen previously, this phenotype was far less strongly genetically
correlated with hypothyroidism (at .53), and as indicated by the r2’s
here, it is also explains less of the genetic variance in hypothyroidism
and diabetes compared to asthma.

We can also condition on both asthma and rheuma (or any number of
phenotypes) in one go:

``` r
run.partial.cor(locus, phenos=c("hypothyroidism","diabetes","rheuma","asthma"))
#> [1] "~ Running partial correlation for 'hypothyroidism' and 'diabetes', conditioned on 'rheuma' + 'asthma'"
#>            phen1    phen2             z r2.phen1_z r2.phen2_z     pcor ci.lower
#> 1 hypothyroidism diabetes rheuma;asthma   0.719291   0.831584 0.502074  0.12643
#>   ci.upper        p
#> 1  0.86768 0.015286
```

------------------------------------------------------------------------

## Example analysis script for running bivariate *r*<sub>g</sub> analysis across all genomic loci

If you are interested in analysing the bivariate local *r*<sub>g</sub>’s
between a large amount of phenotypes, we advice using a cluster
computer. The example analysis script below shows how you might set up
an R script that can be called from the command line, analysing the
local *r*<sub>g</sub>’s across all loci in the locus file.

#### Bash

This is how you may call the R script from the command line

``` bash
Rscript "g1000_test" "test.loci" "input.info.file" "sample.overlap.file" "depression;bmi" "depression.bmi"
```

#### R script

``` r
# command line arguments, specifying input/output file names and phenotype subset
arg = commandArgs(T); ref.prefix = arg[1]; loc.file = arg[2]; info.file = arg[3]; sample.overlap.file = arg[4]; phenos = unlist(strsplit(arg[5],";")); out.fname = arg[6]

library(data.table); library(lava); print(paste("Running LAVA version",packageVersion("lava")))

### Read in data
loci = read.loci(loc.file); n.loc = nrow(loci)
input = process.input(info.file, sample.overlap.file, ref.prefix, phenos)

print(paste("Starting LAVA analysis for",n.loc,"loci"))
progress = ceiling(quantile(1:n.loc, seq(.05,1,.05)))   # (if you want to print the progress)

### Analyse
u=b=list()
for (i in 1:n.loc) {
        if (i %in% progress) print(paste("..",names(progress[which(progress==i)])))     # (printing progress)
        locus = process.locus(loci[i,], input)                                          # process locus
        
        # It is possible that the locus cannot be defined for various reasons (e.g. too few SNPs, negative variance), so the !is.null(locus) check is necessary before calling the analysis functions.
        # Note that it is also possible that individual phenotypes fail, in which case only the remaining phenotypes will be retured from the process.locus() function. 
        # If you require all phenotypes to be analysed (as we do here), also include the all(phenos %in% locus$phenos) condition in the if statement as well
        if (!is.null(locus) & all(phenos %in% locus$phenos)) {
                # extract some general locus info for the output
                loc.info = data.frame(locus = locus$id, chr = locus$chr, start = locus$start, stop = locus$stop, n.snps = locus$n.snps, n.pcs = locus$K)
                
                # run the univariate and bivariate tests
                loc.out = run.univ.bivar(locus, univ.thresh=1e-4)
                u[[i]] = cbind(loc.info, loc.out$univ)
                b[[i]] = cbind(loc.info, loc.out$bivar)
        }
}

# save the output
write.table(do.call(rbind,u), paste0(out.fname,".univ.lava"), row.names=F,quote=F,col.names=T)
write.table(do.call(rbind,b), paste0(out.fname,".bivar.lava"), row.names=F,quote=F,col.names=T)

print(paste0("Done! Analysis output written to ",out.fname,".*.lava"))
```
