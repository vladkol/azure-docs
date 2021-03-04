---
title: Interactive distributed remote debugging with Visual Studio Code and Azure Debug Relay
titleSuffix: Azure Machine Learning
description: Interactively debug distributed remote Azure Machine Learning runs, pipelines, and deployments using Visual Studio Code and Azure Debug Relay
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.topic: conceptual
author: luisquintanilla
ms.author: luquinta
ms.date: 03/05/2021
---

# Interactive distributed remote debugging with Visual Studio Code and Azure Debug Relay



Learn how to interactively debug Azure Machine Learning experiments, pipelines, and deployments using Visual Studio Code (VS Code), [Azure Debug Relay](https://github.com/vladkol/azure-debug-relay), and [debugpy](https://github.com/microsoft/debugpy/).

## Why remote debugging with Azure Debug Relay

When preparing your Machine Learning pipeline code for running in Azure Machine Learning, you may need to troubleshoot it in real environments:
* Across one or more [Azure ML Compute Clusters](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-attach-compute-cluster?tabs=python), attached [Azure Kubernetes](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-attach-kubernetes?tabs=python) or [Azure Batch](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-attach-compute-targets#azbatch)
* In developer or production environment when debugging an issue that must to be reproduced there
* When you prefer to debug straight on Azure Machine Learning Compute Instances or Clusters

In some environments, compute resources and data stores may only be accessible in a virtual network and/or with a managed identity.

[Azure Debug Relay](https://github.com/vladkol/azure-debug-relay) creates secure private tunnels between compute nodes and your Visual Studio Code debug adapter for Python.
It also takes care of identifying which node needs to be connected to for debugging your code.

> [!TIP]
> When debugging on Azure ML Compute Clusters, we recommend to consider setting `Minimum Number of Nodes` to at least 1. It will reduce time of the inner loop every time when you launch a run with that cluster. If you launch a short pipeline multiple times withing a few minutes, you may also increase `Idle seconds before scale down` - by default it is 120 seconds.

> [!IMPORTANT]
> Both operations will result in charges for the number of nodes allocated.

## Prerequisites

* [Python 3](https://www.python.org/downloads/)
* [debugpy](https://pypi.org/project/debugpy/)
* [An existing Azure Relay namespace](https://docs.microsoft.com/en-us/azure/azure-relay/relay-hybrid-connections-http-requests-dotnet-get-started#create-a-namespace) and a [Hybrid Connection](https://docs.microsoft.com/en-us/azure/azure-relay/relay-hybrid-connections-http-requests-dotnet-get-started#create-a-hybrid-connection)
* [Azure Debug Relay (VS Code extension)](https://marketplace.visualstudio.com/items?itemName=VladKolesnikov-vladkol.azure-debug-relay)
* [Azure Debug Relay (Python package)](https://pypi.org/project/azure-debug-relay/)

## Create an Azure Relay Namespace and a Hybrid Connection

## Prepare your pipeline steps code to support Distributed Remote Debugging

## Prepare your Visual Studio Code workspace

## Debug your code

## Next steps

Now that you've set up VS Code with Azure Debug Relay, you can debug code interactively in any run or deployment in Azure Machine Learning.

Learn more about troubleshooting:

* [Local model deployment](how-to-troubleshoot-deployment-local.md)
* [Remote model deployment](how-to-troubleshoot-deployment.md)
* [Machine learning pipelines](how-to-debug-pipelines.md)
* [ParallelRunStep](how-to-debug-parallel-run-step.md)
