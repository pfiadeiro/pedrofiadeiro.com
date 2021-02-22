---
title: "Azure Data Factory Policies"
date: 2021-02-21T07:00:00Z
categories: [How-To, Governance]
tags: [Azure, Data Factory, Azure Policy]
keywords: [Azure, Data Factory, Azure Policy]
thumbnailImage: https://res.cloudinary.com/pedrofiadeiro/image/upload/v1613392669/Blog%20Images/Data-Factory-Policies/azure-data-factory-policies.png
summary: Azure Policies are an essential component of governance in Azure. We can now implement policies in Data Factory and we'll use this blog post to look at a few examples of how they can be applied
---

Azure Policy helps enforcing organizational standards and assess compliance of the different resources created in Azure making it a key component of governance in Azure.

There are now a few **[built-in policy definitions](https://docs.microsoft.com/en-us/azure/data-factory/policy-reference)** for Data Factory and we're going to take a look at one of those policy definitions in this blog post. We'll also create and apply one custom policy definition to understand how it can be done.
<!--more-->

<!-- omit in toc -->
# Table of Contents

- [Prerequisites](#prerequisites)
- [Azure Policy - why is it important?](#azure-policy---why-is-it-important)
- [Assign our first built-in policy - use Key Vault to store Linked Services secrets](#assign-our-first-built-in-policy---use-key-vault-to-store-linked-services-secrets)
    - [Create ADF Linked Services](#create-adf-linked-services)
    - [Quick Look at the Policy Definition](#quick-look-at-the-policy-definition)
    - [Assign Policy Definition](#assign-policy-definition)
- [Fix a Linked Service](#fix-a-linked-service)
    - [Create Key Vault Linked Service](#create-key-vault-linked-service)
    - [Change Azure SQL Database Linked Service Definition](#change-azure-sql-database-linked-service-definition)
    - [Check Linked Services Compliance](#check-linked-services-compliance)
- [Create a custom ADF policy definition](#create-a-custom-adf-policy-definition)
    - [Understanding Linked Services JSON definition](#understanding-linked-services-json-definition)
    - [Build the policy definition](#build-the-policy-definition)
    - [Create a self-hosted integration runtime in Data Factory](#create-a-self-hosted-integration-runtime-in-data-factory)
    - [Create a policy definition in the Portal](#create-a-policy-definition-in-the-portal)
    - [Assign the Custom Policy Definition](#assign-the-custom-policy-definition)
    - [Change one Linked Service to use a self-hosted integration runtime](#change-one-linked-service-to-use-a-self-hosted-integration-runtime)
- [Wrapping Up](#wrapping-up)

# Prerequisites

If you want to follow the examples described in this post, make sure you have the following items available:

- An Azure subscription. You may register for a **[free](https://azure.microsoft.com/en-gb/free/)** Azure subscription if you don’t have one yet.
- An Azure Resource Group where you will place the resources created. Please refer to **[Create a Resource Group](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal#create-resource-groups)**.
- An Azure Data Factory instance. Please refer to **[Create a Data Factory](https://docs.microsoft.com/en-us/azure/data-factory/quickstart-create-data-factory-portal#create-a-data-factory)** to see how to create a new instance using the portal.
- For our custom example, we'll need a Self-Hosted Integration Runtime (IR) but it can only be installed on 64-bit Windows machines (find the **[prerequisites here](https://docs.microsoft.com/en-us/azure/data-factory/create-self-hosted-integration-runtime#prerequisites)**). The installer can be downloaded from this **[location](https://www.microsoft.com/en-us/download/details.aspx?id=39717)** and the setting up of the IR is described **[here](https://docs.microsoft.com/en-us/azure/data-factory/create-self-hosted-integration-runtime#setting-up-a-self-hosted-integration-runtime)**. If you don't have a 64-bit Windows machine, you can use an Azure VM for this purpose. The example should still be clear enough for you to understand it even if you don't install the Self-Hosted IR.

# Azure Policy - why is it important?

I won't be diving into too much detail when it comes to Azure Policy, I'll let that topic for another blog post, but it's important to understand why Azure Policy is a key component in Azure governance.

We all have worked in companies/projects that have a set of defined rules regarding different topics: implementation procedures, security procedures, management procedures. We can also all probably agree that if there are no steps in places to enforce those same procedures, they tend not to be always followed, either by unfamiliarity, oversight or simple disregard of those same procedures.

That's where Azure Policy comes in and can help us. By using it, we can check for resource consistency, regulatory compliance, security, cost and management. Azure provides us a long list of built-in policy definitions that can be applied to different resources and we can always create new policy definitions that suit our needs.

These policy definitions use a JSON format to form the logic the evaluation uses to determine if a resource is compliant or not. If you want to read more about Azure Policy, the [MS Docs](https://docs.microsoft.com/en-us/azure/governance/policy/overview) are a great resource.

# Assign our first built-in policy - use Key Vault to store Linked Services secrets

The first policy definition we're going to use is a **[policy](https://github.com/Azure/azure-policy/blob/master/built-in-policies/policyDefinitions/Data%20Factory/LinkedService_InlineSecrets_Audit.json)** to check whether Azure Data Factory (ADF) linked services are using Key Vault for storing secrets.

Let's first go to ADF and create some linked services that we'll use in this example. 

### Create ADF Linked Services

1. Go to the Azure Portal and select the ADF resource you created and choose the option **Author and monitor**. As an alternative, you can go straight to [adf.azure.com](https://adf.azure.com/) and select your tenancy, subscription and ADF resource created.

{{< image classes="fancybox fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613995718/Blog%20Images/Data-Factory-Policies/Launch_ADF.png" title="Launch ADF" >}}

2. In the ADF page, select the option **Manage** on the left-hand side, the **Linked Services** option within the Connections section and click on the **New** button.

{{< image classes="fancybox fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613504710/Blog%20Images/Data-Factory-Policies/Manage_LinkedServices.png" title="Manage Linked Services" >}}

3. Search for *"azure sql database"* and choose the option available.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613508740/Blog%20Images/Data-Factory-Policies/New_Linked_Service.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1613508740/Blog%20Images/Data-Factory-Policies/New_Linked_Service.png" title="Azure SQL DB Linked Service" >}}

4. Fill in the following details and click the button **Create**. We're obviously inserting fake details but for purposes of building this example, we don't really need to specify real connections.
    - Name - ***AzureSqlDatabase***
    - Account selection method - ***Enter manually***
    - Fully qualified domain name - ***fakeserver.database.windows.net***
    - Database name - ***fakedatabase***
    - Authentication type - ***SQL authentication***
    - User name - ***fakeuser***
    - Password - ***fakepwd***

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613508740/Blog%20Images/Data-Factory-Policies/New_Linked_Service_Connection_Details.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_450,g_face/v1613508740/Blog%20Images/Data-Factory-Policies/New_Linked_Service_Connection_Details.png" title="Linked Service Details" >}}

5. Create two more linked services, one for an Azure Blob Storage and another for an Oracle database using a random value for the password fields. Details can be seen below.

{{< image classes="fancybox fig-50" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613545558/Blog%20Images/Data-Factory-Policies/New_LS_Oracle.png" title="Oracle Linked Service" >}}
{{< image classes="fancybox fig-50" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613545558/Blog%20Images/Data-Factory-Policies/New_LS_Blob.png" title="Blob Storage Linked Service" >}}
{{< image classes="fancybox fig-50" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613545558/Blog%20Images/Data-Factory-Policies/New_LS_Oracle_Details.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,h_462/v1613508740/Blog%20Images/Data-Factory-Policies/New_LS_Oracle_Details.png" title="Oracle Details" >}}
{{< image classes="fancybox fig-50" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613545558/Blog%20Images/Data-Factory-Policies/New_LS_Blob_Details.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_371/v1613508740/Blog%20Images/Data-Factory-Policies/New_LS_Blob_Details.png" title="Blob Storage Details" >}}

6. Click on the **Publish all** button to save the recently created linked services.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613508740/Blog%20Images/Data-Factory-Policies/Publish_All.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1613508740/Blog%20Images/Data-Factory-Policies/Publish_All.png" title="Publish All" >}}

### Quick Look at the Policy Definition

With the linked services created, it's now time to assign the **[policy definition](https://github.com/Azure/azure-policy/blob/master/built-in-policies/policyDefinitions/Data%20Factory/LinkedService_InlineSecrets_Audit.json)** to our subscription/resource group. 

Before assigning the policy, let's just look briefly at some components of the policy to get an understanding of what it will be looking for. This policy has a long JSON definition so we're just going to look at some bits of it and focus on the **policyRule** section.

In Azure Policy, the operator ***allOf*** has the same meaning of a logical ***AND*** (all conditions need to be true) and the operator ***anyOf*** has the same meaning of a logical ***OR*** (one or more conditions need to be true).

The policy rule starts by defining that is's going to look at all linked services. By being included in an ***allOf*** section it means that this policy rule will only evaluate to true if this condition and the following ones are all true. 

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613639783/Blog%20Images/Data-Factory-Policies/policy_rule_ls.png" title="Policy Rule First Condition" >}}

However, the next (long) section is an ***anyOf*** block which means that only one of the many conditions specified need to be true for this policy rule to be fulfilled.

Looking at just one of those conditions evaluated, we can see that it's looking for any linked services that have a property named **connectionString** and, if they do, whether that connection string contains some specific words such as ***AccountKey=***, ***PWD=***, ***Password=***, ***CredString=***, ***pwd=***.

If this condition evaluates to true, we'll know that we have a linked service whose definition is specifying the password of a connection directly in the linked service (even though it will show as an encrypted credential in the linked service JSON) and not making use of Key Vault to store that password.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613639783/Blog%20Images/Data-Factory-Policies/policy_rule2_ls.png" title="Policy Rule Second Condition" >}}

### Assign Policy Definition

We'll assign this policy definition to the resource group we created and see the outcome of our resource compliance.

1. Go to the Azure Portal and select the **Policy** service.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613653819/Blog%20Images/Data-Factory-Policies/Azure_Policy.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1613653819/Blog%20Images/Data-Factory-Policies/Azure_Policy.png" title="Azure Policy Service" >}}

2. Select **Definitions** on the left side of the Azure Policy page. In the drop-down for **Type** choose the option ***Built-in*** and in the drop-down for **Category** choose the option ***Data Factory***. There will be a few policy definitions listed, choose the one named ***"[Preview] Azure Data Factory linked services should use Key Vault for storing secrets"*** (*Note:* It's possible that at the time you're reading this blog post, the *[Preview]* suffix has been removed from the name).

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613653819/Blog%20Images/Data-Factory-Policies/Choose_Policy_Definition.png" title="Choose Policy Definition" >}}

3. In the next page, details of the policy definition are available such as its name, description, type, category and also what type of effects can be selected for this policy definition. Click on the button **Assign**.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_800/v1613654991/Blog%20Images/Data-Factory-Policies/Assign_Policy.png" title="Assign" >}}

4. In the Assign Policy page we need to go through a few tabs:
   - **Basics** tab:
     - ***Scope***: Select the resource group you created in the prerequisites section
     - ***Assignment name***: You can leave the default value or change for something else you rather call your assignment
     - ***Policy enforcement***: Enabled (default value)
   - **Parameters** tab:
     - ***Effect***: Audit (default value)
   - **Remediation** tab:
     - No changes needed
   - **Non-compliance messages** tab:
     - ***Non-compliance message***: Insert message that will show when resource isn't compliant
   - **Review + create** tab:
     - Click on the **Create** button

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613660681/Blog%20Images/Data-Factory-Policies/Policy_Assignment_Basics.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1613660681/Blog%20Images/Data-Factory-Policies/Policy_Assignment_Basics.png" title="Basics tab" >}}
{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613660681/Blog%20Images/Data-Factory-Policies/Policy_Assignment_Parameters.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1613660681/Blog%20Images/Data-Factory-Policies/Policy_Assignment_Parameters.png" title="Parameters tab" >}}
{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613660681/Blog%20Images/Data-Factory-Policies/Policy_Assignment_NonComplianceMessage.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1613660681/Blog%20Images/Data-Factory-Policies/Policy_Assignment_NonComplianceMessage.png" title="Non-compliance messages tab" >}}
{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613660681/Blog%20Images/Data-Factory-Policies/Policy_Assignment_ReviewCreate.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1613660681/Blog%20Images/Data-Factory-Policies/Policy_Assignment_ReviewCreate.png" title="Review + Create tab" >}}

5. If you now go back to the main Policy page and select the option **Assignments**, we can filter by the name you gave your assignment and see, the just created, new assignment.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613663970/Blog%20Images/Data-Factory-Policies/Policy_Assignments.png" title="Policy Assignments" >}}

6. Selecting the option **Compliance**, on the left side menu, will lead us to a page where we can filter again by assignment name and see how many resources are compliant with our policy. It may take up to 15-30mins for the policy assignment to kick in and you will see the compliance state of the policy marked as ***Not started*** until it's evaluated for the first time.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613663970/Blog%20Images/Data-Factory-Policies/Compliance_NotStarted.png" title="Policy Compliance Not Started" >}}

7. Once it gets evaluated, we'll see that we have a **Non-compliant** state and that none of our resources are in a compliant state. Click on the assignment name to see more details.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614029979/Blog%20Images/Data-Factory-Policies/NonCompliant_Resources.png" title="Policy Non-Compliant" >}}

8. Finally, once inside the policy compliance page we can see more details. We have an overall compliance state and how many resources are compliant, exempt or non-compliant. We also have a list, per resource, with their state, scope and when was the last evaluation made. Clicking on the **Details** link of our Azure SQL Database resource, we can see specific information such as resource name, resource type, its compliance state and the custom non-compliance message we entered when creating a new assignment.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613995902/Blog%20Images/Data-Factory-Policies/NonCompliant_Details.png" title="Non-compliance Details" >}}

# Fix a Linked Service

After our assignment evaluation ran, we know that we have 3 resources in a non-compliant state. Let's change that and make sure that one of these linked services gets changed to a compliant state.

### Create Key Vault Linked Service

We'll be using fake Key Vault details (we didn't create one) which is enough to test our policy compliance. However, in a real use case scenario, you would have a proper Key Vault resource to store your secrets.

1. Going back to ADF's linked services page, select the option **New** and search for Key Vault.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613508740/Blog%20Images/Data-Factory-Policies/New_LS_KeyVault.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1613508740/Blog%20Images/Data-Factory-Policies/New_LS_KeyVault.png" title="Key Vault Linked Service" >}}

2. Fill in the required details. Choose the option **Enter manually** and insert a **Base url** such as *https://mykeyvault.vault.azure.net/*. Click on the option **Create** once you're done.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613978739/Blog%20Images/Data-Factory-Policies/New_LS_KeyVault_Details.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,c_thumb,w_500,g_face/v1613978739/Blog%20Images/Data-Factory-Policies/New_LS_KeyVault_Details.png" title="Key Vault Linked Service Details" >}}

### Change Azure SQL Database Linked Service Definition

1. Choose the **AzureSqlDatabase** linked service created previously to open its definition.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613508740/Blog%20Images/Data-Factory-Policies/SQL_LS.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/image/upload/f_auto,q_auto,c_thumb,w_500,g_face/v1613508740/Blog%20Images/Data-Factory-Policies/SQL_LS.png" title="Linked Service Details" >}}

2. Edit the definitions of this linked service and click on the **Save** button.
    - **Azure Key Vault** instead of Connection string
    - For **AKV linked service** select the one created in the step above
    - For **Secret name** insert a random value such as *AzureSqlDBConnectionString* (this would be the name of the secret created in Key Vault)

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613508740/Blog%20Images/Data-Factory-Policies/Edit_Azure_SQLDB_LS.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,c_thumb,w_500,g_face/v1613508740/Blog%20Images/Data-Factory-Policies/Edit_Azure_SQLDB_LS.png" title="Edit Azure SQL DB Linked Service Details" >}}

3. Click on the **Publish all** button to save the most recent changes.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613508740/Blog%20Images/Data-Factory-Policies/Publish_All_2.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1613508740/Blog%20Images/Data-Factory-Policies/Publish_All_2.png" title="Publish All 2" >}}

### Check Linked Services Compliance

Going back to the Policy Compliance page, filtering by our assignment name, we can see that we now have **2** out of 4 resources compliant (again, this may take a few minutes before the evaluation runs again).

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613993434/Blog%20Images/Data-Factory-Policies/Compliance_Revised.png" title="Policy Compliance Revised" >}}

Checking the assignment details and filtering for compliant resources, the Azure SQL Database is now listed as a compliant resource and we know we're following the best practices for this particular resource. The Key Vault itself is also in a compliant state. In a real-world scenario, the same would need to be done for the other two linked services created and in a non-compliant state.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1613992453/Blog%20Images/Data-Factory-Policies/Compliant_Resource.png" title="Azure SQL DB Linked Service Compliant" >}}

# Create a custom ADF policy definition

A few days ago I came across a [topic](https://stackoverflow.com/questions/66117261/how-to-add-azure-custom-policy-for-azure-data-factory-to-only-use-azure-key-vaul/66150499) in Stack Overflow where someone was looking for a way to enforce the use of a Self-Hosted IR for Linked Services. This was the perfect opportunity to make use of this new feature for ADF and implement a custom policy to check for this requirement.

### Understanding Linked Services JSON definition

The first thing we need to look at is linked services JSON definition. We need to understand what to look for and where so we can come up with a custom policy. 

The official documentation is pretty clear when it comes to this and we can see [here](https://docs.microsoft.com/en-us/azure/data-factory/concepts-linked-services#linked-service-json) that the JSON definition of a linked service will contain a block with property **connectVia** when the linked service makes use of a self-hosted integration runtime.

{{< codeblock "Linked Service JSON definition" "JSON" >}}{
    "name": "<Name of the linked service>",
    "properties": {
        "type": "<Type of the linked service>",
        "typeProperties": {
              "<data store or compute-specific type properties>"
        },
        "connectVia": {
            "referenceName": "<name of Integration Runtime>",
            "type": "IntegrationRuntimeReference"
        }
    }
}
{{< /codeblock >}}

### Build the policy definition

The custom policy definition can be found on this [link](https://github.com/pfiadeiro/azure-content/blob/master/azure-policies/DataFactory/audit-integration-runtimes/azurepolicy.json). Let's go through the different components of it.

We'll start with the properties **displayName**, **description**, **mode** and **metadata**. The first 2 are used to identify the policy and give some context about it.  The **mode** tag defines which resources are evaluated by the policy definition, with the 2 options being ***all*** and ***indexed***. 

The **metadata** tag provides option information about the policy definition such as **category** which will detemrine under which category in the Azure portal the policy definition is displayed.

{{< codeblock "General Definition" "JSON" >}}
"displayName":"Azure Data Factory should use Self-Hosted Integration Runtimes for Linked Services definitions",
"description":"All linked services created in Data Factory should connect through a Self-Hosted Integration Runtime when possible",
"mode":"All",
"metadata": {
   "category": "Data Factory"
},
{{< /codeblock >}}

The next block we're going to look at is the **parameters** section. This policy definition will contain 2 parameters: one to define the type of effect used by the policy and the other to define a list of linked services types to be considered for policy evaluation. This last parameter is important because not all linked services support an integration runtime choice, Key Vault being one example of that.

For each parameter we need to define a **name** (effect for example), its **type**, **metadata** containing information like **displayName** or **description** and, optionally, a **defaultValue** and a list of **allowedValues**.

{{< codeblock "Parameters" "JSON" >}}"parameters": {
  "effect": {
    "type": "String",
    "metadata": {
      "displayName": "Effect",
      "description": "Enable or disable the execution of the policy"
    },
    "allowedValues": [
      "Audit",
      "Deny",
      "Disabled"
    ],
    "defaultValue": "Audit"
  },
  "allowedLinkedServiceResourceTypes": {
    "type": "Array",
    "metadata": {
      "displayName": "Linked services types to check for self-hosted IR", 
      "description": "This parameter should contain the list of all possible types of linked services to check for the use of self-hosteIR."
    },
    "allowedValues": [
      "AzureBlobFS",
      "AzureBlobStorage",
      "AzureSqlDatabase",
      "Oracle",
      "PostgreSql"
    ]
 }
{{< /codeblock >}}

As you can see the policy definition only consider five type of linked services. If you need to add more types to the policy definition, you can easily see the type value of a linked service by checking its JSON definition on the Linked services page in ADF.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614009968/Blog%20Images/Data-Factory-Policies/LS_JSON_Definition.png" title="Linked Service JSON Definition" >}}

Finally, we have the **policyRule** block where we define which conditions need to be true for the policy to be enforced.

There are 3 conditions that need to be true:
- the resource type needs to be a **linked service**
- the property **connectVia** won't exist in the JSON definition
- the linked service type is within one of the types selected when assigning the policy

{{< codeblock "Parameters" "JSON" >}}"policyRule": {
  "if": {
    "allOf": [
      {
        "field": "type",
        "equals": "Microsoft.DataFactory/factories/linkedservices"
      },
      {
        "field": "Microsoft.DataFactory/factories/linkedservices/connectVia",
        "exists": "false"
      },
      {
        "field": "Microsoft.DataFactory/factories/linkedservices/type",
        "in": "[parameters('allowedLinkedServiceResourceTypes')]"
      }
    ]
  },
  "then": {
    "effect": "[parameters('effect')]"
  }
}
{{< /codeblock >}}

More information about a policy definition structure can be found [here](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/definition-structure).

### Create a self-hosted integration runtime in Data Factory

As listed on the prerequisites section, you'll need a self-hosted IR to test this example. If you followed the links available on that section, you should have a self-hosted integration runtime created in ADF looking like this.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614012722/Blog%20Images/Data-Factory-Policies/SelfHosted_IR.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1614012722/Blog%20Images/Data-Factory-Policies/SelfHosted_IR.png" title="Self-Hosted Integration Runtime" >}}

If you didn't have the possibility of creating a self-hosted IR, just follow along and the explanation should be clear.

### Create a policy definition in the Portal

We'll now go back to the Azure Portal and the Policy Definitions page where we'll select the option **+ Policy definition**

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614013686/Blog%20Images/Data-Factory-Policies/New_Policy_Definition.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1614013686/Blog%20Images/Data-Factory-Policies/New_Policy_Definition.png" title="New Policy Definition" >}}

In the New Policy definition page, you should choose the **Definition location** as the subscription you're using ((it has to be a subscription or a management group), give a **Name** to the policy and in the **Category** option choose **Use existing** and select **Data Factory** from the drop-down box.

In the **Policy Rule** copy the JSON code available in [GitHub](https://github.com/pfiadeiro/azure-content/blob/master/azure-policies/DataFactory/audit-integration-runtimes/azurepolicy.json) and paste it there. Once all this is done, click on the option **Save**

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614013686/Blog%20Images/Data-Factory-Policies/Policy_Definition_Details1.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1614013686/Blog%20Images/Data-Factory-Policies/Policy_Definition_Details1.png" title="Policy Definition 1" >}}
{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614013686/Blog%20Images/Data-Factory-Policies/Policy_Definition_Details2.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1614013686/Blog%20Images/Data-Factory-Policies/Policy_Definition_Details2.png" title="Policy Definition 2" >}}

Back in the Policy Definitions page, filtering by **Type** custom, you'll see the new policy definition created.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614013687/Blog%20Images/Data-Factory-Policies/Custom_Policy.png"  title="Custom Policies" >}}

### Assign the Custom Policy Definition

We now need to create an assignment with the custom policy definition to evaluate the state of our resources against that policy.

1. Go back to the Policy **Definitions** page, select the **Type** custom and click on the **Name** of the custom policy definition created in the previous section.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614013687/Blog%20Images/Data-Factory-Policies/Custom_Policy.png"  title="Custom Policies" >}}

2. Select the **Assign** option

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614033620/Blog%20Images/Data-Factory-Policies/Assign_Custom_Policy.png"  title="Assign Custom Policy" >}}

3. In the Assign Policy page the only required changes are:
   - **Scope** tab:
     - ***Scope***: Select the resource group you created in the prerequisites section
   - **Parameters** tab:
     - ***Effect***: Audit (default value) 
   - **Non-compliance messages** tab:
     - ***Non-compliance message***: Insert message that will show when resource isn't compliant
   - **Review + create** tab:
     - Click on the **Create** button

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614017089/Blog%20Images/Data-Factory-Policies/Custom_Policy_Basics_Tab.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1614017089/Blog%20Images/Data-Factory-Policies/Custom_Policy_Basics_Tab.png" title="Custom Policy Basics tab" >}}
{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614015737/Blog%20Images/Data-Factory-Policies/Custom_Policy_Parameters_Tab.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1614015737/Blog%20Images/Data-Factory-Policies/Custom_Policy_Parameters_Tab.png" title="Custom Policy Parameters tab" >}}
{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614015841/Blog%20Images/Data-Factory-Policies/Custom_Policy_NonCompliance_Tab.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto,w_550/v1614015841/Blog%20Images/Data-Factory-Policies/Custom_Policy_NonCompliance_Tab.png" title="Custom Policy Non-compliance messages tab" >}}
{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614015737/Blog%20Images/Data-Factory-Policies/Custom_Policy_ReviewCreate_Tab.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1614015737/Blog%20Images/Data-Factory-Policies/Custom_Policy_ReviewCreate_Tab.png" title="Custom Policy Review + Create tab" >}}

4. Selecting the option **Compliance** we can see how many resources are compliant with our new assignment (always taking into consideration it may take a few minutes for the policy to run). **3** out of 4 possible resources are in a non-compliant state which is the expected result, none of our linked services is making use of a self-hosted IR except for the Key Vault linked service which doesn't qualify to be evaluated by this policy.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614029895/Blog%20Images/Data-Factory-Policies/Custom_Policy_Compliance.png" title="Custom Policy Compliance" >}}

5. Checking the assignment details by clicking on its name, we have the 3 linked services not compliant with this policy listed. If we click on the **Details** link, we can see our non-compliance message.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614023200/Blog%20Images/Data-Factory-Policies/Custom_Policy_Compliance_Details.png" title="Custom Policy Compliance Details" >}}


### Change one Linked Service to use a self-hosted integration runtime

Let's now edit the definition of one of our Linked Services to see if its compliance state changes.

1. Back in the ADF Linked Services page, click on the **Oracle** linked service

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614016695/Blog%20Images/Data-Factory-Policies/Edit_Oracle_LS.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1614016695/Blog%20Images/Data-Factory-Policies/Edit_Oracle_LS.png" title="Edit Oracle Linked Service" >}}

2. In the option **Connect via integration runtime**, we're going to change this value to make use of the recently created self-hosted IR. Because we changed a property, we need to input a random password again. Click the **Apply** button after making these 2 changes.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614016695/Blog%20Images/Data-Factory-Policies/Edit_Oracle_LS_Details.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1614016695/Blog%20Images/Data-Factory-Policies/Edit_Oracle_LS_Details.png" title="Edit Oracle Linked Service Details" >}}

3. If we look at the JSON definition of the Oracle Linked Service, we can see that it now included the property **connectVia** as expected.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614016695/Blog%20Images/Data-Factory-Policies/Oracle_LS_JSON.png" thumbnail="https://res.cloudinary.com/pedrofiadeiro/c_thumb,w_550,f_auto,q_auto/v1614016695/Blog%20Images/Data-Factory-Policies/Oracle_LS_JSON.png" title="Oracle Linked Service JSON Definition" >}}

4. Waiting a few more minutes for the policy evaluation to run again, we now have **2** out of 4 resources in a compliant state and by clicking on the assignment name, we can look at the details page and see that the Oracle linked service is now in a compliant state.

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614028986/Blog%20Images/Data-Factory-Policies/Custom_Policy_Compliance_2.png"  title="Custom Policy Compliance Re-evaluated" >}}

{{< image classes="fancybox center fig-100" src="https://res.cloudinary.com/pedrofiadeiro/f_auto,q_auto/v1614028986/Blog%20Images/Data-Factory-Policies/Custom_Policy_Compliance_Details_2.png" title="Custom Policy Compliance Details Re-evaluated" >}}

# Wrapping Up

On this blog post you saw how you can make use of the new Azure Policy definitions for Data Factory and also I how to create custom policy definitions that can be applied to Data Factory resources.

Don't forget to delete any resources you've created to follow this blog post if you no longer need them.

I hope this post was useful, thanks for reading!