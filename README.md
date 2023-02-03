# Axiom-SNParray-Variant-and-Sample-QC-and-visualisation #
by Jennifer Nacimento Schulze

This repository provides the basic steps to export genotyping data from the Axiom Analysis Suite as well as it's manipulation using plink and population structure visualisation in the form of PCA's and Admixture plots.

## Axiom Analysis Suite ##

The [Axas](https://www.thermofisher.com/uk/en/home/life-science/microarray-analysis/microarray-analysis-instruments-software-services/microarray-analysis-software/axiom-analysis-suite.html) is a free-access software developed by ThermoFisher Scientific to analyse genotyping data generated by Axiom microarrays. 

## Generating genotype data from Axas ##

Whilst the software provides standard parameters for sample and array QC, it is possible to extract genotypes from all individuals and filter them manually using [PLINK](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC1950838/). For this, genotyping data must be exported in `PLINK PED`, without specifying thresholds to the sample and variant CR. This will generate two files: a ".map" and ".ped" which must be imported to the HPC for furhter analysis.

## Screening for Sample CR in PLINK ##

Once the files have been imported to the HPC, QC can be done using `PLINK`. <br /> 
Data missingess within individuals and variants can be explored using the command:

``` 
plink --file (yourfilename) --missing 
```
This command will generate two files: <br />
`.imiss` (individual missingness) <br />
`.lmiss` (variant missingness) <br />

From these two files, you can estimate the average CR of your indivduals/variants as well as identify those that fall under the selected threshold. Whilst the threshold for sample CR is normally > 90%, and variant CR of > 95%, depending on your data and aims of your study, you can select different thresholds.

## Excluding individuals and variants with non-desirable CR ##

Individuals with CR below the selected threshold can be excluded using:
```
plink --file myfilename --remove indlist.txt --recode --allow-extra-chr --make-bed --out (outputfile)
```

the flag `--remove` requires a space/tab-delimited text file with different columns including:  <br />
family IDs - column 1 <br />
Within-family IDs - column 2 <br />

the `--recode` flag will generate a generate a new file, excluding the selected genotypes, as PLINK will preserve all genotypes. <br />
the `--allow-extra-chr` flag can be used when chromosome codes are not recognised (e.g. if you're using genomes assembled in a contig level) and their names do not begin with a digit.
the `--make-bed` flag will generate a new **.bed** file with the remaining individuals/variants.
```
plink --file myfilename --exclude variantlist.txt --recode --allow-extra-chr --make-bed --out outputfile
```
The `--exclude` flag will do the same for variants.

## Basic visualusation of population structure with Principal Component Analysis (PCA) ##
### 1. data prep for PCA ###
```
plink --file myfilename --double-id --allow-extra-chr --set-missing-var-ids @:# --indep-pairwise 50 10 0.1 --out outputfile
plink --file myfilename --double-id --allow-extra-chr --set-missing-var-ids @:# --extract outputfile.prune.in --make-bed --pca --out outputfile
```
These commands will generate both `.eigenval` and `.eigenvec`  files, which are needed for the PCA.

The subsequent PCA analysis and data visulatisation will be done in [R](https://www.r-project.org/) which is freely available online.<br />

## PCA analysis in R ##
The `.eigenval` and `.eingenvec` files can be exported to a local computer. Alternatively, `R` can be opened in the HPC, if available.<br />

### 2. R script  ###
```R
#Install packages
install.packages("tidyverse")
install.packages("ggplot2")

#Load packages
library(tidyverse)
library(ggplot2)

#upload data
pca <- read_table2("./myfilename.eigenvec", col_names = FALSE)
eigenval <- scan("./myfilename.eigenval")

#remove first column
pca <- pca[,-1]

#check table
print(pca)

#prep table to name your individuals
names(pca)[2:ncol(pca)] <- paste0("PC", 1:(ncol(pca)-1))

#create a vector to group individuals (e.g. by population)
names(pca)[1] <- "ind"
spp<- rep (NA, length(pca$ind))
spp[grep("(Yoursamplename1)", pca$ind)] <- "Pop_A"
spp[grep("Yoursamplename2", pca$ind)] <- "Pop_B"
#repeat it for as many groups need to be created

#if needed, create a second vector to group individuals in another level (e.g. sampling location)

pop<- rep (NA, length(pca$ind))
pop[grep("Yoursamplename1", pca$ind)] <- "Sampling_location_A"
pop[grep("Yoursamplename2", pca$ind)] <- "Sampling_location_B"
#repeat it for as many groups need to be created

#create vectors for sample variance
pva <- as_tibble(data.frame(pca,spp))

#create vector with egienvalues in percentage
pve <- data.frame(PC = 1:20, pve = eigenval/sum(eigenval)*100)
cumsum(pve$pve)

# set ggplot2 theme for plotting
#modify text/colors/sizes as needed
ggtheme = theme(legend.title = element_blank(),axis.text.y = element_text(colour="black", size=15),axis.text.x
 = element_text(colour="black", size=15),axis.title = element_text(colour="black", size=15),legend.position = "
top",legend.text = element_text(size=15),legend.key.size = unit(0.7,"cm"),legend.box.spacing = unit(0, "cm"), p
anel.grid = element_blank(),panel.background = element_blank(),panel.border = element_rect(colour="black", fill
=NA, size=1),plot.title = element_text(hjust=0.5, size=25)) # title centered

#plot percentage of eigenvalues
pdf("(datasetname)_eigenval.pdf", height = 10, width = 10)
ggplot(pve, aes(PC, pve)) +  geom_hline(yintercept = 0) + geom_vline(xintercept = 0) + geom_bar(stat = "identi
ty") + ylab("Percentage variance explained") + ggtheme
dev.off()

# plot PCA's
#PC1 and PC2

pdf("(yourfilename).pdf", width = 15, height = 10)
ggplot(pca, aes(PC1, PC2, col= spp, label = spp)) +  geom_hline(yintercept = 0) + geom_vline(xintercept = 0) +
geom_point(size = 3) + xlim(-0.40,0.15) + ylim(-0.15,0.15) + coord_equal() +  geom_text(aes(label=spp), hjust=0
, vjust=0) +  theme_bw() + xlab(paste0("PC1 (", signif(pve$pve[1], 3), "%)")) + ylab(paste0("PC2 (", signif(pve
$pve[2], 3), "%)"))
dev.off()

#PC1 and PC3
pdf("(yourfilename).pdf", width = 15, height = 10)
ggplot(pca, aes(PC1, PC2, col= spp, label = spp)) +  geom_hline(yintercept = 0) + geom_vline(xintercept = 0) +
geom_point(size = 3) + xlim(-0.40,0.15) + ylim(-0.15,0.15) + coord_equal() +  geom_text(aes(label=spp), hjust=0
, vjust=0) +  theme_bw() + xlab(paste0("PC1 (", signif(pve$pve[1], 3), "%)")) + ylab(paste0("PC3 (", signif(pve
$pve[3], 3), "%)"))
dev.off()
```
## Admixture analysis ##

Admixture analysis must be run from the HPC server. Program dowloand link, manual and publications are available [here](https://dalexander.github.io/admixture/).

### Running Admixture ###

This software does not accept chromosome names which are not human. Therefore, the first column of the `.bim` file must be exchanged by 0. <br /> 
The final `.bim`, `.bed`, `.fam` files were generated with `PLINK` after removing individuals/variants which did not pass the selected QC filters.

```
awk '{$1=0;print $0}' (yourfile).bim > (yourfile).bim.tmp
mv (yourfile).bim.tmp (yourfile).bim
```
Now, Admixture can be run by applying the following commands: 

```
admixture --cv (yourfile).bed 2 > log2.out

# Two files will be produced
# The .Q file contains the cluster assignments for each individual
# The .P file contains the population allele frequencies for each SNP included in the analysis

# Contrary to Structure analysis, admixutre calculates the best number of K clusters representing the population being analysed, with no prior assumption. 
# Therefore, in this example, a set of k clusters, generally from 2 to 10, will be analysed in a loop and the output directed to a log.out file. However, more this value can be adjusted based on preivous knowledge of the population set being analysed in the study. must test the best fitting K value for the population being analysed.  

for i in {2..10}
do
admixture --cv (myfile).bed $i > log${i}.out
done

# The best fitting number of K clusters will be the one with the lowest number of cross-validation error
# This value can be extracted from the log.out generated files using the following command:

awk '/CV/ {print $3,$4}' *out | cut -c 4,7-20 > (myfile).cv.error
```

### Visualising Admixture output in R ###
Once the Admixture analysis is done and you have defined K value which will be used for visualisation, the data can be exported to into  `R` for visualisation:
```R
# load libraries

library(tidyverse)
library(ggplot2)

# import datasets
# in this case, bestKvalue = the K value with the lowest cross-validation error generated by the admixture analysis
tbl1=read.table("(myfile).bestKvalue.Q")
tbl2=read.table("(myfile).list")
pca <- cbind(tbl1,tbl2)

#Here, bestKvalue + 1 must be added in []
names(pca)[bestKvalue+1] <- "ind"

# #create a vector to group individuals (e.g. by population)
spp <- rep (NA, length(pca$ind))
spp[grep("YourSampleName1", pca$ind)] <- "Pop_A"
spp[grep("YourSampleName2", pca$ind)] <- "Pop_B"

k <- as_tibble(data.frame(pca,spp))

#you can reorder your samples on the X axis by using the following command line
new_order <- c("Pop_A", "Pop_B", "...")
k3$spp <- factor(as.character(k3$spp), levels=new_order)
k <- k[order(k3$spp),]

# define color pallete, if wanted.
pal <- colorRampPalette(colors = c("lightblue", "blue"))(3)

# Alternatively, define the colors manually, one for each group created for the analysis
# Create and export Admixture plot
#in kx, use the k number with the lowest cross-validation error
pdf("myfile.pdf", width=20, height=10)
barplot(t(as.matrix(kX)), col=c("#71D0F5FF","#F05C3BFF", "..."), width=0.1, space=0.2, las=2, xlab="Populations", ylab="Ancestry", border=NA, names.arg=(k3$spp))
dev.off()
```



