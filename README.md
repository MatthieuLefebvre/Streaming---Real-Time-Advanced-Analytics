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


#### Task 1 : deploy an ADLS gen2 storage and create a service principal for OAuth access to the ADLS Gen2 filesystem

1. deploy an ADLS Gen2 Storage account

- add ressource : search for **Storage Account**
- Click on **Create**
- Select the right subscription and the right ressource group and then choose a storage account name like **prefixselabadlsgen2**
- Take the right location (East US) 
- Performance : Keep Standard
- Account Kind : Storage V2
- Replication : you can choose RA-GRS (3 local copie of data + 3 other copie in another Azure Region) if you want to test this capabilities, if not just choose LRS (3 local copie of data)
- In Advanced Tab : make sure to choose **DATA LAKE STORAGE GEN2 Hierarchical namespace enabled**
- Apply Tag if you want to. 
- Then Create !

As an added layer of security when accessing an ADLS Gen2 filesystem using Databricks you can use OAuth 2.0 for authentication. In this task, you will use the Azure CLI to create an identity in Azure Active Directory (Azure AD) known as a service principal to facilitate the use of OAuth authentication.

> **IMPORTANT**: You must have permissions within your Azure subscription to create an App registration and service principal within Azure  Active Directory to complete this task.

2. In the Azure portal, select the Cloud Shell icon in the top toolbar.

2. Ensure PowerShell is selected in the Cloud Shell pane

3. Next, you will issue a command to create a service principal named seanalytics-sp and assign it to the Storage Blob Data Contributor role on your ADLS Gen2 Storage account. The command will be in the following format:

```
az ad sp create-for-rbac -n "seanalytics-sp" --role "Storage Blob Data Contributor" --scopes {adls-gen2-storage-account-resource-id}
```
> **IMPORTANT** : You will need to replace the {adls-gen2-storage-account-resource-id} value with the resource ID of your ADLS Gen2 Storage account.

4. To retrieve the ADLS Gen2 Storage account resource ID you need to replace above, navigate to your Resource groups in the Azure navigation menu.

5. In your resource group, select the ADLS Gen2 Storage account you provisioned previously, and on the ADLS Gen2 Storage account blade select 
- **Properties** under **Settings** in the left-hand menu, 
- and then select the copy to clipboard button to the right of the **Storage account resource ID** value.

6. Paste the Storage account resource ID into the command above, and then copy and paste the updated ```az ad sp create-for-rbac``` command at the Cloud Shell prompt and press Enter. The command should retrieve Json like value with your subscription ID, App ID, etc.

7. Copy the output from the command into a text editor, as you will need it in the following steps.

8. To verify the role assignment, select Access control (IAM) from the left-hand menu of the ADLS Gen2 Storage account blade, and then select the Role assignments tab and locate seanalytics-sp under the STORAGE BLOB DATA CONTRIBUTOR role.

#### Task 2: Add the service principal credentials and Tenant Id to Azure Key Vault

1.Deploy an Azure key Vault ressource into your ressource group
- search for Key Vault
- Enter a name like selab-keyvault
- be carefull and choise the right subscription and the right RG
- Location : East US
- Pricing : Standard (Premium is to connect with an HSM device)
- Access Policies : Keep default
- Vnet : keep default
- Then Create ! 

1. To provide access your ADLS Gen2 account from Azure Databricks you will use secrets stored in your Azure Key Vault account to provide the credentials of your newly created service principal within Databricks. Navigate to your Azure Key Vault account in the Azure portal, then select **Access Policies** and select the **+ Add new** button.

2. Choose the account that you are currently logged into the portal with as the principal and check Select all under ```key permissions, secret permissions, and certificate permissions,``` then click OK and then click **Save**.

3. Now select **Secrets** under Settings on the left-hand menu. On the Secrets blade, select **+ Generate/Import** on the top toolbar.

4. On the Create a secret blade, enter the following:

- **Upload options**: Select Manual.
- **Name**: Enter "selab-SP-Client-ID".
- **Value**: Paste the **appId** value from the Azure CLI output you copied in an earlier step.

5. Select **Create**.

6. Select **+ Generate/Import** again on the top toolbar to create another secret.

7. On the Create a secret blade, enter the following:

- **Upload options**: Select Manual.
- **Name**: Enter "selab-SP-Client-Key".
- **Value**: Paste the password value from the Azure CLI output you copied in an earlier step.

8. Select **Create**.

9. To perform authentication using the service principal account in Databricks you will also need to provide your Azure AD Tenant ID. Select **+ Generate/Import** again on the top toolbar to create another secret.

10. On the Create a secret blade, enter the following:

- **Upload options**: Select Manual.
- **Name**: Enter "Azure-Tenant-ID".
- **Value**: Paste the tenant value from the Azure CLI output you copied in an earlier step.

11. Select **Create**.

#### Task 3: Configure ADLS Gen2 Storage Account in Key Vault

In this task, you will configure the Key for the ADLS Gen2 Storage Account within Key Vault.

1. In the Azure Portal, navigate to the **ADLS Gen2 Storage Account**, then select **Access keys** under Settings on the left-hand menu. You are going to copy the **Storage account name** and **Key** values and add them as secrets in your Key Vault account.

2. Open a new browser tab or window and navigate to your Azure Key Vault account in the Azure portal, then select **Secrets** under Settings on the left-hand menu. On the Secrets blade, select **+ Generate/Import** on the top toolbar.

3. On the Create a secret blade, enter the following:

- **Upload options**: Select Manual.
- **Name**: Enter "ADLS-Gen2-Account-Name".
- **Value**: Paste the Storage account name value you copied in an earlier step.

4. Select **Create**.

5. Select **+ Generate/Import** again on the top toolbar to create another secret.

6. On the Create a secret blade, enter the following:

- **Upload options**: Select Manual.
- **Name**: Enter "ADLS-Gen2-Account-Key".
- **Value**: Paste the Storage account Key value you copied in an earlier step.

7. Select **Create**.

#### Task 4: Create an Azure Databricks cluster
In this task, you will connect to your Azure Databricks workspace and create a cluster to use for this lab.

1. Return to the Azure portal :
- Provision an **Azure Databricks workspace**
- Then navigate into the new ressource, and select **Launch Workspace** from the overview blade, signing into the workspace with your Azure credentials, if required.

2. Welcome to Azure Databricks ! 

3. Select **Clusters** from the left-hand navigation menu, and then select **+ Create Cluster**.

4. On the Create Cluster screen, enter the following:

- **Cluster Name**: Enter a name for your cluster, such as lab-cluster.

- **Cluster Mode**: Select Standard.

- **Databricks Runtime Version**: Select Runtime: 5.2 (Scala 2.11, Spark 2.4.0).

- **Python Version**: Select 3.

- **Enable autoscaling**: Ensure this is checked.

- **Terminate after XX minutes of inactivity**: Leave this checked, and the number of minutes set to 120.

- **Worker Type**: Select Standard_DS4_v2.

- **Min Workers**: Leave set to 2.
- **Max Workers**: Leave set to 8.
- **Driver Type**: Set to Same as worker.

- **Expand Advanced Options** make sure or enter the following into the Spark Config box:
```
spark.databricks.delta.preview.enabled true
```

5. Select **Create Cluster**. It will take 3-5 minutes for the cluster to be created and started.

#### Task 5: Open Azure Databricks and load lab notebooks

In this task, you will import the notebooks contained in this git repo Event Hub real-time advanced analytics into your Azure Databricks workspace.

1. Inside of Databricks workspace, Select **Workspace** from the left-hand menu, then select **Users** and select your user account (email address), and then select the down arrow on top of your user workspace and select **Import** from the context menu.

2. Within the Import Notebooks dialog, select URL for Import from, and then paste the following into the box: ```
```
ADD URL HERE
```

3. Select Import.

You should now see a folder named EventHubAdvancedAnalytics in your user workspace. This folder contains all of the notebooks you will use throughout this lab.

#### Task 6: Configure Azure Databricks Key Vault-backed secrets

In this task, you will connect to your Azure Databricks workspace and configure Azure Databricks secrets to use your Azure Key Vault account as a backing store.

1. Return to the Azure portal, navigate to your newly provisioned Key Vault account and select **Properties** on the left-hand menu.

2. Copy the **DNS Name** and **Resource ID** property values and paste them to Notepad or some other text application that you can reference later. These values will be used in the next section.

3. Navigate to the Azure Databricks workspace you provisioned above, and select Launch Workspace from the overview blade, signing into the workspace with your Azure credentials, if required.

4. In your browser's URL bar, append #secrets/createScope to your Azure Databricks base URL (for example, https://eastus.azuredatabricks.net#secrets/createScope).

5. Enter ```key-vault-secrets``` for the name of the secret scope.

6. Select **Creator** within the Manage Principal drop-down to specify only the creator (which is you) of the secret scope has the MANAGE permission.

> MANAGE permission allows users to read and write to this secret scope, and, in the case of accounts on the Azure Databricks Premium Plan, to change permissions for the scope.

> Your account must have the Azure Databricks Premium Plan for you to be able to select Creator. This is the recommended approach: grant MANAGE permission to the Creator when you create the secret scope, and then assign more granular access permissions after you have tested the scope.

7. Enter the **DNS Name** (for example, https://selab-vault.vault.azure.net/) and **Resource ID** you copied earlier during the Key Vault creation step, for example: **/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourcegroups/lab/providers/Microsoft.KeyVault/vaults/selab-vault**.

8. Select **Create**.

After a moment, you will see a dialog verifying that the secret scope has been created.

#### Task 7: Install the Event Hub Spark Connector in Databricks

1. Select Workspace from the left-hand menu, then select the drop down arrow next to Shared and select **Create** and **Library** from the context menus.

2. On the Create Library page, select **Maven** under Library Source, and then select **Search Packages** next to the Coordinates text box.

3. On the Search Packages dialog, select **Maven Central** from the source drop down, enter **azure-eventhubs-spark** into the search box, and click Select next to Group Id **com.microsoft**, Artifact Id : **azure:azure-eventhubs-spark_2.11** release **2.3.12**

4. Select **Create** to finish installing the library.

5. On the following screen, check the box for **Install automatically on all clusters**, and select Confirm when prompted.



## Optionnal (If we have time)
Consume data from Power BI : 
https://eastus2.azuredatabricks.net:443/sql/protocolv1/o/5123375449920176/0408-155211-bonds806
token
dapi9f57dcb947fe8af555b0ade11b0b5559

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

