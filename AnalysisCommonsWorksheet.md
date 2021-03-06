# Analysis Commons

## Outline
* Introduction to web-interface
* Running a single variant analysis
* Workflows and monitoring jobs
* Running aggregate tests (SKAT)
* Run batch jobs from the command line
* Writing your own Apps
	
## Web Interface and Running an Analysis Application 

### Exercise 1) Run a single variant analysis.  
Note that the job will finish instantaneously if you don’t change the output file name.  It knows that you are running the exact same job and will just reuse results from previous analyses. 


Log into http://dnanexus.com using the user name and password listed on the handout.  
Should be in the form of Username:**topmed\_#** and Password:**Topmed\_#**.
*Ignore warning about default billing account.*
Navigate to and select **(dcc:tools/genesis\_v0.7)**

File inputs:  
* phenofile -> phenotype/1KG_pheno.csv  
* genotypefile -> genotypes/1KG_phase3_subset_chr1.gds  
* kinship -> kinship/1KG_kins.Rda  
* Note: orange aggregation, annotation and genefile can be left empty

Parameter inputs:  
* output folder: output/YOURFOLDERNAME  
* outcome _(Column name of the outcome variable)_: outcome  
* covariates _(case sepecific)_: Population,sex  
* prefix for output filename: single\_chr1  
* test_type: Single  
* pheno_id: sample.id  
* Note: Other options can be left as their defaults, some are only used for aggreagate tests



### Exercise 2) Run SKAT test grouping variants into gene transcript regions and limit the variants to those with a CADD phred score > 2 and MAF <= 5%.
_Italic_ inputs below are the same as single variant; update the parameters & files to change to a SKAT test.  Go to the monitor tab.  Click on the Name of a job ( or someone’s ) that successfully completed the single variant analysis, then click “Launch as new Job” and modify the inputs.   

File inputs:  
* _phenofile -> phenotype/1KG\_pheno.csv_  
* _genotypefile -> genotypes/1KG\_phase3\_subset\_chr1.gds_  
* _kinship -> kinship/1KG\_kins.Rda_  
* annotation -> annotation/1KG\_annotation\_CHR1.txt  
* genefile -> aggregation/AggUnit\_CHR1\_ucscgene.csv  

Parameter inputs:  
* _outcome: outcome_  
* _covariates: Population,sex_  
* _pheno_id: sample.id_  
* output folder: output/YOURFOLDERNAME  
* outputfilename: skat\_chr1\_geneBased\_CADDgt2  
* test_type: SKAT  
* snp_filter: CADD\_phred>2  
* min_mac:0  
* top_maf: 0.05  
* weights: c(1,25)  

## Command line interface

References:  
* Command Line Interface [Quickstart](https://wiki.dnanexus.com/Command-Line-Client/Quickstart)  
* Index of [dx commands](https://wiki.dnanexus.com/Command-Line-Client/Index%20of%20dx%20Commands)  

### Log in to AWI
**Replace topmed_## with the user ID from your handout**
```
$ ssh topmed_##@34.212.243.167 --timeout 2h
You will be prompted for your password, e.g. Topmed_## (Note capitolization)
_Please ignore login warnings

$ source /usr/local/dx-toolkit/environment
```


```
$ dx login 
	Enter the following at the prompts
		username: topmed_##
		password: Topmed_##
		project:dcc ( type 0 to select dcc )

You can select or change project once you are logged in
$ dx select dcc
```



### Exercise 3) Navigate directories, make output directory, examine files

* File paths: \<project\>:/path/to/file.txt
* Example: dcc:/phenotypes/1KG\_pheno.csv


List directory contents:
```
$ dx select dcc
$ dx ls
$ dx ls /tools
$ dx ls dcc:/tools
```
Get results from project
```
$ dx download dcc:/phenotype/1KG_pheno.csv
$ ls
$ head 1KG_pheno.csv
```
### Exercise 4) Run single variant analysis from command line using bash script

Open the single_multichrom.sh bash script and edit to replace the output directory “YOURNAME” to your folder
```
$ dx describe tools/genesis_v0.7
```
Either edit using nano
```
$ nano single_multichrom.sh 

Run the App.  Will loop over 2 chromosomes running the single variant analyses
$ ./single_multichrom.sh
```

## Writing your own Apps 
### Exercise 5) Write an App that creates phenotype residuals and performs an inverse normal transform


Use app wizard to create template
```
$ dx-app-wizard

App Name: make_residuals
Title []: Create inverse normal transformed residuals

1st input name (<ENTER> to finish): phenofile
Label (optional human-readable name) []: CSV phenotype file
Choose a class (<TAB> twice for choices): file
This is an optional parameter [y/n]: n

2nd input name (<ENTER> to finish): model
Label (optional human-readable name) []: model for creating residuals (e.g. outcome~age+Population )
Choose a class (<TAB> twice for choices): string
This is an optional parameter [y/n]: n

3rd input name (<ENTER> to finish): prefix
Label (optional human-readable name) []: Output filename prefix
Choose a class (<TAB> twice for choices): string
This is an optional parameter [y/n]: n

4th input name (<ENTER> to finish): <ENTER>

1st output name (<ENTER> to finish): output
Label (optional human-readable name) []: 
Choose a class (<TAB> twice for choices): file

Timeout policy [48h]: 1h
Programming language: bash

*Use defaults for other options*
```

Look at the files created by the wizard
```
cd make_residuals/
ls
more dxapp.json 
```

Edit App executable to run an R script
```
$ vi src/make_residuals.sh
main() {

    echo "Value of phenofile: '$phenofile'"
    echo "Value of model: '$model'"
    echo "Value of prefix: '$prefix'"

    dx download "$phenofile" -o phenofile

    Rscript /make_resid.R $model

    output=$(dx upload output --brief)

    dx-jobutil-add-output output "$output" --class=file
    dx mv ${output} ${prefix}.csv
}

```

Create an R script that does the 'work'
`
$ vi resources/make_resid.R
`
```
args<-commandArgs(TRUE)
model <- as.formula(args[1])
print(model)
pheno = read.csv("phenofile",as.is=T)
pheno$resid = residuals(lm(model,data=pheno))
pheno$invnt_resid =  with(pheno,qnorm((rank(resid,na.last="keep")-0.5)/sum(!is.na(resid))))

write.csv(pheno,file="output",row.names=F)
```  
Build the App
```
$ dx build -f make_residuals --destination=output/YOURNAME/make_residuals
```

Run the App
```
$ dx run output/YOURNAME/make_residuals -iphenofile=phenotype/1KG_pheno.csv \
-imodel=outcome~sex+Population -iprefix=1kg_pheno_invnt \
--destination=output/YOURNAME --yes
```


Monitor Progress
```
$ dx watch jobid
```

### Optional Exercise 6) Make QQ plot
Make QQ plot of your single variant results.  
Select results from the multiple chromosome run (chr21 and chr22).  

You will need to identify the p-value column name.  Use [dx head](https://wiki.dnanexus.com/Command-Line-Client/Index-of-dx-Commands#head--) to view column names, or [dx download](https://wiki.dnanexus.com/Command-Line-Client/Index-of-dx-Commands#download) to download the results for viewing.  Alternately, view through web interface using Visualize ( next to Monitor near top of the page ) and select [*Gzipped File Previewer*](https://platform.dnanexus.com/projects/F5jVpJ80JXGQV51P8GqVxPPQ/visualize#)

Run first through web interface then try running interactivly from the web interface then from the command line. 

```
$ dx run tools/qqplot
   note t

```
