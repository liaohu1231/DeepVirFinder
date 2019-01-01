# DeepVirFinder: Identifying viruses from metagenomic data by deep learning

Version: 1.0

Authors: Jie Ren, Kai Song, Chao Deng, Nathan Ahlgren, Jed Fuhrman, Yi Li, Xiaohui Xie, Ryan Poplin, Fengzhu Sun

Maintainer: Jie Ren renj@usc.edu, Chao Deng chaodeng@usc.edu


## Description

DeepVirFinder predicts viral sequences using deep learning method. 
The method has good prediction accuracy for short viral sequences, 
so it can be used to predict sequences from the metagenomic data.

DeepVirFinder significantly improves the prediction accuracy compared to our k-mer based method VirFinder by using convolutional neural networks (CNN).
CNN can automatically learn genomic patterns from the viral and prokaryotic sequences and simultaneously build a predictive model based on the learned genomic patterns. 
The learned patterns are represented in the form of weight matrices of size 4 by k, where k is the length of the pattern. 
This representation is similar to the position weight matrix (PWM), the commonly used representation of biological motifs, 
which are also of size 4 by k and each column specifies the probabilities of having the 4 nucleotides at that position.
When only one type of nucleotide can be chosen at each position with probability 1, the motif degenerates to a k-mer. 
Thus, the CNN is a natural generalization of k-mer based model. 
The more flexible CNN model indeed outperforms the k-mer based model on viral sequence prediction problem.


## Dependencies

DeepVirFinder requires Python 3.6 with the packages of numpy, theano, keras, scikit-learn, and Biopython.
We recommand the use [Miniconda](https://conda.io/miniconda.html) to install all dependencies. 
After installing Miniconda, simply run


    conda install python=3.6 numpy theano keras scikit-learn Biopython
    
or create a virtual environment

    conda create --name dvf python=3.6 numpy theano keras scikit-learn Biopython
    source activate dvf



## Installation

Download the package by 

    git clone https://github.com/jessieren/DeepVirFinder
    cd DeepVirFinder
    
    
## Usage

The input of DeepVirFinder is the fasta file containing the sequences to predict, 
and the output is a .txt file containing the predicted score and p-value for each of the input sequences. 
The higher score or lower p-value indicate higher likelihood of being a viral sequence. 
The p-value is compuated by comparing the predicted score with the null distribution for prokaryotic sequences. 

The output file will be in the same directory as the input file by default. Users can also specify the output directory by the option [-o].
The option [-l] is for setting a minimun sequence length threshold so that sequences shorter than this threshold will not be predicted.
The program also supports parallel computing. Using [-c] to specify the number of threads to use. 
The option [-m] is for specifying the directory to the models. The default model directory is ./models, which contains the models we trained and used in the paper.


    python dvf.py [-i INPUT_FA] [-o OUTPUT_DIR] [-l CUTOFF_LEN] [-c CORE_NUM]


#### Options
      -h, --help            show this help message and exit
      -i INPUT_FA, --in=INPUT_FA
                            input fasta file
      -m MODDIR, --mod=MODDIR
                            model directory (default ./models)
      -o OUTPUT_DIR, --out=OUTPUT_DIR
                            output directory
      -l CUTOFF_LEN, --len=CUTOFF_LEN
                            predict only for sequence >= L bp (default 1)
      -c CORE_NUM, --core=CORE_NUM
                            number of parallel cores (default 1)


## Examples

#### Predicting the crAssphage genome

    python dvf.py -i ./test/crAssphage.fa -o ./test/ -l 300
    
     
#### Predicting a set of metagenomically assembled contigs
    
    python dvf.py -i ./test/CRC_meta.fa -l 1000 -c 2
    

#### If you would like to compute q-values (false discovery rate), please use the R package "qvalue". 

  To install the package "qvalue" in R:

  ```
  # try http:// if https:// URLs are not supported; it also checks for out-of-date packages
  source("https://bioconductor.org/biocLite.R")
  biocLite("qvalue")
  ```
  
  To compute the q-values, load the package and call the function 'qvalue'. For example, 

  ```
  # load the package qvalue
  library(qvalue)

  # read the prediction results
  result <- read.csv("./test/CRC_meta.fa_gt1000bp_dvfpred.txt", sep='\t')

  # estimate q-values (false discovery rates) based on p-values
  result$qvalue <- qvalue(result$pvalue)$qvalues

  # sort sequences by q-value in ascending order
  result[order(result$qvalue),]
  ```

## Training the model using customized dataset

If users are interested in training a new deep learning model using their own dataset, 
we provide the scripts for processing the genomic data and training the model. 
Four fasta files are needed for training the model: 
  1. the host genomic sequences for training, 
  2. the host genomic sequences for validation, 
  3. the virus genomic sequences for training, and 
  4. the virus genomic sequences for validation.
 
The script encode.py processes the input genomic sequences by fragmenting them into fixed length sequences [-l], 
and encoding them by one-hot encoding method. The contig type [-p] indicates the type of the sequences, virus or host. 
This indicator will be encoded into the file name and will be used in the following steps for data type recognition.

#### Options
      -h, --help            show this help message and exit
      -i FILENAME, --fileName=FILENAME
                            fileName
      -l CONTIGLENGTH, --contigLength=CONTIGLENGTH
                            contigLength
      -p CONTIGTYPE, --contigType=CONTIGTYPE
                            contigType, virus or host

The script training.py takes the encoded sequences and trains a deep learning model for classifying viruses from hosts. 
The directory of the encoded training data [-i] and the directory of the encoded validation data [-j] need to be specified. 
Hyperparameters of the deep learning model include the number of filters in the convolutional layer [-n], the length of the filter [-f], and the number of neurons in the dense layer [-d]. 
Since viral sequences in real data can be of various lengths, we train multiple models using sequences of different lengths, say 150, 300, 500, 1000 bp for predicting sequences of different length range. The option [-l] specifies the length of the sequences used for training. 

#### Options
      -h, --help            show this help message and exit
      -l CONTIGLENGTH, --len=CONTIGLENGTH
                            contig Length
      -i INDIRTR, --intr=INDIRTR
                            input directory for training data
      -j INDIRVAL, --inval=INDIRVAL
                            input directory for validation data
      -o OUTDIR, --out=OUTDIR
                            output directory
      -f FILTER_LEN1, --fLen1=FILTER_LEN1
                            the length of the filter
      -n NB_FILTER1, --fNum1=NB_FILTER1
                            number of filters in the convolutional layer
      -d NB_DENSE, --dense=NB_DENSE
                            number of neurons in the dense layer
      -e EPOCHS, --epochs=EPOCHS
                            number of epochs

### Example

We prepared an example for a test:

    # encoding sequences for training and validation (may take about 5 minutes)
    for l in 150 300 500 1000 
    do 
    python encode.py -i ./train_example/tr/host_tr.fa -l $l -p host
    python encode.py -i ./train_example/tr/virus_tr.fa -l $l -p virus
    
    python encode.py -i ./train_example/val/host_val.fa -l $l -p host
    python encode.py -i ./train_example/val/virus_val.fa -l $l -p virus
    done

    # training the model for different contig lengths
    # Using GPU, in total the process takes about 30 minutes
    source /<path_to_cuda_setup>/setup.sh
    source /<path_to_cuDNN_setup>/setup.sh
    for l in 150 300 500 1000 
    do 
    THEANO_FLAGS='mode=FAST_RUN,device=cuda0,floatX=float32,GPUARRAY_CUDA_VERSION=80' srun python training.py -l $l -i ./train_example/tr/encode -j ./train_example/val/encode -o ./train_example/models -f 10 -n 500 -d 500 -e 10
    done
    
The trained models will be saved in the output directory. To predict sequences using the newly trained model, specify the model directory using the option -m,
    
    python dvf.py -i ./test/crAssphage.fa -o ./train_example/test -l 300 -m ./train_example/models



<!-- Copyright and License Information
-----------------------------------

Copyright (C) 2017 University of Southern California

Authors: Jie Ren, Fengzhu Sun

This program is freely available under the terms of the GNU General Public (version 3) as published by the Free Software Foundation (http://www.gnu.org/licenses/#GPL) for academic use. 

Commercial users should contact Dr. Sun at fsun@usc.edu, copyright at the University of Southern California. 

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details. -->

<!--You should have received a copy of the GNU General Public License along with this program. If not, see http://www.gnu.org/licenses/.-->

