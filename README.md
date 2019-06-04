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

Before doing this : you may have to add your current IP public adress to the firewall rule. (propagation could take 15 min... BREAK ?)

Once the service is up, using Data Explorer, create 
a new database : 'malefedb'
click on Provision database throughput and set the value to 15000
add a new container : 'transactions'
add a new partition key : '/ipCountryCode'





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

