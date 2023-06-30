---
title: Analyzing and Mitigating Data Stalls in DNN Training
author: Jieun
date: 2023-06-28
layout: post
category: paper_reviews
---

* 요약: DNN training input pipeline 상의 data stalls을 탐지 및 해결하여 CPU/GPU utilization 향상

---

## Introduction
 * Data items are first fetched from storage and then pre-processed in memory.
 * An example of preprocessing: data is first decompressed, and then random perturbations such as cropping the image or rotating it are performed to improve the model’s accuracy.
 * DNN training is often I/O-bound, bottlenecked by fetching the data from storage, or CPU-bound, bottlenecked by applying data pre-processing in memory.
    - We term these bottlenecks data stall
 * Prior works:
	- Quiver — demonstrate how to speed up DNN training by efficiently caching data from remote object storage on local storage.
	- Cerebro and DeepIO — have looked at optimizing data fetch in distributed training.
 * We vary factors such as the storage media, amount of data that can be cached in memory, the number of CPU threads used to fetch and pre-process data, and GPU generation.
 * The data pipelines in popular training frameworks like PyTorch and TensorFlow are inefficient in their use of CPU and memory resources.
 * DS-Analyzer, that uses differential analysis between runs to accurately identify data-stall bottlenecks.

## Background
 * Three constraints:
	- The dataset must be shuffled every epoch to ensure the order in which data items are accessed are random in each epoch
	- An epoch must use all data items in the dataset exactly once 
	- In every epoch, the data transformations(pre-processing) must be random
 * DALI can accelerate data pre-processing operations using GPU-accelerated data pre-processing operations.
 * DALI also prefetches and pipelines the data fetch and pre-processing with the GPU compute.
 * We use DALI, as it is the strongest baseline.

## Data Stalls in DNN Training
 * Training process:
	1. A minibatch of data items is fetched from storage.
	2. The data items are pre-processed, for e.g.,, for image classification, data items are decompressed, and then randomly cropped, resized, and flipped.
	3. The minibatch is then processed at the GPU to obtain the model’s prediction
	4. A loss function is used to determine how much the prediction deviates from the right answer
	5. Model weights are updated using computed gradients
	6. Ideally, most of the time in each epoch should be spent on Steps 3–5
 * The rate at which data items can be fetched from storage (Step 1) depends primarily on the storage media.

## Analyzing Data Stalls
 * For the image classification task, pre-processing includes image decoding, random crop, resizing to a fixed size, and a random horizontal flip of the image.
 * We have empirically verified that DALI’s performance is strictly better than PyTorch, TF and MxNet’s default data loaders.
 * Training environment:
	- All experiments are performed on PyTorch 1.1.0 using the state-of-the-art NVIDIA data loading pipeline, DALI.
	- SSD-V100: 8xV100 GPU, 32GB GPU Mem, SSD, 530 MBps Rand Read
	- HDD-1080Ti: 8x1080Ti, 11GB GPU Mem, HDD, 15-50 MBps Rand Read
 * Training parameters:
	- In SSD-V100: use a batch size of 512 per GPU for all image classification models, 128 per GPU for SSD-Res18, 16 per GPU for M5
	- In HDD-1080Ti: use the maximum batch size that fits the GPU memory
 * Training metrics: The average epoch time of three epochs, ignoring the first epoch. (used for warmup)

## DS-Analyzer: Predictive Analysis
 * Frameworks like PyTorch and TensorFlow provide an approximate time spent on data loading and pre-processing per minibatch, by simply placing timers in the training script. This is insufficient and inaccurate.
	- This technique cannot accurately provide the split up of time spent in data fetch (from disk or cache) and pre-processing operations.
	- Frameworks like PyTorch and libraries like DALI use several concurrent processes (or threads) to fetch and pre-process data.
 * Data stall in one of the data loading processes may reflect as GPU compute time for the other processes, because all GPU processes wait to synchronize weight updates at batch boundaries.
 * DS-Analyzer runs in three phases:
	1. Measure ingestion rate: DS-Analyzer pre-populates synthetic data at the GPUs and runs the job for a fixed number of epochs. (no fetch or prep stalls)
	2. Measure prep stalls: DS-Analyzer runs the training script with a subset of the given dataset, such that it is entirely cached in memory, using all available CPU cores, and estimates the training speed. (1 - 2 = prep stalls)
	3. Measure fetch stalls: DS-Analyzer runs the training script by clearing all caches, and setting maximum cache size to a user-given limit. (2 - 3 = fetch stalls)
 * Data Stalls:
	- Fetch Stalls (Remote): When dataset resides on remote storage.
		- We analyze the impact of two kinds of remote backends; a distributed file system (GlusterFS) and Azure blob object store (accessed via blobfuse)
		- When data resides remotely, the first epoch of training fetches data over the network and stores it locally for subsequent use.
		- The first epoch when it downloads the entire dataset to local storage, and can result in 20x higher training time as compared to GFS.
		- GFS results in more data stalls as it validates metadata of cached data items over the network every item a data item is accessed.
		- Blobfuse does not incur any network cost beyond first epoch, if the datset fits on local SSD.
		- Therefore, a common trining senario is to pay a one-time download cost for the dataset, and reap benefits of local-SSD accesses thereafter.
		- In the rest of the work, we analyze fetch stalls in senarios where dataset is present locally on a server, but is not entirely cached in memory. → 프로젝트에서 고려해야 할 시나리오는 local disks와 local memory 모두에 fit되지 않을 때
	- Fetch Stalls (Local): When datasets cannot be fully cached and the fetch rate < compute rate.
        - Popular dataset like ImageNet-1K cannot be fully cached on commonly used cloud SKUs.
        - DNNs spend 10-70% of their epoch time on blocking I/O, despite pipelining and prefetching, simply because the compute rate is higher than fetch rate.
        - DNN training platforms rely on the operating system's Page Cache to cache raw training data in memory. Unfortunately, the OS Page Cache leads to thrashing as it is not efficient for DNN training.
        - The lack of coordination among caches makes distributed training storage I/O-bound.
    - Data stalls exist across training frameworks.
        - TF shuffles the small random files, serializes it, and stores them as a set of files called TFRecords. TFRecords make reads more sequential.

## Mitigating Data Stalls
 * The MinIO Cache
    - DNN training has a unique data access pattern: it is repetitive across epochs and random within an epoch.
    - The DNN access pattern that is at odds with usual OS cache replacement (e.g. LRU) policies.
    - MinIO recommends a simple and unintutive solution; items, once cached, are never replaced in the DNN cache.
 * Partitioned MinIO Caching
    - Each server operates on a random shard of the dataset per epoch, and this partition changes every epoch. The MinIO cache alone, is not efficient in this setting.
    - Partitioned MinIO caching works as follows:
        - In the first epoch, the dataset is sharded across all servers.
        - Each server populates it's local MinIO cache with data items in the shard assigned to it.
        - At the end of first epoch, we collectively cache a part of the dataset of size equal to the sum of capacities of individual MinIO caches.
        - To route data fetch requests to the appropriate server, we maintain metadata about data items present in each server's cache.
 * We build CoorDL on top of DALI to take adventage of the GPU-accelerated data pre-processing operations.

## Evaluation

## Conclusion