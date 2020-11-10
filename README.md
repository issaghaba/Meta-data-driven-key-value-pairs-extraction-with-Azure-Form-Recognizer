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

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/adbnotebook1.png)

In the popup, enter the name you want to give to the notebook and select Python as default language. If you have a Databricks cluster running you can select it otherwise, leave it empty for now. We’ll create one later. Click Create.

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/adbnotebook2.png)

In the first cell, we’ll retrieve the parameters passed by Azure Data Factory

```
dbutils.widgets.text("form_batch_group_id", "","")
dbutils.widgets.get("form_batch_group_id")
form_batch_group_id = getArgument("form_batch_group_id")

dbutils.widgets.text("model_id", "","")
dbutils.widgets.get("model_id")
model_id = getArgument("model_id")

dbutils.widgets.text("training_container_name", "","")
dbutils.widgets.get("training_container_name")
training_container_name = getArgument("training_container_name")

dbutils.widgets.text("training_blob_root_folder", "","")
dbutils.widgets.get("training_blob_root_folder")
training_blob_root_folder=  getArgument("training_blob_root_folder")

dbutils.widgets.text("scoring_container_name", "","")
dbutils.widgets.get("scoring_container_name")
scoring_container_name=  getArgument("scoring_container_name")

dbutils.widgets.text("scoring_input_blob_folder", "","")
dbutils.widgets.get("scoring_input_blob_folder")
scoring_input_blob_folder=  getArgument("scoring_input_blob_folder")


dbutils.widgets.text("file_type", "","")
dbutils.widgets.get("file_type")
file_type = getArgument("file_type")

dbutils.widgets.text("file_to_score_name", "","")
dbutils.widgets.get("file_to_score_name")
file_to_score_name=  getArgument("file_to_score_name")

```
In the second cell, we retrieve secrets from Key Vault and assign them to some variables.

```
cognitive_service_subscription_key = dbutils.secrets.get(scope = "FormRecognizer_SecretScope", key = "CognitiveserviceSubscriptionKey")
cognitive_service_endpoint = dbutils.secrets.get(scope = "FormRecognizer_SecretScope", key = "CognitiveServiceEndpoint")

training_storage_account_name = dbutils.secrets.get(scope = "FormRecognizer_SecretScope", key = "StorageAccountName")
storage_account_sas_key= dbutils.secrets.get(scope = "FormRecognizer_SecretScope", key = "StorageAccountSasKey")  

ScoredFile = file_to_score_name+ "_output.json"
training_storage_account_url="https://"+training_storage_account_name+".blob.core.windows.net/"+training_container_name+storage_account_sas_key 
```

Now that we completed the Settings notebook, let’s create a notebook to train the model. As mentioned above, we will use files stored in a folder in an Azure Data Lake Gen 2 storage account (training_blob_root_folder). The folder name is passed as a variable. Each set of form types will be in the same folder and, as we loop over the parameter table, we’ll train the model using all of the form types.
Let’s create a new notebook and call it TrainFormRecognizer for example.
The first step is to execute the Settings notebook.

```
%run "./Settings"
```

In the next cell, well assign variables from the Settings file, and dynamically train the model for each form type leveraging the code publish in the this [Quickstart](https://docs.microsoft.com/en-us/azure/cognitive-services/form-recognizer/quickstarts/python-train-extract?tabs=v2-0#get-training-results%20) by our engineering team

```
import json
import time
from requests import get, post

post_url = cognitive_service_endpoint + r"/formrecognizer/v2.0/custom/models"
source = training_storage_account_url
prefix= training_blob_root_folder

includeSubFolders=True
useLabelFile=False
headers = {
    # Request headers
    'Content-Type': file_type,
    'Ocp-Apim-Subscription-Key': cognitive_service_subscription_key,
}
body = 	{
    "source": source
    ,"sourceFilter": {
        "prefix": prefix,
        "includeSubFolders": includeSubFolders
   },
}
if model_id=="-1": # if you don't already have a model you want to retrain. In this case, we create a model and use it to extract the key-value pairs
  try:
      resp = post(url = post_url, json = body, headers = headers)
      if resp.status_code != 201:
          print("POST model failed (%s):\n%s" % (resp.status_code, json.dumps(resp.json())))
          quit()
      print("POST model succeeded:\n%s" % resp.headers)
      get_url = resp.headers["location"]
      model_id=get_url[get_url.index('models/')+len('models/'):]     
      
  except Exception as e:
      print("POST model failed:\n%s" % str(e))
      quit()
else :# if you already have a model you want to retrain, we reuse it and (re)train with the new form types.  
  try:
    get_url =post_url+r"/"+model_id
      
  except Exception as e:
      print("POST model failed:\n%s" % str(e))
      quit()
```
The final step in the training process is to get the training result in a json format.

```
n_tries = 10
n_try = 0
wait_sec = 5
max_wait_sec = 5
while n_try < n_tries:
    try:
        resp = get(url = get_url, headers = headers)
        resp_json = resp.json()
        print (resp.status_code)
        if resp.status_code != 200:
            print("GET model failed (%s):\n%s" % (resp.status_code, json.dumps(resp_json)))
            n_try += 1
            quit()
        model_status = resp_json["modelInfo"]["status"]
        print (model_status)
        if model_status == "ready":
            print("Training succeeded:\n%s" % json.dumps(resp_json))
            n_try += 1
            quit()
        if model_status == "invalid":
            print("Training failed. Model is invalid:\n%s" % json.dumps(resp_json))
            n_try += 1
            quit()
        # Training still running. Wait and retry.
        time.sleep(wait_sec)
        n_try += 1
        wait_sec = min(2*wait_sec, max_wait_sec)     
        print (n_try)
    except Exception as e:
        msg = "GET model failed:\n%s" % str(e)
        print(msg)
        quit()
print("Train operation did not complete within the allocated time.")

```

## Extract key-value pairs from multiple forms in batch mode
Now that we’ve trained the model for all of our forms, the next steps are to score the different forms you have using the trained model. Your form can be on premise or in a storage account in Azure (ADLS or blob storage). In this post, we’ll use a data lake gen2 storage account. We’ll mount the storage account in Databricks and refer to the mount during the inferencing process.

Just like for the training of the forms, we’ll use Azure Data Factory to invoke the extraction of the key-value pairs from the forms. We’ll loop over the forms in the folders specified in the param table. We’ll also specify the degree of parallelism during the scoring. 

Let’s create the create the notebook to mount the storage account in Databricks. We’ll call it MountAdls 😉.
You will need to call the settings notebook in the mount Adls notebook as well, to set the variable.

```
%run "./Settings"
```

In the second cell, we’ll define variables (storage account, SAS key…). As this is sensitive information, we’ll retrieve them from our Key Vault secrets.

```
cognitive_service_subscription_key = dbutils.secrets.get(scope = "FormRecognizer_SecretScope", key = "CognitiveserviceSubscriptionKey")
cognitive_service_endpoint = dbutils.secrets.get(scope = "FormRecognizer_SecretScope", key = "CognitiveServiceEndpoint")

scoring_storage_account_name = dbutils.secrets.get(scope = "FormRecognizer_SecretScope", key = "StorageAccountName")
scoring_storage_account_sas_key= dbutils.secrets.get(scope = "FormRecognizer_SecretScope", key = "StorageAccountSasKey")  

scoring_mount_point = "/mnt/"+scoring_container_name 
scoring_source_str = "wasbs://{container}@{storage_acct}.blob.core.windows.net/".format(container=scoring_container_name, storage_acct=scoring_storage_account_name) 
scoring_conf_key = "fs.azure.sas.{container}.{storage_acct}.blob.core.windows.net".format(container=scoring_container_name, storage_acct=scoring_storage_account_name)
```
Next, we’ll try to unmount the storage account in case it was previously mounted.


```
try:
  dbutils.fs.unmount(scoring_mount_point) # Use this to unmount as needed
except:
  print("{} already unmounted".format(scoring_mount_point))
  
```
Finally, we’ll mount the storage account.

```
try: 
  dbutils.fs.mount( 
    source = scoring_source_str, 
    mount_point = scoring_mount_point, 
    extra_configs = {scoring_conf_key: scoring_storage_account_sas_key} 
  ) 
except Exception as e: 
  print("ERROR: {} already mounted. Run previous cells to unmount first".format(scoring_mount_point))

```
Note that we only mounted the training storage account as, in this case, the training and the files we want to extract key-value pairs are in the same storage account. If your scoring and training storage accounts are different, you will have to mount the two storage accounts.
Now, we can create a scoring notebook. Similarly, to the training notebook, we will use files stored in folders in the Azure Data Lake Gen 2 storage account we just mounted. The folder name is passed as a variable. We will loop over all the forms in the specified folder and extract the key-value pairs from its content.
Let’s create a new notebook and call it ScoreFormRecognizer for example.
The first step is to execute the Settings and the MountAdls.

```
%run "./Settings"
%run "./00_MountAdls"
```
```
########### Python Form Recognizer Async Analyze #############
import json
import time
from requests import get, post 


#prefix= TrainingBlobFolder
post_url = cognitive_service_endpoint + "/formrecognizer/v2.0/custom/models/%s/analyze" % model_id
source = r"/dbfs/mnt/"+scoring_container_name+"/"+scoring_input_blob_folder+"/"+file_to_score_name
output = r"/dbfs/mnt/"+scoring_container_name+"/scoringforms/ExtractionResult/"+os.path.splitext(os.path.basename(source))[0]+"_output.json"

params = {
    "includeTextDetails": True
}

headers = {
    # Request headers
    'Content-Type': file_type,
    'Ocp-Apim-Subscription-Key': cognitive_service_subscription_key,
}

with open(source, "rb") as f:
    data_bytes = f.read()

try:
    resp = post(url = post_url, data = data_bytes, headers = headers, params = params)
    if resp.status_code != 202:
        print("POST analyze failed:\n%s" % json.dumps(resp.json()))
        quit()
    print("POST analyze succeeded:\n%s" % resp.headers)
    get_url = resp.headers["operation-location"]
except Exception as e:
    print("POST analyze failed:\n%s" % str(e))
    quit() 
 ```
 In the next cell, we’ll get the results of the key-value pair extraction. The first cell will output the result in the Databricks notebook but, as we want the result in a json file to process further into Azure SQL Database, or Cosmos DB, we’ll write the result in a file. The output file name will be the name of the scored file concatenated with “_output.json". The file will be stored in the same folder as the source file.

 ```
n_tries = 10
n_try = 0
wait_sec = 2
max_wait_sec = 6
while n_try < n_tries:
    try:
        resp = get(url = get_url, headers = {"Ocp-Apim-Subscription-Key": cognitive_service_subscription_key})
        resp_json = resp.json()
        if resp.status_code != 200:
            print("GET analyze results failed:\n%s" % json.dumps(resp_json))
            n_try += 1
            quit()
        status = resp_json["status"]
        if status == "succeeded":
            print("Analysis succeeded:\n%s" % json.dumps(resp_json))
            n_try += 1
            quit()
        if status == "failed":
            print("Analysis failed:\n%s" % json.dumps(resp_json))
            n_try += 1
            quit()
        # Analysis still running. Wait and retry.
        time.sleep(wait_sec)
        n_try += 1
        wait_sec = min(2*wait_sec, max_wait_sec)     
    except Exception as e:
        msg = "GET analyze results failed:\n%s" % str(e)
        print(msg)
        n_try += 1
        print("Analyze operation did not complete within the allocated time.")
        quit()
 ```
  ```
  import requests
file = open(output, "w")
file.write(str(resp_json))
file.close()
 
   ```
# Training and scoring orchestration with Azure Data Factory
At this point, the only remaining part is the Data Factory for the orchestration of the training and the scoring of cognitive service model. The steps to create a Data Factory are described [here](https://docs.microsoft.com/en-us/azure/data-factory/quickstart-create-data-factory-portal#create-a-data-factory).
After you create the ADF resource, you will need to create three pipelines: one for the training and two for the scoring. We’ll explain later why we need two pipelines for the scoring.
## Training pipeline

The first activity in the training pipeline is a Lookup to read and return the values in the parametrization table in the Azure SQL database.  As all the training datasets will be in the same storage account and container, potentially different folders, we’ll keep the default value First row only attribute in the lookup activity settings. 
For each type of form to train the model against, we’ll train the model using all the files in the training_blob_root_folder.  

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/ParamTable.png)


![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/TrainingPipeline.png)

The stored procedure takes two parameters : model_id and the form_batch_group_id. The code to return the model id from the databricks notebook is dbutils.notebook.exit(model_id) and the code to read the code in stored procedure activity in data factory is @activity('GetParametersFromMetadaTable').output.firstRow.form_batch_group_id

## Scoring pipelines

To extract the key value pairs, we’ll scan all the folders in the parametrization table and, for each folder, we’ll extract the key-value pairs of all the files in it. As of today, ADF does not support nested ForEach Loops. The workaround is to create two pipelines. The first will do the Lookup from the parametrization table and pass the folders list as parameter to the second pipeline. 

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/ScoringPipeline1.png)

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/ScoringPipeline2.png)

The second pipeline will use a GetMeta activity to get the list of the files in the folder and pass it as a parameter to the scoring Databricks notebook we created earlier.

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/ScoringPipeline3.png)

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/ScoringPipeline4.png)

# Degree of parallelism

In both the training and scoring pipelines, specify the degree of parallelism to process multiple forms simultaneously. 

To set the degree of parallelism in the ADF pipeline : 
* select the Foreach activity
* uncheck the “sequential box
* set the degree of parallelism in the batch count text box.

![alt text](https://github.com/issaghaba/FormRecognizer/blob/main/images/DegreOfParallelism.png)

We recommend a maximum batch count of 15 for the scoring. 

Congratulations! You can now digitize all your backlog of form and run some analytics on top of it. 

