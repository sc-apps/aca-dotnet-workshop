---
title: Module 4 - ACA State Store With Dapr State Management API
has_children: false
nav_order: 4
canonical_url: 'https://bitoftech.net/2022/08/29/azure-container-apps-state-store-with-dapr-state-management-api/'
---
# Module 4 - ACA State Store With Dapr State Management API

In this module, we will switch the in-memory store of tasks and use a key/value persistent store (Azure Cosmos DB) by using the [Dapr State Management Building Block](https://docs.dapr.io/developing-applications/building-blocks/state-management/state-management-overview/), we will see how we can store the data in Azure Cosmos DB without installing any Cosmos DB SDK or write specific code to integrate our Backend API with Azure Cosmos DB.
Moreover, we will use Redis to store tasks when we are running the application locally, you will see that we can switch between different stores without any code changes, thanks to the [Dapr pluggable state stores feature](https://docs.dapr.io/developing-applications/building-blocks/state-management/state-management-overview/#pluggable-state-stores)! It is a matter of adding new Dapr Component files and the underlying store will be changed. This page shows the [supported state stores](https://docs.dapr.io/reference/components-reference/supported-state-stores/) in Dapr.

![dapr-stateapi-cosmosdb](../../assets/images/04-aca-dapr-stateapi/dapr-stateapi-cosmosdb.jpg)

### Overview of Dapr State Management API

Dapr's state management API to save, read, and query key/value pairs in the supported state stores. To try this out and without doing any code changes or installing any NuGet packages we can directly invoke the State Management API and store the data on Redis locally. When you  initialized Dapr in your local development environment, it installed Redis container instance locally, so we can use Redis locally to store and retrieve state. If you navigate to the path `<UserProfile>\.dapr\components` you find a file named `statestore.yaml`. Inside this file, you will see the properties needed to access the local Redis instance. The [state store template component file structure](https://docs.dapr.io/operations/components/setup-state-store/) can be found on this link.

To try out the State Management APIs, run the Backend API from VS Code by running the below command or using the Run and Debug tasks we have created in the previous post.

```powershell
~\TasksTracker.ContainerApps\TasksTracker.TasksManager.Backend.Api> dapr run --app-id tasksmanager-backend-api --app-port 7088 --dapr-http-port 3500 --app-ssl dotnet run
```

Now from any rest client, invoke the below POST request to the endpoint `http://localhost:3500/v1.0/state/statestore`

```http
POST /v1.0/state/statestore HTTP/1.1
Host: localhost:3500
Content-Type: application/json
[
    {
        "key": "Book1",
        "value": {
            "title": "Parallel and High Performance Computing",
            "author": "Robert Robey",
            "genre": "Technical"
        }
    },
    {
        "key": "Book2",
        "value": {
            "title": "Software Engineering Best Practices",
            "author": "Capers Jones",
            "genre": "Technical"
        }
    },
    {
        "key": "Book3",
        "value": {
            "title": "The Unstoppable Mindset",
            "author": "Jessica Marks",
            "genre": "Self Improvement",
            "formats":["kindle", "audiobook", "papercover"]
        }
    }
]
```

What we've done here is the following:
- The value `statestore` in the endpoint should match the `name` value in the global component file `statestore.yaml`
- We have sent a request to store 3 entries of books, you can put any JSON representation in the value property

To see the results visually, you can install a VS Code extension to connect to Redis DB and see the results, in my case, I'm using the extension named "Redis Xplorer", after you connect to Redis locally, you should see the 3 entries similar to the below image. Notice how each entry key is prefixed by the Dapr App Id, in our case it is `tasksmanager-backend-api`. More about [key prefix strategy](#key-prefix-strategies) later in the post.

![dapr-stateapi-redis](../../assets/images/04-aca-dapr-stateapi/dapr-api-redis.jpg)

To get the value of a key, you need to issue a GET request to the endpoint `http://localhost:3500/v1.0/state/statestore/{YourKey}` and you should receive the value of the key store, in our case if we do a GET `http://localhost:3500/v1.0/state/statestore/Book3` the results will be the below object:

```JSON
{
    "formats": [
        "kindle",
        "audiobook",
        "papercover"
    ],
    "title": "The Unstoppable Mindset",
    "author": "Jessica Marks",
    "genre": "Self Improvement"
}
```

### Use Dapr Client SDK for State Store Management

Now we will introduce a change on the Backend API and create a new service named `TasksStoreManager.cs` which will implement the interface `ITasksManager.cs` to start storing tasks data on the persistence store. Locally we will start testing with Redis, then we are going to change the state store to use Azure Cosmos DB.

##### 1. Add Dapr Client SDK to the Backend API
Similar to what we have done in the Frontend Web App, we need to use Dapr Client SDK to manage the state store, to do so, open the file named `TasksTracker.TasksManager.Backend.Api.csproj` and Install the NuGet package `Dapr.AspNetCore` below:

```json
<ItemGroup>
    <PackageReference Include="Dapr.AspNetCore" Version="1.9.0" />
    <!-- Other packages are removed for brevity -->
</ItemGroup>
```

##### 2. Create a new concrete implementation to manage tasks persistence
As you recall from the previous module, we were storing the tasks in memory, now we need to store them in Redis and later on Azure Cosmos DB. To do so add a new file named `TasksStoreManager.cs` under the folder named `Services` and this file will implement the interface `ITasksManager`. Copy & Paste the code below:

```csharp
namespace TasksTracker.TasksManager.Backend.Api.Services
{
    public class TasksStoreManager : ITasksManager
    {
        private static string STORE_NAME = "statestore";
        private readonly DaprClient _daprClient;
        private readonly IConfiguration _config;
        private readonly ILogger<TasksStoreManager> _logger;

        public TasksStoreManager(DaprClient daprClient, IConfiguration config, ILogger<TasksStoreManager> logger)
        {
            _daprClient = daprClient;
            _config = config;
            _logger = logger;
        }
        public async Task<Guid> CreateNewTask(string taskName, string createdBy, string assignedTo, DateTime dueDate)
        {
            var taskModel = new TaskModel()
            {
                TaskId = Guid.NewGuid(),
                TaskName = taskName,
                TaskCreatedBy = createdBy,
                TaskCreatedOn = DateTime.UtcNow,
                TaskDueDate = dueDate,
                TaskAssignedTo = assignedTo,
            };

            _logger.LogInformation("Save a new task with name: '{0}' to state store", taskModel.TaskName);
            await _daprClient.SaveStateAsync<TaskModel>(STORE_NAME, taskModel.TaskId.ToString(), taskModel);
            return taskModel.TaskId;
        }

        public async Task<bool> DeleteTask(Guid taskId)
        {
            _logger.LogInformation("Delete task with Id: '{0}'", taskId);
            await _daprClient.DeleteStateAsync(STORE_NAME, taskId.ToString());
            return true;
        }

        public async Task<TaskModel?> GetTaskById(Guid taskId)
        {
            _logger.LogInformation("Getting task with Id: '{0}'", taskId);
            var taskModel = await _daprClient.GetStateAsync<TaskModel>(STORE_NAME, taskId.ToString());
            return taskModel;
        }

        public async Task<List<TaskModel>> GetTasksByCreator(string createdBy)
        {
            var query = "{" +
                    "\"filter\": {" +
                        "\"EQ\": { \"taskCreatedBy\": \"" + createdBy + "\" }" +
                    "}}";

            var queryResponse = await _daprClient.QueryStateAsync<TaskModel>(STORE_NAME, query);

            var tasksList = queryResponse.Results.Select(q => q.Data).OrderByDescending(o=>o.TaskCreatedOn);
            return tasksList.ToList();
        }

        public async Task<bool> MarkTaskCompleted(Guid taskId)
        {
            _logger.LogInformation("Mark task with Id: '{0}' as completed", taskId);
            var taskModel = await _daprClient.GetStateAsync<TaskModel>(STORE_NAME, taskId.ToString());
            if (taskModel != null)
            {
                taskModel.IsCompleted = true;
                await _daprClient.SaveStateAsync<TaskModel>(STORE_NAME, taskModel.TaskId.ToString(), taskModel);
                return true;
            }
            return false;
        }

        public async Task<bool> UpdateTask(Guid taskId, string taskName, string assignedTo, DateTime dueDate)
        {
            _logger.LogInformation("Update task with Id: '{0}'", taskId);
            var taskModel = await _daprClient.GetStateAsync<TaskModel>(STORE_NAME, taskId.ToString());
            var currentAssignee = taskModel.TaskAssignedTo;
            if (taskModel != null)
            {
                taskModel.TaskName = taskName;
                taskModel.TaskAssignedTo = assignedTo;
                taskModel.TaskDueDate = dueDate;
                await _daprClient.SaveStateAsync<TaskModel>(STORE_NAME, taskModel.TaskId.ToString(), taskModel);
                return true;
            }
            return false;
        }
    }
}
```
Looking at the code above, we have injected the `DaprClient` into the new service and DaprClient has a set of [methods to support CRUD operations](https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-get-save-state/). Notice how we are using the state store named `statestore`  which should match the name in the component file.

{: .note }
The query API will not work against the local Redis store as you need to install [RediSearch](https://redis.io/docs/stack/search/) locally on your machine which is out of the scope for this workshop. It will work locally once we switch to Azure Cosmos DB.

##### 3. Register the TasksStoreManager new service and DaprClient

Now we need to register the new service named `TasksStoreManager` and `RedisClient` when the Backend API app starts up, to do so open the file `Program.cs` and register both as the below. Do not forget to comment out the registration of the `FakeTasksManager` service as we don’t want to store tasks in memory anymore.

```csharp
var builder = WebApplication.CreateBuilder(args);
// Add services to the container.
builder.Services.AddDaprClient();
builder.Services.AddSingleton<ITasksManager, TasksStoreManager>();
//builder.Services.AddSingleton<ITasksManager, FakeTasksManager>();
//Code removed for brevity
```

Now you are ready to run both applications and debug them, you can store new tasks, update them, delete existing tasks and mark them as completed, the data should be stored on your local Redis instance.

### Use Azure Cosmos DB with Dapr State Store Management API

**1. Provision Cosmos DB Resources**
Now we will create an Azure Cosmos DB account, Database, and a new container that will store our tasks, you can use the below PowerShell script to create the Cosmos DB resources on the same resource group we used in the previous module. You need to change the variable name of the `$COSMOS_DB_ACCOUNT` to a unique name as this name is already used under my account.

```powershell
$COSMOS_DB_ACCOUNT="taskstracker-state-store" `
$COSMOS_DB_DBNAME="tasksmanagerdb" `
$COSMOS_DB_CONTAINER="taskscollection" 

## Check if Cosmos account name already exists
az cosmosdb check-name-exists `
--name $COSMOS_DB_ACCOUNT

## returns false

## Create a Cosmos account for SQL API
az cosmosdb create `
--name $COSMOS_DB_ACCOUNT `
--resource-group $RESOURCE_GROUP

## Create a SQL API database
az cosmosdb sql database create `
--account-name $COSMOS_DB_ACCOUNT `
--resource-group $RESOURCE_GROUP `
--name $COSMOS_DB_DBNAME

## Create a SQL API container
az cosmosdb sql container create `
--account-name $COSMOS_DB_ACCOUNT `
--resource-group $RESOURCE_GROUP `
--database-name $COSMOS_DB_DBNAME `
--name $COSMOS_DB_CONTAINER `
--partition-key-path "/id" `
--throughput 400
```

{: .note }
The `primaryMasterKey` connection string is only needed for our local testing on the development machine, we'll be using a different approach (**Managed Identities**) when deploying Dapr component to Azure Container Apps Environment.

Once the scripts execution is completed, we need to get the `primaryMasterKey` of the CosmosDB account, to do this you can use the below PowerShell script. Copy the value of `primaryMasterKey` as we will use it in the next step.

```powershell
## List Azure CosmosDB keys
az cosmosdb keys list `
--name $COSMOS_DB_ACCOUNT `
--resource-group $RESOURCE_GROUP
```

**2. Create a Component file for State Store Management**
Dapr uses a modular design where functionality is delivered as a component. Each component has an interface definition. All of the components are pluggable so that you can swap out one component with the same interface for another

Components are configured at design-time with a YAML file which is stored in either a components/local folder within your solution, or globally in the `.dapr` folder created when invoking `dapr init`. These YAML files adhere to the generic [Dapr component schema](https://docs.dapr.io/operations/components/component-schema/), but each is specific to the component specification.

It is important to understand that the component spec values, particularly the spec `metadata`, can change between components of the same component type As a result, it is strongly recommended to review a component’s specs, paying particular attention to the sample payloads for requests to set the metadata used to interact with the component.

I'm borrowing the diagram below from Dapr official documentation which shows some examples of the components for each component type, we are now looking at the State Stores components, specifically the [Azure Cosmos DB](https://docs.dapr.io/reference/components-reference/supported-state-stores/setup-azure-cosmosdb/).

![dapr-components](../../assets/images/04-aca-dapr-stateapi/dapr-components.jpg)

To add the component file state store add a new folder named `components` under the directory `TasksTracker.ContainerApps` and add a new yaml file named `dapr-statestore-cosmos.yaml`. Paste the code below:
```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.azure.cosmosdb
  version: v1
  metadata:
  - name: url
    value: https://taskstracker-state-store.documents.azure.com:443/
  - name: masterKey
    value: "<value>"
  - name: database
    value: tasksmanagerdb
  - name: collection
    value: taskscollection
scopes:
- tasksmanager-backend-api
```
Few things to note about this yaml file:
- We've used the name `statestore` which should match the name of statestore we've used in the `TaskStoreManager.cs` file. As well we have set the metadata key/value to allow us to connect to Azure Cosmos DB. You need to replace the `masterKey` value with your Cosmos Account key. Remember this is only needed for local development debugging, we will not be using the masterKey when we deploy to ACA. 
- We've updated the other metadata keys such as `database`, `collection`, etc... to match the values of your Cosmos DB instance. For full metadata specs, you can check this [page](https://docs.dapr.io/reference/components-reference/supported-state-stores/setup-azure-cosmosdb/).
- By default, all dapr-enabled container apps within the same environment will load the full set of deployed components. By adding `scopes` to a component, you tell the Dapr sidecars for each respective container app which components to load at runtime. Using scopes is recommended for production workloads. In our case, we have set the scopes to read `tasksmanager-backend-api` as this will be the application that needs access to Azure Cosmos DB State Store. More about scopes can be found on this [link](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview?tabs=bicep1%2Cyaml#component-scopes).

{: .note }
Dapr component scopes correspond to the Dapr application ID of a container app, not the container app name.

Now you should be ready to launch both applications and start doing CRUD operations from the Frontend Web App including querying the store, all your data will be stored in Cosmos DB Database you just provisioned, it should look like the below:
![cosmos-db-dapr-state-store](../../assets/images/04-aca-dapr-stateapi/cosmos-db-dapr-state-store.jpg)

### Key Prefix Strategies

When you look at the key stored per entry and for example `tasksmanager-backend-api||aa3eb856-8309-4e68-93af-119be0d400e8`, you will notice that the key is prefixed with the Dapr application App Id responsible to store this entry, in our case, it will be `tasksmanager-backend-api`, there might be some scenarios which you need to have another service to access the same data store (not recommended as each service should be responsible about its own data store) then you can change the default strategy, this can be done by adding the below meta tag to the component file, for example, if we need to set the value of the prefix to a constant value such as  `TaskId` we can do the following:
```yaml
spec:
  metadata:
  - name: keyPrefix
  - value: TaskId
```
If we need to totally omit the key prefix so it is accessed across multiple Dapr applications, we can set the value to `none`.

### Configure Managed Identities in Container App

As we highlighted earlier, we'll not use a connection strings to establish the relation between our Container App and Azure Cosmos DB when we deploy to ACA, Cosmos DB Master Key/Connection string was only used when debugging locally. Now we will rely on Managed Identities to allow our container app to access Cosmos DB. With [Manged Identities](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview?tabs=bicep1%2Cyaml) you do't worry about storing the keys securely and rotate them, this approach is safer and more easier to manage.

We will be using a `system-assigned` identity with a role assignments to grant our Backend API container app permissions to access data stored in Cosmos DB, we need to assign it a custom role for the Cosmos DB data plane. In this example ae are going to use a [built-in role](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-setup-rbac#built-in-role-definitions), named `Cosmos DB Built-in Data Contributor`, which grants our application full read-write access to the data; you can optionally create custom, fine-tuned roles following the instructions in the [official docs](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-setup-rbac).

##### 1. Create system-assigned identity for our container app

Run the command below to create `system-assigned` identity for our container app:

```Powershell
az containerapp identity assign `
  --resource-group $RESOURCE_GROUP `
  --name $BACKEND_API_NAME `
  --system-assigned
```

This command will create an Enterprise Application (so a Service Principal) within Azure AD, which is linked to our container app, the output of this command will be as the below, keep a note of the property `principalId` as we are going to use it in the next step.

```json
{
  "principalId": "774aaaab-e3a9-452c-a168-54dcc88a8500",
  "tenantId": "0a02a8b1-xxx-xxxx-xxxx-67ceb9132d81",
  "type": "SystemAssigned"
}
```

##### 2. Assign the container app system-identity to built in Cosmos DB role
Next, we need to associate the container app system-identity with the target Cosmos DB resource. You can read more about Azure built-in roles for Cosmos DB or how to create custom fine-tuned roles [here](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-setup-rbac#built-in-role-definitions). Run the command below to associate the container app `system-assigned` identity with `Cosmos DB Built-in Data Contributor` role.

```powershell
 $COSMOS_DB_ACCOUNT="aca-workshop-state-store" # Cosmos DB account name
 $PRINCIPAL_ID = "774aaaab-e3a9-452c-a168-54dcc88a8500" # Principal Id after creating system identity for container app 
 $ROLE_ID = "00000000-0000-0000-0000-000000000002" #"Cosmos DB Built-in Data Contributor" 

az cosmosdb sql role assignment create `
--account-name  $COSMOS_DB_ACCOUNT `
--resource-group $RESOURCE_GROUP `
--scope "/" `
--principal-id $PRINCIPAL_ID `
--role-definition-id $ROLE_ID
```

### Deploy the Backend API and Frontend Web App Projects to ACA

Now we are ready to deploy all local changes from this module and previous module to ACA, but we need to do one addition before deploying, we have to create a [component schema file](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview?tabs=bicep1%2Cyaml#component-schema) for Azure Cosmos DB which is meeting the specs defined by Azure Container Apps. Reason for this variance that ACA Dapr schema is slightly simplified to support Dapr components and remove unnecessary fields, including `apiVersion`, `kind`, and redundant metadata and spec properties.

##### 1. Create a ACA-Dapr Component file for State Store Management
Personally, I prefer to separate the component files that will be used when deploying to Azure Container Apps from the ones which we will use when running our application locally (Dapr self-hosted). So go ahead and create a new folder named `aca-components` under the directory `TasksTracker.ContainerApps`, then add a new file named `containerapps-statestore-cosmos.yaml` and paste the code below:

```yaml
componentType: state.azure.cosmosdb
version: v1
metadata:
- name: url
  value: https://taskstracker-state-store.documents.azure.com:443/
- name: database
  value: tasksmanagerdb
- name: collection
  value: taskscollection
scopes:
- tasksmanager-backend-api
```

Few things to notice here:
- We didn't specify the Cosmos DB component name `state-store` when we created this component file, we are going to specify it once we add this dapr component to Azure Container Apps Environment via CLI.
- We are not referencing any Cosmos DB Keys/Connection strings as the authentication between Dapr and Cosmos DB will be configured using Managed Identities. 
- We are setting the `scopes` array value to `tasksmanager-backend-api` to ensure Cosmos DB component is loaded at runtime by only the appropriate container apps. In our case it will be needed only for the container apps with Dapr application IDs `tasksmanager-backend-api`. In future module we are going to include another container app which needs to access Cosmos DB.

##### 2. Build Frontend Web App and Backend API App images and push them to ACR
As we have done previously we need to build and deploy both app images to ACR so they are ready to be deployed to Azure Container Apps, to do so, continue using the same PowerShell console and paste the code below (Make sure you are on directory `TasksTracker.ContainerApps`):

```powershell
az acr build --registry $ACR_NAME --image "tasksmanager/$BACKEND_API_NAME" --file 'TasksTracker.TasksManager.Backend.Api/Dockerfile' .

az acr build --registry $ACR_NAME --image "tasksmanager/$FRONTEND_WEBAPP_NAME" --file 'TasksTracker.WebPortal.Frontend.Ui/Dockerfile' .
```

##### 3. Add Cosmos DB Dapr State Store to Azure Container Apps Environment
We need to run the command below to add the yaml file `.\aca-components\containerapps-statestore-cosmos.yaml` to Azure Container Apps Environment, to do so run the below PowerShell command:

```powershell
az containerapp env dapr-component set `
 --name $ENVIRONMENT --resource-group $RESOURCE_GROUP `
 --dapr-component-name statestore `
 --yaml '.\aca-components\containerapps-statestore-cosmos.yaml'
```

##### 4. Enable Dapr for the Frontend Web App and Backend API Container Apps
Until this moment Dapr was not enabled on the Container Apps we have provisioned,  you can check this by running the below command and notice the result:

```powershell
az containerapp show -n $BACKEND_API_NAME -g $RESOURCE_GROUP --query properties.configuration.dapr
```

What we need to do now is to enable Dapr for bot Container Apps, to do so, run the below command in the PowerShell console:

```powershell
az containerapp dapr enable --name $BACKEND_API_NAME `
                            --resource-group $RESOURCE_GROUP `
                            --dapr-app-id  $BACKEND_API_NAME `
                            --dapr-app-port  80

az containerapp dapr enable --name $FRONTEND_WEBAPP_NAME `
                            --resource-group $RESOURCE_GROUP `
                            --dapr-app-id  $FRONTEND_WEBAPP_NAME `
                            --dapr-app-port  80
```

Few things to notice here:
- We've enabled Dapr on both container apps and specified a unique Dapr identifier for the Back End API and Front End Web App container apps. This `dapr-app-id` will be used for service discovery, state encapsulation and the pub/sub consumer ID.
- We've set the `dapr-app-port` which is the port our applications are listening on which will be used by Dapr for communicating to our applications.

For a complete list of the supported Dapr sidecar configurations in Container Apps, you can refer to [this link](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview?tabs=bicep1%2Cyaml#dapr-enablement).

##### 5. Deploy new revisions of the Frontend Web App and Backend API to Container Apps

The last thing we need to do here is to update both container apps and deploy the new images from ACR, to do so we need to run the below command. I've used a `revision-suffix` property so it will append to the revision name and gives you better visibility on which revision you are looking at

```powershell
## Update Frontend web app container app and create a new revision 
az containerapp update `
--name $FRONTEND_WEBAPP_NAME  `
--resource-group $RESOURCE_GROUP `
  --revision-suffix v20230218 `
  --cpu 0.25 --memory 0.5Gi `
  --min-replicas 1 `
  --max-replicas 1

## Update Backend API App container app and create a new revision 
az containerapp update `
--name $BACKEND_API_NAME  `
--resource-group $RESOURCE_GROUP `
  --revision-suffix v20230218-1 `
  --cpu 0.25 --memory 0.5Gi `
  --min-replicas 1 `
  --max-replicas 1
```

With this final step, we should be able to access the Frontend Web App, call the backend API app using Dapr sidecar, and store tasks to Azure Cosmos DB.

In the next module, we will introduce the Dapr Pub/Sub Building block which we will publish messages to Azure Service Bus when a task is saved, and a new background service will process those incoming messages and send an email to the task assignee.