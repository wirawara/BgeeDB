---
output: html_document
---
# BgeeDB, an R package for retrieval of curated expression datasets and for gene list enrichment tests
##### Andrea Komljenovic, Julien Roux, Marc Robinson-Rechavi, Frédéric Bastian [F1000Research 2016, 5:2748]([https://doi.org/10.12688/f1000research.9973.2](https://f1000research.com/articles/5-2748/v1))


```BgeeDB``` is a collection of functions to import data from the Bgee database (<http://bgee.org/>) directly into R, and to facilitate downstream analyses, such as gene set enrichment test based on expression of genes in anatomical structures. Bgee provides annotated and processed expression data and expression calls from curated wild-type healthy samples, from human and many other animal species.
 
The package retrieves the annotation of RNA-seq or Affymetrix experiments integrated into the Bgee database, and downloads into R the quantitative data and expression calls produced by the Bgee pipeline. The package also allows to run GO-like enrichment analyses based on anatomical terms, where genes are mapped to anatomical terms by expression patterns, based on the ```topGO``` package. This is the same as the TopAnat web-service available at (<http://bgee.org/?page=top_anat#/>), but with more flexibility in the choice of parameters and developmental stages.

In summary, the BgeeDB package allows to: 
* 1. List annotation of RNA-seq and microarray data available the Bgee database
* 2. Download the processed gene expression data available in the Bgee database
* 3. Download the gene expression calls and use them to perform TopAnat analyses 

## Installation

In R:
``` {r}
if (!require("BiocManager", quietly = TRUE))
    install.packages("BiocManager")

BiocManager::install("BgeeDB")
```

## How to use BgeeDB package

### Load the package
``` {r, message = FALSE, warning = FALSE}
library(BgeeDB)
```

### Running example: downloading and formatting processed RNA-seq data

#### List available species in Bgee
The ```listBgeeSpecies()``` function allows to retrieve available species in the Bgee database, and which data types are available for each species. 

``` {r}
listBgeeSpecies()
```

It is possible to list all species from a specific release of Bgee with the ```release``` argument (see ```listBgeeRelease()``` function), and order the species according to a specific columns with the ```ordering``` argument. For example:

``` {r}
listBgeeSpecies(release = "13.2", order = 2)
```

#### Create a new Bgee object

In the following example we will choose to focus on mouse ("Mus\_musculus") RNA-seq. Species can be specified using their name or their NCBI taxonomic IDs. To specify that RNA-seq data want to be downloaded, the ```dataType``` argument is set to "rna\_seq". To download Affymetrix microarray data, set this argument to "affymetrix". 

``` {r}
bgee <- Bgee$new(species = "Mus_musculus", dataType = "rna_seq")
```

*Note 1*: It is possible to work with data from a specific release of Bgee by specifying the ```release``` argument, see ```listBgeeRelease()``` function. 

*Note 2*: The functions of the package will store the downloaded files in a versioned folder created by default in the working directory. These cache files allow faster re-access to the data. The directory where data are stored can be changed with the ```pathToData``` argument.

#### Retrieve the annotation of mouse RNA-seq datasets

The ```getAnnotation()``` function will output the list of RNA-seq experiments and libraries available in Bgee for mouse. 

``` {r}
annotation_bgee_mouse <- getAnnotation(bgee)
# list the first experiments and libraries
lapply(annotation_bgee_mouse, head)
```

#### Download the processed mouse RNA-seq data

The ```getData()``` function will download processed RNA-seq data from all mouse experiments in Bgee as a list.

``` {r}
# download all RNA-seq experiments from mouse
data_bgee_mouse <- getData(bgee)
# number of experiments downloaded
length(data_bgee_mouse)
# check the downloaded data
lapply(data_bgee_mouse, head)
# isolate the first experiment
data_bgee_experiment1 <- data_bgee_mouse[[1]]
```

The result of the ```getData()``` function is, for each experiment, a data frame with the different samples listed in rows, one after the other. Each row is a gene and the expression levels are displayed as raw read counts or RPKMs. A detection flag indicates is the gene is significantly expressed above background level of expression. 

*Note 1*: An additional column in the data frame including expression in the TPM unit will be available from Bgee release 14 (planned for the end of 2016). 

*Note 2*: If microarray data are downloaded, rows correspond to probesets and log2 of expression intensities are available instead of read counts/RPKMs.

Alternatively, you can choose to download data from only one particular RNA-seq experiment from Bgee with the `experimentId` parameter:

``` {r}
# download data for GSE30617
data_bgee_mouse_gse30617 <- getData(bgee, experimentId = "GSE30617")
```

#### Format the RNA-seq data

It is sometimes easier to work with data organized as a matrix, where rows represent genes or probesets and columns represent different samples. The ```formatData()``` function reformats the data into an ExpressionSet object including:
* An expression data matrix, with genes or probesets as rows, and samples as columns (```assayData``` slot). The ```stats``` argument allows to choose if the matrix should be filled with read counts, RPKMs (and soon TPMs) for RNA-seq data. For microarray data the matrix is filled with log2 expression intensities.
* A data frame listing the samples and their anatomical structure and developmental stage annotation (```phenoData``` slot)
* For microarray data, the mapping from probesets to Ensembl genes (```featureData``` slot)

The ```callType``` argument allows to retain only actively expressed genes or probesets, if set to "present" or "present high quality". Genes or probesets that are absent in a given sample are given ```NA``` values.

```{r}
# use only present calls and fill expression matric with RPKM values
gene.expression.mouse.rpkm <- formatData(bgee, data_bgee_mouse_gse30617, callType = "present", stats = "rpkm")
gene.expression.mouse.rpkm 
```

### Running example: TopAnat gene expression enrichment analysis

For some documentation on the TopAnat analysis, please refer to our publications, or to the web-tool page (<http://bgee.org/?page=top_anat#/>).

#### Create a new Bgee object
Similarly to the quantitative data download example above, the first step of a topAnat analysis is to built an object from the Bgee class. For this example, we will focus on zebrafish:

```{r}
# Creating new Bgee class object
bgee <- Bgee$new(species = "Danio_rerio")
```

*Note* : We are free to specify any data type of interest using the ```dataType``` argument among `rna_seq`, `affymetrix`, `est` or `in_situ`, or even a combination of data types. If nothing is specified, as in the above example, all data types available for the targeted species are used. This equivalent to specifying `dataType=c("rna_seq","affymetrix","est","in_situ")`.

#### Download the data allowing to perform TopAnat analysis

The ```loadTopAnatData()``` function loads a mapping from genes to anatomical structures based on calls of expression in anatomical structures. It also loads the structure of the anatomical ontology and the names of anatomical structures. 

```{r}
# Loading calls of expression
myTopAnatData <- loadTopAnatData(bgee)
# Look at the data
str(myTopAnatData)
```

The strigency on the quality of expression calls can be changed with the ```confidence``` argument. Finally, if you are interested in expression data coming from a particular developmental stage or a group of stages, please specify the a Uberon stage Id in the ```stage``` argument. 

```{r, eval=FALSE}
## Loading only high-quality expression calls from affymetrix data made on embryonic samples only 
## This is just given as an example, but is not run in this vignette because only few data are returned
## bgee <- Bgee$new(species = "Danio_rerio", dataType="affymetrix")
## myTopAnatData <- loadTopAnatData(bgee, stage="UBERON:0000068", confidence="high_quality")
```

*Note*: As mentioned above, the downloaded data files are stored in a versioned folder that can be set with the ```pathToData``` argument when creating the Bgee class object (default is the working directory). If you query again Bgee with the exact same parameters, these cached files will be read instead of querying the web-service again. **It is thus important, if you plan to reuse the same data for multiple parallel topAnat analyses, to plan to make use of these cached files instead of re-downloading them for each analysis.** The cached files also give the possibility to repeat analyses offline.

#### Prepare a topGOdata-like object allowing to perform TopAnat analysis

First we need to prepare a list of genes in the foreground and in the background. The input format is the same as the gene list required to build a ```topGOdata``` object in the ```topGO``` package: a vector with background genes as names, and 0 or 1 values depending if a gene is in the foreground or not. In this example we will look at genes, annotated with "spermatogenesis" in the Gene Ontology (using the ```biomaRt``` package). We expect these genes to be enriched for expression in male tissues, notably testes. The background list of genes is set to all genes annotated to at least one Gene Ontology term, allowing to account for biases in which types of genes are more likely to receive Gene Ontology annotation.

```{r}
# source("https://bioconductor.org/biocLite.R")
# biocLite("biomaRt")
library(biomaRt)
ensembl <- useMart("ensembl")
ensembl <- useDataset("drerio_gene_ensembl", mart=ensembl)

# Foreground genes are those with GO annotation "spermatogenesis"
myGenes <- getBM(attributes= "ensembl_gene_id", filters=c("go_id"), values=list(c("GO:0007283")), mart=ensembl)

# Background are all genes with GO annotation
universe <- getBM(attributes= "ensembl_gene_id", filters=c("with_go_go"), values=list(c(TRUE)), mart=ensembl)

# Prepare the gene list vector 
geneList <- factor(as.integer(universe[,1] %in% myGenes[,1]))
names(geneList) <- universe[,1]
head(geneList)
summary(geneList == 1)

# Prepare the topGO object
myTopAnatObject <-  topAnat(myTopAnatData, geneList)
```

*Warning*: This can be long, especially if the gene list is large, since the Uberon anatomical ontology is large and expression calls will be propagated through the whole ontology (e.g., expression in the forebrain will also be counted as expression in parent structures such as the brain, nervous system, etc). Consider running a script in batch mode if you have multiple analyses to do.

#### Launch the enrichment test

For this step, see the vignette of the ```topGO``` package for more details, as you have to directly use the tests implemented in the ```topGO``` package, as shown in this example:

```{r}
results <- runTest(myTopAnatObject, algorithm = 'classic', statistic = 'fisher')
```

You can also choose one of the topGO decorrelation methods, for example the "weight" method, allowing to avoid redundant results induced by the structure of the ontology.
```{r, eval=FALSE}
results <- runTest(myTopAnatObject, algorithm = 'weight', statistic = 'fisher')
```

*Warning*: This can be long because of the size of the ontology. Consider running scripts in batch mode if you have multiple analyses to do.

#### Format the table of results after an enrichment test for anatomical terms

The ```makeTable``` function allows to filter and format the test results, and calculate FDR values. 

```{r}
# Display results sigificant at a 5% FDR threshold
makeTable(myTopAnatData, myTopAnatObject, results, cutoff = 0.05)
```

There is only one significant term, `testis`, which makes biological sense: there is an expression bias for testis of genes involved in spermatogenesis.

By default results are sorted by p-value, but this can be changed with the ```ordering``` parameter by specifying which column should be used to order the results (preceded by a "-" sign to indicate that ordering should be made in decreasing order). For example, it is often convenient to sort  significant structures by decreasing enrichment fold, using `ordering = -6`. The full table of results can be obtained using `cutoff = 1`.

*Warning*: it is debated whether FDR correction is appropriate on enrichment test results, since tests on different terms of the ontologies are not independent. A nice discussion can be found in the vignette of the ```topGO``` package.
