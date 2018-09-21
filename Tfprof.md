# Profile TennsorFlow Application using tfprof
Tfprof profiler and advisor for TensorFlow  application distributed and maintained by TensorFlow's developers.
For further information about tfprof use the project page on [Github](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/core/profiler)
## Installation
Since tfprof is not packages with tensorflow on SCS5 you have to install it on your own.

```
git clone https://github.com/tensorflow/tensorflow.git
cd tensorflow

module load TensorFlow/1.10.0-fosscuda-2018b-Python-3.6.6 Bazel/0.16.0-GCCcore-7.3.0

bazel --output_user_root=/projects/p_tensorflow/sw/TensorFlow-1.10.0-Utils-fosscuda-2018b-Python-3.6.6/bazel/ build  tensorflow/core/profiler:profiler
```

Use the following modfile
```
#%Module1.0######################################################################
##
## This is a modules template. Adept it to your requirements.
## Module file for <software xyz>
## 

## Source zih-modules helper script. It provides the framework for an uniform modules system.
## Additionally, it sets global variables that provide you information about the 
## application to be loaded and some other information. See the following list:
##
## soft_arch      Provides information about the architecture.
## soft_class     Provides the software class. E.g. [applications|compiler|tools]
## soft_host      Provides the host name.
## soft_machine   Provides the machine name.
## soft_version   Provides the software version number to be loaded. 
## soft_ware      Provides the software name to be loaded.
## user           Provides the user name.

source /sw/modules/global/modulescripts/zih_modules.tcl

set tfprof_root /projects/p088/p_tensorflow/sw/TensorFlow-1.10.0-Utils-fosscuda-2018b-Python-3.6.6/bazel/99590a2b09501727d433b0894530308c/execroot/org_tensorflow/bazel-out/k8-opt/bin/tensorflow/core/profiler
     

## Set modules dependencies with version information !!! e.g. intel/11.1.069

#set soft_dependencies " "

switch $soft_machine {
   "taurus"  {
       append soft_dependencies "TensorFlow/1.10.0-fosscuda-2018b-Python-3.6.6"
   }
}

## Set variable shortDescription if you want to add specific information that 
## should be displayed while the module is loaded.

#set shortDescription "Use \"bsub -n<number-of-cpus> ...\"  to start the application xyz"


## Set variable longDescription if you want to add further information about the 
## software that should be displayed at the module help command.

set longDescription "scorep is a module that allows tracing of python scripts using Score-P."

append longDescription "\nRun with python m scorep  ."


## Set the specific environment variables for your software package

#set lib_path /sw/<global|$soft_machine>/$soft_class/$soft_ware/$soft_version/$soft_arch

#environment settings for Score-P

prepend-path PATH $tfprof_root/


## Source modules action information
source /sw/modules/global/modulescripts/zih_modules_action.tcl
```


## Using Tfprof
### Modify your code to create a profile
Except taken from [Github](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/core/profiler)

Usuall you can go with the first method to get a full profile.
```
# When using high-level API, session is usually hidden.
#
# Under the default ProfileContext, run a few hundred steps.
# The ProfileContext will sample some steps and dump the profiles
# to files. Users can then use command line tool or Web UI for
# interactive profiling.
with tf.contrib.tfprof.ProfileContext('/tmp/train_dir') as pctx:
  # High level API, such as slim, Estimator, etc.
  train_loop()
```

If want to have a fine tuned profile use one of these methods

```
# When using lower-level APIs with a Session object. User can have
# explicit control of each step.
#
# Create options to profile the time and memory information.
builder = tf.profiler.ProfileOptionBuilder
opts = builder(builder.time_and_memory()).order_by('micros').build()
# Create a profiling context, set constructor argument `trace_steps`,
# `dump_steps` to empty for explicit control.
with tf.contrib.tfprof.ProfileContext('/tmp/train_dir',
                                      trace_steps=[],
                                      dump_steps=[]) as pctx:
  with tf.Session() as sess:
    # Enable tracing for next session.run.
    pctx.trace_next_step()
    # Dump the profile to '/tmp/train_dir' after the step.
    pctx.dump_next_step()
    _ = session.run(train_op)
    pctx.profiler.profile_operations(options=opts)
```

```
# For more advanced usage, user can control the tracing steps and
# dumping steps. User can also run online profiling during training.
#
# Create options to profile time/memory as well as parameters.
builder = tf.profiler.ProfileOptionBuilder
opts = builder(builder.time_and_memory()).order_by('micros').build()
opts2 = tf.profiler.ProfileOptionBuilder.trainable_variables_parameter()

# Collect traces of steps 10~20, dump the whole profile (with traces of
# step 10~20) at step 20. The dumped profile can be used for further profiling
# with command line interface or Web UI.
with tf.contrib.tfprof.ProfileContext('/tmp/train_dir',
                                      trace_steps=range(10, 20),
                                      dump_steps=[20]) as pctx:
  # Run online profiling with 'op' view and 'opts' options at step 15, 18, 20.
  pctx.add_auto_profiling('op', opts, [15, 18, 20])
  # Run online profiling with 'scope' view and 'opts2' options at step 20.
  pctx.add_auto_profiling('scope', opts2, [20])
  # High level API, such as slim, Estimator, etc.
  train_loop()
```

### Analyse using tfprof CLI
```
profiler --profile_path=/tmp/train_dir/profile_100
```

