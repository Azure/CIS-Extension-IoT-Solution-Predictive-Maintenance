# Cortana Intelligence Suite Extension for the IoT Predictive Maintenance Solution  

The [Azure IoT Suite](https://www.azureiotsuite.com/) contains a solution for [Aerospace Predictive Maintenance](https://www.azureiotsuite.com/#solutions/types/PM) built
extending the work done in the [Predictive Maintenance for Aerospace – A Cortana Analytics Solution Template](https://blogs.technet.microsoft.com/machinelearning/2016/02/23/predictive-maintenance-for-aerospace-a-cortana-analytics-solution-template/)
by the data science group. 

The [Azure IoT Suite](https://www.azureiotsuite.com/) solution uses a different architecture to reach a similar experience with the core differences being:


 - Predictions are made for a single two-engine aircraft opposed to an entire fleet of aircraft. 
 - Event ingestion is performed using an [Azure IoT Hub](https://azure.microsoft.com/en-us/services/iot-hub/) opposed to an [Azure Event Hub](https://azure.microsoft.com/en-us/documentation/articles/event-hubs-overview/).
 - Event processing is performed using an [Azure WebJob](https://azure.microsoft.com/en-us/documentation/articles/web-sites-create-web-jobs/) opposed to an [Azure Data Factory](https://azure.microsoft.com/en-us/services/data-factory/).
 - The dashboard view is an [Azure Web App](https://azure.microsoft.com/en-us/services/app-service/web/) opposed to a [Power BI](https://powerbi.microsoft.com/en-us/) dashboard which
   allows for a more dynamic view of the events as they occur. 

This tutorial will guide you through the process of modifying, post deployment, the Iot Predictive Maintenance solution to utilize the [Cortana Intelligence Suite](aka.ms/cortanaintelligence) for 
event processing replacing the Azure Web Job.

## Modifying the IoT Suite Predictive Maintenance Solution 

### Prerequisites

This tutorial will require:

 - An Azure subscription, which will be used to deoploy the IoT suite project 
   (a [one-month free
   trial](https://azure.microsoft.com/en-us/pricing/free-trial/) is
   available for new users)
 - An [Azure Machine Learning Studio](https://studio.azureml.net/) account.   
 - A Windows Desktop with a network connection to run helper applications in the deployment modification process.
    - [Azure Powershell](https://azure.microsoft.com/en-us/documentation/articles/powershell-install-configure/) to obtain your [Azure Active Directory Tenent Id](https://blogs.technet.microsoft.com/heyscriptingguy/2013/12/31/get-windows-azure-active-directory-tenant-id-in-windows-powershell/)
 - [SQL Server Management Studio](https://msdn.microsoft.com/en-us/library/mt238290.aspx) installed.
 - A succesfully deployed [Azure IoT Suite](https://www.azureiotsuite.com/)  [Aerospace Predictive Maintenance](https://www.azureiotsuite.com/#solutions/types/PM) project.
   - Log into the IoT Suite to have link to Aerospace Predicitve Maintenance open from the link or when logged in 
   click on *Create a new solution* and choose Predictive Maintenance.
### Modification Requirements

To allow a seamless transition from the IoT project to a fully utilize the [Cortana Intelligence Suite](aka.ms/cortanaintelligence) services, several steps need to
be performed. To better understand the need for these changes it is best to describe how the existing architecture is working today. The following description 
is a high level definition of the event processing.

 - Events arrive at the Azure IoT Hub via the first Azure Web Job which acts as an event generator.
 - Azure Stream Analytics performs two queries on the Azure IoT Stream.
   - Raw "sensor" data is written to an Azure Storage Table used to drive graphs in the Azure Web App associated with the project.
   - Aggregated "sensor" data is sent to an Azure Event Hub. 
 - The second Azure Web Job then processes the events from the Azure Event Hub. As events arrive on the Azure Event Hub the Azure Web Job
   - Unpacks the Azure Event Hub message and calls the Azure Machine Learning experiment backing the project via the [Azure ML Request-Response Service (RRS)](https://azure.microsoft.com/en-us/documentation/articles/machine-learning-consume-web-services/) service.
   - Places the prediction result from the RRS call into another Azure Storage Table that drives other graphs in the Azure Web App associated with the project.          

The core steps here are to utilize an Azure Data Factory to perform the message processing. To do so, the following modifications must be made to accomplish that goal.

1. Disable the Azure Web Job that is processing events from the Azure Event Hub.
2. Modify the existing Azure Stream Analytics Job that is producing the aggregated "sensor" data being sent to the Azure Event Hub.
    - This is neccesary to create the column names expected by the Azure ML Experiment. 
3. Create an Azure SQL Server and Database to recieve the aggregated sensor readings.
    - This step also requires creating a SQL Table in the database.
4. Create a new Azure Stream Analytics Job that will move the events from the Azure Event Hub to the Azure SQL Database table. 
5. Create a series of Azure Data Factories that will process the events from the Azure SQL Database
    - Events will be processed using the [Azure ML Batch Execution Service (BES)](https://azure.microsoft.com/en-us/documentation/articles/machine-learning-consume-web-services/) service

#### So why a "series" of Auzre Data Factories you ask?
Azure Data Factories can scheduled over a variety of schedules. However, the shortest time frame to schedule a data factory is 15 minutes. 

To create a similar behavior with a dynamically updating graph in the Azure Web App, events need to be processed at a much faster pace than the 15 minutes allowed by 
the Azure Data Factory suite. To solve that issue we need to create multiple Azure Data Factory pipelines to process the events to replicate the dynamic feel
that the initial IoT Suite project has.

For data integrity purposes, however, two pipelines that produce the same data set (i.e. the Auzre Storage Table for ML results) are not allowed to be scheduled for 
the same time slice in the same Azure Data Factory. 

To work around that design, a complete data factory for each time window we wish to process must be created. For this tutorial, we require 15 data factories, one for
each minute of the 15 minute time slice. Doing so will result in a near-realtime experience.     

# Manual Modification Steps
This section will walk you through the steps to manually modify your deployed IoT Suite Predictive Maintenance project. While you will perform many steps manually we
wont have you manually create 15 Azure Data Factories. We have an application for that.

In the directions given below we will refer frequently to your ***[iotproject]***. This is the project name you chose for your IoT deployment, which also happens to be the Azure Resources
Group name that was created on your behalf. So, where you see ***[iotproject]*** in the documentation below, replace it with your project name.

## Disable the Azure Web Job that is processing events from the Azure Event Hub
 - Log into the [Azure Management Portal](https://ms.portal.azure.com) 
 - In the left hand menu select *Resource groups*
 - Locate the resource group  ***[iotproject]*** and click on it displaying the resources associated with the group in the resource group blade.
 - Locate the *App Service* ***[iotproject]***-jobhost and click on it displaying the app service blade.
 - On the *App Service* blade, locate WebJobs and click on it.
 - Click on the web job *EventProcessor-WebJob* and click Stop.

## Modify the existing Azure Stream Analytics Job
 - Log into the [Azure Management Portal](https://ms.portal.azure.com) 
 - In the left hand menu select *Resource groups*
 - Locate the resource group  ***[iotproject]*** and click on it displaying the resources associated with the group in the resource group blade.
 - Locate the *Stream Analytics job* ***[iotproject]***-Telemetry and click on it displaying the Stream Analytics job blade.
 - Click the *Stop* button
 - When the job stops, click on the *Query* button and replace the query with the query below.
 - Click *Save* and then *Start* 

### Stream Analytics Query Replacement
        WITH
            [StreamData] AS (
                SELECT
                    *
                FROM
                    [IoTHubStream]
                WHERE
                    [ObjectType] IS NULL -- Filter out device info and command responses
            )

        SELECT
            DeviceId,
            Counter,
            Cycle,
            Sensor9,
            Sensor11,
            Sensor14,
            Sensor15
        INTO
            [Telemetry]
        FROM
            [StreamData]

        SELECT
            DeviceId AS id,
            Cycle,
            AVG(Sensor9) AS s9,
            AVG(Sensor11) AS s11,
            AVG(Sensor14) AS s14,
            AVG(Sensor15) AS s15
        INTO
            [TelemetrySummary]
        FROM
            [StreamData]
        GROUP BY
            DeviceId,
            Cycle,
            SLIDINGWINDOW(minute, 2) -- Duration must cover the longest possible cycle
        HAVING
            SUM(EndOfCycle) = 2 -- Sum when EndOfCycle contains both start and end events


## Create SQL Server, SQL Database and SQL Database Table
 - Log into the [Azure Management Portal](https://ms.portal.azure.com) 
 - In the left hand menu select *Resource groups*
 - Locate the resource group  ***[iotproject]*** and click on it displaying the resources associated with the group in the resource group blade.
 - Click the *Add* button at the top of the resource group blade.
 - In the Search blade that appears, enter *SQL Database* and choose *SQL Database* from the selection.
 - Click Create
 - Enter a unique database name and record that name in the table above.
 - Keep the subscription and resource group the same as the IOT deployment.
 - Click on the Server 
 - You can choose an existing database or a create a new one. In either case, record the server name, database name, user name and password as you will need this information later.
 - Return to the ***[iotproject]*** resource group, locate the SQL Server and click on it displaying the SQL Server blade..
 - On the SQL Server blade, locate *Firewall* button under settings to display the Firewall blade.
   - In the *Firewall* blade create a new rule (by any name) with a start IP of 0.0.0.0 and end IP of 255.255.255.255. While this is NOT a secure way to configure
   your server, for the purposes of this tutorial it will reduce the friction of further steps.  
 - Using SQL Server Management Studio log into the server
   - Right click on the database that has been created for this modification and click New Query
   - Copy the SQL Table Script (below) into the query window and execute. 
   - Ensure that the table in the database has been created
 
 ### SQL Table Script
        CREATE TABLE [dbo].[iotextendtbl](
 	        eventprocessedutctime [datetime2] NULL,
 	        id varchar(256) NULL,
 	        cycle int NULL,
 	        s9 float NULL,
 	        s11 float NULL,
 	        s14 float NULL,
 	        s15 float NULL,
        )
 
        CREATE CLUSTERED INDEX [IX_iotextendtbl_Column]
            ON [dbo].[iotextendtbl]([id] ASC, [cycle] ASC, [eventprocessedutctime] ASC);


## Create a new Azure Stream Analytics Job
 - Log into the [Azure Management Portal](https://ms.portal.azure.com) 
 - In the left hand menu select *Resource groups*
 - Locate the resource group  ***[iotproject]*** and click on it displaying the resources associated with the group in the resource group blade.
 - Click the *Add* button at the top of the resource group blade.
 - In the Search blade that appears, enter *Stream Analytics job* and choose *Stream Analytics job* from the selection.
 - Click Create
 - Create a unique name for this stream job, choosing the ***[iotproject]*** resource group (implying the same subscription).
 - Navigate back to the ***[iotproject]*** resource group.
 - Click on the new Stream Analytics job just created (NOTE: you may need to refresh the Resource Group blade) displaying the Stream Analytics job 
 blade where the remainder of these instructions are based.
 - Click Inputs and on the exposed blade, click Add
   - Input Alias = *InputHub*
   - Source Type = *Data Stream*
   - Source = *Event hub*
   - Subscription  = *Use event hub from current subscription*
   - Service bus namespace = ***[iotproject]*** x
   - Event hub = ***[iotproject]***-ehdata
   - The remaining settings can remain as they are (key=RootManageSharedAccessKey, format=json)
   - Click *Create*
 - Click Outputs and on the exposed blade, click Add
   - Output Alias = *InputHub*
   - Sink = *SQL Database*
   - Database = **The database you created earlier**
   - The remaining settings (Username, Password) should be written down from the previous step of creating the server. The Table property should be
    **iotextendtbl** from the SQL script run previously.
   - Click *Create*
 - Click Query and on the exposed blade and copy and paste the New Stream Analytics job query below.
 - Click Save
 - Start the stream job

 ### New Stream Analytics job Query
        SELECT
            EventProcessedUtcTime,id,cycle,s9,s11,s14,s15
        INTO
            [OutputTable]
        FROM
            [InputHub]

## Create the new Azure Data Factories
As promised, this step will be fairly straightforward - and automated!

 - From this repository download the *IotUpdateAdf.zip* file and unzip the contents to your local drive.
 - The application will require you enter in information regarding your subscription, active directory tenent id, resource group (***[iotproject]***), 
 and SQL server information. To save yourself from having to enter this information in each time you run the application, these settings can be entered in the file
 *IotUpdateAdf.exe.config* file in the *IotUpdateAdf* directory. It is simply an XML based file where you can manually make edits and save them.
   - **NOTE SqlServer is simply the server name, i.e. myserveronazure and **not** the full server path as myserveronazure.database.windows.net** 
- Launch the *IotUpdateAdf.exe* and click the **Upgrade IOT Deployment** button when the information is entered and the button becomes enabled. The deployment will take approximately 15
minutes to complete. The remaining informaiton needed for the deployment will be obtained from your Azure Subscription while the application runs.    

**NOTE** Running the applicaiton more than once is supported. That is, if the factories have been created running the application will NOT cause your deployment any harm. 
If the tool partially ran, using the Azure Managment Portal, delete the data factories in the  ***[iotproject]*** resource group and run the application again.

# Automated Modification Steps
The automated modification steps perform all of the above steps for you through the use of a single application. 


 - From this repository download the *IotUpdateFull.zip* file and unzip the contents to your local drive.
 - The application will require you enter in information regarding your subscription, active directory tenent id, and resource group (***[iotproject]***).
  To save yourself from having to enter this information in each time you run the application, these settings can be entered in the file
  *IotUpdateFull.exe.config* file in the *IotUpdateFull* directory. It is simply an XML based file where you can manually make edits and save them.
- Launch the *IotUpdateFull.exe* and click the **Upgrade IOT Deployment** button when the information is entered and the button becomes enabled. The deployment will take approximately 15-20
minutes to complete.     

**NOTE** Running the applicaiton more than once is supported. That is, if a partial run occured running it again will complete the process with the following caveats:
 - If the SQL server was created but the create table script did not execute, you will need to manually run the script. 
 - If the data factories were not completely created, i.e. a partial factory deploymentusing the Azure Managment Portal, delete the data factories in the  
 ***[iotproject]*** resource group and run the application again.
