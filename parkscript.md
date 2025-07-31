##From erikpark
* Notes on the data and analysis for this project *

- This analysis is being done entirely on the IU clusters (specifically Carbonate as of 2023-05), and all relevant data are stored on the IU Slate storage system.

- There is no single script which executes and completes this project, but all relevant code (which is run in the terminal on the clusters) is collected below.

###############
#### Code #####
###############

First I needed to install both Juicer (https://github.com/aidenlab/juicer/wiki/Installation) and HiC-Pro (https://github.com/nservant/HiC-Pro) from github onto the carbonate cluster.

####################
# HiC-Pro2Juicebox #
####################

Then I ran the hicpro2juicebox.sh three times (once for each allValidpairs file) using three separate slurm scripts on Carbonate.
This needed a recent installation of the Juicer_tools.jar, taken from here: https://github.com/aidenlab/Juicebox/releases

The general slurm script code is as follows (using the CTRL-allValidpairs file for an example)

> module load miniconda/python3.9/4.12.0
> module load cudatoolkit
> module load java/1.8.0_131
> /N/u/hicpro2juicebox.sh -i /N/slate/erikpark/CTRL.allValidPairs_original -g hg19 -t /N/tmp/tmp-CTCFKO -o /N/slate/erikpark/ -j /N/Juicer/scripts/juicer_tools.2.20.00.jar

Where, in order, the arguments say:
/N/bin/utils/hicpro2juicebox.sh = location of script to run
-i /N/CTRL.allValidPairs_original = location of input file 
-g hg19 = genome to use (I used default)
-t /N/tmp/tmp-CTCFKO = location of temporary directory
-o /N/out/ = location of output directory
-j /N/u/Juicer/scripts/juicer_tools.2.20.00.jar = location of juicer tools jar file, from Juicer install

**********  NOTE  ***************
On 2023-05-17 I started a new run of this using two additional files (genome size, and restriction fragment files) newly provided by collaborators who originally ran the HiCPro code. Additionally I also ran this with an older version of Juicer_tools (version 1.19.02) that was successfully used in the past by collaborators. This code is largely similar and is as follows:

> /N/u/erikpark/Carbonate/HiC/HiC-Pro-master/bin/utils/hicpro2juicebox.sh -i /N/slate/erikpark/CTCFKO_allValidPairs -g /N/u/erikpark/Carbonate/HiC/mm10_sorted_genome.fa.fai -t /N/slate/erikpark/tmp/tmp-CTCFKO -o /N/slate/erikpark/hic2juicer-oldjar/ -j /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar -r /N/u/erikpark/Carbonate/HiC/mm10_sorted_genome.fa.mm10_GATC_GANTC_no_chr_with_MT.bed

The only differences are the changed -g (genome size file) and -r (restriction site file) arguments, and the change to the older version of juicer tools

################
# Juicer tools #
################

Then after the .hic files were generated, I annotated and processed them using Juicer Tools, on the cluster.

### HiCCUPS

First I did the chromatin loop finding program (https://github.com/aidenlab/juicer/wiki/HiCCUPS), once for each sample (of CTRL, CTFKO, and TAC). 
I did this using an interactive job on the Carbonate cluster. I started the interactive job with the following command, while logged into the carbonate cluster:

> srun -p gpu -A general --gpus-per-node 1 --time 05:00:00 --pty bash

This code says:
srun = I want to request an interactive session
-p gpu = that is on a compute node that has a gpu
-A general = using a general account (this is a formality, you need to put it for this cluster but it doesn't matter)
--gpus-per-node 1 = I only need one gpu
--time 05:00:00 = I want it to run for up to 5 hours
--pty bash = I will execute the job in the terminal using bash programming

I then ran the following script to actually run the hiccup, loop finding program. This generally followed the following code, with the only differences being the name of the input file, and the output directory:

> module load cudatoolkit
> module load java/1.8.0_131
> module unload gcc # the program only works with versions of gcc ≥ 11, but it loads 12 by default.
> module load gcc/10.2.0
> java -Xmx100g -jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar hiccups /N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic /N/slate/erikpark/hiccups-output-oldjar/CTRL

Where, in order the arguments say: 
java = start java
-Xmx100g = run with 100GB of ram
-jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar = location of the juicer_tools.jar file to use
hiccups = run the hiccups program
/N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic = location of input file
/N/slate/erikpark/hiccups-output-oldjar/CTRL = location of output directory

### Arrowhead

I then ran interactive jobs for the contact domain finding algorithm (https://github.com/aidenlab/juicer/wiki/Arrowhead).

I used the following script (example below) to actually run the algorithm:

> module load cudatoolkit
> module load java/1.8.0_131
> module unload gcc # the program only works with versions of gcc ≥ 11, but it loads 12 by default.
> module load gcc/10.2.0
> java -Xmx100g -jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar arrowhead /N/slate/erikpark/CTRL.allValidPairs_original.hic /N/slate/erikpark/contact-domains-list/CTRL

The arguments are almost identical to those used by hiccups, here the only difference is the program name and the output file directory.


### HiCCUPSDiff

Next I ran HiCCUPSDiff (https://github.com/aidenlab/juicer/wiki/HiCCUPSDiff) to look for differential loops between the two treatments (TAC and CTCFKO) and the control sample.
Syntax was like this:

> java -Xmx100g -jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar hiccupsdiff /N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic /N/slate/erikpark/hic2juicer-oldjar/CTCFKO_allValidPairs.hic /N/slate/erikpark/hiccups-output-oldjar/CTRL/merged_loops.bedpe /N/slate/erikpark/hiccups-output-oldjar/CTCFKO/merged_loops.bedpe /N/slate/erikpark/hiccups-diff-output/CTRL-TAC

where in order:
-jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar = the jar file
hiccupsdiff = run the HiCCUPSDiff command
/N/slate/erikpark/CTRL.allValidPairs_original.hic = the first .hic file to compare
/N/slate/erikpark/TAC.allValidPairs_original.hic = the second .hic file to compare
/N/slate/erikpark/hiccups-output/CTRL/merged_loops.bedpe = the file resulting from the earlier HiCCUPS run, corresponding to the first .hic file
/N/slate/erikpark/hiccups-output/TAC/merged_loops.bedpe = HiCCUPS output file for the second .hic file
/N/slate/erikpark/hiccups-diff-output/CTRL-TAC = output directory

** NOTE though that as of 2023-05-16 this code is NOT working - there seems to be a known issue with the command that has not been addressed on the github (https://github.com/aidenlab/juicer/issues/312)
*** This was fixed by using the older version of juicer_tools (version 1.19.02) and including some extra files in the hicpro2juicebox.sh script. This document has been updated to reflect those changes.


### APA (Aggregate Peak Analysis)

Then I ran APA (https://github.com/aidenlab/juicer/wiki/APA) to plot the peaks different between the treatment samples and the controls. We need to call this twice, once for each of the samples in the hiccupsdiff call. Per the conventions I am using, hiccupsdiff lists the control sample differential loops as '1' (differential_loops1.bedpe) and the treatment as '2' (differential_loops2.bedpe).

** Note that in order to run this we need to mess with Java a little. Before we run the apa command we need to run:
> unset DISPLAY
This seems to be because it is making plots and trying to display them, but there is no display hooked up to the super computer, so it throws errors.

APA Syntax looks like:

> java -Xmx100g -jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar apa /N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic+/N/slate/erikpark/hic2juicer-oldjar/CTCFKO_allValidPairs.hic /N/slate/erikpark/hiccups-diff-output/CTRL-CTCFKO/differential_loops1.bedpe /N/slate/erikpark/APA-output/CTRL-CTCFKO/CTRL-loops 

Where in order the syntax says:
java -Xmx100g = start java
/N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar = run on the old jar file
apa = run apa analysis
/N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic+/N/slate/erikpark/hic2juicer-oldjar/CTCFKO_allValidPairs.hic = point to the two .hic files used in hiccupsdiff. Note the use of the + between them
/N/slate/erikpark/hiccups-diff-output/CTRL-CTCFKO/differential_loops1.bedpe = point to the peaks file, in this case the differential peaks from the control sample
/N/slate/erikpark/APA-output/CTRL-CTCFKO/CTRL-loops = location of output

to then run it for the paired treatment sample (CTCFKO in this case) we would run:

> java -Xmx100g -jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar apa /N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic+/N/slate/erikpark/hic2juicer-oldjar/CTCFKO_allValidPairs.hic /N/slate/erikpark/hiccups-diff-output/CTRL-CTCFKO/differential_loops2.bedpe /N/slate/erikpark/APA-output/CTRL-CTCFKO/CTCFKO-loops 


### Eigenvector analysis 

This analysis will be used to find "compartments" in the .hic files. We will use a loop to look at each chromosome individually. Code looks like this, generally:

# Run in loop for 1 through 19, X, Y
# (might need to name chr1, chr2, ... , chrX, chrY)
# Example for CTRL matrix:


> for i in chr1 chr2 chr3 chr4 chr5 chr6 chr7 chr8 chr9 chr10 chr11 chr12 chr13 chr14 chr15 chr16 chr17 chr18 chr19 chrY chrX
	do
 	 current_chr=i
  	java -Xmx100g -jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar eigenvector KR /N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic $i BP 500000 /N/slate/erikpark/Eigen-output/CTRL/CTRL_${i}_eigenvector.txt
	done

This code sets up a for loop where it reads in elements of the .hic file for each chromosome, and then executes the eigenvector analysis at a resolution of 500000, and saves the results for each chromosome individually in a text file with the chromosome that was queried as the title (ex. CTRL_chr1_eigenvector.txt as the output for the eigenvector analysis for chromosome 1).


### APA (Aggregate Peak Analysis) in individual samples

Next, we wanted to do APA in specific samples, to find loops which are detected in one group and not others - for example in the control group but not in CTCF or TAC.
To do this, we used the following strategy as outlined by a collaborator:
"You may simply run the pairwise analyses one at a time. For example, hiccupsdiff between CTRL and CTCFKO conditions will result in CTRL-only loops and CTCF-only loops. Then you can visualize the CTRL and CTCFKO signal in these loop sets, which can be achieved through separate APA analyses:

1) Use CTRL only loops, plot CTRL signal using APA
2) Use CTRL only loops, plot CTCFKO signal using APA
3) Use CTCFKO only loops, plot CTCFKO signal using APA
4) Use CTCFKO only loops, plot CTRL signal using APA"

In practice, this looks like the following code, to plot CTRL only loops from the CTRL only loops:

> java -Xmx100g -jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar apa /N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic /N/slate/erikpark/hiccups-diff-output/CTRL-CTCFKO/differential_loops1.bedpe /N/slate/erikpark/APA-output-individual-samples/CTRL-signals/CTRL-CTCFKO/CTRL-loops

Where in order the syntax says:
java -Xmx100g = start java
/N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar = run on the old jar file
apa = run apa analysis
/N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic = point to the .hic file of interest that was used in hiccupsdiff. Note here that I am just calling the CTRL .hic file.
/N/slate/erikpark/hiccups-diff-output/CTRL-CTCFKO/differential_loops1.bedpe = point to the peaks file, in this case the differential peaks from the control sample
/N/slate/erikpark/APA-output-individual-samples/CTRL-signals/CTRL-CTCFKO/CTRL-loops = location of output, here for the plots of CTRL signal from the CTRL only loops, from specifically the differential analysis comparison between CTRL and CTCFKO samples

* NOTE that for the CTRL-TAC/TAC specific differential expression loops, there was nothing found at the 25000 resolution. So when we do this APA analysis on the TAC loops file, we need to add the resolution flag to exclude the 25000 resolution, like -r 10000, 5000


### HiCCUPSDiff to find loops only in CTRL samples

Now we want to find loops specific to just the CTRL sample, not seen in TAC or CTCFKO.
To do this, we are starting with the CTRL-CTCFKO hiccups-diff-output/CTRL-CTCFKO/differential_loops1.bedpe file - this is the list of loops found in CTRL and not CTCFKO. We will compare this list against the list of TAC loops (seen at hiccups-output-oldjar/TAC/merged_loops.bedpe) to find loops present in CTRL but not CTCFKO or TAC.
Code to do this looks like the following:

> java -Xmx100g -jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar hiccupsdiff /N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic /N/slate/erikpark/hic2juicer-oldjar/TAC.allValidPairs_original.hic /N/slate/erikpark/hiccups-diff-output/CTRL-CTCFKO/differential_loops1.bedpe /N/slate/erikpark/hiccups-output-oldjar/TAC/merged_loops.bedpe /N/slate/erikpark/Individual-loops/CTRL-only-loops

where in order:
-jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar = the jar file
hiccupsdiff = run the HiCCUPSDiff command
/N/slate/erikpark/CTRL.allValidPairs_original.hic = the first .hic file to compare, in this case the control file
/N/slate/erikpark/TAC.allValidPairs_original.hic = the second .hic file to compare, the TAC file that we are looking for loops to exclude
/N/slate/erikpark/hiccups-diff-output/CTRL-CTCFKO/differential_loops1.bedpe = the file resulting from the earlier HiCCUPSDiff run, this corresponds to the loops found in CTRL but not CTCFKO
/N/slate/erikpark/hiccups-output/TAC/merged_loops.bedpe = HiCCUPS output file for the second .hic file, the loops seen in TAC
/N/slate/erikpark/Individual-loops/CTRL-only-loops = output directory


### APA to plot loops only in CTRL samples

Now we are going to use APA to plot the CTRL only loops found in the step above
Code looks like:

> java -Xmx100g -jar /N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar apa /N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic /N/slate/erikpark/Individual-loops/CTRL-only-loopsdifferential_loops1.bedpe /N/slate/erikpark/Individual-loops/CTRL-only-loops/CTRL-only-APA-plots 

Where in order the syntax says:
java -Xmx100g = start java
/N/u/erikpark/Carbonate/Juicer/scripts/juicer_tools_1.19.02.jar = run on the old jar file
apa = run apa analysis
/N/slate/erikpark/hic2juicer-oldjar/CTRL.allValidPairs_original.hic = point to the .hic file for the sample of interest, here the CTRL file, because we just want to plot the CTRL only loops.
/N/slate/erikpark/Individual-loops/CTRL-only-loopsdifferential_loops1.bedpe = point to the loops file, in this case the loops found just from the control sample
/N/slate/erikpark/Individual-loops/CTRL-only-loops/CTRL-only-APA-plots = location of output


### HiCCUPSDiff to find loops in common between samples TAC and CTCFKO samples, but not seen in CTRL

Here we needed to get a bit creative. 
First we run HiCCUPSDiff on TAC vs. CTCFKO, to find loops unique to one conditon. 
Then I loaded the original loops list of one of the samples (I used TAC), and the list of loops unique to TAC (result of HiCCUPSDiff output described on prior line) into R. I then found which loops were present in the main loop list, but NOT present in the unique list. This result is hypothetically the loops which are shared between TAC and CTCFKO.
From there I then ran HiCCUPSDiff again on the new "shared" loops list I manually made, against the CTRL .hic and loop list, to find which loops were in TAC and CTCFKO but not in control.
