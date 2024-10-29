---
layout: post
author: Ben Clifford
title: Removing Channels
excerpt: In November 2024, Parsl's little used Channel abstraction is going away. Here are some details.
---

Since the beginning, Parsl has had facilities for executing code somewhere else: for example, the High Throughput Executor helps you run tasks on different nodes within a supercomputer. Beyond that, the <em>channel</em> abstraction was aimed at getting the High Throughput Executor (and it's predecessors) running on some remote site - to support use cases such as running a workflow on two supercomputers at once, or running your workflow on your desktop PC with tasks executing on a supercomputer backend.

This multisite behaviour is complicated, and the channels ignores most of the complications, instead focusing on a tiny piece of the problem: how to run a batch job submission on a remote site. This goes back decades: for example the Globus Toolkit GRAM component, from largely the same groups as Parsl and released in the late 1990s, had the same focus. In part because Parsl doesn't do a good job (or any job, in some cases) with the other complications, not many people run multisite workloads.

More recently, the groups which might have pushed on Parsl's remote execution have put their energy into developing <a href="https://www.globus.org/compute">Globus Compute</a> and imagining the way forward as Parsl submitting tasks into Globus Compute. Yadu gave a talk on the most recent work (<a href="https://youtu.be/6TFTzIdWwUg">video</a>, <a href="https://parsl-project.org/parslfest/2024/Babuji-Yadu_PF24-Channels-and-GCE.pdf">slides</a> ) at <a href="https://parsl-project.org/parslfest/parslfest2024.html">ParslFest 2024</a>, and Reid wrote this blog post about <a href="https://parsl-project.org/2024/06/26/parsl-globus-compute.html">earlier techniques</a>.

So with a strong abandonware vibe around channels, and with that code tangled throughout the codebase complicating understanding and maintenance, I advocated for removal of channels in <a href="https://github.com/Parsl/parsl/issues/3515">GitHub issue #3515</a>. This is scheduled for 7th of November, 2024 - a week in the future, at time of writing.


=== bits

channel removal:
prototype PR https://github.com/Parsl/parsl/pull/3650
discussion issue: https://github.com/Parsl/parsl/pull/3650



=== reid's original post

<a href="https://www.globus.org/compute" target="_blank">Globus Compute</a> provides a unified interface for securely and reliably accessing remote compute resources, from laptops to campus clusters, clouds, and supercomputers. Users expose their resources as function-serving endpoints by installing the Globus Compute endpoint agent, which utilizes Parsl to handle the various execution models of each resource. At a high level, we can conceptualize Globus Compute as an extension of Parsl, transforming a distributed network of Parsl executors into a Function as a Service (FaaS) platform.

<img src="/images/blog/2024-06-26/globus-compute.png" width="95%" style="border:0px solid black;" alt="Illustration of Globus Compute services.">

Let's walk through a simple example to help illustrate how this works. Imagine you are a researcher performing compute-intensive tasks on experiment data. You have secured allocations on <a href="https://www.alcf.anl.gov/polaris" target="_blank">Polaris at ALCF</a> and <a href="https://www.nersc.gov/systems/perlmutter/" target="_blank">Perlmutter at NERSC</a>, and are looking for a platform to help construct a workflow to simultaneously utilize both systems. You are familiar with Parsl and want to run the workflow from your laptop, so Globus Compute is a natural choice.

First, you will need to `pip install globus-compute-endpoint` on each allocation. Polaris uses a PBS scheduler, while Perlmutter uses Slurm. The YAML configuration file for your Polaris endpoint might look like the following:

```yaml
display_name: My Polaris Endpoint 
engine:
    type: GlobusComputeEngine
    max_workers_per_node: 1
    address:
        type: address_by_interface
        ifname: bond0
    provider:
        type: PBSProProvider
        account: example_alcf_account
        queue: preemptable
        cpus_per_node: 32
        select_options: ngpus=4
        scheduler_options: "#PBS -l filesystems=home:grand:eagle"
        worker_init: "module load Anaconda; source activate compute_venv"
        walltime: 01:00:00
        nodes_per_block: 1
        init_blocks: 0
        min_blocks: 0
        max_blocks: 2
        launcher:
            type: MpiExecLauncher
            bind_cmd: --cpu-bind
            overrides: --depth=64 --ppn 1
```

When activated, this Globus Compute endpoint will launch a `GlobusComputeEngine`, which subsequently launches Parsl's `HighThroughputExecutor`, `PBSProProvider`, and `MpiExecLauncher`. In this example, the only configuration option that the endpoint will not pass through to Parsl is `display_name`, which specifies the name that is visible in the <a href="https://app.globus.org/compute" target="_blank">Globus Compute web interface</a>.

You may also want to share access to your endpoints with specific research colleagues using <a href="https://www.globus.org/globus-auth-service" target="_blank">Globus Auth</a>. You can do this by setting the `multi_user` configuration option to `true`, then modifying a few other files. The details on multi-user endpoints are beyond the scope of this blog post, but you can learn more <a href="https://globus-compute.readthedocs.io/en/main/endpoints/multi_user.html" target="_blank">here</a>.

On the client side, Parsl apps and the Globus Compute SDK can work well together. For example, Parsl apps can delegate tasks to Globus Compute endpoints:

```python
import pathlib

import globus_compute_sdk as gc
import parsl

polaris_gce = gc.Executor(endpoint_id=...)
perlmutter_gce = gc.Executor(endpoint_id=...)


def pre_process(data):
	...
	return data


def analysis(data):
	...
	return data


@parsl.join_app
def run_pre_process(data):
	return perlmutter_gce.submit(pre_process, data)


@parsl.join_app
def run_analysis(data):
	return polaris_gce.submit(analysis, data)


@parsl.python_app
def post_process(data):
	...
	return data


futures = []
dir_path = pathlib.Path("./experiments")
with parsl.load():
    for file_path in dir_path.iterdir():
        data = file_path.read_text()
            futures.append(post_process(analysis(pre_process(data))))
        for f in as_completed(futures):
            print(f.result())

```

This workflow delegates pre-processing to your Globus Compute endpoint on Perlmutter at NERSC, analysis to your endpoint on Polaris at ALCF, and post-processing to Parsl workers on your local machine. We utilize Parsl's `@join_app` decorator to avoid allocating Parsl workers while waiting for task results from your Globus Compute endpoints.

It is also possible to launch Parsl workflows from within Globus Compute tasks. The recommended setup is to run a Globus Compute endpoint on a login node with default configuration, then submit a task containing the entire Parsl workflow. You can learn more about this methodology <a href="https://globus-compute.readthedocs.io/en/latest/tutorial.html#running-parsl-workflows" target="_blank">here</a>.  

The final step is to kick back, grab a drink, and wait for your distributed workflow to return some groundbreaking results. You can rely on Parsl and Globus Compute to handle the rest.

Interoperability between these platforms will continue to expand over the next few years, enabling new and exciting capabilities for researchers. We will do our best to keep this blog up-to-date as we roll out related features and improvements.

👋 Until next time.
