# Deploying to AI2's Cluster via Beaker

AI2 offers a gpu job queueing system through Beaker. This repo templatizes and abstracts away some complexity in managing the beaker deployments.

Generally, here is how you should incorporate your model into this repository:
1. Place your model in the `model/` folder
2. Edit the `build.sh` [here](scripts/build.sh) to specify what files are relevant to your image build.  At build time these files will get copied into the build context directory `./artifacts`, which is cleared out each time an image build is performed.
3. Edit the `Dockerfile` [here](docker/amd64/Dockerfile) to specify two important things. First, some base installations (python version, any system packages, etc.). Second, what files you want to copy into the image from the build context, and where specifically these should go inside the image filesystem.
4. Edit the `container-entrypoint.sh` [here](scripts/container-entrypoint.sh) to specify anything you want to install at runtime, and what should be called when the container starts.

## Prerequisites

- [ ] AI2 access (access to their okta portal and beaker)
- [ ] tailscale vpn setup
- [ ] ssh key for your device added to beaker
- [ ] docker
- [ ] unix based shell (i.e. Mac, Linux)
- [ ] beaker CLI (see Beaker docs obtained from AI2 POC)

## Useful Links

- add ssh key: https://bridge.allenai.org/ssh
- slack: https://app.slack.com/client/T0A198UU9/C07S3D4D8JZ
- beaker: https://beaker.allen.ai/
- beaker docs: https://beaker-docs.apps.allenai.org/

For troubleshooting, if you have problems, ask on beaker slack.

## High Level Deployment Overview

Clusters
- saturn: a100s (allows non-preemptible jobs)
- jupiter: h100s
- neptune: less powerful GPUs (24GB VRAM)

The deployment process to a cluster is like this:
1. Build an image for your workload
2. Push your image to the beaker container registry
3. Deploy to beaker using their yaml deployment syntax

## Specific Guide

### Deployment to Beaker

All commands should be run from the root of this repo.

1. First you will need to mirror the S3 data to Weka. Open the `./scripts/s3.sh` and edit the WEKA_PREFIX to point to any prefix (i.e. can be your username).

2. Source the s3 related environment variables.

```
# source environment variables
source ./scripts/s3.sh
```

3. Kick off a beaker job to conduct the transfer.

```
beaker experiment create beaker/yaml/s3.yaml 
```

4. Build the docker image. This build will produce an image named `dna-seq-models`. Once this image is pushed into Beaker, it does not allow you to overwrite it. So if this becomes an issue (changes to the model necessitating new image), you will need to edit the script to use a different image name.

```
./scripts/build.sh my-image-name
```

5. Push the image to beaker. If using a different image name fix below.

```
beaker image create --name my-image-name my-image-name
```

6. Configure the experiment to your needs by editing `./beaker/yaml/beaker-config.yaml`. Then run the command to create the experiment.
```
beaker experiment create ./beaker/yaml/beaker-config.yaml
```

### Distributed Training with Beaker

You can use the [beaker distributed config](beaker/yaml/beaker-config-distrib.yaml) to deploy your distributed experiment. However, you will need to set some important things up correctly.
1. NCCL / IB Env Vars: Depending on what cluster you are running on, you will need to comment / uncomment the correct block in the yaml.
2. Replica Count: The number of replicas (nodes).
3. GPU Count: The number of GPUs per node
4. Beaker User Information: Distributed training utilizes service discovery in order to discover other nodes. You will need to fill this information out [here](model/work.ipynb).

ATTENTION: Beaker engineers have suggested they will expose the hostnames of all nodes in a future update, so this service discovery will no longer be needed. When they add this we will remove the service discovery referenced in the sample notebook. Removing this is ideal as we hardcode ports which may cause port synchronization issues with many running experiments. Until they add this, you will need to replicate the service discovery method implemented in [the notebook](model/work.ipynb), but only if your distributed training jobs require explicit hostnames for all nodes. This reference service discovery implementation is setup for tensorflow, but if using something else you would just need to prepare the environment variable differently.

For any questions, refer to beaker docs:
https://beaker-docs.apps.allenai.org/experiments/distributed-training.html

### Expert Tip: Fast deployments

If you find yourself constantly running the above commands, these themselves can be automated by this [script](scripts/fast-deploy.sh). This is helpful as beaker does not allow you to overwrite image tags when uploading to beaker's container registry, forcing you to go through a feedback loop where you constantly have to push to new image tags and then edit your deployment yaml. This can ideally be one step.

Conceptually, here is how it works:
1. Calls the build script.
2. Pushes that image to beaker, under a specific hash. (Beaker won't allow image overwrites with the same name).
3. Edits your deployment script to overwrite the previous image has with the freshly generated one.
4. Deploys to beaker.

For the linked script to work you will need to do the following:
1. Specify the path to the deployment YAML you are using.
2. Your deployment yaml should have a dash followed by a hash at the end of the image tag, so that the script is set up to work in its expected state. If this is unclear take a look at what the script is doing.

