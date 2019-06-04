# Streaming - Real-Time Advanced Analytics

Goal of this workshop will be to build an Azure infrastruture full PaaS oriented and create a kind og “hello world” streaming application in Azure.

Something like: simple producer -> to Event Hubs -> to Databricks -> Datalake -> SQL Data Warehouse.


## Getting Started

These instructions will get you ....

### Prerequisites

What things you need to install the software and how to install them

```
Give examples
```

### Setup
#### Step 1 : Ressource Group
Is to create a Ressource Group where you will put every Azure ressource we will used for this workshop.
Be carefull to select the right Azure Subscription and prefered Region (East US)
Please respect any naming convention SCCoE have defined like : "rg-projectname"

#### Step 2 : Event Hub
Deploy an Azure Event Hub into your existing Ressource Group
Provide a relevant name. Be careful the convention name for this ressource should respect this rule : 
  The namespace can contain only letters, numbers, and hyphens.
  The namespace must start with a letter, and it must end with a letter or number.
 example : malefe-transac-eh
 Pricing Tier : select Strandard
 Do not enable Kafka or namespace zone redundant
 Location : East US
 Throughput Units = 1
 Do not enable auto-inflate.
 
 
 
 When the Event Hub is deploy, go to the ressource and add a new Event Hub Entity : "transactions" which will be related to our event producer. Set the partition count to 10 and retention period to 1 day. Do not enable Event Hub Capture. 
 
----- TIPS : ----- 
The partition count on an event hub cannot be modified after setup. With that in mind, it is important to think about how many partitions you need before getting started.
Important point to remember is, TU is configured at namespace level. And, ONE Event Hubs namespace can have multiple eventhubs. Each eventhub can have different no. of partitions.
If you select 5 or more TUs on the Namespace and have only 1 EventHub with 4 partitions - you will get a max. of 4 MBPS or 4K msgs/sec.

Ideally, you will need more partitions than TUs. First, model your partition count as mentioned above. Start with 1 TU while you are developing your Solution. Once done, when you are doing load testing or going live, increase TUs to tune to your load. Remember, you could have multiple eventhubs in a Namespace. So, having 20 TUs at Namespace level and 10 eventhubs with 4 partitions each - can deliver 20 MBPS across the Namespace
You can't have more concurrent readers than you have partitions. If you want to have 5 concurrent readers, you need 5 partitions.
----- END OF TIPS -----
 
Then Select it then select Shared access policies under Settings in the left-hand menu
Select Add to Add SAS Policies. 
1 - Sender
```
Policy name: Enter "Sender".
Manage: Unchecked
Send: Checked
Listen: Unchecked
```
2 - Listener
```
Policy name: Enter "Listener".
Manage: Unchecked
Send: Unchecked
Listen: Checked
```
Then Select Sender access policy and copy Connection string-primary key value and save this value for the Sender policy in Notepad.

 #### Step 3 : Deploy CosmosDB
Deploy an Azure CosmosDB into your existing Ressource Group
Provide a relevant db account name : 'malefe-cosmosdb-analytics'
Select the API : Core (SQL)
Location : East US
Click on create.

Once the service is up, using Data Explorer, create 
a new database : 'malefedb'
click on Provision database throughput and set the value to 15000
add a new container : 'transactions'
add a new partition key : '/ipCountryCode'

Then, you will configure Cosmos DB's time-to-live (TTL) settings to On with no default. This will allow the data generator to expire, or delete, the ingested messages after any desired period of time by setting the TTL value (object property of ttl) on individual messages as they are sent.

Next you will pass in the Azure Cosmos DB URI and Key values to the data generator so it can connect to and send events to your collection.

Navigate to your Azure Cosmos DB account in the Azure portal, then select Settings on the left-hand menu.
Under Settings within the Settings blade, select the On (no default) option for Time to Live. This setting is required to allow documents added to the container to be configured with their own TTL values.

Select Save to apply your settings

On the Azure Cosmos DB Account blade, select Keys.
Copy the endpoint URI and Primary Key for Cosmos DB. Save this value to notepad or similar for use later


 #### Step 4 : Configuring Event Hubs and the transaction generator
 In this step, you will configure the payment transaction data generator and connect it to your Event Hub.
 Download repo code 
 open TransactionGenerator.sln in the TransactionGenerator directory.
 This will open the solution in Visual Studio.
 
Double-click appsettings.json in the Solution Explorer to open it. This file contains the settings used by the console app to connect to your Azure services and to configure application behavior settings. 

The appsettings.json file contains the following:

```
{
    "EVENT_HUB_1_CONNECTION_STRING": "",
    "EVENT_HUB_2_CONNECTION_STRING": "",
    "EVENT_HUB_3_CONNECTION_STRING": "",

    "COSMOS_DB_ENDPOINT": "",
    "COSMOS_DB_AUTH_KEY": "",

    "SECONDS_TO_LEAD": "0",
    "SECONDS_TO_RUN": "600",
    "ONLY_WRITE_TO_COSMOS_DB": "false"
}
```

SECONDS_TO_LEAD is the amount of time to wait before sending payment transaction data. Default value is 0.

SECONDS_TO_RUN is the maximum amount of time to allow the generator to run before stopping transmission of data. The default value is 600. Data will also stop transmitting after the included Untagged_Transactions.csv file's data has been sent.

If the ONLY_WRITE_TO_COSMOS_DB property is set to true, no records will be sent to the Event Hubs instances. Default value is false.

Copy your Event Hub connection string value you copied from the Sender policy and saved during the steps you completed in the before the hands-on lab setup guide. Paste this value into the double-quotes located next to EVENT_HUB_1_CONNECTION_STRING.
BE-CARREFULL here : we have to add ";EntityPath=transactions" after ;SharedAccessKey=[REDACTED];EntityPath=transactions

Example :
```
"EVENT_HUB_1_CONNECTION_STRING": "Endpoint=sb://malefe-transac-eh.servicebus.windows.net/;SharedAccessKeyName=Sender;SharedAccessKey=f7MflrK/...=",
  "EVENT_HUB_2_CONNECTION_STRING": "",
  "EVENT_HUB_3_CONNECTION_STRING": "",
```

Save the file

Open ```Program.cs``` in the Visual Studio Solution Explorer
In Visual Studio, select View, then Task List from the dropdown menu
This will display the TODO items within the code comments as a list of tasks you can double-click to jump to its location.
Go to TODO 1 located in Program.cs by double-clicking the item in the Task List. Paste the following code under TODO 1, which uses the Event Hub client to send the event data, setting the partition key to IpCountryCode:
```
await eventHubClient.SendAsync(eventData: eventData,
    partitionKey: transaction.IpCountryCode).ConfigureAwait(false);
```
```
Note: The /ipCountryCode partition was selected because the data will most likely include this value, and it allows us to partition by location from which the transaction originated. This field also contains a wide range of values, which is preferable for partitions.
```

Paste the code below under TODO 2 to increment the count of the number of Event Hub requests that succeeded:
```
_eventHubRequestsSucceededInBatch++;
```

Paste the code below under TODO 3 to instantiate a new Event Hub client and add it to the eventHubClients collection:
```
EventHubClient.CreateFromConnectionString(
    arguments.EventHubConnectionString
),
```
Save your changes.

 #### Step 5 : Configuring CosmosDB and the transaction generator
Open the appsettings.json file once more. Paste your Cosmos DB endpoint value next to COSMOS_DB_ENDPOINT, and the Cosmos DB authorization key next to COSMOS_DB_AUTH_KEY.

Save your changes.

Open Program.cs and paste the code below under TODO 4 to send the generated transaction data to Cosmos DB and store the returned ResourceResponse object into a new variable for statistics about RU/s used:
```
var response = await _cosmosDbClient.CreateDocumentAsync(collectionUri, transaction)
    .ConfigureAwait(false);
```

Paste the code below under TODO 5 to append the number of RU/s consumed to the _cosmosRUsPerBatch variable:
```
_cosmosRUsPerBatch += response.RequestCharge;
```

Paste the code below under TODO 6 to set the Cosmos DB connection policy:
```
var connectionPolicy = new ConnectionPolicy
{
    ConnectionMode = ConnectionMode.Direct,
    ConnectionProtocol = Protocol.Tcp
};
```


line 29 : change private const string DatabaseName to paste the exact dbname you have configured.

Save your changes.

### Run

Run the console app by clicking Debug, then Start Debugging in the top menu in Visual Studio, or press F-5 on your keyboard.

The PaymentGenerator console window will open, and you should see it start to send data after a few seconds. You may close the window or press Ctrl+C or Ctrl+Break at any time to stop sending data to Event Hubs and Cosmos DB.

The top of the output displays information about the Cosmos DB container you created (transactions), the requested RU/s as well as estimated hourly and monthly cost. After every 1,000 records are requested to be sent, you will see output statistics so you can compare Event Hubs to Cosmos DB. Be on the lookout for the following:

Compare Event Hub to Cosmos DB statistics. They should have similar processing times and successful calls.
Inserted line shows successful inserts in this batch and throughput for writes/second with RU/s usage and estimated monthly ingestion rate added to Cosmos DB statistics.
Processing time: Shows whether the processing time for the past 1,000 requested inserts is faster or slower than the other service.
Total elapsed time: Running total of time taken to process all documents.
If this value continues to grow higher for Cosmos DB vs. Event Hubs, that is a good indicator that the Cosmos DB requests are being throttled. Consider increasing the RU/s for the container.
Succeeded shows number of accumulative successful inserts to the service.
Pending are items in the bulkhead queue. This amount will continue to grow if the service is unable to keep up with demand.
Accumulative failed requests that encountered an exception.
The obvious and recommended method for sending a lot of data is to do so in batches. This method can multiply the amount of data sent with each request by hundreds or thousands. However, the point of our exercise is not to maximize throughput and send as much data as possible, but to compare the relative performance between Event Hubs and Cosmos DB.

As an experiment, you can scale down the number of requested RU/s for your Cosmos DB container to 700. After doing so, you should see increasingly slower transfer rates to Cosmos DB due to throttling. You will also see the pending queue growing at a higher rate. The reason for this is because when the number of writes (remember, writes typically use 5 RU/s vs. just 1 RU/s for reads on 1 KB-sized documents) exceeds the allotted amount of RU/s, Cosmos DB sends a 429 response with a retry_after header value to tell the consumer that it is resource-constrained. The SDK automatically handles this by waiting for the specified amount of time, then retrying. After you are done experimenting, set the RU/s back to 15,000.

### Choosing between Cosmos DB and Event Hubs for ingestion

Depending of the requirements around ingesting data, including data retention of the hot data and geographic locations to which the data is replicated for high availability and global distribution of the data for processing. There are many similarities between Event Hubs and Cosmos DB that allow both to work well for data ingestion. However, these services have some significant differences in their overall feature set that you need to evaluate to choose the best option for this situation.

Cosmos DB allows for more flexible, and longer, TTL (message retention) than Event Hubs, which is capped at 7 days, or 4 weeks when you contact Microsoft to request the extra capacity. Another option for Event Hubs is to use a dedicated Event Hubs or Event Hub Capture to simultaneously save ingested data to Blob Storage or Azure Data Lake Store for longer retention and cold storage. However, this will require additional development, including automatic clearing of the data after a period of time.

If you want to be able to easily query this data during the 60-day message retention period, from a database. This could also be accomplished through Azure Data Warehouse using Polybase to query the files, we will configure all of taht. 
Using CosmosDB, this another service they may otherwise not need, as well as additional development, administration, and cost.

Finally, the requirement to synchronize/write the ingested data to multiple regions, which could grow at any time, makes Cosmos DB a more favorable choice.

## Understanding and preparing the transaction data at scale
In this exercise, you will create connections from your Databricks workspace to ADLS Gen2 and Event Hub. 
Then, using Azure Databricks you will import and explore some of the historical raw transaction data provided to gain a better understanding of the preparation that needs to be done prior to using the data for building and training a machine learning model. 

You will then use the connection to Event Hub from Databricks to read streaming transactions directly. Finally, you will write the incoming streaming transaction data into an Azure Databricks Delta table stored in your data lake.

### deploy Azure Data Store gen2

#### Create a service principal for OAuth access to the ADLS Gen2 filesystem

### deploy Azure Databricks


And repeat

```
until finished
```

End with an example of getting some data out of the system or using it for a little demo

## Running the tests

Explain how to run the automated tests for this system

### Break down into end to end tests

Explain what these tests test and why

```
Give an example
```

### And coding style tests

Explain what these tests test and why

```
Give an example
```

## Deployment

Add additional notes about how to deploy this on a live system

## Built With

* [Dropwizard](http://www.dropwizard.io/1.0.2/docs/) - The web framework used
* [Maven](https://maven.apache.org/) - Dependency Management
* [ROME](https://rometools.github.io/rome/) - Used to generate RSS Feeds

## Contributing

Please read [CONTRIBUTING.md](https://gist.github.com/PurpleBooth/b24679402957c63ec426) for details on our code of conduct, and the process for submitting pull requests to us.

## Versioning

We use [SemVer](http://semver.org/) for versioning. For the versions available, see the [tags on this repository](https://github.com/your/project/tags). 

## Authors

* **Billie Thompson** - *Initial work* - [PurpleBooth](https://github.com/PurpleBooth)

See also the list of [contributors](https://github.com/your/project/contributors) who participated in this project.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details

## Acknowledgments

* Hat tip to anyone whose code was used
* Inspiration
* etc

