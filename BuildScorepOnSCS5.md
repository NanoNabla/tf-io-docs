# Building Scorep-IO for Tensorflow on SCS5 for Tracing TensorFlow
Using scorep with tensorflow is nothing special towards using it with any other python application. So you can use this explaination to analyze all applications build with python.

* [Build](#build)
    * [Build Dev-Tools](#build-dev-tools)
    * [Build OTF2](#build-otf2)
    * [Build Scorep](#build-scorep)
    * [Build Scorep Python Binding](#build-scorep-python-binding)
* [Usage](#usage)
    * [Without using IO support](#without-using-io-support)
     * [Using IO support](#using-io-support)
     * [Known problems](#known-problems)


## Build
This aims to build Scorep with Python support to use with TensorFlow

```
svn co https://silc.zih.tu-dresden.de/svn/silc-root/branches/TRY_TUD_io_recording_for_trunk_reintegration 
svn up -r 13824
```

```
module load  PAPI/5.6.0-foss-2018b fosscuda/2018b libunwind/1.2.1-foss-2018b OPARI2/2.0.3-foss-2018b CubeLib/4.4-foss-2018b CubeW/4.4-foss-2018b Python/3.6.6-fosscuda-2018b
```

### Build Dev-Tools
There is no dev-tools module on SCS5 available. You have build them on your own

```
wget https://silc.zih.tu-dresden.de/trac-silc/attachment/wiki/BuildSystem/scorep-dev-06.tar.gz
tar xfvz scorep-dev-06.tar.gz
cd scorep-dev-06.tar.gz
./install-scorep-dev.06.sh --prefix=/projects/p_tensorflow/sw/scorep-dev/
```
Create a modfile

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
## soft_arch	  Provides information about the architecture.
## soft_class     Provides the software class. E.g. [applications|compiler|tools]
## soft_host	  Provides the host name.
## soft_machine   Provides the machine name.
## soft_version   Provides the software version number to be loaded.
## soft_ware	  Provides the software name to be loaded.
## user           Provides the user name.

source /sw/modules/global/modulescripts/zih_modules.tcl

set scorep_dev_root    /projects/p_tensorflow/sw/scorep-dev

## Set modules dependencies with version information !!! e.g. intel/11.1.069

#set soft_dependencies " "

## Set variable shortDescription if you want to add specific information that
## should be displayed while the module is loaded.

#set shortDescription "Use \"bsub -n<number-of-cpus> ...\"  to start the application xyz"


## Set variable longDescription if you want to add further information about the
## software that should be displayed at the module help command.

set longDescription "This $soft_ware provides requirements of the Score-P build system."

## Set the specific environment variables for your software package

#set lib_path /sw/<global|$soft_machine>/$soft_class/$soft_ware/$soft_version/$soft_arch

prepend-path LD_LIBRARY_PATH $scorep_dev_root/lib
prepend-path PATH $scorep_dev_root/bin


## Source modules action information
source /sw/modules/global/modulescripts/zih_modules_action.tcl
```


```
module use 
module load scorep-dev/06
```

### Build OTF2
Since TensorFlow required Python 3 and there are only OTF2 builds requiring Python 2.7 there will be a conflict in the module system. Therefore you have to build your own OTF2

```
./bootstrap -j 24
mkdir build-otf2
cd build-otf2
../vendor/otf2/configure --prefix=/projects/p_tensorflow/sw/OTF2-Python3.6 --with-nocross-compiler-suite=gcc
make -j 24
make install
```
I copied the original modfile and change root path and python dependency.

```
help([==[

Description
===========
The Open Trace Format 2 is a highly scalable, memory efficient event
 trace data format plus support library. It is the new standard trace format for
 Scalasca, Vampir, and TAU and is open for other tools.


More information
================
 - Homepage: http://www.score-p.org
]==])

whatis([==[Description: The Open Trace Format 2 is a highly scalable, memory efficient event
 trace data format plus support library. It is the new standard trace format for
 Scalasca, Vampir, and TAU and is open for other tools.]==])
whatis([==[Homepage: http://www.score-p.org]==])

local root = "/projects/p_tensorflow/sw/OTF2-Python3.6"

conflict("OTF2")

depends_on("fosscuda/2018b")

depends_on("Python/3.6.6-fosscuda-2018b")

-- depends_on("future/0.16.0-foss-2018b-Python-2.7.15")

-- depends_on("Six/1.11.0-foss-2018b-Python-2.7.15")

prepend_path("CPATH", pathJoin(root, "include"))
prepend_path("LD_LIBRARY_PATH", pathJoin(root, "lib"))
prepend_path("LIBRARY_PATH", pathJoin(root, "lib"))
prepend_path("PATH", pathJoin(root, "bin"))
setenv("EBROOTOTF2", root)
setenv("EBVERSIONOTF2", "2.1.1")
setenv("EBDEVELOTF2", pathJoin(root, "easybuild/OTF2-2.1.1-foss-2018b-easybuild-devel"))
```

### Build Scorep
```
module use /projects/p_tensorflow/modenv-scs5/
module load  PAPI/5.6.0-foss-2018b fosscuda/2018b libunwind/1.2.1-foss-2018b OPARI2/2.0.3-foss-2018b CubeLib/4.4-foss-2018b CubeW/4.4-foss-2018b Python/3.6.6-fosscuda-2018b
module load OTF2/Python3.6 scorep-dev/06

./bootstrap -j 24

mkdir build
cd build

  ../configure                  '--prefix=/projects/p_tensorflow/sw/scorep-io2-foss-2018b' \
                                '--with-nocross-compiler-suite=gcc' \
                                '--with-mpi=openmpi' \
                                '--with-libcudart=/sw/installed/CUDA/9.2.88-GCC-7.3.0-2.30' \
                                '--disable-silent-rules' \
                                '--with-otf2=/projects/p_tensorflow/sw/OTF2-Python3.6/bin' \
                                '--with-opari2=/sw/installed/OPARI2/2.0.3-foss-2018b/bin' \
                                '--with-cubew=/sw/installed/CubeW/4.4-foss-2018b' \
                                '--with-cubelib=/sw/installed/CubeLib/4.4-foss-2018b' \
                                '--with-papi-header=/sw/installed/PAPI/5.6.0-foss-2018b/include/' \
                                '--with-papi-lib=/sw/installed/PAPI/5.6.0-foss-2018b/lib' \
                                '--with-libunwind=/sw/installed/libunwind/1.2.1-foss-2018b' \
                                '--with-machine-name=taurus.hrsk.tu-dresden.de' \
                                '--without-gui' \
                                '--enable-static' \
                                '--enable-shared'

make -j24
make install
```

```
help([==[

Description
===========
The Score-P measurement infrastructure is a highly scalable and
 easy-to-use tool suite for profiling, event tracing, and online analysis of HPC
 applications.


More information
================
 - Homepage: http://www.score-p.org
]==])

whatis([==[Description: The Score-P measurement infrastructure is a highly scalable and
 easy-to-use tool suite for profiling, event tracing, and online analysis of HPC
 applications.]==])
whatis([==[Homepage: http://www.score-p.org]==])

local root = "/projects/p_tensorflow/sw/scorep-io2-foss-2018b"

conflict("Score-P")

depends_on("libunwind/1.2.1-foss-2018b")

depends_on("PAPI/5.6.0-foss-2018b")

depends_on("CubeLib/4.4-foss-2018b")

depends_on("CubeW/4.4-foss-2018b")

depends_on("OPARI2/2.0.3-foss-2018b")

depends_on("OTF2/Python3.6")

depends_on("fosscuda/2018b")

prepend_path("CPATH", pathJoin(root, "include"))
prepend_path("LD_LIBRARY_PATH", pathJoin(root, "lib"))
prepend_path("LIBRARY_PATH", pathJoin(root, "lib"))
prepend_path("PATH", pathJoin(root, "bin"))
setenv("EBROOTSCOREMINP", root)
setenv("EBVERSIONSCOREMINP", "4.0")
setenv("EBDEVELSCOREMINP", pathJoin(root, "easybuild/Score-P-4.0-foss-2018b-easybuild-devel"))

prepend_path("CUBE_DOCPATH", pathJoin(root, "share/doc/scorep/profile"))
```
### Build Scorep Python Binding
The original Python Binding developed by [Andreas Gocht](https://github.com/score-p/scorep_binding_python) is working with IO tracking. 
If you need IO data you have to use the fork developed by [Christian Herold](https://github.com/harryherold/scorep_binding_python)
The installation procedure is same for both.

```
git clone https://github.com/score-p/scorep_binding_python.git
# For IO support use
#git clone https://github.com/harryherold/scorep_binding_python 
cd scorep_binding_python

module load CMake/3.11.4-GCCcore-7.3.0
python setup.py install --prefix=/projects/p_tensorflow/sw/scorep-binding-python-cs5/
```

Create modfile

```
help([==[

Description
===========
The Score-P measurement infrastructure is a highly scalable and
 easy-to-use tool suite for profiling, event tracing, and online analysis of HPC
 applications.


More information
================
 - Homepage: http://www.score-p.org
]==])

whatis([==[Description: The Score-P measurement infrastructure is a highly scalable and
 easy-to-use tool suite for profiling, event tracing, and online analysis of HPC
 applications.]==])
whatis([==[Homepage: http://www.score-p.org]==])

local root = "/projects/p_tensorflow/sw/scorep-binding-python-cs5"

--conflict("Score-P")

depends_on("Score-P/io-foss-2018b")

depends_on("Python/3.6.6-fosscuda-2018b")

--prepend_path("CPATH", pathJoin(root, "include"))
--prepend_path("LD_LIBRARY_PATH", pathJoin(root, "lib"))
--prepend_path("LIBRARY_PATH", pathJoin(root, "lib"))
--prepend_path("PATH", pathJoin(root, "bin"))
--setenv("EBROOTSCOREMINP", root)
--setenv("EBVERSIONSCOREMINP", "4.0")
--setenv("EBDEVELSCOREMINP", pathJoin(root, "easybuild/Score-P-4.0-foss-2018b-easybuild-devel"))

--prepend_path("CUBE_DOCPATH", pathJoin(root, "share/doc/scorep/profile"))

prepend_path("PYTHONPATH", pathJoin(root, "lib/python3.6/site-packages"))
prepend_path("LD_LIBRARY_PATH", pathJoin(root, "lib"))
```

## Usage
Since the native python binding does not support IO tracking you have to choose which version you whant to use.

### Without using IO support
```
module use module use /projects/p_tensorflow/modenv-scs5/
module load Score-P-Python-Binding/1.0 TensorFlow/1.10.0-fosscuda-2018b-Python-3.6.6

# set scorep environment variables e.g.
#export SCOREP_ENABLE_TRACING=true

python -m scorep <script.py>
```

If your application used MPI use `python -m scorep --mpi <script.py>`.

### Using IO support
```
module use module use /projects/p_tensorflow/modenv-scs5/
module load Score-P-Python-Binding/io TensorFlow/1.10.0-fosscuda-2018b-Python-3.6.6

# copy the LD_PRELAOD string from output of
scorep-preload-init --user --thread=pthread --io=runtime:posix

# set scorep environment variables e.g.
#export SCOREP_ENABLE_TRACING=true

LD_PRELOAD=<your output str> python -m scorep <scripy.py>
```

 If your application used MPI use `LD_PRELOAD=<your output str>  python -m scorep --mpi <script.py>`.

If your work is done exit `bash .scorep_preload/unknown_<XXX>.clean` to clean up scorep's preload files.
 
### Known problems
For some TensorFlow aplications  my application crashed throwing the error 
`application.py: missing name of file to run`
I got rid of it, by replacing 
`tf.app.run()` with `tf.app.run(main=main)` in the `main`-method.