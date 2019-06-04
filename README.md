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
The partition count on an event hub cannot be modified after setup. With that in mind, it is important to think about how many partitions you need before getting started.
Important point to remember is, TU is configured at namespace level. And, ONE Event Hubs namespace can have multiple eventhubs. Each eventhub can have different no. of partitions.
If you select 5 or more TUs on the Namespace and have only 1 EventHub with 4 partitions - you will get a max. of 4 MBPS or 4K msgs/sec.

Ideally, you will need more partitions than TUs. First, model your partition count as mentioned above. Start with 1 TU while you are developing your Solution. Once done, when you are doing load testing or going live, increase TUs to tune to your load. Remember, you could have multiple eventhubs in a Namespace. So, having 20 TUs at Namespace level and 10 eventhubs with 4 partitions each - can deliver 20 MBPS across the Namespace
You can't have more concurrent readers than you have partitions. If you want to have 5 concurrent readers, you need 5 partitions.
 
 
 #### Step 3 : Configuring Event Hubs and the transaction generator
 In this step, you will configure the payment transaction data generator and connect it to your Event Hub.
 Download repo code 
 open TransactionGenerator.sln in the files\TransactionGenerator directory.
 This will open the solution in Visual Studio.
 
 


```
Give the example
```

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

