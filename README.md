﻿# Configuration-driven development
 "Configuration-driven development," refers to an approach in software development where the behavior and functionality of an application are primarily controlled through configuration files, rather than writing code. It focuses on separating application logic from configuration parameters, allowing developers to easily modify the behavior of the software without the need for extensive code changes.  [Xenhey.BPM.Core.Net6](https://www.nuget.org/packages/Xenhey.BPM.Core.Net6) offers a orchrestration pipeline using configuration to drive the execution of business logic, providing a tailored solution for each Line Of Business(LOB). The following are some benefits. 

 1. Flexibility: By using configuration files, developers can easily tweak and adjust the application's behavior without modifying the underlying code. It allows for quick customization and adaptation to different scenarios or client requirements.

2. Maintenance: Separating configuration from code simplifies maintenance processes. Updates and modifications can be made by adjusting the configuration files, reducing the risk of introducing bugs or breaking existing functionality. It also facilitates version control and collaboration, as changes to configuration can be tracked separately from code changes.

3. Scalability: Configuration-driven development promotes scalability by enabling the application to handle different environments, configurations, or user preferences. It allows for seamless deployment across multiple instances or environments with minimal code changes.

4. Testing and Validation: Configuration-driven development enhances testing and validation processes. Since configuration changes are isolated from the codebase, it becomes easier to verify the impact of different configurations on the application's behavior. It also facilitates A/B testing or experimentation by quickly switching between different configurations.

5. Domain-Specific Customization: Configuration-driven development enables domain-specific customization by providing options and settings tailored to specific use cases. This empowers non-technical users or administrators to configure the application according to their specific requirements without the need for coding expertise
# Example
## Use case. ETL to Azure SQL DB.
![image](https://github.com/oigbokwe73/documentation/assets/15838780/d8e06136-f4a8-4bdf-8942-b21c62fed6e4)
## Code snippet
Http Trigger 
```c#
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using System.Collections.Specialized;
using System.Linq;
using System.IO;
using Xenhey.BPM.Core.Net6.Implementation;
using Xenhey.BPM.Core.Net6;

namespace AzureServiceBusToSQL
{
    public class Search
    {
        private HttpRequest _req;
        private NameValueCollection nvc = new NameValueCollection();
        [FunctionName("search")]
        public async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = null)]
            HttpRequest req, ILogger log)
        {
            _req = req;

            log.LogInformation("C# HTTP trigger function processed a request.");
            string requestBody = await new StreamReader(_req.Body).ReadToEndAsync();
            _req.Headers.ToList().ForEach(item => { nvc.Add(item.Key, item.Value.FirstOrDefault()); });
            var results = orchrestatorService.Run(requestBody);
            return resultSet(results);

        }

        private ActionResult resultSet(string reponsePayload)
        {
            var returnContent = new ContentResult();
            var mediaSelectedtype = nvc.Get("Content-Type");
            returnContent.Content = reponsePayload;
            returnContent.ContentType = mediaSelectedtype;
            return returnContent;
        }
        private IOrchestrationService orchrestatorService
        {
            get
            {
                return new ManagedOrchestratorService(nvc);
            }
        }

    }
}
```

# Data Flow Process

```
{
  "Id": "UploadFileProcess",
  "LineOfBusinessProcessData": [
    {
      "Key": "object",
      "Type": "Xenhey.BPM.Core.Net6.Processes.ProcessData"
    }
  ],
  "Type": "",
  "DataFlowProcess": [
    {
      "Key": "CreateCSVBatchFilesForProcessing",
      "Type": "Xenhey.BPM.Core.Net6.Processes.CSVProcess",
      "Async": "false",
      "IsEnable": "true",
      "DataFlowProcessParameters": [
        {
          "Key": "StorageAccount",
          "Value": "AzureWebJobsStorage"
        },
        {
          "Key": "WriteCsvToStorageAsBatch",
          "Value": "Yes"
        },
        {
          "Key": "BatchSize",
          "Value": "201"
        },
        {
          "Key": "FolderName",
          "Value": "CSVFiles"
        },
        {
          "Key": "TableName",
          "Value": "csvbatchfiles"
        },
        {
          "Key": "Container",
          "Value": "processed"
        },
        {
          "Key": "FileExtension",
          "Value": ".csv"
        },
         {
          "Key": "ContentType",
          "Value": "csv/text"
        }	
      ]
    }
  ]
}
```
| Key |Value | Description|
|--|--|--|
| StorageAccount | AzureWebJobsStorage  |  Storage account name used in app setting
| WriteCsvToStorageAsBatch | yes  |  This entry is required. Calling out the feature to execute.
| BatchSize | 201  |  Required to parse your file into batches. The  "1" is for the headers
| Container | processed  |  This is the container name in the Blob.
| FolderName | CSVFiles  |  This name is created in the container within Blob Storage.
| FileExtension | .csv  |  This name created for the file extension when writing to disk
| ContentType | csv/text  |  This is the name of the content type used to render the file



```
using System.Collections.Specialized;
using System.IO;
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using Xenhey.BPM.Core.Net6;
using Xenhey.BPM.Core.Net6.Implementation;

namespace AzureServiceBusToSQL
{
    public static class BlobTrigger
    {
        [FunctionName("BlobTrigger")]
        public static void Run([BlobTrigger("processed/{name}", Connection = "AzureWebJobsStorage")] Stream myBlob, string name, ILogger log)
        {
            string ApiKeyName = "x-api-key";
            log.LogInformation("C# blob trigger function processed a request.");
            NameValueCollection nvc = new NameValueCollection();
            nvc.Add(ApiKeyName, "[API_KEY]"); // [API_KEY] Enter your api key
            IOrchestrationService orchrestatorService = new ManagedOrchestratorService(nvc);
            var processFiles = orchrestatorService.Run(myBlob);
        }
    }
}
```
