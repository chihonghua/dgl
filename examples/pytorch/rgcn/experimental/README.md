## Distributed training

This is an example of training RGCN node classification in a distributed fashion. Currently, the example train RGCN graphs with input node features. The current implementation follows ../rgcn/entity_claasify_mp.py.

Before training, please install some python libs by pip:

```bash
sudo pip3 install ogb
sudo pip3 install pyinstrument
sudo pip3 install pyarrow
```

To use the example and tools provided by DGL, please clone the DGL Github repository.

```bash
git clone --recursive https://github.com/dmlc/dgl.git
```
Below, we assume the repository is cloned in the home directory.

To train RGCN, it has four steps:

### Step 0: set IP configuration file.

User need to set their own IP configuration file before training. For example, if we have four machines in current cluster, the IP configuration
could like this:

```bash
172.31.0.1
172.31.0.2
172.31.0.3
172.31.0.4
```

Users need to make sure that the master node (node-0) has right permission to ssh to all the other nodes.

### Step 1: partition the graph.

The example provides a script to partition some builtin graphs such as ogbn-mag graph.
If we want to train RGCN on 4 machines, we need to partition the graph into 4 parts.

In this example, we partition the ogbn-mag graph into 4 parts with Metis. The partitions are balanced with respect to
the number of nodes, the number of edges and the number of labelled nodes.
```bash
python3 partition_graph.py --dataset ogbn-mag --num_parts 4 --balance_train --balance_edges
```

This script generates partitioned graphs and store them in the directory called `data`.

### Step 2: copy the partitioned data to the cluster
DGL provides a script for copying partitioned data to the cluster. Before that, copy the training script to a local folder:


```bash
mkdir ~/dgl_code
cp ~/dgl/examples/pytorch/rgcn/experimental/entity_classify_dist.py ~/dgl_code
```



The command below copies partition data, ip config file, as well as training scripts to the machines in the cluster.
The configuration of the cluster is defined by `ip_config.txt`.
The data is copied to `~/rgcn/ogbn-mag` on each of the remote machines.
`--rel_data_path` specifies the relative path in the workspace where the partitioned data will be stored.
`--part_config` specifies the location of the partitioned data in the local machine (a user only needs to specify
the location of the partition configuration file). `--script_folder` specifies the location of the training scripts.
```bash
python ~/dgl/tools/copy_files.py --ip_config ip_config.txt \
                                 --workspace ~/rgcn \
                                 --rel_data_path data \
				 --part_config data/ogbn-mag.json \
			         --script_folder ~/dgl_code
```

**Note**: users need to make sure that the master node has right permission to ssh to all the other nodes.

Users need to copy the training script to the workspace directory on remote machines as well.

### Step 3: Launch distributed jobs

DGL provides a script to launch the training job in the cluster. `part_config` and `ip_config`
specify relative paths to the path of the workspace.

The command below launches one training process on each machine and each training process has 4 sampling processes.
**Note**: There is a known bug in Python 3.8. The training process hangs when running multiple sampling processes for each training process.
Please set the number of sampling processes to 0 if you are using Python 3.8.

```bash
python3 ~/dgl/tools/launch.py \
--workspace ~/rgcn/ \
--num_trainers 1 \
--num_servers 1 \
--num_samplers 4 \
--part_config data/ogbn-mag.json \
--ip_config ip_config.txt \
"python3 dgl_code/entity_classify_dist.py --graph-name ogbn-mag --dataset ogbn-mag --fanout='25,25' --batch-size 512  --n-hidden 64 --lr 0.01 --eval-batch-size 16  --low-mem --dropout 0.5 --use-self-loop --n-bases 2 --n-epochs 3 --layer-norm --ip-config ip_config.txt  --num-workers 4 --num-servers 1 --sparse-embedding  --sparse-lr 0.06 --node-feats"
```

We can get the performance score at the second epoch:
```
Val Acc 0.4323, Test Acc 0.4255, time: 128.0379
```

## Partition a graph with ParMETIS

It has four steps to partition a graph with ParMETIS for DGL's distributed training.
More details about the four steps are explained in our
[user guide](https://doc.dgl.ai/guide/distributed-preprocessing.html).

### Step 1: write the graph into files.

The graph structure should be written as a node file and an edge file. The node features and edge features
can be written as DGL tensors. `write_mag.py` shows an example of writing the OGB MAG graph into files.

```bash
python3 write_mag.py
```

### Step 2: partition the graph with ParMETIS
Run the program called `pm_dglpart` in ParMETIS to read the node file and the edge file output in Step 1
to partition the graph.

```bash
pm_dglpart mag 2
```
This partitions the graph into two parts with a single process.

```
mpirun -np 4 pm_dglpart mag 2
```
This partitions the graph into eight parts with four processes.

```
mpirun --hostfile hostfile -np 4 pm_dglpart mag 2
```
This partitions the graph into eight parts with four processes on multiple machines.
`hostfile` specifies the IPs of the machines; one line for a machine. The input files
should reside in the machine where the command line runs. Each process will write
the partitions to files in the local machine. For simplicity, we recommend users to
write the files on NFS.

### Step 3: Convert the ParMETIS partitions into DGLGraph

DGL provides a tool called `convert_partition.py` to load one partition at a time and convert it into a DGLGraph
and save it into a file.

```bash
python3 ~/dgl/tools/convert_partition.py --input-dir . --graph-name mag --schema mag.json --num-parts 2 --num-node-weights 4 --output outputs
```

### Step 4: Read node data and edge data for each partition

This shows an example of reading node data and edge data of each partition and saving them into files located in the same directory as the DGLGraph file.

```bash
python3 get_mag_data.py
```

### Step 5: Verify the partition result (Optional)

```bash
python3 verify_mag_partitions.py 
```

## Distributed code runs in the standalone mode

The standalone mode is mainly used for development and testing. The procedure to run the code is much simpler.

### Step 1: graph construction.
When testing the standalone mode of the training script, we should construct a graph with one partition.
```bash
python3 partition_graph.py --dataset ogbn-mag --num_parts 1
```

### Step 2: run the training script
```bash
python3 entity_classify_dist.py --graph-name ogbn-mag  --dataset ogbn-mag --fanout='25,25' --batch-size 512 --n-hidden 64 --lr 0.01 --eval-batch-size 128 --low-mem --dropout 0.5 --use-self-loop --n-bases 2 --n-epochs 3 --layer-norm --ip-config ip_config.txt --conf-path 'data/ogbn-mag.json' --standalone  --sparse-embedding  --sparse-lr 0.06 --node-feats
```
