# Data Integration Pipelines For NYC Payroll Data Analytics

## Context of the project

The City of New York wants to create a data analytics platform using Azure Synapse Analytics to achieve two main goals:

- Examine how the city's funds are distributed and how much of its budget is set aside for overtime.

- Make the information accessible to the public so that they may see how the City's budget is being used to pay salaries and overtime to all municipal employees.

The source data resides in **Azure Data Lake** and needs to be processed in a NYC data warehouse in **Azure Synapse Analytics**. The source datasets consist of CSV files with Employee master data and monthly payroll data entered by various City agencies.


![DB Schema](/images/db-schema.jpeg "DB Schema")

## Project resources

For this project, we'll work in the Azure Portal, using several Azure resources including:

- **Azure Data Lake Gen2**
- **Azure SQL DB**
- **Azure Data Factory**
- **Azure Synapse Analytics**


## Step 1 : Prepare the Data Infrastructure


1. Create the data lake and upload data. For all sub-steps bellow, run *commands\storage_gen2.ps1*

- Create an **Azure Data Lake Storage Gen2** (storage account) and associated storage container. 

![ADLS](/images/adls2.PNG "ADLS")


- Create the three following directories in this storage container :

    - *dirpayrollfiles*
    - *dirhistoryfiles*
    - *dirstaging*

![Containers](/images/containers.PNG "Containers")

- Upload these files from the */data* folder to the *dirpayrollfiles* folder :

    - *EmpMaster.csv*
    - *AgencyMaster.csv*
    - *TitleMaster.csv*
    - *nycpayroll_2021.csv*

![files](/images/files.PNG "files")

- Upload the file *nycpayroll_2020.csv* from the project data to the *dirhistoryfiles* folder.

![files](/images/historyfiles.PNG "files")


2. Create an **Azure Data Factory** Resource.

![adf](/images/adf.PNG "adf")


3. Create a **SQL Database** to store the current year of the payroll data. For the two sub-steps bellow

- In the Azure portal, create a SQL Database resource named *nycpayroll-db*

- Add client IP address to the SQL DB firewall

- Create a table called *NYC_Payroll_Data* in *db_nycpayroll* in the Azure Query Editor using the SQL Script *nyc_payroll_data.sql*

![sqldatabase](/images/sqlserver.PNG "SQL Database")


4. Create A **Synapse Analytics workspace**. 

![synapse](/images/synapse.PNG "Synapse Workspace")

- Create a new **Azure Data Lake Gen2** and file system for **Synapse Analytics** when you are creating the Synapse Analytics workspace in the Azure portal.

- Create a **SQL dedicated pool** in the **Synapse Analytics** workspace. Select DW100c as performance level. Keep defaults for other settings.

![sqlpool](/images/sqlpool.PNG "SQL Pool")

- Go into the **Networking** section of your **Synapse Workspace** and update the firewall rules :

![Clientip](/images/clientip.PNG "Client IP")


## Step 2: Create Linked Services


1. In **Azure Data Factory**, create a **Linked Service** for **Azure Data Lake**

- create a linked service to the data lake that contains the data files

- From the data stores, select **Azure Data Lake Gen 2**. Name : *DataLakeFiles*

![linkedservices](/images/lsadls.PNG "Linked Services")


2. In **Azure Data Factory**, create a **Linked Service** to SQL Database that has the current (2021) data. If you get a connection error, remember to add the IP address to the firewall settings in SQL DB in the Azure Portal. 

![linkedservices](/images/lssqldb.PNG "Linked Services")


3. In **Azure Data Factory**, create a **Linked Service** : Create the linked service to the SQL pool. 

![linkedservices](/images/lssynapse.PNG "Linked Services")

In the end, you should have this :

![linkedservices](/images/lsall.PNG"Linked Services")


## Step 3: Create Datasets in Azure Data Factory

1. Create the datasets for the 2021 Payroll file on **Azure Data Lake Gen2**. 

- Select DelimitedText
- Set the path to the *nycpayroll_2021.csv* in the Data Lake

![Dataset](/images/dataset2021.PNG "Dataset")


2. Repeat the same process to create datasets for the rest of the data files in the Data Lake

- *EmpMaster.csv*. 

![Dataset](/images/empmaster.PNG "Dataset")

- *TitleMaster.csv*. 

![Dataset](/images/titlemaster.PNG "Dataset")

- *AgencyMaster.csv*. 

![Dataset](/images/agencymaster.PNG "Dataset")

- Publish all the datasets
![Dataset](/images/datasets.PNG "Dataset")



## Step 4: Create Data Flows

1. In **Azure Data Factory**, create the data flow to load *2021 Payroll* Data to SQL DB transaction table (in the future NYC will load all the transaction data into this table).

- Create a new data flow
- Select the dataset for the 2021 payroll file as the source

![Dataflow](/images/workflowcsv.PNG "Dataflow")

![Dataflow](/images/workflows.PNG "Dataflow")


5. Create pipelines for *Employee*, *Title*, *Agency*, and *year 2021 Payroll transaction* data to **Synapse Analytics** containing the data flows. Optionally you can also create one master pipeline to invoke all the Data Flows.

- Select the dirstaging folder in the data lake storage for staging

![Pipeline](/images/csv2021pipeline.PNG "Pipeline")

- Validate and publish the pipelines

![Pipeline](/images/masterpipeline.PNG "Pipeline")


## Step 5: Data Aggregation and Parameterization

In this step, we'll extract the 2021 year data and historical data, merge, aggregate and store it in **Synapse Analytics**. The aggregation will be on Agency Name, Fiscal Year and TotalPaid.

1. Create a Summary table in Synapse with the SQL script 

2. Create a new dataset for the **Azure Data Lake Gen2** folder that contains the historical files.

- Select *dirhistoryfiles* in the data lake as the source

3. Create new data flow and name it Dataflow Aggregate Data

- Create a data flow level parameter for Fiscal Year
- Add first Source for nyc payroll SQL table
- Add second Source for the Azure Data Lake history folder

4. Create a new Union activity in the data flow and Union with history files

5. Add a Filter activity after Union

- In Expression Builder, enter 
```
toInteger(FiscalYear) >= $dataflow_param_fiscalyear
```

6. Derive a new *TotalPaid* column

- In Expression Builder, enter 
```
RegularGrossPaid + TotalOTPaid+TotalOtherPay
```

7. Add an Aggregate activity to the data flow next to the *TotalPaid* activity

- Under Group By, Select AgencyName and Fiscal Year

8. Add a Sink activity to the Data Flow

- Select the dataset to target (sink) the data into the Synapse Analytics Payroll Summary table.
In Settings, select Truncate Table

9. Create a new Pipeline and add the Aggregate data flow

- Create a new Global Parameter (This will be the Parameter at the global pipeline level that will be passed on to the data flow)
- In Parameters, select **Pipeline Expression**
- Choose the parameter created at the Pipeline level

10. Validate, Publish and Trigger the pipeline. Enter the desired value for the parameter.

11. Monitor the Pipeline run and take a screenshot of the finished pipeline run.

**Aggregation Dataflow**
![Aggregation Flow](/images/aggregateflow.PNG "Aggregation Flow")

**Final Summary in Synapse**
![final summary](/images/finalsummary.PNG "Final Output")

## Step 6: Connect the Project to Github

In this step, we'll connect Azure Data Factory to Github

- Login to your Github account and create a new Repo in Github
- Connect Azure Data Factory to Github
- Select your Github repository in Azure Data Factory
- Publish all objects to the repository in Azure Data Factory
