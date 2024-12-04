# Productionizing your AI workloads on GKE

The exercises in this repo cover a range of topics to help you run production
quality AI workloads in Googke Kubernetes Engine.

They are intended as a guide for developers and operators of "traditional" (that
is, non-AI) services that want to include AI workloads in their existing
Kubernetes clusters in a reliable and scalable way.

## Prerequisites

- A GKE cluster that includes GPUs and/or TPUs
    - If you don't already have one, you can [use this tutorial]() to create one.
- A http://huggingface.co account

## Exercises

These exercises are intended to (mostly) stand alone, though some will require
previous steps. I've done my best to be explicit about any dependencies below,
but if I missed any, feel free to file an issue.


### [Self Hosted Gemma2](TODO)

- **Objective**: Deploy your own version of Google's Gemma model to an existing
    GKE cluster.

This exercise deploys our LLM and discusses some of the first choices you'll
make in running AI workloads, like choosing accelerators and your model-serving
layer.

This establishes our starting point for the remaining exercises. If you already
have a model to play with, you're welcome to skip this step - though some
changes may be necessary.

### [Server Telemetry](TODO)

- **Objective**: Capture and explore metrics, traces, and logs that your model
    produces.

This exercise covers OpenTelemetry metrics and traces that are available from
HuggingFace's TGI server, and how to create SLOs to ensure your AI is performing
as expected.

### [03 - Scaling with expected traffic]()

- **Objective**: Understand the scaling characteristics of your model, and what
    factors affect the 'performance curve'.

Here we introduce the TGI benchmarking tool, and discuss how the shape of your
expected traffic affects your scaling choices. It also covers TGI's available
options that influence production behavior (for example, maximum concurrent
requests).

## Future Topics

### [Versioning and Rollouts](TODO)

We should cover some options for versioning and releasing, as well as
comparative analysis of two models.

### [Data management and global services](TODO)

LLMs are large bundles of data, and deploying them as a global service can lead
to unexpectedly large network usage, and slow deployments. We should discuss
some strategies for managing data on multiple continents and available
packaging strategies. (base models and PEFT layers)

### [Client Telemetry](TODO)

Understand what telemetry is available from clients that call your model. This
is a broad topic and needs to target one or two specific client libraries (e.g.
Langchain, etc).
