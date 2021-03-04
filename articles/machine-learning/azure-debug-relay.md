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
ms.date: 05/04/2021
---

# Interactive distributed remote debugging with Visual Studio Code and Azure Debug Relay



Learn how to interactively debug Azure Machine Learning experiments, pipelines, and deployments using Visual Studio Code (VS Code), [Azure Debug Relay](https://github.com/vladkol/azure-debug-relay), and [debugpy](https://github.com/microsoft/debugpy/).

## Why remote debugging

Use the Visual Studio code and Azure Debug Relay to troubleshoot and validate your machine learning experiments and pipelines.

### Prerequisites

* [Python 3](https://www.python.org/downloads/)
* [debugpy](https://pypi.org/project/debugpy/)
* [Azure Debug Relay (VS Code extension)(https://marketplace.visualstudio.com/items?itemName=VladKolesnikov-vladkol.azure-debug-relay)
* [Azure Debug Relay (Python package)](https://pypi.org/project/azure-debug-relay/)

## Next steps

Now that you've set up VS Code Remote, you can use a compute instance as remote compute from VS Code to interactively debug your code. 

Learn more about troubleshooting:

* [Local model deployment](how-to-troubleshoot-deployment-local.md)
* [Remote model deployment](how-to-troubleshoot-deployment.md)
* [Machine learning pipelines](how-to-debug-pipelines.md)
* [ParallelRunStep](how-to-debug-parallel-run-step.md)
