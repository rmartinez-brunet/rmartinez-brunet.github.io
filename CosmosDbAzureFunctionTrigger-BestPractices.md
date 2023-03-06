# CosmosDb Azure Function Trigger
## Opinionated Best Practices

Version 1.0 2023/03/06

Azure CosmosDb Azure Trigger is based on the CosmosDb Change Feed and is commonly used to further process the changing items. For example, to replicate the changes to another database/container.

Due to the schema-less of CosmosDb and the pposibility of storing different entity types in the same database/container, the processing of the changes in the CosmosDb trigger usually implies a check by some 'docType' property of the entity. The consequences of these are: 

* Is not possible to bind the trigger input type to an specific type.
* Need to do a lot of Deserialization (to check the type) and Serialization (to further submit the change -- to EventGrid for instance). 

It begs the question what approach would be the best in term of simplicity of code and performance.

We considered three options regarding the processing of the trigger input: 

1. **dynamic** object
1. **Newtonsoft.Json.Linq.JArray** 
1. **System.Text.Json.JsonDocument**

I must admit that I try to use System.Text.Json whenever possible considering that at some future point Json.Net could be deprecated.

## Implementations

The implementations for an hypothetic **Person** CosmosDb change feed don't differ much:

For the **dynamic** version:

``` CSharp
    [FunctionName("PeopleChangeFeedProcessor")]
    public static void Run([CosmosDBTrigger(
        databaseName: "%DatabaseName%",
        containerName: "%ContainerName%",
        Connection = "CosmosDbConnectionString",
        LeaseContainerName = "leases",
        CreateLeaseContainerIfNotExists = true,
        LeaseContainerPrefix = "People_")]string input,
        ILogger log)
    {
        if (!string.IsNullOrWhiteSpace(input))
        {
            var document = JsonConvert.DeserializeObject<IList<dynamic>>(input);

            foreach (var element in document)
            {
                if (element.docType == "Person")
                {
                    string json = JsonConvert.SerializeObject(element);

                    // Further process the json

                    log.LogInformation(json);

                }
            }

        }
    }
```

For the Newtonsoft **JArray** version:

``` CSharp
 [FunctionName("PeopleChangeFeedProcessor")]
    public static void Run([CosmosDBTrigger(
        databaseName: "%DatabaseName%",
        containerName: "%ContainerName%",
        Connection = "CosmosDbConnectionString",
        LeaseContainerName = "leases",
        CreateLeaseContainerIfNotExists = true,
        LeaseContainerPrefix = "People_")]string input,
    ILogger log)
    {
        if (!string.IsNullOrWhiteSpace(input))
        {
            var document = JArray.Parse(input);

            foreach (var element in document)
            {
                if ((string)element["docType"] == "Person")
                {
                    string json = element.ToString();

                    // Further process the json

                    log.LogInformation(json);

                }
            }

        }
    }
```

For the **JsonDocument** version:

``` CSharp
[FunctionName("PeopleChangeFeedProcessor")]
public static void Run([CosmosDBTrigger(
    databaseName: "%DatabaseName%",
    containerName: "%ContainerName%",
    Connection = "CosmosDbConnectionString",
    LeaseContainerName = "leases",
    CreateLeaseContainerIfNotExists = true,
    LeaseContainerPrefix = "People_")]string input,
    ILogger log)
{
    if (!string.IsNullOrWhiteSpace(input))
    {
        var document = JsonDocument.Parse(input);

        foreach (var element in document.RootElement.EnumerateArray())
        {
            if (element.TryGetProperty("docType", out JsonElement item) 
                && item.ValueKind == JsonValueKind.String 
                && item.GetString() == "Person")
            {
                string json = element.ToString();

                // Further process the json

                log.LogInformation(json);
            }
        }
    }
}
```

The code is very similar in any of the three options. 

## Benchmarks

In order to compare the performance of the different implementations I stripped the functions of the Azure Function attributes and benchmarked the functions with BenchMarkDotNet using an input of 100 generated **Person** records. The results are below.

|Case|Mean|Min|Max|Range|AllocatedBytes|Operations|
|---|---|---|---|---|---|---|
|RunJsonDocument|42.05 μs|41.37 μs|43.26 μs|4%|119,176|245,760|
RunJArray|301.19 μs|293.58 μs|315.98 μs|7%|516,560|30,720|
|RunDynamic|309.77 μs|302.27 μs|336.26 μs|11%|459,736|30,720|

## Conclusions.

Using JsonDocument yields a 7.5 fold increase in processing speed and 25% reduction in allocated memory.

