# Parallelizing a Containerized Application on Azure Batch using Python SDK
This guide is meant to act as a 0-100 guide to parallelize a containerized application using the Azure Batch Resource. We will step through pre-requisites, then break the problem into steps.  

0. Prerequisites
1. Break the problem into Parallelizable Tasks
2. Containerizing the Application
3. Setup Azure Resources 
4. Using the Python SDK to orchestrate the Azure Batch Resource.

# 0. Pre-Requisites:
###  Install Windows Subsystem for Linux [powershell]
    wsl --install
---
### Install Docker on Windows

> More detailed directions [here](https://docs.docker.com/desktop/windows/wsl/)

----
### Install Python 3.9
| At the time of writing the latest microsoft batch supported version python 3.9
> https://www.python.org/downloads/release/python-3913/
---
### Install Azure Python SDK Packages

```bash
py -m pip install azure-batch
py -m pip install azure-storage-blob
py -m pip install azure-identity
py -m pip install azure-key vault-secrets
py -m pip install azure-common
```

# 1. Break the problem into Parallelizable Tasks
## Running the Application:
Your batch job on Azure must consist of individual tasks, each running asynchronously. 

This requires your exectuable to:
- Take input and output through the command line parameters.
- Access any other resources it requires through command line parameters.


It’s good to build this out this way as it’s quick and easy to test as you break the problem into individual tasks.

> myapp.exe -i input.txt -o output.txt -c config.json

-i represents input data

-o represents the produced data

-c represents a json containing some configuration for the run, or some reference data which is used by the application on every run.


## Splitting the Job then Combining the Results:
Because this is a batch job, you'll want to be able to split the larger job into smaller tasks in some automated way, likely involving some splitting script. You identify some unit, which is the smallest task you can do in parallel with the other tasks, this becomes the focus of your splitting and your individual tasks. The further you split down your tasks, the more that it will cost you, but the faster your tasks will complete. There is a flat cost for every minute that you save by splitting your job further. If you split a job containing 1000 parallelizable units into 100 tasks of 10, you will complete it much slower than if you had 1000 tasks of 1. You incur a spin up time cost for the time that each node takes to start up its OS and pull the docker image. 

> py splitter.py BigJob.json -o input_folder/

You’ll also want a well-defined merging script to take the completed outputs and merge them meaningfully. This script will put out a lot of .txt files which represent the individual task each node will complete ie. input1.txt, input2.txt, ... 

> py recombiner.py output_folder/

Once your batch application can be split, run, and the results combined, you are ready to containerize and use Azure Batch.

# 2. Containerizing the Application

Azure nodes pull from a docker image to have all the dependencies and set up the environment to run the application. To build this image, you'll need to set up the dockerfile and build the docker image. 

---

## Application Wrapper
The application you're using must run its parameters from environment variables. An excellent way to do this is to set up a wrapper to pull environment variables and then call your command line application.

In this script, you copy over files from wherever they are stored to the node, then run them using your app. If any files are really large and stored in a zipped way, it’s best to unzip them here on the node. So they get copied to the node small, then ran.

myapp_wrapper.sh

```sh
#!/bin/bash
cp "${INPUT_PATH}" "."
cp "${CONFIG_PATH}" "."

exec myapp.exe -i "${INPUT_PATH}" -o "${OUTPUT_PATH}" -c "config.json"
```

> INPUT_PATH is the path to the file which contains instructions on the task the node is to run

> OUTPUT_PATH is the path to the file which will export the results of the given task

> The path of the config.json is the same for each task, so it can be a constant here. It just contains read only reference for this particular job.

---

## Dockerfile Setup 

You'll need to make your dockerfile to set up whatever environment your application requires. You pull from some base image, install whatever packages you need, then run your application. The base image is a docker image which is publically accessable which contains the environment you'll need to run your code. A base image follows the FROM keyword. Base images are availible in dockerhub for almost any environment. 

Here is a straightforward example of a dockerfile which uses the python base image. 

```docker
FROM python

RUN python -m pip install cowsay
COPY hello_world.py /hello_world.py

ENTRYPOINT ["python", "hello_world.py"]
```

> More detailed guides on building dockerfiles can be found [here](https://docs.docker.com/engine/reference/builder/)

For our case:

```docker
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get needed_package
COPY myapp.exe /myapp.exe
COPY myapp_wrapper.sh /myapp_wrapper.sh
ENTRYPOINT ["myapp_wrapper.sh"]
```

Once you have your docker file, you build an image using docker build.

    docker build -t myapp:v0.1 .

Once you have your docker image, you can call your app locally with 

    docker run myapp:v0.1 -v /Users/andy/mydata:/mydata -e INPUT_PATH=mydata/input.txt -e OUTPUT_PATH=mydata/ -e CONFIG_PATH=mydata/config.json
    

# 3. Setup Azure Resources

You will need to create the following Azure resources.

- Container Registry
- Storage Account
- Batch Resource
- Key Vault

We’ll go through the creation steps for setting up these resources one at a time.

---
## Azure Setup - Container Registry
This resource contains the docker image for your particular run, containing the environment for your application. This resource can be accessed by other Azure resources easily, and will be the place from which nodes pull in their environments. 

You can create this resource through the Azure Portal. Creation for this is relatively simple. You just need to select a Subscription/Resource group, then enter a name. 

Once this is created, you must push your built docker image to this container. 

This can be done by tagging your docker image with the name of the registry, then using the docker push command. 

You will first need to sign in by finding the access username and password of the container registry. **Username and Password can be found under the "Access keys" tab in the container registry on the Azure Portal.** 

```bash
docker login indatascience.azurecr.io -u USERNAME -p PASSWORD 
docker build -t myapp:v0.1 .
docker image tag myapp:v0.1 indatascience.azurecr.io/myapp:v0.1
docker push indatascience.azurecr.io/myapp:v0.1
```

You will also want to tag the image with the latest tag, then push that as the latest version, this will be the image version that each node will pull.

```bash
docker image tag myapp:v0.1 indatascience.azurecr.io/myapp:latest
docker push indatascience.azurecr.io/myapp:latest
```

---

## Azure Setup - Storage Account
This is where the Azure Batch resource will access the input and output data.

Creating one of these is pretty straightforward. You can do so through the Azure Portal.

--- 

## Azure Setup - Batch Service

This is the Azure Resource responsible for allocating Virtual Machines and performing and tracking the job. 

The creation is straightforward and can also be done through the Azure Portal.

Once created, some additional configuration steps must be taken.

1. You will need to connect the Storage Account you have made by going into the batch resource, selecting the Storage account, and setting an authentication mode (Storage Keys)

2. You will also need to go to quotas and request a quota increase (increase the allocated amount of nodes from azure, either spot or dedicated.) This can sometimes take a couple of days, depending on the number of nodes needed and Microsoft’s current availability.

---

## Azure Setup - Key Vault

This resource will hold your secrets for each other resources in your use case. You'll need to give permissions to those who will need to use this resource to authenticate.

You can create an Azure Key Vault using the Azure Portal.

Some additional configuration steps must be taken as well:

1. You will then need to put the secrets from the Container Registry, the Storage Account, and the Batch Service inside the Key Vault. Under the Secrets Tab.

    These can be found under each resource’s access key tabs. The Key is the value you are looking for here. 

2. You will also need to authenticate users who will be using this under the 
Access Policies Tab. You'll need to update this when you want to permit people to use this.

# 4. Orchestrating Azure Resources with the Python SDK

To use Azure Batch, you need to perform several actions:
- Authenticate
- Upload files
- Create a pool, a job, and tasks
- Monitor Tasks
- Download Output Files
- Clean Up Azure Resources

You can orchestrate these tasks using the Azure Python SDK. These can be part of the same script or broken into different functions. 

These are the imports used for this tutorial:

```python
import azure.batch._batch_service_client as batch
import azure.batch.models as batchmodels
from azure.storage.blob import ContainerClient, BlobServiceClient
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
from azure.common.credentials import ServicePrincipalCredentials
from os import listdir
import os
from uuid import uuid4
import re
import tempfile

TENANT_ID = "your-tenant-id"
CLIENT_ID = "your-client-id"
RESOURCE = "https://batch.core.windows.net/"

```
You will also want to figure out what your TENANT_ID and CLIENT_ID are, as they are used in the following Authentication step. You can find these in the Azure portal under Azure Active Directory.

---

### Authentication:
When you are authenticating through Azure, you should use the default Azure Credential, which will allow you to authenticate the user using their Microsoft Account.

Once you've authenticated with your given credential, you can fetch the secrets for the other resources you'll need.

```python

credential = DefaultAzureCredential(
    exclude_interactive_browser_credential=False
)
keyVaultName = "dev-mykeyvault-kv"
KVUri = f"https://{keyVaultName}.vault.azure.net"
secret_client = SecretClient(vault_url=KVUri, credential=credential)
container_registry_secret = secret_client.get_secret("container-registry")
storage_account_secret = secret_client.get_secret("storage-account")
batch_resource_secret = secret_client.get_secret("batch-resource")

# Setup Service Principal Credentials
credential = ServicePrincipalCredentials(
    tenant=TENANT_ID,
    client_id=CLIENT_ID,
    secret=heavy_lift_secret.value,
    resource=RESOURCE,
)
```

---

### Uploading Files:

You will need to upload files unique to each task to Azure, so that they can be processed. You create a container, then upload the files your application needs to that container. You also set up the output container during this step.

```python
# generate UUID for the job
uuid = uuid4()

# make directories and upload input files/database files
connect_str = f"DefaultEndpointsProtocol=https;AccountName=mystorageaccount;AccountKey={blob_file_system_secret.value};EndpointSuffix=core.windows.net"
blob_client = BlobServiceClient.from_connection_string(connect_str)
input_container_name = f"{uuid}-input"
input_container_client = blob_client.create_container(input_container_name)
output_container_name = f"{uuid}-output"
output_container_client = blob_client.create_container(output_container_name)

# Get config file names, upload config files
print("Uploading input Files")
config_paths = listdir(config_directory_path)
for config_file in config_paths:
    with open(config_directory_path + "\\" + config_file, "rb") as data:
        upload_name = config_file.split("\\")[-1]
        input_container_client.upload_blob(f"config/{upload_name}", data)
print("Uploaded Files to Azure")
print(f"Successfully Uploaded Job :{str(uuid)}")
```

---

### Create Pool, a Job, and Tasks
This is the most complicated part of using azure batch to parallelize an application. 

Now that you've submitted your files, and they are up on an Azure Storage Container, you'll need to create a Pool of VMs, specify your Job, and create your Tasks. This is where you will set your parameters for your VMs and set up autoscaling.

> Pool: Collection of virtual machines, pulls from a docker image on spin up.

> Job: Collection of Tasks, assigned to a pool.

> Task: Individual command line call assigned to a single node, representing the single unit of work in a parallelized task.

You first need to create a pool. This will be populated with individual nodes.

The following is a code snippet that creates pools, jobs, and tasks, all named with a UUID to keep track of the Azure Resources associated with a particular job.

```python
input_container_client = blob_client.get_container_client(
    f'{uuid}-input'
)

# Setup Service Principal Credentials
credential = ServicePrincipalCredentials(
    tenant=TENANT_ID,
    client_id=CLIENT_ID,
    secret=heavy_lift_secret.value,
    resource=RESOURCE,
)

# Setup needed clients:
batch_client = batch.BatchServiceClient(
    credential, "https://pcldevindliftsbatchba.westus.batch.azure.com"
)

pool_id = f"{uuid}"
job_id = f"{uuid}"

# make directories and upload config files/database files
input_container_client = ContainerClient.from_connection_string(
    connect_str, container_name="heavylift-input"
)
blob_client = BlobServiceClient.from_connection_string(connect_str)
input_container_name = f"{uuid}-input"
input_container_client = ContainerClient.from_connection_string(
    connect_str, container_name=input_container_name
)
output_container_name = f"{uuid}-output"
output_container_client = ContainerClient.from_connection_string(
    connect_str, container_name=output_container_name
)

# Specify Image
image_ref_to_use = batchmodels.ImageReference(
    publisher="microsoft-azure-batch",
    offer="ubuntu-server-container",
    sku="20-04-lts",
    version="latest",
)

# Specify a container registry
container_registry = batchmodels.ContainerRegistry(
    registry_server="indatascience.azurecr.io",
    user_name="indatascience",
    password=container_registry_secret.value,
)

# Create container configuration, prefetching Docker images from the container registry
container_conf = batchmodels.ContainerConfiguration(
    container_image_names=["indatascience.azurecr.io/heavylift:latest"],
    container_registries=[container_registry],
)

# Create input storage account mapping
heavylift_input = batchmodels.AzureBlobFileSystemConfiguration(
    account_name="pcldevindliftsbatchba",
    container_name=input_container_name,
    account_key=blob_file_system_secret.value,
    relative_mount_path=input_container_name,
)

# Create output storage account mapping
heavylift_output = batchmodels.AzureBlobFileSystemConfiguration(
    account_name="pcldevindliftsbatchba",
    container_name=output_container_name,
    account_key=blob_file_system_secret.value,
    relative_mount_path=output_container_name,
)

# Create Pool add configurations
new_pool = batchmodels.PoolAddParameter(
    id=pool_id,
    virtual_machine_configuration=batchmodels.VirtualMachineConfiguration(
        image_reference=image_ref_to_use,
        container_configuration=container_conf,
        node_agent_sku_id="batch.node.ubuntu 20.04",
    ),
    mount_configuration=[
        batchmodels.MountConfiguration(
            azure_blob_file_system_configuration=heavylift_input
        ),
        batchmodels.MountConfiguration(
            azure_blob_file_system_configuration=heavylift_output
        ),
    ],
    vm_size=vm_size,
    target_low_priority_nodes=0,
)

"""Creates the specified pool if it doesn't already exist

:param batch_client: The batch client to use.
:type batch_client: `batchserviceclient.BatchServiceClient`
:param pool: The pool to create.
:type pool: `batchserviceclient.models.PoolAddParameter`
"""
try:
    print("Attempting to create pool:", new_pool.id)
    batch_client.pool.add(new_pool)
    print("Created pool:", new_pool.id)
except batchmodels.BatchErrorException as e:
    if e.error.code != "PoolExists":
        raise
    else:
        print("Pool {!r} already exists".format(new_pool.id))
        raise

# Set up autoscale formula

formula = f"""maxNumberofVMs = {num_nodes};
            maxPendingTasks = $PendingTasks.GetSample(1);
            $TargetLowPriorityNodes=min(maxNumberofVMs, maxPendingTasks);
            $NodeDeallocationOption=taskcompletion;"""

response = batch_client.pool.enable_auto_scale(
    pool_id,
    auto_scale_formula=formula,
    auto_scale_evaluation_interval=timedelta(minutes=5),
    pool_enable_auto_scale_options=None,
    custom_headers=None,
    raw=False,
)

# Set maximum number of task retries.

job_constraints = batchmodels.JobConstraints(max_task_retry_count=3)

# Create Job
job = batchmodels.JobAddParameter(
    id=job_id,
    pool_info=batchmodels.PoolInformation(
        pool_id=pool_id,
    ),
    constraints=job_constraints,
)

batch_client.job.add(job)

# Create Tasks

blob_list = input_container_client.list_blobs()

user = batchmodels.UserIdentity(
    auto_user=batchmodels.AutoUserSpecification(
        elevation_level=batchmodels.ElevationLevel.admin,
        scope=batchmodels.AutoUserScope.task,
    )
)

taskNum = 0

for inputFile in blob_list:
    # TODO Verify inputFiles have needed data only.
    if re.match("config/Config_file[0-9]*.json", inputFile["name"]):
        taskNum = inputFile["name"].split("config/Config_file")[-1]
        taskNum = taskNum[:-5]
        taskName = "HeavyLift-Task" + taskNum
        outPath = "OUTPUT_PATH=/output/" + "outFile" + str(taskNum)

        containerOptions = f'-w /heavylift/ -v $AZ_BATCH_NODE_MOUNTS_DIR/{input_container_name}:/input -v $AZ_BATCH_NODE_MOUNTS_DIR/{output_container_name}:/output -e DB_PATH=/input/database/HeavyLiftSQLite.zip -e CONFIG_PATH=/input/{inputFile["name"]} -e OUTPUT_PATH=/output/'
        task_container_settings = batchmodels.TaskContainerSettings(
            image_name="indatascience.azurecr.io/heavylift:latest",
            container_run_options=containerOptions,
        )

        task = batchmodels.TaskAddParameter(
            id=taskName,
            command_line="",
            user_identity=user,
            container_settings=task_container_settings,
        )

        batch_client.task.add(job_id, task)
print(f"Successfully started job: {uuid}")
```

---

### Monitoring Task Progression
The simplest way to monitor running tasks on azure batch is to use either the  [Batch Explorer App](https://azure.github.io/BatchExplorer/microsoft) , or the [Azure Portal Desktop App](https://azure.microsoft.com/en-us/get-started/azure-portal/). These let you watch your nodes/jobs complete, and let you monitor the nodes as they work. You can also access the other resources used in this tutorial through the azure portal, which can help diagnose problems. 

---

### Download Output Files

Once a run has completed, the nodes output the files up into an Azure Storage Container, where they can stay until you need to pull them, then merge the results into a single solution.

```python
# Get blob client, and list names of output files
connect_str = f"DefaultEndpointsProtocol=https;AccountName=pcldevindliftsbatchba;AccountKey={blob_file_system_secret.value};EndpointSuffix=core.windows.net"
blob_client = BlobServiceClient.from_connection_string(connect_str)
output_container_name = f"{uuid}-output"
output_container_client = blob_client.get_container_client(
    output_container_name
)

output_blobs = output_container_client.list_blobs()

for a blob in output_blobs:
    blob_path = export_path + r"\\" + blob.name
    with open(blob_path, "wb") as download_file:
        download_file.write(
            output_container_client.download_blob(blob.name).readall()
        )
    print(f"Downloaded {blob.name}")
print(f"Download completed at: {export_path}")
```

### Merge Results 
Now you can merge these results, this may be sending them to an SQL server, forwarding them to some application, or producing some other artifact.

> py recombiner.py output_folder/

---

### Cleaning Up Azure Resources

After a run has been completed and the data has been pulled. However, the resources continue to exist, the job remains, and the pool (although it will no longer have nodes) remains on Azure. 

- The input data in the Azure Storage Container
- The output data in the Azure Storage Container
- The batch job
- The batch pool

These artifacts must be cleaned up after you are finished with them; the following code snippet details how you can clear the data associated with a particular job.

> Not cleaning up the azure resources over time can accumulate additional costs, as the Azure Storage Containers incur cost proportional to the amount of stored data. Batch also incurs additional costs. Additionally, Batch resources can have limits on number of pools created, so at some point you may be forced to clean these up.

```python
print("Deleting azure storage containers")
# delete used directories and upload config files/database files
connect_str = f"DefaultEndpointsProtocol=https;AccountName=pcldevindliftsbatchba;AccountKey={blob_file_system_secret.value};EndpointSuffix=core.windows.net"
blob_client = BlobServiceClient.from_connection_string(connect_str)
output_container_name = f"{uuid}-output"
output_container_client = blob_client.get_container_client(
    output_container_name
)
output_container_client.delete_container()
input_container_name = f"{uuid}-input"
input_container_client = blob_client.get_container_client(input_container_name)
input_container_client.delete_container()

print("Deleting batch jobs")
batch_client = batch.BatchServiceClient(
    credential, "https://pcldevindliftsbatchba.westus.batch.azure.com"
)

# Get the list of jobs that match the UUID
uuid_jobs = []
job_list = batch_client.job.list()
for job in job_list:
    job_name = job.as_dict()["id"]
    if uuid in job.as_dict()["id"]:
        uuid_jobs.append(job_name)
for job in uuid_jobs:
    job_status = batch_client.job.delete(job)

print("Deleting batch pools")
uuid_pools = []
pool_list = batch_client.pool.list()
for pool in pool_list:
    pool_name = pool.as_dict()["id"]
    if uuid in pool.as_dict()["id"]:
        uuid_pools.append(pool_name)

for pool in uuid_pools:
    pool_status = batch_client.pool.delete(pool)

print(f"Project {uuid} deleted from batch.")
```

You have completed your Parallelized Run, merged your Results, and cleaned up the Associated Azure Resources.

# Additional Resources

> [Quickstart - Use Python API to run an Azure Batch job](https://docs.microsoft.com/en-us/azure/batch/quick-run-python) - A quick guide on how to spin up a very simple Batch job, useful for basic syntax. 


> [Azure Batch Libraries for Python Reference Documentation](https://docs.microsoft.com/en-us/python/api/overview/azure/batch?view=azure-python) - Specific documentation for the Azure Batch Python SDK, useful for looking into specific datatypes in the SDK.

> [Batch Explorer App](https://azure.github.io/BatchExplorer/microsoft) - A tool for managing the Batch resource.


> [Azure Portal Desktop App](https://azure.microsoft.com/en-us/get-started/azure-portal/) - A tool for managing Azure resources.

> [Batch Pricing](https://azure.microsoft.com/en-ca/pricing/details/batch/) - Pricing for Azure Nodes, useful for deciding what node is best for your application (given memory/cpu restrictions).

> [Batch Auto-Scaling Documentation](https://docs.microsoft.com/en-us/azure/batch/batch-automatic-scaling) - 
Instructions on how to setup an autoscale function, so that nodes can automatically dealocate as tasks complete.


