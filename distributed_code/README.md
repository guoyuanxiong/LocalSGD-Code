# Getting started
Our experiments heavily rely on `Docker` and `Kubernetes`. For the detailed experimental environment setup, please refer to dockerfile under the `environments` folder.


## Use case of distributed training (centralized)
Some simple explanation of the arguments used in the code.
* Arguments related to *distributed training*:
    * The `n_mpi_process` and `n_sub_process` indicates the number of nodes and the number of GPUs for each node. The data-parallel wrapper is adapted and applied locally for each node.
        * Note that the exact mini-batch size for each MPI process is specified by `batch_size`, while the mini-batch size used for each GPU is `batch_size/n_sub_process`.
    * The `world` describes the GPU topology of the distributed training, in terms of all GPUs used for the distributed training.
    * The `hostfile` from `mpi` specifies the physical location of the MPI processes.
    * We provide two use cases here:
        * `n_mpi_process=2`, `n_sub_process=1` and `world=0,0` indicates that two MPI processes are running on 2 GPUs with the same GPU id. It could be either 1 GPU at the same node or two GPUs at different nodes, where the exact configuration is determined by `hostfile`.
        * `n_mpi_process=2`, `n_sub_process=2` and `world=0,1,0,1` indicates that two MPI processes are running on 4 GPUs and each MPI process uses GPU id 0 and id 1 (on 2 nodes).
* Arguments related to *communication compression*:
    * The `graph_topology` 
    * The `optimizer` will decide the type of distributed training, e.g., centralized SGD, decentralized SGD
    * The `comm_op` specifies the communication compressor we can use, e.g., `sign+norm`, `random-k`, `top-k`.
* Arguments related to *learning*:
    * The `lr_scaleup`, `lr_warmup` and `lr_warmup_epochs` will decide if we want to scale up the learning rate, or warm up the learning rate. For more details, please check `pcode/create_scheduler.py`.

### Examples
The script below trains `ResNet-20` with `CIFAR-10`, as an example of centralized training algorithm `(post-)local SGD`.
```bash
OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
    --arch resnet20 --optimizer local_sgd \
    --avg_model True --experiment demo --manual_seed 6 \
    --data cifar10 --pin_memory True \
    --batch_size 128 --base_batch_size 64 --num_workers 2 \
    --num_epochs 300 --partition_data random --reshuffle_per_epoch True --stop_criteria epoch \
    --n_mpi_process 16 --n_sub_process 1 --world 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 \
    --on_cuda True --use_ipc False \
    --lr 0.1 --lr_scaleup True --lr_warmup True --lr_warmup_epochs 5 \
    --lr_scheduler MultiStepLR --lr_decay 0.1 --lr_milestones 150,225 \
    --local_step 32 --turn_on_local_step_from 150 \
    --weight_decay 1e-4 --use_nesterov True --momentum_factor 0.9 \
    --hostfile hostfile --graph_topology complete --track_time True --display_tracked_time True \
    --python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/
```

The script below trains `ResNet-20` with `CIFAR-10`, as an example of centralized training algorithm `post-local SGD` with `sign` based communication compressor.
```bash
OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
    --arch resnet20 --optimizer local_sign_sgd \
    --avg_model True --experiment demo --manual_seed 6 \
    --data cifar10 --pin_memory True \
    --batch_size 128 --base_batch_size 128 --num_workers 2 \
    --num_epochs 300 --partition_data random --reshuffle_per_epoch True --stop_criteria epoch \
    --n_mpi_process 16 --n_sub_process 1 --world 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 \
    --on_cuda True --use_ipc False \
    --lr 0.01 --lr_scaleup False --lr_warmup False --lr_warmup_epochs 5 \
    --lr_scheduler MultiStepLR --lr_decay 0.1 --lr_milestones 150,225 \
    --local_step 16 --turn_on_local_step_from 150 \
    --weight_decay 1e-4 --use_nesterov True --momentum_factor 0.9 \
    --hostfile hostfile --graph_topology complete --track_time True --display_tracked_time True \
    --python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/
```

The script below trains `ResNet-20` with `CIFAR-10`, as an example of centralized training algorithm `post-local SGD` with `sign+norm` based communication compressor.
```bash
OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
    --arch resnet20 --optimizer local_ef_sign_sgd \
    --avg_model True --experiment demo --manual_seed 6 \
    --data cifar10 --pin_memory True \
    --batch_size 128 --base_batch_size 128 --num_workers 2 \
    --num_epochs 300 --partition_data random --reshuffle_per_epoch True --stop_criteria epoch \
    --n_mpi_process 16 --n_sub_process 1 --world 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 \
    --on_cuda True --use_ipc False \
    --lr 0.1 --lr_scaleup True --lr_warmup True --lr_warmup_epochs 5 \
    --lr_scheduler MultiStepLR --lr_decay 0.1 --lr_milestones 150,225 \
    --local_step 64 --turn_on_local_step_from 150 \
    --weight_decay 1e-4 --use_nesterov True --momentum_factor 0.9 \
    --hostfile hostfile --graph_topology complete --track_time True --display_tracked_time True \
    --python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/
```

The script example of `LSTM` on `wikitext2` for `SGD` follows:
```bash
OMP_NUM_THREADS=2 MKL_NUM_THREADS=2 $HOME/conda/envs/pytorch-py3.6/bin/python run.py \
    --arch rnn_lm --rnn_n_hidden 650 --rnn_n_layers 3 --rnn_bptt_len 30 \
    --rnn_clip 0.4 --rnn_use_pretrained_emb False --rnn_tie_weights True --drop_rate 0.40 \
    --optimizer sgd --avg_model True --experiment demo \
    --data wikitext2 --pin_memory True \
    --batch_size 32 --base_batch_size 24 --num_workers 2 \
    --num_epochs 300 --partition_data random --reshuffle_per_epoch False --stop_criteria epoch \
    --n_mpi_process 32 --n_sub_process 1 --world 0,0,0,0,0,0,0,0,1,1,1,1,1,1,1,1,0,0,0,0,0,0,0,0,1,1,1,1,1,1,1,1 \
    --on_cuda True --use_ipc False --comm_device cuda \
    --lr 2.5 --lr_scaleup True --lr_warmup True --lr_warmup_epochs 5 \
    --lr_scheduler MultiStepLR --lr_decay 0.1 --lr_milestones 150,225 \
    --weight_decay 0 --use_nesterov False --momentum_factor 0 \
    --hostfile hostfile --graph_topology complete --track_time True --display_tracked_time True \
    --python_path $HOME/conda/envs/pytorch-py3.6/bin/python --mpi_path $HOME/.openmpi/
```
