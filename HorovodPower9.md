# Horovod mit TensorFlow auf Power 9
Bauen und Verwenden von Horovod für verteiltes Machine-Learning mit TensorFlow auf der IBM Power9-Architektur.

## Virtualenv installieren
Virtualenvironment sind grob gesagt Python-Umgebungen, in denen Pakete installiert werden können. Die Umgebungen sind voneinander unabhängig. Da die Pakete in die Umgebung installiert werden, ist keine Root-Berechtigung notwendig.

Auf der Power9-Partition muss das Virtualenv noch installiert werden.

	module load modenv/ml
	module load TensorFlow/1.14.0-PythonAnaconda-3.6
	pip install --user virtualenv # installiert ins eigene home
	# cd to your workspace
	ln -s ~/.local/bin/virtualenv . # nicht notwendig, aber bequemer

### Verwenden
	source myvirtualenv/bin/activate
	# do your work
	deactivate
		
	
## Horovod mit NCCL
### Installation
Unbedingt darauf achten, erst die Module zu Laden und dann erst ins Virtualenvironment zu wechseln.

Da NCCL scheinbar nur knotenlokal funktioniert, wird trotzdem MPI für Kommunikation über Knotengrenzen hinweg benötigt. Dies sollte aber noch einmal geklärt werden. Um MPI nicht zu verwenden muss lediglich 
`HOROVOD_WITH_MPI=1 ` gegen `HOROVOD_WITHOUT_MPI=1 ` getauscht werden


	module load modenv/ml
	module load TensorFlow/1.14.0-PythonAnaconda-3.6
	module load NCCL/2.3.7-fosscuda-2018b
	module load CMake/3.12.1-GCCcore-7.3.0

	# cd to your workspace

	./virtualenv horovod_nccl # neue Umgebung erstellen
	source horovod_nccl/bin/activate

	python -c 'import tensorflow as tf; print(tf.__version__)' # Test ob TensorFlow in der Umgebung steckt

	 
	HOROVOD_NCCL_HOME=/sw/installed/NCCL/2.4.2-gcccuda-2019a HOROVOD_GPU_ALLREDUCE=NCCL \
	HOROVOD_WITH_MPI=1 \
	HOROVOD_WITH_TENSORFLOW=1 HOROVOD_WITHOUT_PYTORCH=1 HOROVOD_WITHOUT_MXNET=1 \
	python3 -m pip install --no-cache-dir horovod
	
	python -c "import tensorflow as tf; import horovod.tensorflow as hvd" # Test ob Horovod in der Umgebung steckt

### Verwenden
Die Hostliste muss man sich aus der Umgebungsvariable `SLURM_JOB_NODELIST` zusammenbauen.

	module load modenv/ml
	module load NCCL/2.3.7-fosscuda-2018b
	module load TensorFlow/1.14.0-PythonAnaconda-3.6

	# cd to your workspace
	source horovod_nccl/bin/activate
	
	python -c "import tensorflow as tf; import horovod.tensorflow as hvd" # Test ob Horovod in der Umgebung steckt
	
	horovodrun -np 16 -H server1:4,server2:4,server3:4,server4:4 python train.py # 4 Knoten mit je 4 GPUs

	
	
## Horovod mit OpenMPI
**Achtung:** Nicht das Modul OpenMPI/3.1.3-gcccuda-2019a verwenden, da OpenMPI 3.1.3 laut Horovod auf Grund eines Bugs nicht funktioniert.
> Note: Open MPI 3.1.3 has an issue that may cause hangs. The recommended fix is to downgrade to Open MPI 3.1.2 or upgrade to Open MPI 4.0.0.
### Installation
	module load modenv/ml
	module load TensorFlow/1.14.0-PythonAnaconda-3.6
	module load OpenMPI/3.1.4-gcccuda-2018b
	module load NCCL/2.3.7-fosscuda-2018b
	module load CMake/3.12.1-GCCcore-7.3.0

	./virtualenv horovod_mpi # neue Umgebung erstellen
	source horovod_mpi/bin/activate

	python -c 'import tensorflow as tf; print(tf.__version__)' # Test ob TensorFlow in der Umgebung steckt

	HOROVOD_NCCL_HOME=/sw/installed/NCCL/2.3.7-gcccuda-2018b \
	HOROVOD_WITH_MPI=1 \
	HOROVOD_WITH_TENSORFLOW=1 HOROVOD_WITHOUT_PYTORCH=1 HOROVOD_WITHOUT_MXNET=1 \
    HOROVOD_GPU_ALLREDUCE=MPI HOROVOD_GPU_ALLGATHER=MPI HOROVOD_GPU_BROADCAST=MPI \
	python3 -m pip install --no-cache-dir horovod
	 
	 python -c "import tensorflow as tf; import horovod.tensorflow as hvd" # Test ob Horovod in der Umgebung steckt
	 
	 
### Verwenden
Der `horovodrun`-Wrapper benötigt eine Liste der Hosts. Um dies zu Umgehen sollte direkt `mpirun` verwendet werden, da hier die Hostliste per SLURM ins MPI kommen.
	
	module load modenv/ml
	module load TensorFlow/1.14.0-PythonAnaconda-3.6
	module load OpenMPI/3.1.4-gcccuda-2018b
	module load NCCL/2.3.7-fosscuda-2018b
	module load CMake/3.12.1-GCCcore-7.3.0
	
	source horovod_mpi/bin/activate

	mpirun -np 16 \
	-bind-to none -map-by slot \
	-x NCCL_DEBUG=INFO -x LD_LIBRARY_PATH -x PATH \
	-mca pml ob1 -mca btl ^openib \
	python my_application.py
	
	
>Open MPI with RDMA
>
>As noted above, using TCP for MPI communication does not have any significant effects on performance in the majority of cases. Models that make heavy use of hvd.broadcast() and hvd.allgather() operations are exceptions to that rule.
>
>Default Open MPI openib BTL that provides RDMA functionality does not work well with MPI multithreading. In order to use RDMA with openib, multithreading must be disabled via the -x HOROVOD_MPI_THREADS_DISABLE=1 option. See the example below:
>
>mpirun -np 16 \
>    -H server1:4,server2:4,server3:4,server4:4 \
>    -bind-to none -map-by slot \
>    -x NCCL_DEBUG=INFO -x LD_LIBRARY_PATH -x HOROVOD_MPI_THREADS_DISABLE=1 -x PATH \
>    -mca pml ob1 \
>    python train.py
>
>Other MPI RDMA implementations may or may not benefit from disabling multithreading, so please consult vendor documentation.	


## Horovod with Score-P
Score-P 


## Weitere Infos
### NCCL
Horovod bringt auch selbst NCCL mit. Wenn nicht die auf das vorhandene NCCL verlinkt oder explizit deaktiviert wird, wird dieses scheinbar verwendet.

> NCCL (pronounced "Nickel") is a stand-alone library of standard collective communication routines for GPUs, implementing all-reduce, all-gather, reduce, broadcast, and reduce-scatter. It has been optimized to achieve high bandwidth on platforms using PCIe, NVLink, NVswitch, as well as networking using InfiniBand Verbs or TCP/IP sockets. NCCL supports an arbitrary number of GPUs installed in a single node or across multiple nodes, and can be used in either single- or multi-process (e.g., MPI) applications.

Es müsste noch mal untersucht werden, in wie weit wir NCCL-Kommunikation mit Score-P aufzeichnen können.

### Verwendet NCCL MPI?
Ich bin mir nach dem Lesen der Docs nicht sicher, ob NCCL und MPI direkt voneinander abgegrenzt werden können oder ob NCCL-Kommunikation teilweise doch über MPI abläuft bzw. trotz des HOROVOD_GPU_*=MPI NCCL verwendet wird, wenn nicht explizit `HOROVOD_WITHOUT_MPI=1` gesetzt ist.

Das müsste mal untersucht werden.

### nv_peer_memory
Für GPUDirect RDMA Unterstützung braucht man unter Umständen zu NCCL noch [nv_peer_memory](https://github.com/Mellanox/nv_peer_memory) von Mellanox.
Müsste mal getestet werden.

### Gloo
Horovod bringt Gloo mit, welches von Facebook für Machine-Learning-Anwendungen entwickelt wurde und als MPI-Ersatz verwendet werden kann.
Es kann mittel `horovodrun --gloo -np 2` verwendet bzw. beim Bauen mittels `HOROVOD_WITHOUT_GLOO=1` deaktiviert werden.
(https://github.com/horovod/horovod#id11)

> Gloo is a collective communications library. It comes with a number of collective algorithms useful for machine learning applications. These include a barrier, broadcast, and allreduce.
>
>Transport of data between participating machines is abstracted so that IP can be used at all times, or InifiniBand (or RoCE) when available. In the latter case, if the InfiniBand transport is used, GPUDirect can be used to accelerate cross machine GPU-to-GPU memory transfers.
>
>Where applicable, algorithms have an implementation that works with system memory buffers, and one that works with NVIDIA GPU memory buffers. In the latter case, it is not necessary to copy memory between host and device; this is taken care of by the algorithm implementations.
[Gloo](https://github.com/facebookincubator/gloo)

### Durch TensorFlow-Modul kommt auch MPI
Durch das Modul `TensorFlow/1.14.0-PythonAnaconda-3.6` kommt auch `xlC_r` aus der PowerAI. Daher muss `HOROVOD_WITHOUT_MPI=1` gesetzt werden, um MPI zu deaktivieren.


### IBM DDL-Framework
DDL ist IBMs eigenes Distributed Deep Learning Framework, was eine von IBM überarbeitetes TensorFlow bereitstellt.
Mittels `HOROVOD_DDL_{HOME,INCLUDE,LIB}` kann man da auch Horovod runter packen.

(https://developer.ibm.com/linuxonpower/2018/08/24/distributed-deep-learning-horovod-powerai-ddl)
