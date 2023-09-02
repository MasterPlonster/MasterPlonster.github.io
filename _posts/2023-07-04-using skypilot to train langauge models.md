---
layout: post
title: Training Large Language Models with Skypilot
date: 2023-08-04 08:57:00-0400
description: a useful tool to help with hardware provisioing and environment setup for training large language models.
tags: skypilot training llm tips
categories: llm
giscus_comments: true
related_posts: false
---


## About SkyPilot 

If you ever tried to finetune a large language model, you probably know that provisioning and preparing the hardware and environment can be a pain. GPU availability is limited, and you need to make sure you have the right CUDA version, the right drivers, and the right libraries. 

I've been using [SkyPilot](https://skypilot.readthedocs.io/en/latest/) for a while now, and it made a big difference in my life. Initially it was just a convinience tool for me to quickly search for a GCP zone with A100 availability, but I quickly discovered that relying on it forces me to be much more organized and disciplined in organizing my training runs.

Technically speaking, SkyPilot is an abstraction of Ray + Cloud provider SDK + some niceities, including folder syncing, a task queue and managment of spot instances. It's free, open source, and easy to use. So let's get started. 

## First Steps 

First, you need to [install SkyPilot](https://skypilot.readthedocs.io/en/latest/getting-started/installation.html) and configure the CLIs for your favourite cloud providers. 

In you are using AWS:

```bash
pip install skypilot
pip install skypilot[aws]

pip install boto3
aws configure
```

You also need to make sure that you've got the relevant quota set up for your account. With GCP there's some API activation you need to do, and with AWS you need to request a quota increase. The documentation explains how to do that.

Next, it's time to configure your project. You can do that by creating a `skypilot.yaml` file in your project directory. Here's an example:

```yaml
resources:
  cloud: aws
  accelerators: A100:4 # Pick the kind and number of GPUs you want to use.

# Working directory (optional) containing the project codebase
# Its contents are synced to ~/sky_workdir/ on the cluster.
workdir: .

# Invoked under the workdir (i.e., can use its files)
# This is where you set up your environment and install dependencies
# The machine will already have conda install so you can use that to set up your environment with the correct python version.
setup: |
    conda create -n skypilot python=3.10
    conda activate skypilot

    pip install -r requirements.txt

# This is the main action, where you run your job.
# Invoked under the workdir (i.e., can use its files).
run: |
    python train.py
```

Now creating a cluster and running the job is a simple as:

```bash
sky launch -c mycluster skypilot.yaml
```

SkyPilot will also helpfully add the host to your ssh config, so you can easily ssh into it with `ssh mycluster`.

To see your running clusters, it's as simple as `sky status`, and you can terminate the cluster with `sky down mycluster`.

## Autostop 

As we all know, forgetting your training machine on can be expensive. SkyPilot has a nice feature called `autostop` that will automatically stop the machine after a certain amount of idle time. You can launch a task with autostop using the flag `-i` like this:

```bash
sky launch -d -c mycluster cluster.yaml -i 10 --down
```

## Saving Logs and Artefacts

Terminating a cluster when your task is done obviously means that everything stored on the machine is lost. The way I like to work is to save the Logs to Weights and Biases, and save the model checkpoints to either a bucket or HuggingFace Hub. 

If using a HuggingFace trainer this is as simple as addign `report_to=wand` and `push_to_hub=True` to the `TrainingArguments`. You do need to make sure you are logged in to both, or pass the relevant authentication tokens. 

While SkyPilot provides can use an env section in the skypilot.yaml pass environment variables, I like to store my template in github, which is not a safe place for secrets. Instead, I use a `.env` file in the project directory, which is synced by SkyPilot directly to the cluster, and then I can use it to set the environment variables like this:

```yaml
resources:
  cloud: aws
  accelerators: A100:4 # Pick the kind and number of GPUs you want to use.

# Working directory (optional) containing the project codebase
# Its contents are synced to ~/sky_workdir/ on the cluster.
workdir: .

# Invoked under the workdir (i.e., can use its files)
# This is where you set up your environment and install dependencies
# The machine will already have conda install so you can use that to set up your environment with the correct python version.
setup: |
    conda create -n skypilot python=3.10
    conda activate skypilot

    pip install -r requirements.txt

    export $(grep -v '^#' .env | xargs)
    wandb login $WANB_TOKEN

# This is the main action, where you run your job.
# Invoked under the workdir (i.e., can use its files).
run: |
    python train.py
```

## Using The Job Queue

The job queue comes without any additional configuration, and is a great way to manage multiple jobs. After you spin up your cluster you can just send additional jobs with:

```bash
sky exec mycluster task.yaml -d
```

Where `task.yaml` is a file with the same structure as `skypilot.yaml` containing a task configuration.