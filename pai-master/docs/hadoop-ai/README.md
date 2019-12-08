# Hadoop AI Enhancement
## Overview ##

We enhance Hadoop with GPU and Port support for better AI job scheduling.
Currently, YARN-3926 also supports GPU scheduling, which treats GPU as countable resource. 
However, GPU placement is very important to deep learning job for better efficiency. 
For example, a 2-GPU job runs on gpu {0,1} could be faster than run on gpu {0, 7}, 
if GPU 0 and 1 are under the same PCI-E switch while 0 and 7 are not. 

We add the GPU support to Hadoop 2.7.2 to enable GPU locality scheduling, which support fine-grained GPU placement. 
A 64-bits bitmap is added to yarn Resource, which indicates both GPU usage and locality information in a node
 (up to 64 GPUs per node). ‘1’ means available and ‘0’ otherwise in the corresponding position of the bit.

We add the Port support to Hadoop 2.7.2 to enable Port allocation. Client(application master) can submit request with specific port ranges

The AI enhancement patch was upload to:
https://issues.apache.org/jira/browse/YARN-7481

Usually there will have multiple patch files, the newest one is the last known good patch. We also integrated the AI enhancement to hadoop-2.9.0, so you will find patchs with name start with hadoop-2.9.0



## How to Build in Linux environment

   Let's use hadoop-2.9.0 patch as an example, hadoop-2.7.2 patch has the similar steps.
  there are two ways to build:

  **quick build**

   Please refer to this [readme](./hadoop-build/README.md) to get the quick way(automation) to do the build.
  

   **Step by step build**

   Below are step-by-step build for advance user:

 1. Prepare linux environment
 
       Ubuntu 16.04 is the default system. This dependencies must be installed:

  	    sudo apt-get install -y git openjdk-8-jre openjdk-8-jdk maven \
	    	cmake libtool automake autoconf findbugs libssl-dev pkg-config build-essential zlib1g-dev

	    wget https://github.com/google/protobuf/releases/download/v2.5.0/protobuf-2.5.0.tar.gz
	    tar xzvf protobuf-2.5.0.tar.gz
	    cd protobuf-2.5.0
	    ./configure
	    make -j $(nproc)
	    make check -j $(nproc)
	    sudo make install
	    sudo ldconfig
 

 2. Download hadoop AI Enhancement

    Please download "hadoop-2.9.0.gpu-port.patch" from https://issues.apache.org/jira/browse/YARN-7481 to your local.
   
 3. Get hadoop 2.9.0 source code
    
	       git clone https://github.com/apache/hadoop.git
	       cd hadoop
	       git checkout branch-2.9.0
	
 4. Build the official hadoop in your linux develop environment

   	Run command “mvn package -Pdist,native -DskipTests -Dmaven.javadoc.skip=true -Dtar”
   
    Please make sure you can pass result before move to next steps, you can search the internet to find how to set up the environment and build the official hadoop.
   
   
5. Apply hadoop AI enhancement patch file
   
    copy the downloaded file into your linux hadoop root folder and run:

    git apply hadoop-2.9.0.port-gpu.patch

    if you see the output similar with below information, it means you have successfully applied this patch

		../../hadoop-2.9.0.port-gpu:276: trailing whitespace.
		../../hadoop-2.9.0.port-gpu:1630: trailing whitespace.
		../../hadoop-2.9.0.port-gpu:1631: trailing whitespace.
		  public static final long REFRESH_GPU_INTERVAL_MS = 60 * 1000;
		../../hadoop-2.9.0.port-gpu:1632: trailing whitespace.
		../../hadoop-2.9.0.port-gpu:1640: trailing whitespace.
		          Pattern.compile("^\\s*([0-9]{1,2})\\s*,\\s*([0-9]*)\\s*MiB,\\s*([0-9]+)\\s*MiB");
		warning: squelched 94 whitespace errors
		warning: 99 lines add whitespace errors.

   
6. Build hadoop AI enhancement
  
     	Run command “mvn package -Pdist,native -DskipTests -Dmaven.javadoc.skip=true -Dtar”

      You will get the `hadoop-2.9.0.tar.gz` under `hadoop-dist/target` folder if everything is good.

      Use hadoop-2.9.0.tar.gz to udpate the hadoop-binary settings in services-configuration.yaml under your cluster configs path:

                custom-hadoop-binary-path: ***/hadoop-dist/target/hadoop-2.9.0.tar.gz

   

## Yarn Interface ##
1. Add GPUs, GPUAttribute and ports into `yarn_protos` as interface.

    sourcefile:
    hadoop-yarn-project/hadoop-yarn/hadoop-yarn-api/src/main/proto/yarn_protos.proto
    ```
		 message ResourceProto {
		   optional int32 memory = 1;
		   optional int32 virtual_cores = 2;
		   optional int32 GPUs = 3;
		   optional int64 GPUAttribute = 4;
		   optional ValueRangesProto ports = 5;
		 }
    ```

2.	Interface to get/set the GPU, GPU attribute and Ports

    GPUAttribute, the bitmap, is represented as a long variable. 
    
    sourcefile: hadoop-yarn-project/hadoop-yarn/hadoop-yarn-api/src/main/java/org/apache/hadoop/yarn/api/records/Resource.java	
    ```
		 1. public static Resource newInstance(int memory, int vCores, int GPUs, long GPUAttribute, ValueRanges ports)
		 2. public abstract int getGPUs();
		 3. public abstract void setGPUs(int GPUs);
		 4. public abstract long getGPUAttribute();
		 5. public abstract void setGPUAttribute(long GPUAttribute);
		 6. public abstract ValueRanges getPorts();
		 7. public abstract void setPorts(ValueRanges ports);
    ```
3.	Yarn configuration
    
        sourcefile: hadoop-yarn-project/hadoop-yarn/hadoop-yarn-common/src/main/resources/yarn-default.xml
        
        Below are some GPU and Port properties required by the revised yarn Resource Manager (RM).
        ```
            <property>
                <description>The minimum allocation for every container request at the RM,  in terms of GPUs. Requests lower than this will throw an InvalidResourceRequestException. </description>
                <name>yarn.scheduler.minimum-allocation-gpus</name>
                <value>0</value>
            </property>
            <property>
                <description>The maximum allocation for every container request at the RM, in terms of GPUs. Requests higher than this will throw an InvalidResourceRequestException. </description>
                <name>yarn.scheduler.maximum-allocation-gpus</name>
                <value>8</value>
            </property>		
            <property>
                <description>Percentage of GPU that can be allocated  for containers. This setting allows users to limit the amount of  GPU that YARN containers use. Currently functional only on Linux using cgroups. The default is to use 100% of GPU.
                </description>
                <name>yarn.nodemanager.resource.percentage-physical-gpu-limit</name>
                <value>100</value>
            </property>
            <property>
                <description>exclude the gpus which is used by unknown process</description>
                <name>yarn.gpu_exclude_ownerless_gpu.enable</name>
                <value>false</value>
            </property>
            <property>
                <description>the gpu memory threshold to indicate a gpu is used by unknown process</description>
                <name>yarn.gpu_not_ready_memory_threshold-mb</name>
                <value>20</value>
            </property>
            <property>
                <description>enable port as resource</description>
                <name>yarn.ports_as_resource.enable</name>
                <value>true</value>
            </property>
            <property>
                <description>the max port range available for resource allocation</description>
                <name>yarn.nodemanager.resource.ports</name>
                <value>[1-65535]</value>
            </property>
        ```

## Yarn Client: Resource Request ##

   The Resource request is sent to RM in a Resource object described in *org.apache.hadoop.yarn.client.api.AMRMClient.ContainerRequest*, through the call *org.apache.hadoop.yarn.client.api.YarnClientaddContainerRequest*
   
   There are several GPU request scenarios.
      
1. Request GPU by count only: 

    If a job only care about the number of GPU, not GPU placement, GPUAttribute must set to 0 in the request resource instance. 

		1.Resource res = Resource.newInstance(requireMem, requiredCPU, requiredGPUs, 0)
		2.Res.setGPUAttribute((long)0)

2. Request GPU with locality (GPU attribute):

    The GPU locality information is stored in a 64-bit bitmap (long), each bit represents a GPU. The GPU IDs map to the bitmap as follows.

		1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111 1111
		 |                                                                |         | 
		 |                                                                |         V
		 V                                                                V       gpu:0-3
		gpu:60-63                                                      gpu:8-11
		
	In a locality-aware request, the bitmap is included in the Resource instance. For example, If 4 GPUs are required, and the GPU placement requirement is GPU 0-3, the corresponding request instance will look like:		
		
		```
		Resource res = Resource.newInstance(requireMem, requiredCPU, 4, 15)
		#4 is the request GPU count.
		15 for “1111” is the GPU locality.
		```

     If the GPU attribute is set to none-zero, the requested GPU count must match with it. Otherwise the request will not success. 

3. Request with Node/Label

    This is also supported in GPU, and the behavior is the same as the official Hadoop 2.7.2.

4. Request with Relax 
  
	In GPU scheduling, the Relax is node level relax, GPU locality cannot be relaxed. 
	For example, if it is requested for Node 1 with a GPU count 2, GPU attribute set to 3 for GPU 0, 1.
	When Relax is enabled in the container request, if Node 1’s GPU 0 or 1 is unavailable, 
	yarn RM might relax to other node's GPU 0 and 1 (if both are available). 
	But yarn RM will not relax to other GPUs in Node 1, or to GPUs other than 0 and 1 in other nodes. 

5. Request with Ports:

		1.ValueRanges valueRanges;
		2.Resource res = Resource.newInstance(requireMem, requiredCPU);
		3.fill the value ranges.
		4.res.setPorts(valueRanges)

## Resource manger ##

1. GPU allocation algorithm

	a.	If GPU locality specified, the algorithm performs a simple “match” for the requested locality with Nodes’ GPU attribute. RM will only return a container with the resource exactly matches the requirement. 

	b.	If requested without GPU locality, the algorithm only compares the requested GPU count with Nodes’s free GPU count.
	
2. Scheduler

	CapacityScheduler, FairScheduler, and FifoScheduler are all revised to support GPU scheduling. In the FairScheduler, preemption is also supported. 

3. Resource Calculator 

	The GPU count is considered in *DominantResourceCalculator* for the mix resource environment. 
	Meanwhile,  a new Calculator *org.apache.hadoop.yarn.api.records.Resource.GPUResourceCalculator* is created to only calculate resource by GPU.


## Node Manager Plugin ##

  sourcefile: org.apache.hadoop.yarn.util.LinuxResourceCalculatorPlugin

   The node's GPU capacity is collected by running a nvidia-smi command when the NodeManager service starts.
   NodeManager heartbeat will also report the GPU utilization status to ResourceManager. User can config the yarn.gpu_exclude_ownerless_gpu.enable and yarn.gpu_not_ready_memory_threshold-mb to exclude the GPU which is not ready for serving.

   The node's Port using information are collected through command "netstat -anlut" in NodeManager, The NodeManager heartbeat will also report the Port real time
   using status to ResourceManager. user can config the yarn.nodemanager.resource.ports to set the serve port range.

## Web apps   ##

   The GPU count and GPU attribute information are also displayed in the Hadoop web. User can check the overall capacity, used, free information in app information page. User can also check each node’s GPUs utilization information in bitmap format in Node Information page. 

## Known Issues ##

1. Scheduling by GPUType

   Currently the GPU configuration file cannot be put to the cluster correctly so scheduling jobs by GPUType cannot work.
   A workaround for this is to manually update the configuration file to the cluster. This can be done in following steps:
   ```bash
     # In the OpenPAI source code folder where you do the deployment,
     # there should be a GPU configuration file under path src/cluster-configuration/deploy/gpu-configuration/gpu-configuration.json.
     # Or you can start cluster-configuration to generate it.
     sudo python paictl.py service start -p your_configuration_dir -n cluster-configuration
     # Make sure launcher is configured to logged in with admin user and get the admin user name. Those can be retrieved by running following command. The default user name is root.
     curl "http://master_address:9086/v1/LauncherStatus"
     # put the configuration to cluster
     curl -X PUT -H "Content-Type: application/json" -H "UserName: cluster_admin_user" \
      -d @src/cluster-configuration/deploy/gpu-configuration/gpu-configuration.json "http://master_address:9086/v1/LauncherRequest/ClusterConfiguration"
     # check the configuration
     curl "http://master_address:9086/v1/LauncherRequest/ClusterConfiguration"
   ```

