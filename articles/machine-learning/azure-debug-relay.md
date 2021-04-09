---
title: Interactive distributed remote debugging with Visual Studio Code and Azure Debugging Relay
titleSuffix: Azure Machine Learning
description: Interactively debug distributed remote Azure Machine Learning runs, pipelines, and deployments using Visual Studio Code and Azure Debugging Relay
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.topic: conceptual
author: vladkol
ms.author: vladkol
ms.date: 03/05/2021
---

# Interactive distributed remote debugging with Visual Studio Code and Azure Debugging Relay

Learn how to interactively debug distributed Azure Machine Learning experiments, pipelines, and deployments using Visual Studio Code (VS Code), [Azure Debugging Relay](https://github.com/vladkol/azure-debug-relay), and [debugpy](https://github.com/microsoft/debugpy/).

## Why remote debugging with Azure Debugging Relay

When preparing your Machine Learning pipeline code for running in Azure Machine Learning, you may need to troubleshoot it in real environments:

* Across one or more [Azure ML Compute Clusters](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-attach-compute-cluster?tabs=python), attached [Azure Kubernetes](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-attach-kubernetes?tabs=python) or [Azure Batch](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-attach-compute-targets#azbatch)
* In developer or production environment when debugging an issue that must to be reproduced there
* When you prefer to debug straight on Azure Machine Learning Compute Instances or Clusters

In some environments, compute resources and data stores may only be accessible in a virtual network and/or with a managed identity.

[Azure Debugging Relay](https://github.com/vladkol/azure-debug-relay) creates secure private tunnels between compute nodes and your Visual Studio Code debug adapter for Python.
It also takes care of identifying which node needs to be connected to for debugging your code.

> [!TIP]
> When debugging on Azure ML Compute Clusters, we recommend to consider setting `Minimum Number of Nodes` to at least 1. It will reduce time of the inner loop every time when you launch a run with that cluster. If you launch a short pipeline multiple times withing a few minutes, you may also increase `Idle seconds before scale down` - by default it is 120 seconds.

> [!IMPORTANT]
> Both operations will result in charges for the number of nodes allocated.

## Prerequisites

* [Python 3](https://www.python.org/downloads/)
* [debugpy](https://pypi.org/project/debugpy/)
* [An existing Azure Relay namespace](https://docs.microsoft.com/en-us/azure/azure-relay/relay-hybrid-connections-http-requests-dotnet-get-started#create-a-namespace) and a [Hybrid Connection](https://docs.microsoft.com/en-us/azure/azure-relay/relay-hybrid-connections-http-requests-dotnet-get-started#create-a-hybrid-connection)
* [Azure Debugging Relay (VS Code extension)](https://marketplace.visualstudio.com/items?itemName=VladKolesnikov-vladkol.azure-debug-relay)
* [Azure Debugging Relay (Python package)](https://pypi.org/project/azure-debug-relay/)

## Create an Azure Relay Namespace and a Hybrid Connection

> [!NOTE]
> Here you are going to use a Shared Access Policy on the Azure Relay namespace level,
assuming that a single namespace will be shared between multiple developers working within an Azure Machine Learning workspace.
Each developers is expected to have an individual Hybrid Connection.

You can create an Azure Relay namespace and an Azure Relay Hybrid Connection by many ways:
manually in Azure Portal, via Azure CLI, Azure SDK, ARM templates, etc.

### In Azure Portal

1. [Create Azure Relay resource](https://ms.portal.azure.com/#create/Microsoft.Relay). Better make one in a region closest to your location.
1. Once created, switch to the resource, and add a hybrid connection (`+ Hybrid Connection` button), give it a memorable name (e.g. `test`) - this is your **Hybrid Connection Name**.
1. Switch to `Shared access policies` under `Settings` in the vertical panel.
1. Add a new policy with `Send` and `Listen` permissions.
1. Once created, copy its `Primary Connection String`, this is your **Connection String**.

### Using Azure CLI

Choose your name instead of `mydebugrelay1` for an Azure Relay resource, and your custom name for Hybrid Connection instead of `debugrelayhc1`. Same applies to `debugRelayResourceGroup` as resource group.

```cmd
az group create --name debugRelayResourceGroup --location westus2

az relay namespace create --resource-group debugRelayResourceGroup --name mydebugrelay1 --location westus2

az relay hyco create --resource-group debugRelayResourceGroup --namespace-name mydebugrelay1 --name debugrelayhc1

az relay namespace authorization-rule create --resource-group debugRelayResourceGroup --namespace-name mydebugrelay1 --name sendlisten1 --rights Send Listen

az relay namespace authorization-rule keys list --resource-group debugRelayResourceGroup --namespace-name mydebugrelay1 --name sendlisten1
```

Last command will show you something like this:

```json
{
  "keyName": "sendlisten",
  "primaryConnectionString": "Endpoint=sb://mydebugrelay1.servicebus.windows.net/;SharedAccessKeyName=sendlisten1;SharedAccessKey=REDACTED1",
  "primaryKey": "REDACTED1",
  "secondaryConnectionString": "Endpoint=sb://mydebugrelay1.servicebus.windows.net/;SharedAccessKeyName=sendlisten1;SharedAccessKey=REDACTED2",
  "secondaryKey": "REDACTED2"
}
```

Use `primaryConnectionString` or `secondaryConnectionString` value as your **Connection String**.

**Hybrid Connection Name** would be the one you choose instead of `debugrelayhc1`.

## Prepare your pipeline steps code to support Distributed Remote Debugging

### Add Azure Relay Shared Access Policy **Connection String** to your Azure Machine Learning Workspace Key Vault

```python
debug_connection_string = "" # Put your Connection String here
debug_connection_string_secret_name = "debugarglobalsecret"
aml_workspace.get_default_keyvault().set_secret(
        debug_connection_string_secret_name, debug_connection_string)
```

### Add debugging code to your steps and pipelines

> [!NOTE]
> You can find end-to-end example of a debug-ready Azure ML pipeline [here in Azure Debugging Relay repository](https://github.com/vladkol/azure-debug-relay/tree/main/samples/azure_ml_advanced).

#### Add the following packages to the environment conda dependencies of every step

* debugpy
* azure-debug-relay
* argparse

#### Add step arguments

* `--is-debug` - `True` if you want to debug this step, may be passed from a pipeline parameter
* `--debug-relay-connection-name`, Azure Relay Hybrid Connection name, may be passed from a pipeline parameter
* `--debug-port`, a debugging port (e.g. `5678`) for this step, **every step must use an individual debugging port**
* `--debug-relay-connection-string-secret`, workspace's Azure Relay connection name secret (`debug_connection_string_secret_name` with `debugarglobalsecret` value above)

Debugging flag and the hybrid connection name may be added as pipeline parameters.

Example (with PythonScriptStep)

```python
# Pipeline parameters to use with every run
is_debug = PipelineParameter("is_debug", default_value=False)
relay_connection_name = PipelineParameter(
    "debug_relay_connection_name", default_value="none")

# Create a step
single_step = PythonScriptStep(
        name=f"single-step",
        script_name="steps/single_step.py",
        source_directory=".",
        runconfig=run_config,
        arguments=[
            "--is-debug", is_debug,
            "--debug-relay-connection-name", relay_connection_name,
            "--debug-port", 5678,
            "--debug-relay-connection-string-secret", debug_connection_string_secret_name
        ],
        compute_target=aml_compute,
        allow_reuse=False
    )
```

#### Add the debugging code that handles `--is-debug` argument, initializes Azure Debugging Relay, and starts debugging

You can use [this example from Azure Debugging Relay repo](https://github.com/vladkol/azure-debug-relay/blob/main/samples/azure_ml_advanced/steps/amldebugutils/debugutils.py) as a reference. Once added that, all you need is to call `start_remote_debugging_from_args`.

Example:

```python
parser = argparse.ArgumentParser()
parser.add_argument('--is-debug', type=str, required=True)
    args, _ = parser.parse_known_args()

if args.is_debug.lower() == 'true':
    print("Let's start debugging")
    if start_remote_debugging_from_args():
        debugpy.breakpoint()
        print("We are debugging!")
    else:
        print("Could not connect to a debugger!")
```

## Prepare your Visual Studio Code workspace

Your code is ready for debugging. Now prepare Visual Studio Code.

Every step you want to debug requires a separate [launch configuration](https://code.visualstudio.com/docs/python/debugging) in Visual Studio Code. You add them to `.vscode/launch.json` file.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Listen 5678",
            "type": "python",
            "request": "attach",
            "listen": {
                "host": "127.0.0.1",
                "port": 5678
            },
            "pathMappings": [
                {
                    "localRoot": "${workspaceFolder}",
                    "remoteRoot": "."
                }
            ]
        },
        {
            "name": "Python: Listen 5679",
            "type": "python",
            "request": "attach",
            "listen": {
                "host": "127.0.0.1",
                "port": 5679
            },
            "pathMappings": [
                {
                    "localRoot": "${workspaceFolder}",
                    "remoteRoot": "."
                }
            ]
        }
    ]
}
```

> [!IMPORTANT]
> Each step's launch configuration must use an individual debugging port (`5678` and `5679` above).

Notice how the debugger maps paths on the local and the remote machines.
If your code has a different structure remotely, you may need to provide more sophisticated path mappings. Here is that piece in `.vscode/launch.json`:

```json
"pathMappings": [
{
    "localRoot": "${workspaceFolder}",
    "remoteRoot": "."
}]
```

It tells VS Code that the workspace directory locally is mapped to the "current" directory remotely.

If you want to debug a run with multiple steps, all individual step configurations should be grouped in a [compound configuration](https://code.visualstudio.com/docs/editor/debugging#_compound-launch-configurations):

```json
"compounds": [
{
    "name": "Python: 3 step pipeline",
    "configurations": [
    "Python: Listen 5678", 
    "Python: Listen 5679",
    "Python: Listen 5680"
    ]
}
]
```

[Here](https://github.com/vladkol/azure-debug-relay/blob/main/.vscode/launch.json) you may look at a combined example.

## Debug your code

### Start Visual Studio Code debugging configuration

Launch debugging in Visual Studio Code with previously created `AzureML Debug` configuration.

### Run the pipeline with parameters

Now your code is ready for debugging

* `is_debug` - True if this is a debugging run, supposed to be passed as `--is-debug` to debugged steps
* `debug_relay_connection_name` - Azure Relay Hybrid Connection name, supposed to be passed as `--debug-relay-connection-string-secret` to debugged steps

Example of the launch script:

```python
parser = argparse.ArgumentParser()
parser.add_argument("--is-debug", type=bool, required=False, default=False)
parser.add_argument("--debug-relay-connection-name",
                    type=str, required=False, default="")
options, _ = parser.parse_known_args()

pipeline_parameters = {
    "is_debug": options.is_debug
    }
if options.is_debug:
    if options.debug_relay_connection_name == "":
        raise ValueError("Hybrid connection name cannot be empty!")

    pipeline_parameters.update({
        "debug_relay_connection_name": options.debug_relay_connection_name
        })

published_pipeline.submit(workspace=aml_workspace, experiment_name=experiment_name,
                            pipeline_parameters=pipeline_parameters,
                            continue_on_step_failure=True)
```

### Debugging code

Once the pipeline reaches steps you want to debug, `debugpy.breakpoint()` call is supposed to invoke a breakpoint in Visual Studio Code. Alternatively, you can put breakpoints in VS Code editor.

While debugging, you can leverage all features benefits of interactive debugging in Visual Studio Code, its Python extension, and debugpy.

## Debugging pipelines with ParallelRunStep and MPIConfiguration

> [!IMPORTANT]
> When running distributed pipelines with [ParallelRunStep](https://docs.microsoft.com/en-us/python/api/azureml-pipeline-steps/azureml.pipeline.steps.parallel_run_step.parallelrunstep?preserve-view=true&view=azure-ml-py) or [MPIConfiguration](https://docs.microsoft.com/en-us/python/api/azureml-core/azureml.core.runconfig.mpiconfiguration?view=azure-ml-py),
you need to make sure that debugging only happens on a single node, and not all nodes that steps is running on.
You also need to set `process_count_per_node` parameter in `ParallelRunConfig` or `MpiConfiguration` to `1` (one).
Otherwise there will be network port conflicts.

With ParallelRunStep, to make sure we only debug it on a single node, we check that `AZ_BATCH_IS_CURRENT_NODE_MASTER` environment variable equals `true`.

```python
# debug mode and on the master node
if args.is_debug.lower() == 'true' and bool(os.environ.get('AZ_BATCH_IS_CURRENT_NODE_MASTER')):
    is_debug = True
    print("This is a mater node. Start a debugging session.")
    start_remote_debugging_from_args()
```

With MPIConfiguration, it depends on the distributed training framework. Ultimately, you need identify an instance with ** ran equal to zero*.
[Look into this guide](https://azure.github.io/azureml-web/docs/cheatsheet/distributed-training/) to understand how to detect the rank.

For example, in Horovod MPI Tensorflow steps in would be `horovod.tensorflow.rank()`, so the debugging check may look like the following example:

```python
import argparse
import horovod.tensorflow as hvd

# ...
# ...
if args.is_debug.lower() == 'true' and hvd.rank() == 0:
    print("Let's start debugging")
    if start_remote_debugging_from_args():
        debugpy.breakpoint()
```

> [!TIP]
If you need to handle these situations differently, and full support steps distributed across multiple nodes and processes per node,
you may need to add your own code for picking individual debugging ports for every instance of your training steps. Instead of passing port number as a parameter, steps can choose ports and "reserve" them by [adding an Azure ML Run property](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-manage-runs?tabs=python#tag-and-find-runs) - if a property with a certain name was already added, the port has been utilized and therefore cannot be used.
Multiple processes per node require first starting process to initialize a `DebugRelay` object with a list of ports to connect to.
You may need to employ a shared data structure and a locking mechanism to make sure processes know when DebugRelay
has been already initialized (checking for azrelay process also works).

> [!IMPORTANT]
> You still need to come up with a definitive number of ports because it configures how many debugging listeners start in Visual Studio Code. Every remote process you debug require its own port. Every port requires a configuration in `.vscode/launch.json`.
And there must be a compound configuration that includes all of them.

## Next steps

Now that you've set up VS Code with Azure Debugging Relay, you can interactively debug code in any run or deployment in Azure Machine Learning.

Learn more about troubleshooting:

* [Local model deployment](how-to-troubleshoot-deployment-local.md)
* [Remote model deployment](how-to-troubleshoot-deployment.md)
* [Machine learning pipelines](how-to-debug-pipelines.md)
