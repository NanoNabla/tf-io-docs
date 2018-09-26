# TensorFlow-Inception-IO-Analysis

##  Model
We use TensorFlow's Inception network which is an image recognition network for academic purpose developed by Google.
"Rethinking the Inception Architecture for Computer Vision"  C. Szegedy, V. Vanhoucke, S. Ioffe, J. Shlens, Z. Wojna, 2015, http://arxiv.org/abs/1512.00567

Moreover we use the TensorFlows's Slim model which offers an easy to use frontend to work with Inception.
"TensorFlow-Slim image classification model library" N. Silberman and S. Guadarrama, 2016. https://github.com/tensorflow/models/tree/master/research/slim 

Both models can be found in the TensorFlow Github-repository for TensorFlow models: https://github.com/tensorflow/models/

##TensorFlow-Environment
Pre SCS5 node (z.b. tauruslogin3)
```
module load modenv/both
module load tensorflow/1.3.0-Python-3.5.2
```

## Dataset
We use the dataset from
It's a large dataset of about 140GiB.

## Installation
    git clone https://github.com/tensorflow/models/
    cd models/research/slim

### Download the dataset	
The dataset is available under: `//lustre/ssd/p_tensorflow/imagenet-dataset/tfrecords/`

If you wish to download the dataset by yourself you have to create an account on [Image-Net](http://image-net.org/).
Since the data will be preprocessed using TensorFlow you have make sure TensorFlow is available in your Python environment. 


```
module load modenv/both
modu load tensorflow/1.3.0-Python-3.5.2 bazel/0.5.2 java/jdk1.8.0_66

cd models/research/inception
# location of where to place the ImageNet data
DATA_DIR=/lustre/ssd/p_tensorflow/imagenet-dataset/

# build the preprocessing script.
cd tensorflow-models/inception
bazel build //inception:download_and_preprocess_imagenet

export IMAGENET_USERNAME={YOUR_USERNAME}
export IMAGENET_ACCESS_KEY={YOUR_ACCESS_KEY}

# run it
bazel-bin/inception/download_and_preprocess_imagenet "$
{DATA_DIR}"

# clean up
mkdir ${DATA_DIR}/tfrecords
mv ${DATA_DIR}/train-* ${DATA_DIR}/validation-00* ${DATA_DIR}/tfrecords
```
Since the dataset is large and the download server is rather slow (50kb/s), which leads to problem on Taurus-Cluster with Slurm job sheduling and maximal runtime of jobs we recommand to modify the script.
As wget has no native support for downloading a single file in parallel we used [Axel](https://github.com/axel-download-accelerator/axel). It can be found in `/projects/p_tensorflow/sw/axel/bin/axel`
.
In `inception/data/download_imagenet.sh` replace every `wget` call  with 
```axel -n {VERY_LARGE_NUMBER_OF_THREADS e.g. } {URL}```

It is recommand to request about 16 cpu cores on slurm to run it as `tar` will also speed up.

After this the  dataset in TFRecord-format which is used by Slim and Inception can be found unter `DATA_DIR/tfrecords`.
	
## Usage
Es muss mindestens eine GPU verfügbar sein.
### Training from scratch
```
DATASET_DIR=/lustre/ssd/p_tensorflow/imagenet-dataset/tfrecords
TRAIN_DIR=/tmp/s8916149/inception_v3

python train_image_classifier.py \
--train_dir=${TRAIN_DIR}\
--dataset_dir=${DATASET_DIR} \
--dataset_name='imagenet' \
--dataset_split_name=train \
--model_name=inception_v3 \
--max_number_of_steps=200
```

Training the net to an appropriate state would take weeks of processing, so we limit the number of learning iteration with the parameter `--max_number_of_steps`.

### Evaluating an existing graph 
 to be done
 
 
 ### Fine-Tune a Pre-Trained Model on a New Task
 to be done
 
 ## Profiling / Tracing with Scorep