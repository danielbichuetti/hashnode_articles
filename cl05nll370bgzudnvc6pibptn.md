# Serverless Web API with Azure Function v4 and CosmosDB

Today we will together build our first serverless Web API using Azure Functions v4 and CosmosDB as a serverless database.

### 1. Setup

  First, make sure .NET 6 SDK is installed on your machine. You can get guidance [here](Link).
  Then you will need to install [Azure Functions Core Tools 4.x](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local#v2). These tools include a version of the runtime that powers Azure Functions. You will be able to create and deploy functions.

### 2. Create a local project

  Each project may contain one or more individual functions, each supporting different triggers if desired.  
  Let's create one in-process Azure Function with the runtime I love (.NET).

```csharp
func init FirstFunction --dotnet
```

  Then you get into the folder and can add functions to the project. We will add a function named WebApiExample and use the HTTP Trigger template.


```csharp
func new --name WebApiExample --template "HTTP trigger" --authlevel "anonymous"
```

  I have chosen the authentication level as anonymous since I'm not using it at the moment. In the future, I will add an article about JWT Bearer authentication on Azure Functions, keep an eye out!

  The default template will give you this starting point:

```csharp
namespace FirstFunction
{
    public static class WebApiExample
    {
        [FunctionName("WebApiExample")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
            ILogger log)
        {
            log.LogInformation("C# HTTP trigger function processed a request.");

            string name = req.Query["name"];

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);
            name = name ?? data?.name;

            string responseMessage = string.IsNullOrEmpty(name)
                ? "This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response."
                : $"Hello, {name}. This HTTP triggered function executed successfully.";

            return new OkObjectResult(responseMessage);
        }
    }
}
```

### 3. Run your function

At this point, you can already make a simple test and run locally the function. Run this command from your project's folder:


```csharp
func start
``` 

Now we can see some output that contains information about the functions that are running, triggers, methods, and the path. You access it passing the query string that is by default defined (name). Let's give it a try.


```
http://localhost:7071/api/WebApiExample?name=Daniel
```

The response should be:

> Hello, Daniel

### 4. Personalize the route

For those who are uncomfortable with the URL pattern **/api/{function name}*, it's pretty simple to change it. Just change the *Route parameter* on the trigger attribute. The result will be something like this:

```
[HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = "test")]
```

And we would access Function using this path


>http://localhost:7071/api/test?name=Daniel


You may be questioning yourself about the *api* prefix. It can be removed, but not here. You need to open the *host.json* file and alter it to something like this:

```json
{
  "version": "2.0",
  "extensions": {
    "http": {
      "routePrefix": ""
    }
  },
    "logging": {
        "applicationInsights": {
            "samplingSettings": {
                "isEnabled": true,
                "excludedTypes": "Request"
            }
        }
    }
}
```
As you may be thinking, we have set the route prefix to an empty string. We could have set it to any string allowed on an URL.
By now, we have created a simple Azure Function WebApi and learned how to set up the route, prefix, and suffix. 

### 5. Refactoring for Dependency Injection

To make D.I. work we need to make some adjustments. We will add 2 packages to this project.

```csharp
dotnet add package Microsoft.Extensions.DependencyInjection
dotnet add package Microsoft.Azure.Functions.Extensions
```
Then we will add one file called *Startup.cs* (the name is just for better understanding). We need to make some changes to it, so Functions runtime knows where to look for the Startup class and the DI configuration. This is one example of the result.

```csharp
[assembly:FunctionsStartup(typeof(FirstFunction.Startup))]

namespace FirstFunction
{
    public class Startup : FunctionsStartup
    {
        public override void Configure(IFunctionsHostBuilder builder)
        {
            // Setup your service collection here, AddLogging() is just example
            builder.Services.AddLogging();
        }
    }
}
```
You need to remember these 3 parts. You add the attribute, you *inherit from FunctionsStartup* and *override the Configure method*. Then you set up your service collection with the desired lifetime.
We also need to refactor our main function class. It's using a static class and we have to change this. Let's modify it.

```csharp
namespace FirstFunction
{
    public class WebApiExample
    {
        public WebApiExample()
        {
        }  
      
        [FunctionName("WebApiExample")]
        public async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
            ILogger log)
        {
            log.LogInformation("C# HTTP trigger function processed a request.");

            string name = req.Query["name"];

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);
            name = name ?? data?.name;

            string responseMessage = string.IsNullOrEmpty(name)
                ? "This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response."
                : $"Hello, {name}. This HTTP triggered function executed successfully.";

            return new OkObjectResult(responseMessage);
        }
    }
}
```

### 6. Injecting CosmosClient

So we have prepared the field for using Cosmos. Now we will add Cosmos SDK and implement its D.I. We start by adding the package to the project since we will be using Cosmos SQL API.

```csharp
dotnet add package Microsoft.Azure.Cosmos
```

In the Startup class, we make some changes to the Configure method.

```csharp
public override void Configure(IFunctionsHostBuilder builder)
{
    builder.Services.AddSingleton<CosmosClient>(sp => new CosmosClient(
        Environment.GetEnvironmentVariable("COSMOSDB_CONNECTIONSTRING")));
    builder.Services.AddLogging();
}
```
Cosmos team instructs to *inject CosmosClient as a singleton* to avoid issues. We will follow this advice always. The connection string can be found on the Azure portal easily. Maybe you like to test locally and want to give a try to Azure Cosmos DB Emulator. It allows us to run a Cosmos emulator locally so we can make many tests, including throttling requests. You can find a nice guide [here](https://docs.microsoft.com/en-us/azure/cosmos-db/local-emulator?tabs=ssl-netstd21Link).

We start to make final arrangements for our function class. Let's use the constructor to inject CosmosClient and get the container.

```csharp
private readonly CosmosClient _cosmosClient;
private readonly Container _container;

public WebApiExample(CosmosClient cosmosClient)
{
    _cosmosClient= cosmosClient;

    _container = _cosmosClient.GetContainer("test-db", "test-container");
}
```

### 7. Adding an item to Cosmos

One very important thing is to create the database and container first on Cosmos. We can do it using the portal or programmatically. Avoid doing it programmatically in production on your function. It causes very poor performance. Scenarios you may want to create programmatically (but not inside the function) may be on deployment or seed code. More information can be found [here](https://docs.microsoft.com/en-us/azure/cosmos-db/sql/how-to-create-container). For this example code to work you need to create a Container with the PartitonKey set to /id.
One important note is that CosmosDB demands you to set up a unique "id" property. The partition key is obligatory when creating the container. If you use it when making queries you get better performance and costs.

On the function itself, we will change the accepted method to POST exclusively, since we will be using it to add an item to the database. 

```csharp
[FunctionName("WebApiExample")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "test")] HttpRequest req,
    ILogger log)
{
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    dynamic data = JsonConvert.DeserializeObject(requestBody);
    string name = data?.name ?? "Daniel";

    string uId = Guid.NewGuid().ToString();

    dynamic newItem = new
    {
        id = uId,
        Name = name

    };
    ItemResponse<dynamic> response = await _container.CreateItemAsync<dynamic>(newItem, new PartitionKey(newItem.id));

    return new OkObjectResult($"Request cost: {response.RequestCharge}");
}
```

On the above code, we post a JSON body that is similar to:

```json
{
    "Name":"Paulo"    
}
```

Our code then creates a new item with a unique id and adds it to the Cosmos container. If you have followed exactly what I said, your PartitionKey is the same as the unique id. This is not a need, just an example.
Some of you may be wondering what is this response.

>{"RequestCost":5.9}

CosmosDB SDK returns to us how much *RU (Request Units)* it has to spend for that request you just made. In a production scenario, we would monitor these costs to set them as low as possible to improve performance and costs.

### 8. Conclusion

In this article, we have been instructed on how to create a serverless Web API that has all basics to build complex ones. We learn how to process JSON requests and return responses, how to personalize routes, and to make some database operations using the blazing fast CosmosDB. The next article will be about JWT Bearer authentication in serverless architecture, see you!



