# Dynamically train and score Form Recognizer model to extract key-value pairs from different type of forms at scale using REST API with Python

In my first blog about the automated form processing, I described how you can extract key-value pairs from your forms in real-time using the Azure Form Recognizer cognitive service. We successfully implemented that solution for many customers. 
In this part 2 of the blog about the Form Recognizer Cognitive service, we will discuss common requirements across many customers we came across and how we address them as the product itself evolves. 
Often, after a successful PoC or MVP, our customers realize that, not only they need this real time solution to process their forms but, they also have a huge backlog of forms they would like to ingest into their relational or NoSQL databases, in a batch fashion. In the backlog, they different types of forms and they don’t want to build a model for each type.
In this blog, we’ll describe how to automatically (re)train existing models or create new ones extract the key-value pairs of off different type of forms and at scale.

# Business Problem

Most organizations are now aware of how valuable the data they have in different format (pdf, images, videos…) in their closets are. They are looking for best practices and most cost-effective ways and tools to digitize those assets.  By extracting the data from those forms and combining it with existing operational systems and data warehouses, they can build powerful AI and ML models to get insights from it to deliver value to their customers and business users.
With the [Form Recognizer Cognitive Service](https://docs.microsoft.com/en-us/azure/cognitive-services/form-recognizer/overview), we help organizations to harness their data, automate processes (invoice payments, tax processing …), save money and time and get better accuracy.

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/BusinessProblem.png)

In this section, we’ll describe how to dynamically ingest millions of forms, of different types using Azure Services.
Your backlog of forms might be in your on-premise environment or in a (s)FTP server. We assume that you were able to upload them into an Azure Data Lake Store Gen 2, using [Azure Data Factory](https://docs.microsoft.com/en-us/azure/data-factory/quickstart-create-data-factory-portal), [Storage Explorer](https://docs.microsoft.com/en-us/azure/vs-azure-tools-storage-manage-with-storage-explorer?tabs=windows) or [AzCopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-blobs). Therefore, the solution we’ll describe here will focus on the data ingestion from the data lake to the (No)SQL database.
Our product team published a great tutorial on how to [Train a Form Recognizer model and extract form data by using the REST API with Python](https://docs.microsoft.com/en-us/azure/cognitive-services/form-recognizer/quickstarts/python-train-extract?tabs=v2-0). The solution described here demonstrates the approach for one model and one type of forms. Often, our customers have different type of forms coming from their many clients and customers. The value-add of the post is show you how to automatically train a model with new and different type of forms using a meta-data driven approach. If you do not have an existing model, the program will create one for you and give you the model id.
The REST AP requires some parameters as input. For security reasons, some of these parameters will be store in Azure Key Vault and others, less sensitive, like the blob folder name, will be in a parametrization table in an Azure SQL DB. 
For each form type, Data engineers or data scientists will populate the param table. Will use Azure data factory to iterate over the list of forms type and pass the relevant parameters to an Azure Databricks notebook to (re)train the model.


![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/HighLevelArchitecture.png)

# Below is the list of parameters:

## In Azure Key Vault
* CognitiveServiceEndpoint: The endpoint of the form recognizer cognitive service. This value will be stored in Azure Key Vault for security reasons.
* CognitiveServiceSubscriptionKey: The access key of the cognitive service. This value will be stored in Azure Key Vault for security reasons. The below screenshot shows how to get the key and endpoint of the cognitive service


![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/CognitiveService.png)

* StorageAccountName: The storage account where the training dataset and forms we want to extract the key value pairs from are stored. The two storage accounts can be different. The training dataset must be in the same container for all form types. They can be in different folders.
* StorageAccountSasKey : the shared access signature of the storage account
The below screen shows the key vault after you create all the secrets

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/KeyVault.png)

## In the parametrization table

* form_description: This field is not required as part of the training of the model the inference. It just to provide a description of the type of forms we are training the model for (example client A forms, Hotel B forms,...)
* training_container_name: This is the storage account container name where we store the training dataset. It can be the same as scoring_container_name
* training_blob_root_folder: The folder in the storage account where we’ll store the files for the training of the model. 
* scoring_container_name: This is the storage account container name where we store the files we want to extract the key value pairs from.  It can be the same as the training_container_name
* scoring_input_blob_folder: The folder in the storage account where we’ll store the files to extract key-value pair from.
model_id: The identify of model we want to retrain. For the first run, the value must be set to -1 to create a new custom model to train. The training notebook will return the newly created model id to the data factory and, using a stored procedure activity, we’ll update the meta data table with in the Azure SQL database.
Whenever you had a new form type, you need to reset the model id to -1 and retrain the model.
* file_type: The supported types are application/pdf, image/jpeg, image/png, image/tif. 
* form_batch_group_id : Over time, you might have multiple forms type you train against different models. The form_batch_group_id will allow you to specify all the form types that have been training using a specific model. 

# Pre-requisites

To implement this solution, you will need to create the below services: 
* Form Recognizer resource 
* Azure Data Factory
* Azure Databricks
* Azure SQL single database
* Azure Key Vault


## Create a Form Recognizer resource from the portal

Navigate to the Azure portal: portal.azure.com
In the top left menu, select create a resource.

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/CreateFormRecognizer1.png)

In the marketplace menu, enter Form Recognizer. The cognitive service creation page will appear. Click Create
Fill out the form with the usual information for the creation of an azure resource and click Create.

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/CreateFormRecognizer2.png)

As mentioned above, part of the parameters will be in a param table and the the rest in Azure Key Vault.

## Create a parametrization table.

Create a table in your Azure SQL DB using the below script


```
CREATE TABLE dbo.ParamFormRecogniser(
	form_description varchar(50) NULL,
  training_container_name varchar(50) NOT NULL,
	training_blob_root_folder varchar(50) NULL,
	scoring_container_name varchar(50) NOT NULL,
	scoring_input_blob_folder varchar(50) NOT NULL,
	scoring_output_blob_folder varchar(50) NOT NULL,
	model_id varchar(50) NULL,
	file_type varchar(50) NULL
) ON PRIMARY
GO
```

For the first run, the model id is set to -1. This value is updated using a stored procedure.

```
CREATE PROCEDURE [dbo].[update_model_id] ( @form_batch_group_id  varchar(50),@model_id varchar(50))
AS
BEGIN 
	UPDATE [dbo].[ParamFormRecogniser]   
		SET [model_id] = @model_id  
	WHERE form_batch_group_id =@form_batch_group_id
 END
```
## Create a key Vault
Create an Azure key vault to host the sensitive parameters: StorageAccountSasKey, StorageAccountName, CognitiveServiceEndpoint and CognitiveserviceSubscriptionKey.
Navigate to the Azure portal: portal.azure.com. 
In the top left menu, select create a resource. 


![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/CreateKeyVault1.png)

In the marketplace menu, enter Key Vault. In the new page, enter click Create.
Fill out the form with the usual information for the creation of an azure resource and click Create

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/CreateKeyVault2.png)

Navigate to the Key Vault resource after it is created and, in the settings section select secrets to add the parameters.

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/CreateKeyVault3.png)

A new window will appear, select Generate/import.
Enter the name of the parameter and its value and click create. Repeat this process for the SAS URL and the endpoint.

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/CreateKeyVault4.png)

# Train the model for different type of forms
## Create a notebook in Databricks

Create an Azure Databricks service the same way you created the Cognitive service and the key vault. 
Navigate to the Databricks after it has been created and launch the workspace. 

### Create a secret scope backed by Azure Key Vault
To reference the secrets in the Azure Key Vault we created above, you will need to create a secret scope in Azure Databricks. You can create a secret scope back by Azure Key Vault or backed by Azure Databricks. In the post, we used a secret backed by Azure Key Vault. The steps to create a secret scope are detailed [here](https://docs.microsoft.com/en-us/azure/databricks/security/secrets/secret-scopes#--create-an-azure-key-vault-backed-secret-scope).

### Create Databricks notebooks

This is where the magic happens. First, we’ll create a notebook to called Settings to assign the values in the Param table to variables. We could have two settings files, one for the training and another one for the inference but, in this blog, we’ll only have one for simplify the process. The values will be passed as parameters by Azure Data Factory. We’ll detail the approach in the orchestration section. We’ll also assign variables values read from the secrets the Key Vault.
To create the Settings notebook, click on the workspace button, in the new tab, click on the dropdown list and select create and then notebook.




