# TensorFlow-Inception-IO-Analysis

##  Model
We use TensorFlow's Inception network which is an image recognition network for academic purpose developed by Google.
"Rethinking the Inception Architecture for Computer Vision"  C. Szegedy, V. Vanhoucke, S. Ioffe, J. Shlens, Z. Wojna, 2015, http://arxiv.org/abs/1512.00567

Moreover we use the TensorFlows's Slim model which offers an easy to use frontend to work with Inception.
"TensorFlow-Slim image classification model library" N. Silberman and S. Guadarrama, 2016. https://github.com/tensorflow/models/tree/master/research/slim 

Both models can be found in the TensorFlow Github-repository for TensorFlow models: https://github.com/tensorflow/models/

##TensorFlow-Environment
SCS5 node (z.b. tauruslogin6)
```
module load TensorFlow/1.10.0-fosscuda-2018b-Python-3.6.6
```

## Dataset
We use the Image-Net dataset from (http://image-net.org) 
It's a large dataset of about 140GiB.

## Installation
```    
git clone https://github.com/tensorflow/models/
cd models 
git checkout r1.10.0
git checkout b07b494 # last release commit removes only research folder
cd research/slim
```

### Download the dataset	
The dataset is available under: `/lustre/ssd/p_tensorflow/imagenet-dataset/tfrecords/`

If you wish to download the dataset by yourself you have to create an account on [Image-Net](http://image-net.org/).
Since the data will be preprocessed using TensorFlow you have make sure TensorFlow is available in your Python environment. 


```
module load TensorFlow/1.10.0-fosscuda-2018b-Python-3.6.6

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
Since the dataset is large and the download server is rather slow (50kb/s), which leads to problem on Taurus-Cluster with Slurm job scheduling and maximal runtime of jobs I recommend to modify the script.
As wget has no native support for downloading a single file in parallel I used [Axel](https://github.com/axel-download-accelerator/axel). It can be found in `/projects/p_tensorflow/sw/axel/bin/axel`.

In `inception/data/download_imagenet.sh` replace every `wget` call  with 
`axel -n {VERY_LARGE_NUMBER_OF_THREADS e.g. } {URL}`

It is recommand to request about 16 cpu cores on slurm to run it as `tar` will also speed up.

After this the  dataset in TFRecord-format which is used by Slim and Inception can be found unter `DATA_DIR/tfrecords`.
	
## Usage
There has to be at least 1 gpu be available.

### Training from scratch
```
DATASET_DIR=/lustre/ssd/p_tensorflow/imagenet-dataset/tfrecords
TRAIN_DIR=/tmp/s8916149/inception_v3

python train_image_classifier.py \
--train_dir=${TRAIN_DIR} \
--dataset_dir=${DATASET_DIR} \
--dataset_name='imagenet' \
--dataset_split_name=train \
--model_name=inception_v3 \
--max_number_of_steps=2000
```

Training the net to an appropriate state would take weeks of processing, so we limit the number of learning iteration with the parameter `--max_number_of_steps`.

### Evaluating an existing graph 
 to be done
  

### Fine-Tune a Pre-Trained Model on a New Task
 to be done
 

## Profiling / Tracing with Scorep
### Lo2s
[https://github.com/tud-zih-energy/lo2s](https://github.com/tud-zih-energy/lo2s)

```
module load lo2s/1.1.1-GCCcore-6.4.0
srun lo2s -- python ../train_image_classifier.py \
--train_dir [....]
```

### Lo2s Lustre-Counter
```
module load scorep_plugin_fileparser
module load lo2s
module load TensorFlow/1.10.0-fosscuda-2018b-Python-3.6.6
 
grep_line_num() {
line_num=$(grep -m 1 -n "$2" $1 | sed "s/^\([0-9]\\+\):.*/\\1/;");
   if [ -n ${line_num} ]; then
       echo $((${line_num} - 1));
   fi;
}
 
iostats_file="$(srun find /proc/fs/lustre/llite -type d -iname "highiops-*")/stats";
filter=""
filter+="read bytes:uint@${iostats_file}+c=6;r=$(grep_line_num "${iostats_file}" "read_bytes");s= ;d,";
filter+="write bytes:uint@${iostats_file}+c=6;r=$(grep_line_num "${iostats_file}" "write_bytes");s= ;d,";
filter+="open:uint@${iostats_file}+c=1;r=$(grep_line_num "${iostats_file}" "open");s= ;d,";
filter+="close:uint@${iostats_file}+c=1;r=$(grep_line_num "${iostats_file}" "close");s= ;d,";
filter+="page_fault:uint@${iostats_file}+c=1;r=$(grep_line_num "${iostats_file}" "page_fault");s= ;d";

 
export SCOREP_ENABLE_TRACING=true;
export SCOREP_ENABLE_PROFILING=false;
export SCOREP_METRIC_FILEPARSER_PLUGIN_PERIOD=10000;
export SCOREP_METRIC_FILEPARSER_PLUGIN="${filter}";
 

srun lo2s -- python ../train_image_classifier.py \
```
Make sure both `srun` commands will be executed on the same node.

### Score-P
You have to build your own setup. For detailed information see [BuildScorepOnSCS5.md](BuildScorepOnSCS5.md)

