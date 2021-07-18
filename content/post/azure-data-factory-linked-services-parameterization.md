---
title: "Azure Data Factory Linked Services Parameterization"
date: 2021-07-19T08:00:00Z
categories: [How-To]
tags: [Azure, Data Factory]
keywords: [Azure, Data Factory, Linked Services]
thumbnailImage: https://res.cloudinary.com/pedrofiadeiro/q_auto,f_auto,w_600,h_375,c_crop,g_face/v1616871096/Blog%20Images/Linked-Services-Parameters/LinkedServices.png
summary: Azure Data Factory linked services define your connections to external resources. Being able to parameterize them reduces the number of linked services needed and helps from a management perspective.
---

Linked services (LS) define the connection information to external resources, like connection strings. In Azure Data Factory (ADF) there are, currently, close to **100** different LS.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1616871096/Blog%20Images/Linked-Services-Parameters/LinkedServices.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1616871096/Blog%20Images/Linked-Services-Parameters/LinkedServices.png" title="Linked Services" >}}

These connections can be parameterized and values passed at runtime. For some of them, the parameterization can be done using the [ADF UI directly](https://docs.microsoft.com/en-us/azure/data-factory/parameterize-linked-services#supported-linked-service-types) . For the remaining LS, that parameterization can be achieved by manipulating the LS JSON.

In this blog post, we'll go through both options.

<!-- omit in toc -->
# Table of Contents

- [Prerequisites](#prerequisites)
- [Why parameterize a Linked Service?](#why-parameterize-a-linked-service)
- [How does it work?](#how-does-it-work)
- [Building the Demo Scenario](#building-the-demo-scenario)
    - [Parameterize the Azure SQL Database Linked Service](#parameterize-the-azure-sql-database-linked-service)
    - [Parameterize the Azure Data Lake Storage Linked Service](#parameterize-the-azure-data-lake-storage-linked-service)
    - [Create an Azure SQL Database Dataset](#create-an-azure-sql-database-dataset)
    - [Create an Azure Data Lake Storage Gen2 Dataset](#create-an-azure-data-lake-storage-gen2-dataset)
    - [Create Pipeline to Copy SQL Database Data](#create-pipeline-to-copy-sql-database-data)
    - [Create Pipeline to Copy Storage Account Blob](#create-pipeline-to-copy-storage-account-blob)
- [Executing and Testing the Demo Scenario](#executing-and-testing-the-demo-scenario)
    - [Copy Azure SQL Database data](#copy-azure-sql-database-data)
    - [Copy Storage Account Blob](#copy-storage-account-blob)
- [Wrapping Up](#wrapping-up)

# Prerequisites

If you want to follow the examples described in this post, I suggest you follow the instructions available **[here](https://github.com/pfiadeiro/blog-content/tree/master/Azure%20Data%20Factory%20Linked%20Services%20Parameterization)** which will deploy all the objects required for this blog post. All you need is an Azure subscription.

If you don't want to use that option, make sure you have the following items available:

- An Azure subscription. You may register for a **[free](https://azure.microsoft.com/en-gb/free/)** Azure subscription if you don’t have one yet
- An Azure Resource Group where you will place the resources created. Please refer to **[Create a Resource Group](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal#create-resource-groups)**
- An Azure Data Factory instance. Please refer to **[Create a Data Factory](https://docs.microsoft.com/en-us/azure/data-factory/quickstart-create-data-factory-portal#create-a-data-factory)** to see how to create a new instance using the portal
- Two Azure SQL Servers, each with a basic database (cheapest option). Please refer to **[Create a Single Database](https://docs.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart?tabs=azure-portal#create-a-single-database)**. Also, make sure to allow all Azure services in the server-level firewall rules, as well as your own IP as seen **[here](https://docs.microsoft.com/en-us/azure/azure-sql/database/firewall-create-server-level-portal-quickstart)** 
- Two Azure Data Lake Storage Gen2, each with a container created. Please refer to **[Create a Data Lake Storage Account](https://docs.microsoft.com/en-us/azure/storage/blobs/create-data-lake-storage-account)**
- Upload to one of the storage accounts created, the following **[file](https://github.com/pfiadeiro/blog-content/blob/master/Azure%20Data%20Factory%20Linked%20Services%20Parameterization/BlobObjects/FootballClubs.csv)** into the container created in the previous step
- Create in both databases the following **[table](https://github.com/pfiadeiro/blog-content/blob/master/Azure%20Data%20Factory%20Linked%20Services%20Parameterization/SQLScripts/CreateTable.sql)** and insert these **[records](https://github.com/pfiadeiro/blog-content/blob/master/Azure%20Data%20Factory%20Linked%20Services%20Parameterization/SQLScripts/InsertRecords.sql)** into just **one** of them
- Assign to the Azure Data Factory managed instance, the role ***Storage Blob Data Contributor*** in the 2 storage accounts. Please refer to **[Assign Azure roles](https://docs.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal?tabs=current)**

# Why parameterize a Linked Service?

If you use ADF to orchestrate the movement of data between several locations, it's quite easy to start accumulating a lot of different LS. You can easily use 4 or 5 Azure SQL Databases, 4 or 5 storage accounts, 4 or 5 REST APIs, etc.
 This makes the management of an ADF instance harder, not only from a development perspective but also when it comes to migrate objects to other environments.

This is the reason why the parameterization of LS is good. We can create only one LS per different type of connection and by making use of parameters, define which location we're trying to reach at runtime. These parameters can easily be defined and controlled through metadata and incorporated into the data flow process.

# How does it work?

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626538991/Blog%20Images/Linked-Services-Parameters/Runtime_Parameters.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626538991/Blog%20Images/Linked-Services-Parameters/Runtime_Parameters.png" title="Runtime Parameters" >}}

The capability of defining parameters in our LS to make them dynamic means that all subsequent components (datasets, activities and pipelines) will inherit that dynamic capability. It will be through a set of variables passing between each component that we define what is going to be executed at runtime.

Looking at the image above, assuming a Data Lake Storage Gen2 example, we have the following:
  - A parameter ***LS_AccountURL*** defined at LS level which is then assigned to the attribute ***URL*** on the LS
  - A parameter ***DS_AccountURL*** defined at dataset level which is assigned to the dataset LS property ***LS_AccountURL***
  - Two parameters, ***PL_SourceAccountURL*** and ***PL_TargetAccountURL***, defined at pipeline level which will be used in a copy activity to define the dataset property ***DS_AccountURL***

When we execute the pipeline, we assign values to the 2 pipeline parameters which will then be passed to the copy activity (and any other activities that may make use of them), passed to the dataset and, finally, passed to the LS. This allows us to reach 2 different storage accounts but have only 1 LS created.

# Building the Demo Scenario

The examples we're going to build are simple, we'll be moving some data between 2 Azure SQL Databases and also copy a .csv file from one storage account to another.

We'll parameterize the Azure SQL Database LS by using ADF UI and parameterize the Storage Account LS by manipulating the JSON directly. After that, we'll create 2 datasets (one for each type of linked service) and 2 pipelines to move the data.

### Parameterize the Azure SQL Database Linked Service

If you used the GitHub content to deploy the objects required to follow this blog post, you'll have two LS already created: one for an Azure SQL Database and another one for an Azure Data Lake Storage Gen2 (if you didn't, you'll need to create [them](https://docs.microsoft.com/en-us/azure/data-factory/quickstart-create-data-factory-portal#create-a-linked-service)). 

1. We'll start by editing the SQL Database connection by clicking on the object. 

{{< image classes="fancybox fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625596050/Blog%20Images/Linked-Services-Parameters/Manage_Linked_Services.png" title="Manage Linked Services" >}}

2. Create 4 parameters named ***ServerName***, ***DatabaseName***, ***Username*** and ***Password***. After that, fill in the details of the LS as seen in the image below and click on the ***Apply*** button.
   - **Fully qualified domain name** - *@{linkedService().ServerName}*
   - **Database name** - *@{linkedService().DatabaseName}*
   - **Authentication type** - leave default (SQL Authentication)
   - **User name** - *@{linkedService().Username}*
   - **Password** - *@{linkedService().Password}*

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/image/upload/v1625596925/Blog%20Images/Linked-Services-Parameters/LS_SQL_Details.png"  title="Linked Service Details" >}}

### Parameterize the Azure Data Lake Storage Linked Service

We're going to edit this specific LS by manipulating directly the JSON defining the object.

1. Click on the **Code** button to see the JSON definition of the object.

{{< image classes="fancybox fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625811746/Blog%20Images/Linked-Services-Parameters/LS_Open_Code.png" title="LS Code Definition" >}}

```JSON
{
    "name": "LS_DataLakeStorage",
    "type": "Microsoft.DataFactory/factories/linkedservices",
    "properties": {
        "type": "AzureBlobFS",
        "typeProperties": {
            "url": "https://youraccountnamehere.dfs.core.windows.net"
        },
        "annotations": []
    }
}
```

2. To parameterize this LS, we'll need to add a parameter, alongside its datatype, to the JSON definition and replace the ***url*** value with the name of that parameter. The definition will look like what you can see below. Click on the ***Apply*** button.

```JSON
{
    "name": "LS_DataLakeStorage",
    "properties": {
        "parameters": {
            "AccountUrl": {
                "type": "string"
            }
        },
        "annotations": [],
        "type": "AzureBlobFS",
        "typeProperties": {
            "url": "@{linkedService().AccountUrl}"
        }
    },
    "type": "Microsoft.DataFactory/factories/linkedservices"
}
```

### Create an Azure SQL Database Dataset

1. Go to the ***Author*** page and select the option to create a ***New dataset***. Once the window opens, choose the ***Azure SQL Database*** option and click on ***Continue***.

{{< image classes="fancybox fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625961826/Blog%20Images/Linked-Services-Parameters/New_SQL_Dataset.png" title="New Azure SQL Dataset" >}}

2. On the next screen, name the dataset ***GenericSQLTable*** and choose the LS already created, ***LS_SQLServer***. Click on the ***OK*** button.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625962590/Blog%20Images/Linked-Services-Parameters/Define_SQL_Dataset.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1625962590/Blog%20Images/Linked-Services-Parameters/Define_SQL_Dataset.png" title="Define Azure SQL Dataset" >}}

3. Once you click on the OK button, a new tab will open with the new dataset just created. The first thing you'll immediately notice is that the dataset has **4** LS properties corresponding to the 4 parameters we used in the definition of the LS. Choose the ***Parameters*** tab and create 5 parameters:
    - **LS_ServerName**
    - **LS_DatabaseName**
    - **LS_Username**
    - **LS_Password**
    - **TableName**

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625964849/Blog%20Images/Linked-Services-Parameters/SQL_Dataset_Parameters.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1625964849/Blog%20Images/Linked-Services-Parameters/SQL_Dataset_Parameters.png" title="SQL Dataset Parameters" >}}

4. Go back to the ***Connection*** tab and input the 5 parameters we just defined as seen below.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625964850/Blog%20Images/Linked-Services-Parameters/SQL_Dataset_ConnectionDetails.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1625964850/Blog%20Images/Linked-Services-Parameters/SQL_Dataset_ConnectionDetails.png" title="SQL Dataset Connection Definition" >}}

### Create an Azure Data Lake Storage Gen2 Dataset

1. Go to the ***Author*** page and select the option to create a ***New dataset***. Once the window opens, choose the ***Azure Data Lake Storage Gen2*** option and click on ***Continue***.

{{< image classes="fancybox fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625962039/Blog%20Images/Linked-Services-Parameters/New_Data_Lake_Dataset.png" title="New Azure Data Lake Dataset" >}}

2. On the next screen, select the option ***Delimited Text*** and click on the ***Continue*** button.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625965298/Blog%20Images/Linked-Services-Parameters/Dataset_FileFormat.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1625965298/Blog%20Images/Linked-Services-Parameters/Dataset_FileFormat.png" title="Dataset File Format" >}}

3. On the next screen, name the dataset ***GenericDelimitedFile*** and choose the LS already created, ***LS_DataLakeStorage***. Click on the ***OK*** button.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625965501/Blog%20Images/Linked-Services-Parameters/Define_DataLake_Dataset.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1625965501/Blog%20Images/Linked-Services-Parameters/Define_DataLake_Dataset.png" title="Define Azure Data Lake Dataset" >}}

4. As before, a new tab will open with the new dataset just created with 1 LS property. Choose the ***Parameters*** tab and create 3 parameters:
    - **LS_AccountUrl**
    - **ContainerName**
    - **FileName**

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625965900/Blog%20Images/Linked-Services-Parameters/DataLake_Dataset_Parameters.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1625965900/Blog%20Images/Linked-Services-Parameters/DataLake_Dataset_Parameters.png" title="Data Lake Dataset Parameters" >}}

5. Go back to the ***Connection*** tab and input the 3 parameters we just defined as seen below. Leave all other options as default.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625965900/Blog%20Images/Linked-Services-Parameters/DataLake_Dataset_ConnectionDetails.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1625965900/Blog%20Images/Linked-Services-Parameters/DataLake_Dataset_ConnectionDetails.png" title="Data Lake Dataset Connection Definition" >}}

6. Click the ***Publish*** button to save the 2 datasets just created. When a new window opens on the right-side, click the ***Publish*** button again.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1625999122/Blog%20Images/Linked-Services-Parameters/Publish.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1625999122/Blog%20Images/Linked-Services-Parameters/Publish.png" title="Publish" >}}

### Create Pipeline to Copy SQL Database Data

1. Go to the ***Author*** page and select the option to create a ***New pipeline***. A new tab will open with the new pipeline created.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626000727/Blog%20Images/Linked-Services-Parameters/New_Pipeline.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626000727/Blog%20Images/Linked-Services-Parameters/New_Pipeline.png" title="New Pipeline" >}}

2. Add a ***Copy data*** activity to the pipeline.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626012334/Blog%20Images/Linked-Services-Parameters/Add_Copy_Data_Activity.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626012334/Blog%20Images/Linked-Services-Parameters/Add_Copy_Data_Activity.png" title="Add Copy Data Activity" >}}

3. On the ***Parameters*** tab of the pipeline, define the following **10** parameters:
    - **SourceServer**
    - **SourceDatabase**
    - **SourceUsername**
    - **SourcePassword**
    - **SourceTable**
    - **TargetServer**
    - **TargetDatabase**
    - **TargetUsername**
    - **TargetPassword**
    - **TargetTable**

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626014285/Blog%20Images/Linked-Services-Parameters/Pipeline_Parameters.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626014285/Blog%20Images/Linked-Services-Parameters/Pipeline_Parameters.png" title="Pipeline Parameters" >}}

4. Click on the Copy data activity in the pipeline and in the ***Source*** tab of the activity, select the ***GenericSQLTable*** as the source dataset and input the following parameters we just defined as the Dataset properties (see image below)

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626014286/Blog%20Images/Linked-Services-Parameters/Copy_Source_Definitions.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626014286/Blog%20Images/Linked-Services-Parameters/Copy_Source_Definitions.png" title="Copy Activity Source Definitions" >}}

5. In the ***Sink*** tab of the Copy data activity, select the ***GenericSQLTable*** as the sink dataset and input the following parameters we just defined as the Dataset properties (see image below)

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626014286/Blog%20Images/Linked-Services-Parameters/Copy_Sink_Definitions.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626014286/Blog%20Images/Linked-Services-Parameters/Copy_Sink_Definitions.png" title="Copy Activity Sink Definitions" >}}

### Create Pipeline to Copy Storage Account Blob

1. As in the previous section, create a new pipeline and add a Copy data activity to it.

2. On the ***Parameters*** tab of the pipeline, define the following **6** parameters:
    - **SourceAccountUrl**
    - **SourceContainer**
    - **SourceFileName**
    - **TargetAccountUrl**
    - **TargetContainer**
    - **TargetFileName**

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626016083/Blog%20Images/Linked-Services-Parameters/Pipeline2_Parameters.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626016083/Blog%20Images/Linked-Services-Parameters/Pipeline2_Parameters.png" title="Pipeline Parameters" >}}

3. Click on the Copy data activity in the pipeline and in the ***Source*** tab of the activity, select the ***GenericDelimitedFile*** as the source dataset and input the following parameters we just defined as the Dataset properties (see image below).

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626016084/Blog%20Images/Linked-Services-Parameters/Copy2_Source_Definitions.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626016084/Blog%20Images/Linked-Services-Parameters/Copy2_Source_Definitions.png" title="Copy Activity Source Definitions" >}}

4. In the ***Sink*** tab of the Copy data activity, select the ***GenericDelimitedFile*** as the sink dataset and input the following parameters we just defined as the Dataset properties (see image below).

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626016083/Blog%20Images/Linked-Services-Parameters/Copy2_Sink_Definitions.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626016083/Blog%20Images/Linked-Services-Parameters/Copy2_Sink_Definitions.png" title="Copy Activity Sink Definitions" >}}

5. Click the ***Publish*** button to save the 2 pipelines just created. When a new window opens on the right-side, click the ***Publish*** button again.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626016376/Blog%20Images/Linked-Services-Parameters/Publish_Pipelines.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626016376/Blog%20Images/Linked-Services-Parameters/Publish_Pipelines.png" title="Publish" >}}

# Executing and Testing the Demo Scenario

We're now going to execute the 2 pipelines created. The first will copy records from a table in our source database to our target database. The second will copy a blob from our source storage account to our target storage account.

First, let's check our database tables to make sure we only have records in our source table. The easiest way to do that is going to the Azure portal, select the database resource and choose the option ***Query editor(preview)***. You'll see a SQL server authentication area where you need to enter the password defined when deploying the resources.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626269358/Blog%20Images/Linked-Services-Parameters/Query_Editor.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626016376/Blog%20Images/Linked-Services-Parameters/Query_Editor.png" title="Query Editor" >}}

After logging in, execute the following query  
```SQL 
SELECT * FROM dbo.FootballClubs
```
In the source database, you should see 9 records being returned.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626269259/Blog%20Images/Linked-Services-Parameters/Source_Query_Results.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626269259/Blog%20Images/Linked-Services-Parameters/Source_Query_Results.png" title="Source Data" >}}

In the target database, there won't be any records.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626269259/Blog%20Images/Linked-Services-Parameters/Target_Query_Results.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626269259/Blog%20Images/Linked-Services-Parameters/Target_Query_Results.png" title="Target Data" >}}

To check our storage accounts, the best option is again to use the portal, select the storage resource and choose the option ***Containers***. You should then see a container named ***sourcecontainer*** or ***targetcontainer*** depending on which storage account you're looking at.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626287237/Blog%20Images/Linked-Services-Parameters/Storage_Account_Containers.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626287237/Blog%20Images/Linked-Services-Parameters/Storage_Account_Containers.png" title="Storage Account Containers" >}}

Within the container ***sourcecontainer***, you should see the file ***FootballClubs.csv***.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626287237/Blog%20Images/Linked-Services-Parameters/Source_Container.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626287237/Blog%20Images/Linked-Services-Parameters/Source_Container.png" title="Source Container Blobs" >}}

The container ***targetcontainer*** should be empty.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626287237/Blog%20Images/Linked-Services-Parameters/Target_Container.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626287237/Blog%20Images/Linked-Services-Parameters/Target_Container.png" title="Target Container Blobs" >}}

### Copy Azure SQL Database data

1. Back in ADF, open ***pipeline1*** and choose the option ***Trigger now***

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346539/Blog%20Images/Linked-Services-Parameters/Execute_SQL_Pipeline.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346539/Blog%20Images/Linked-Services-Parameters/Execute_SQL_Pipeline.png" title="Trigger Pipeline" >}}

2. A window will open on the right-hand side asking for values of all the parameters we defined in the pipeline. Input the following values and click on the ***OK*** button.
    - **SourceServer** - fully qualified name of the server such as ***sourcesqlsrvXXXXXX.database.windows.net***
    - **SourceDatabase** - ***SourceDB***
    - **SourceUsername** - ***saXXXXXX*** (name of the server admin that can be seen on the portal's server page)
    - **SourcePassword** - password you defined when deploying the resources
    - **SourceTable** - ***FootballClubs***
    - **TargetServer** - fully qualified name of the server such as ***targetsqlsrvXXXXXX.database.windows.net***
    - **TargetDatabase** - ***TargetDB***
    - **TargetUsername** - ***saXXXXXX*** (name of the server admin that can be seen on the portal's server page)
    - **TargetPassword** - password you defined when deploying the resources
    - **TargetTable** - ***FootballClubs***

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/SQL_Pipeline_Parameters.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/SQL_Pipeline_Parameters.png" title="SQL Pipeline Parameters" >}}

3. Since the pipeline is now running, we need to go to the ***Monitor*** page and after a few seconds you should see an indication of ***Succeeded*** for the pipeline. Click on the pipeline name.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/SQL_Pipeline_Status.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/SQL_Pipeline_Status.png" title="SQL Pipeline Status" >}}

4. Once you click on the pipeline name, you'll go to another page where you can see the copy activity completed successfully and you can see the details of the copy activity.
   
{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Copy_Activity_Details.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Copy_Activity_Details.png" title="Activity Details" >}}

5. We can see in the copy activity details that **9** rows were copied. To make sure that happened, we go back to our target database in the portal and run the same query we executed before running the pipeline. You should get 9 rows returned.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Copy_Details.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Copy_Details.png" title="Copy Activity Details" >}}

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Target_Query_Results_After_Pipeline.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Target_Query_Results_After_Pipeline.png" title="SQL Target Data after Pipeline Execution" >}}

### Copy Storage Account Blob

1. Back in ADF, open ***pipeline2*** and choose the option ***Trigger now***.

2. A window will open on the right-hand side asking for values of all the parameters we defined in the pipeline. Input the following values and click on the ***OK*** button.
    - **SourceAccountUrl** - fully qualified name of storage account as ***https://stsourceXXXX.dfs.core.windows.net/***
    - **SourceContainer** - ***sourcecontainer***
    - **SourceFilename** - ***FootballClubs.csv***
    - **TargetAccountUrl** - fully qualified name of storage account as ***https://sttargetXXXX.dfs.core.windows.net/***
    - **TargetContainer** - ***targetcontainer***
    - **TargetFilename** - ***FootballClubs.csv***

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Blob_Pipeline_Parameters.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Blob_Pipeline_Parameters.png" title="Blob Pipeline Parameters" >}}

3. We go back to the ***Monitor*** page to see the status of the pipeline. Click on the pipeline name.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Blob_Pipeline_Status.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Blob_Pipeline_Status.png" title="Blob Pipeline Status" >}}

4. We can see in the copy activity details that **1** file was copied. To make sure that happened, we go to our target storage account in the portal and check the contents of the container. There should be now one file named ***FootballClubs.csv***.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Blob_Copy_Details.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Blob_Copy_Details.png" title="Copy Activity Details" >}}

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Target_Container_After_Pipeline.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1626346340/Blog%20Images/Linked-Services-Parameters/Target_Container_After_Pipeline.png" title="Storage Container after Pipeline Execution" >}}

# Wrapping Up

On this blog post we saw how we can make use of parameterized LS in ADF. The parameterization of LS will give a lot of flexibility when designing data flows and will be an important item in achieving a metadata driven process.

Don't forget to delete any resources you've created to follow this blog post if you no longer plan on using them.

Thanks for reading!

