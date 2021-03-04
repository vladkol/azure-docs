---
title: Data prep with Spark pools (preview)
titleSuffix: Azure Machine Learning
description: Learn how to attach Spark pools for data preparation with Azure Synapse and Azure Machine Learning
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.topic: conceptual
ms.author: nibaccam
author: nibaccam
ms.reviewer: nibaccam
ms.date: 03/02/2021
ms.custom: how-to, devx-track-python, data4ml


# Customer intent: As a data scientist, I want to prepare my data at scale and to train my machine learning models from a single notebook.
---

# Attach Synapse Spark pools for data preparation with Azure Synapse (preview)

In this article, you learn how to attach and launch an Apache Spark pool backed by [Azure Synapse](/synapse-analytics/overview-what-is.md) for data preparation. 

>[!IMPORTANT]
> The Azure Machine Learning and Azure Synapse integration is in preview. The capabilities presented in this article employ the `azureml-synapse` package which contains [experimental](/python/api/overview/azure/ml/?preserve-view=true&view=azure-ml-py#stable-vs-experimental) preview features that may change at any time.

## Azure Machine Learning and Azure Synapse integration (preview)

The Azure Synapse integration with Azure Machine Learning (preview) allows you to attach an Apache Spark pool backed by Azure Synapse for interactive data exploration and preparation. With this integration, you can have a dedicated compute for data preparation at scale, all within the same Python notebook you use for training your machine learning models.

## Prerequisites

* [Create an Azure Machine Learning workspace](how-to-manage-workspace.md?tabs=python).

* [Create a Synapse workspace in Azure portal](../synapse-analytics/quickstart-create-workspace.md).

* [Create Apache Spark pool using Azure portal, web tools, or Synapse Studio](../synapse-analytics/quickstart-create-apache-spark-pool-portal.md)

* [Install the Azure Machine Learning Python SDK](/python/api/overview/azure/ml/install?preserve-view=true&view=azure-ml-py) which includes the `azureml-synapse` package (preview). 
    * You can also install it yourself, but it's only compatible with SDK versions 1.20 or higher. 
        ```python
        pip install azureml-synapse
        ```

## Link machine learning workspace and Synapse assets

Before you can attach a Synapse Spark pool for data preparation, your Azure Machine Learning workspace must be linked with your Azure Synapse workspace. 

You can link your ML workspace and Synapse workspace via the [Python SDK](#link-sdk) or the [Azure Machine Learning studio](#link-studio). 

> [!IMPORTANT]
> To link to the Synapse workspace successfully, you must be granted the **Owner** role of the Synapse workspace. Check your access in the [Azure portal](https://ms.portal.azure.com/).
>
> If you are not an **Owner** of the Synapse workspace, but want to use an existing linked service, see [Get an existing linked service](#get-an-existing-linked-service).


<a name="link-sdk"></a>
### Link workspaces with the Python SDK

The following code employs the [`LinkedService`](/python/api/azureml-core/azureml.core.linked_service.linkedservice?preserve-view=true&view=azure-ml-py) and [`SynapseWorkspaceLinkedServiceConfiguration`](/python/api/azureml-core/azureml.core.linked_service.synapseworkspacelinkedserviceconfiguration?preserve-view=true&view=azure-ml-py) classes to, 

* Link your machine learning workspace, `ws` with your Azure Synapse workspace. 
* Register your Synapse workspace with Azure Machine Learning as a linked service.

``` python
import datetime  
from azureml.core import Workspace, LinkedService, SynapseWorkspaceLinkedServiceConfiguration

# Azure Machine Learning workspace
ws = Workspace.from_config()

#link configuration 
synapse_link_config = SynapseWorkspaceLinkedServiceConfiguration(
    subscription_id=ws.subscription_id,
    resource_group= 'your resource group',
    name='mySynapseWorkspaceName')

# Link workspaces and register Synapse workspace in Azure Machine Learning
linked_service = LinkedService.register(workspace = ws,              
                                            name = 'synapselink1',    
                                            linked_service_config = synapse_link_config)
```
> [!IMPORTANT] 
> A managed identity, `system_assigned_identity_principal_id`, is created for each linked service. This managed identity must be granted the **Synapse Apache Spark Administrator** role of the Synapse workspace before you start your Synapse session. [Assign the Synapse Apache Spark Administrator role to the managed identity in the Synapse Studio](../synapse-analytics/security/how-to-manage-synapse-rbac-role-assignments.md).
>
> To find the `system_assigned_identity_principal_id` of a specific linked service, use `LinkedService.get('<your-mlworkspace-name>', '<linked-service-name>')`.

<a name="link-studio"></a>
### Link workspaces via studio

Link your machine learning workspace and Synapse workspace via the Azure Machine Learning studio with the following steps: 

1. Sign in to the [Azure Machine Learning studio](https://ml.azure.com/).
1. Select **Linked Services** in the **Manage** section of the left pane.
1. Select **Add integration**.
1. On the **Link workspace** form, populate the fields

   |Field| Description    
   |---|---
   |Name| Provide a name for your linked service. This name is what will be used to reference to this particular linked service.
   |Subscription name | Select the name of your subscription that's associated with your machine learning workspace. 
   |Synapse workspace | Select the Synapse workspace you want to link to. 
   
1. Select **Next** to open the **Select Spark pools (optional)** form. On this form, you select which Synapse Spark pool to attach to your workspace

1. Select **Next** to open the **Review** form and check your selections. 
1. Select **Create** to complete the linked service creation process.

## Get an existing linked service

To retrieve and use an existing linked service requires **User or Contributor** permissions to the Synapse workspace.

This example retrieves an existing linked service, `synapselink1`, from the workspace, `ws`, with the [`get()`](/python/api/azureml-core/azureml.core.linkedservice?preserve-view=true&view=azure-ml-py#get-workspace--name-) method.
```python
linked_service = LinkedService.get(ws, 'synapselink1')
```

### Manage linked services

To unlink your workspaces, use the `unregister()` method

``` python
linked_service.unregister()
```

View all the linked services associated with your machine learning workspace. 

```python
LinkedService.list(ws)
```
 
## Attach Synapse Spark pool as a compute

Once your workspaces are linked, attach a Synapse Spark pool as a dedicated compute resource for your data preparation tasks. 

You can attach Synapse Spark pools via,
* Azure Machine Learning studio
* [Azure Resource Manager (ARM) templates](https://github.com/Azure/azure-quickstart-templates/blob/master/101-machine-learning-linkedservice-create/azuredeploy.json)
* The Python SDK 

Follow these steps to attach a Synapse Spark pool using the studio. 

1. Sign in to the [Azure Machine Learning studio](https://ml.azure.com/).
1. Select **Linked Services** in the **Manage** section of the left pane.
1. Select your Synapse workspace.
1. Select **Attached Spark pools** on the top left. 
1. Select **Attach**. 
1. Select your Synapse Spark pool from the list and provide a name.  
    1. This list identifies the available Synapse Spark pools that can be attached to your compute. 
    1. To create a new Synapse Spark pool, see [Create Apache Spark pool with the Synapse Studio](../synapse-analytics/quickstart-create-apache-spark-pool-portal.md)
1. Select **Attach selected**. 


You can also employ the **Python SDK** to attach a Synapse Spark pool. 

The follow code, 
1. Configures the SynapseCompute with,

   1. The  LinkedService, `linked_service` that you either created or retrieved in the previous step. 
   1. The type of compute target you want to attach, `SynapseSpark`
   1. The name of the Synapse Spark pool. This must match an existing Apache Spark pool that is in your Synapse workspace.
   
1. Creates a machine learning ComputeTarget by passing in, 
   1. The machine learning workspace you want to use, `ws`
   1. The name you'd like to refer to the compute within the machine learning workspace. 
   1. The attach_configuration you specified  when configuring your SynapseCompute.
       1. The call to ComputeTarget.attach() is asynchronous, so the sample blocks until the call completes.

```python
from azureml.core.compute import SynapseCompute, ComputeTarget

attach_config = SynapseCompute.attach_configuration(linked_service, #Linked synapse workspace alias
                                                    type='SynapseSpark', #Type of assets to attach
                                                    pool_name="<Synapse Spark pool name>") #Name of Synapse spark pool 

synapse_compute = ComputeTarget.attach(workspace= ws,                
                                       name='<Synapse Spark pool alias in Azure ML>', 
                                       attach_configuration=attach_config
                                      )

synapse_compute.wait_for_completion()
```

Verify the Synapse Spark pool is attached.

```python
ws.compute_targets['Synapse Spark pool alias']
```

## Launch Synapse Spark pool for data preparation tasks

You can specify an [Azure Machine Learning environment](concept-environments.md) to use during your Synapse session. Only Conda dependencies specified in the environment will take effect. Docker image is not supported.

>[!WARNING]
>  Python dependencies specified in environment Conda dependencies are not supported in Synapse Spark pools. Currently, only fixed Python versions are supported. 
> Check your Python version by including  `sys.version_info` in your script.

The following code, creates the environment, `myenv`, which installs `azureml-core` version 1.20.0 and `numpy` version 1.17.0 before the session begins. You can then include this environment in your Synapse session `start` statement.

```python

from azureml.core import Workspace, Environment

# creates environment with numpy and azureml-core dependencies
ws = Workspace.from_config()
env = Environment(name="myenv")
env.python.conda_dependencies.add_pip_package("azureml-core==1.20.0")
env.python.conda_dependencies.add_conda_package("numpy==1.17.0")
env.register(workspace=ws)
```

To begin data preparation with the Synapse Spark pool, specify the Synapse Spark pool name and provide your subscription ID, the machine learning workspace resource group, the name of the machine learning workspace, and which environment to use during the Synapse session. 

> [!IMPORTANT]
> To continue use of the Synapse Spark pool you must indicate which compute resource to use throughout your data preparation tasks with `%synapse` for single lines of code and `%%synapse` for multiple lines. 

```python
%synapse start -c SynapseSparkPoolAlias -s AzureMLworkspaceSubscriptionID -r AzureMLworkspaceResourceGroupName -w AzureMLworkspaceName -e myenv
```

After the session starts, you can check the session's metadata.

```python
%synapse meta
```

## Load data from storage

Once your Synapse Spark session starts, read in the data that you wish to prepare. Data loading is supported for Azure Blob storage and Azure Data Lake Storage Generations 1 and 2.

There are two ways to load data from these storage services: 

* Directly load data from storage using its Hadoop Distributed Files System (HDFS) path.

* Read in data from an existing [Azure Machine Learning dataset](how-to-create-register-datasets.md).

To access these storage services, you need **Storage Blob Data Reader** permissions. If you plan to write data back to these storage services, you need **Storage Blob Data Contributor** permissions. [Learn more about storage permissions and roles](../storage/common/storage-auth-aad-rbac-portal.md#azure-roles-for-blobs-and-queues).

### Load data with Hadoop Distributed Files System (HDFS) path

To load and read data in from storage with the corresponding HDFS path, you need to have your data access authentication credentials readily available. These credentials differ depending on your storage type.  

The following code demonstrates how to read data from an **Azure Blob storage** into a Spark dataframe with either your shared access signature (SAS) token or access key. 

```python
%%synapse

# setup access key or SAS token
sc._jsc.hadoopConfiguration().set("fs.azure.account.key.<storage account name>.blob.core.windows.net", "<access key>")
sc._jsc.hadoopConfiguration().set("fs.azure.sas.<container name>.<storage account name>.blob.core.windows.net", "sas token")

# read from blob 
df = spark.read.option("header", "true").csv("wasbs://demo@dprepdata.blob.core.windows.net/Titanic.csv")
```

The following code demonstrates how to read data in from **Azure Data Lake Storage Generation 1 (ADLS Gen 1)** with your service principal credentials. 

```python

# setup service principal which has access of the data
sc._jsc.hadoopConfiguration().set("fs.adl.account.<storage account name>.oauth2.access.token.provider.type","ClientCredential")

sc._jsc.hadoopConfiguration().set("fs.adl.account.<storage account name>.oauth2.client.id", "<client id>")

sc._jsc.hadoopConfiguration().set("fs.adl.account.<storage account name>.oauth2.credential", "<client secret>")

sc._jsc.hadoopConfiguration().set("fs.adl.account.<storage account name>.oauth2.refresh.url",
https://login.microsoftonline.com/<tenant id>/oauth2/token)

df = spark.read.csv("adl://<storage account name>.azuredatalakestore.net/<path>")

```

The following code demonstrates how to read data in from **Azure Data Lake Storage Generation 2 (ADLS Gen 2)** with your service principal credentials. 

```python
# setup service principal which has access of the data
sc._jsc.hadoopConfiguration().set("fs.azure.account.auth.type.<storage account name>.dfs.core.windows.net","OAuth")
sc._jsc.hadoopConfiguration().set("fs.azure.account.oauth.provider.type.<storage account name>.dfs.core.windows.net", "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider")
sc._jsc.hadoopConfiguration().set("fs.azure.account.oauth2.client.id.<storage account name>.dfs.core.windows.net", "<client id>")
sc._jsc.hadoopConfiguration().set("fs.azure.account.oauth2.client.secret.<storage account name>.dfs.core.windows.net", "<client secret>")
sc._jsc.hadoopConfiguration().set("fs.azure.account.oauth2.client.endpoint.<storage account name>.dfs.core.windows.net",
https://login.microsoftonline.com/<tenant id>/oauth2/token)


df = spark.read.csv("abfss://<container name>@<storage account>.dfs.core.windows.net/<path>")

```

### Read in data from registered datasets

You can also get an existing registered dataset in your workspace and perform data preparation on it by converting it into a spark dataframe.  

The following example gets a registered TabularDataset, `blob_dset`, that references files in blob storage, and converts it into a spark dataframe. When you convert your datasets into a spark dataframe, you can leverage `pyspark` data exploration and preparation libraries.  

``` python

%%synapse
from azureml.core import Workspace, Dataset

dset = Dataset.get_by_name(ws, "blob_dset")
spark_df = dset.to_spark_dataframe()
```

## Perform data preparation tasks

After you've retrieved and explored your data, you can perform data preparation tasks.

The following code, expands upon the HDFS example in the previous section and filters the data in spark dataframe, `df`, based on the **Survivor** column and groups that list by **Age**

```python
%%synapse
from pyspark.sql.functions import col, desc

df.filter(col('Survived') == 1).groupBy('Age').count().orderBy(desc('count')).show(10)

df.show()

```

## Save data to storage and stop spark session

Once your data exploration and preparation is complete, store your prepared data for later use in your storage account on Azure.

In the following example, the prepared data is written back to Azure Blob storage and overwrites the original `Titanic.csv` file in the `training_data` directory. To write back to storage, you need **Storage Blob Data Contributor** permissions. [Learn more about storage permissions and roles](../storage/common/storage-auth-aad-rbac-portal.md#azure-roles-for-blobs-and-queues).

```python
%% synapse

df.write.format("csv").mode("overwrite").save("wasbs://demo@dprepdata.blob.core.windows.net/training_data/Titanic.csv")
```

When you've completed data preparation and saved your prepared data to storage, stop using your Synapse Spark pool with the following command.

```python
%synapse stop
```

## Create dataset to represent prepared data

When you're ready to consume your prepared data for model training, connect to your storage with an [Azure Machine Learning datastore](how-to-access-data.md), and specify which file(s) you want to use with an [Azure Machine Learning dataset](how-to-create-register-datasets.md).

The following code example,

* Assumes you already created a datastore that connects to the storage service where you saved your prepared data.  
* Gets that existing datastore, `mydatastore`, from the workspace, `ws` with the get() method.
* Creates a [FileDataset](how-to-create-register-datasets.md#filedataset), `train_ds`, that references the prepared data files located in the `training_data` directory in `mydatastore`.  
* Creates the variable `input1`, which can be used at a later time to make the data files of the `train_ds` dataset available to a compute target.

```python
from azureml.core import Datastore, Dataset

datastore = Datastore.get(ws, datastore_name='mydatastore')

datastore_paths = [(datastore, '/training_data/')]
train_ds = Dataset.File.from_files(path=datastore_paths, validate=True)
input1 = train_ds.as_mount()

```

## Example notebook

See this [end to end notebook](../synapse-analytics/overview-what-is.md) for a detailed code example of how to perform data preparation and model training from a single notebook with Azure Synapse and Azure Machine Learning.

## Next steps

* [Train a model](how-to-set-up-training-targets.md).
* [Train with Azure Machine Learning dataset](how-to-train-with-datasets.md)
* [Create an Azure machine learning dataset](how-to-create-register-datasets.md).

