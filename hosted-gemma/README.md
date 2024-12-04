# Self-hosting Gemma2 on GKE

*Objective:* Create a GKE cluster with access to Nvidia L4 GPUs, and deploy the
[Gemma 2 open model](https://ai.google.dev/gemma/docs/base).

#### Prerequisites:

- a [Huggingface](http://huggingface.co) account

### Background

For this portion of the lab, you'll be creating a GKE cluster with GPU
accelerators, and deploying the Gemma-2 model to it. To serve the model, we'll
be using the Text Generation Inference model server (TGI).

TGI is able to download models automatically from HuggingFace (or other
sources), so the container we're deploying does not need to contain the model
data.

> [!NOTE]
> Throughout these exercises I use shell-style placeholders like
> `${my-project-name}` - you should replace these with your actual project name,
> removing the surrounding `${}`.

### Create a suitable GKE cluster

Most AI workloads use some kind of hardware accelerator - GPUs and TPUs are most
common. In order to run AI workloads, we'll need some of those accelerators
available in our cluster.

Since VMs with accelerators are [not live
migrated](https://cloud.google.com/kubernetes-engine/docs/how-to/gpus#limitations),
we're going to keep our GPU workloads in their own node pool, which we add
below.

If you have an existing GKE cluster you want to use, you can skip down to
creating a node pool.

[GKE Autopilot](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview) clusters will automatically obtain GPUs for your cluster
when/if they are available. This is fine for casual use, but to ensure reliable,
"production grade" services, you should reserve the GPUs you need in a dedicated
node pool.

<!-- The following command will create a Autopilot GKE cluster -->

<!-- ``` -->
<!-- gcloud container clusters create-auto ${my-gke-cluster} --location=us-central1 --project ${my-project-name} -->
<!-- ``` -->

<!-- See the documentaion about [creating GKE autopilot -->
<!-- clusters](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-an-autopilot-cluster) -->
<!-- for more available options. -->

To create a GKE Standard cluster with a minimal default nodepool, run the
following command:

```
gcloud container clusters create ${my-gke-cluster} --region us-central1
--node-locations us-central1-f --num-nodes 2
```

#### Adding your GPU Node Pool

For this lab, we'll be using 4 `nvidia-l4` GPUs.
Locating available capacity for GPUs is surprisingly complex. First, lets get a
list of zones where our L4 GPUs exist:

```
gcloud compute accelerator-types list --filter 'name=nvidia-l4'
```

This lists all the zones where Google has deployed the desired GPU type. But
there may not be available GPU capacity in all these zones. AFAIK, there's no
way to inspect capacity without creating resources, so we'll have a bit of a
trial-and-error game to play with our node pool creation.

Google Cloud also has regional usage quotas for GPUs, especially the more
popular types of GPU. To inspect regional quotas, we'll use the `gcloud compute
regions describe` command (and a little `jq` to keep the output short)

```
gcloud compute regions describe us-central1 --format=json \
  | jq '{ name: .name, zones: .zones, quota: (.quotas[]| select(.metric=="NVIDIA_L4_GPUS") ) }'
```

This command gives us the region name, the list of zones in that region (which
we'll need at node-pool creation time), and the quota limit for Nvidia L4 GPUs.

```
{
  "name": "us-central1",
  "zones": [
    "https://www.googleapis.com/compute/v1/projects/MY_PROJECT/zones/us-central1-a",
    "https://www.googleapis.com/compute/v1/projects/MY_PROJECT/zones/us-central1-b",
    "https://www.googleapis.com/compute/v1/projects/MY_PROJECT/zones/us-central1-c",
    "https://www.googleapis.com/compute/v1/projects/MY_PROJECT/zones/us-central1-f"
  ],
  "quota": {
    "limit": 8.0,
    "metric": "NVIDIA_L4_GPUS",
    "usage": 0.0
  }
}
```

Nvidia L4 GPUs require the use of VMs in the [G2 Machine
series](https://cloud.google.com/compute/docs/gpus#l4-gpus)
Since our Gemma2 model uses 4 GPUs, we need to use the `g2-standard-48`
machine type.

This means our Node pool can only have 2 `g2-standard-48` machines in it,
before we hit our Regional GPU quota. With this in mind, we can now create our
Node pool with the following command:

```
gcloud container node-pools create my-gpu-pool \
   --accelerator type=nvidia-l4,count=4,gpu-driver-version=default \
   --machine-type g2-standard-48 \
   --region us-central1 \
   --cluster ${my-gke-cluster} \
   --node-locations us-central1-c,us-central1-b \
   --num-nodes=2
```

Our new node pool is automatically given a [kubernetes taint](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) to keep jobs
from scheduling there by default. The taint looks like this:

```
    taints:
    - effect: NoSchedule
      key: nvidia.com/gpu
      value: present
```

To schedule jobs on these nodes, our jobs must have Tolerations added to them.
The provided gemma pod definition includes the following toleration to schedule
on our gpu-enabled nodepool:

```
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
```

### Kubectl access

Now that our cluster has been created, and our GPU-accelerate node pool
attached, let's configure `kubectl` to talk to our cluster.

`gcloud container clusters get-credentials ${my-gke-cluster} --location
us-central1`

You should now be able to use `kubectl cluster-info` to view basic information
about your cluster. :tada:

### Set Up HuggingFace access

Workloads we deploy on our cluster will download the [Gemma2 model from
HuggingFace](https://huggingface.co/google/gemma-2-2b) when deployed. To allow
this, we'll need to create a HuggingFace Access Token, and add it to Secret
Manager.

First, we'll [create a HuggingFace access
token](https://huggingface.co/docs/hub/en/security-tokens) by following their
instructions. For this use case, the token requires only Read permissions.

Once you have the token, the following commands will store the token in a
kubernetes secret in your GKE cluster.

> [!warning]
> Kubernetes secrets are not encrypted by default. To ensure the safety of your
> secrets, follow the guidance in the [kubernetes Secrets docs](https://kubernetes.io/docs/concepts/configuration/secret/).

```
kubectl create secret generic hf-secret --from-literal=access_token=${my-access-token}
```

## Deploy Gemma2 from HuggingFace Hub

Next, we'll be deploying a pre-built image of TGI, and asking it to download a
pre-built Gemma2 model from HuggingFace Hub.

- `kubectl apply -f ./gemma-nvidia-l4.yaml`

It will take several minutes for your pods to become healthy, since the pod will
need to fetch the model data before serving.

Once the pods are ready, you can access your new Gemma through the
`gemma-service` kubernetes service, on port 8000.

## Recap

Running AI workloads is very similar to running traditional workloads on
kubernetes, but does require learning about the details of Taints and
Tolerations.

With Google Cloud, creating a cluster with available accelerators is quite a
complex and frustrating experience, especially in smaller regions with less
accelerator inventory. Unfortunately, there's no way to know if accelerators are
available except for trying to create a nodepool that reserves them. :sad:
