## Cloud functions interated with cloud storage in GCP

### 1. Introduction

This blog is used to express how to combine cloud functions to load files stored in storage into Bigquery.

Will cover bellow products in the rest of the blog:
- Cloud functions
- Cloud storage 
- Bigquery
- Pubsub


As we first face with `Cloud funtions`, this is a severless products that we could use for some use cases that don't need for the real compute resource and auto-scale, as the `Cloud functions` is a **event-driven** product, there will come with event-driven, but what is event-driven? What is an event? Then we put a file into a bucket, file added, that’s an event; when we delete a file, file changed, that’s an event; when we modify a file, file changed, that’s an event. We just need to pay for the resource we used. 


When we are in GCP, we could use the Cloud function to implement this event-driven logic. The base is we will put a file in the bucket, then we will trigger a Pubsub message, then Cloud function will monitor this Pubsub topic, if we get that info, the Cloud function logic will be triggered.

![Whole process steps](https://miro.medium.com/max/1750/1*22N8pdH4Oop1gn8ZIZoInw.png)

This is based on your own GCP project, just follow the below steps to make things happen, but if in real project that you could ignore many of them step, but here is used to make the step as completed as possible if you would like to hands-on work to try by yourself, so if you follow these steps face any problems so please raise your hands, I will try my best to solve them.

I highly recommend to use **Notebook** to implement the logic as we could get interactive actions with code, we could use our **Local jupyter notebook server** or we could use **cloud-hosted notebook** like [Colab research](https://colab.research.google.com/) by Google, it’s free! 

If you would like to use your own compute resource for the rest, so please do first install **GCP SDK** first based on your system with google is easy enough.


### 2. Install dependencies

Assumption: we will use **Python** code to implement the whole logic.

We will use `google-cloud-storage` and `google-cloud-bigquery` to do the data loading logic, also this is also based on that you have already installed `GCP SDK` in your local machine if you use `jupyter notebook`, if with `colab`, the `SDK` is already installed, you don't need to care about the SDK stuff. That's the reason that I recommend to use **Colab**.

First to ensure that we should have `google-cloud-storage` and `google-cloud-bigquery` to be installed first.

```
! pip install google-cloud-bigquery google-cloud-storage — quiet
```


### 3. Get Service account key

We need to get a `Service account` to get interaction with the `GCP` project as this is the authentication way to get access to your project, so the first step is that we need to apply a `SA` first. You could just follow the `SA` creation step.

#### 3.1 Create SA

In the `IAM` tab, click with the `service account` tab, then just click `create`, give a name, then grant the SA account with the proper power you want, here for simplicity, I just provide with **Owner** account for convinience.

![SA account](https://miro.medium.com/max/1750/1*P0t5P_5Cm3zmFsYJZ8bl4Q.png)


#### 3.2 Get SA key

After the account is created, then we need to get the `SA` account key with a `JSON` file, we will use this file to get interaction with `GCP` in fact, so please keep in mind to keep this file secretly.

![service account key](https://miro.medium.com/max/1750/1*KFBeyiNpqdj_Rab-0jhbig.png)


After we have already got the key, then we should first insert project information into our running time environment, that’s easy, just add an environment variable with `GOOGLE_APPLICATION_CREDENTIALS` with the file, that’s it.

```python
import os
from google.cloud import bigquery
# Get SA file name, this is based on the same folder that we run the code
sa_file_name = [x for x in os.listdir('.') if x.endswith('.json')][0]
os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = sa_file_name
```

### 4. Create a bucket and push notification for the bucket

As when we change the bucket file like `upload`, `change`, `delete` or other modifications in the bucket file, we know that there are some changes in the bucket, but how to notice **Cloud functions**? That’s **Pubsub** comes in.

When there is a modification, then we could create a message that contains some information that: oh, you have added a file called `data.csv` at what time by who, like this information. Then we will use **Cloud functions** to get the published message from that topic.


#### Tips

After we have already created the bucket, then we should create a `notification` from the bucket into a `topic`, so we could just use this command:
```
gsutil notification create -f json -t topic_name gs://bucket_name
```

You could reference this API link: [Bucket notification API](https://cloud.google.com/storage/docs/gsutil/commands/notification), if you want to know bucket notification logic, please find it [Here](https://cloud.google.com/storage/docs/pubsub-notifications).


### 5. Create datasets, table, and load data into BQ with cloud function

As we need to load the file into the bucket, so we could create the `dataset` and `table` that we need, we could both use the `portal` or `API`, here I just use `Python API` to do this.

After we have created the table, then we will use the cloud function to do the data loading logic.

Before we do anything, please do keep in mind that we should provide with a `requirements.txt` file that contains packages that you use for could functions, as cloud functions are event-driven, we don’t hold environment, so we need to tell the platform that we need these packages, please help me to install them.

```python
%%writefile requirements.txt
google-cloud-bigquery
google-cloud-storage
```

The last step is that we should implement the logic with Python code that we would like to use, I have uploaded my code into my Github, you could find the whole code from my [Github](https://github.com/lugq1990/GCP_tutorial), also you could find many other tutorials about GCP with hands-on python code.


Bellow is the main function that we will use to do the load file from GCS into BQ directly, the logic is as bellow:

1. Provide some basic information that use for project
2. Provide job information like: table schema; how to load data; file type and file delimiter etc.
3. Create dataset and table in case that they don't exist.
4. Load file into BQ directly with Bigquery API.

```
%%writefile main.py
from google.cloud import bigquery

project_name = "loyal-weaver-296802"
dataset_name = "cloud_dataset"
table_name = "iris"
bucket_name = "gcs_to_bq_cloud_func_new"
file_name = "data.csv"

# where is the file in bucket
file_uri = "gs://{}/{}".format(bucket_name, file_name)
# where to load the data into BQ
dataset_table_name = "{}.{}.{}".format(project_name, dataset_name, table_name)

# BQ client
client = bigquery.Client()

columns_name = ['a', 'b', 'c', 'd', 'label']
schema_list = [bigquery.SchemaField(x, 'float') for x in columns_name]


def create_dataset_and_table():
  """
  Create dataset and table
  """
  # create dataset and table
  try:
    dataset = client.create_dataset(dataset_name)
    print("dataset has been created!")
  except Exception as e:
    print("When try to create dataset: {} with error:{}".format(dataset_name, e))
    pass

  dataset = client.get_dataset(dataset_name)

  # table_ref = dataset.table(table_name)
  table = bigquery.Table(table_ref, schema=schema_list)

  # create table
  try:
    table = client.create_table(table, exists_ok=True)
    print("table:{} has been created.".format(table_name))
  except Exception as e:
    print("When create table with error:{}".format(e))


def load_gcs_file_into_bq_new(event, context):
  """
  Based on the pubsub topic to trigger cloud function to load
  files in bucket into bigquery directly.
  """
  # first to create dataset and table
  create_dataset_and_table()

  # config job config, this is whole thing that we need to config
  # during the loading step: where the file and where to put data,
  # and the file information, how to load the file: `append` or `truncate`
  job_config = bigquery.LoadJobConfig(
      schema=schema_list,
      skip_leading_rows=1,
      write_disposition=bigquery.WriteDisposition.WRITE_APPEND,
      source_format=bigquery.SourceFormat.CSV,
      field_delimiter=','
  )

  load_job = client.load_table_from_uri(file_uri, dataset_table_name, job_config=job_config)

  # wait to finish
  load_job.result() 

  print("Load action has finished without error")

  print("Whole step finished.")
```

### 6. Deploy cloud function

The last step is to deploy our code into the GCP project, here we could use `gcloud` command to deploy, you could reference the official web [Pubsub Trigger](https://cloud.google.com/functions/docs/calling/pubsub)

This is the command that we will use to deploy the function that we have written.

The parameters that we use here is `function name`, which version of python engine to use and the bucket we want to put files and trigger-event as what type of event that we want to follow.

```
gcloud functions deploy load_gcs_file_into_bq_new \
--runtime python37 \
--trigger-resource gcs_to_bq_cloud_func_new \
--trigger-event google.storage.object.finalize \
--allow-unauthenticated
```

### 7. Load file from bucket into BQ with cloud functions

The last step that we should take is to provide the file that we need to upload it into the bucket we created, we could use `GCP Portal` or use `gsutil` command to upload the file into that bucket directly(but we do need to ensure that SDK is installed in the computer(SDK install) or in Colab it’s installed already.)

```
! gsutil cp data.csv gs://gcs_to_bq_cloud_func_new/
```

#### Tips

Please do keep in mind that we should provide the right bucket name and the source file name in that bucket when write python code.


### 8. Check logs and dataset result

Alright, we have already deployed the code and moved a file into that bucket, but how about BigQuery? We could get the running log in GCP portal with Cloud functions, click the function we created and in **LOGS**, we could get the running status and result.

![log result](https://miro.medium.com/max/2188/1*Uc8HBiGVRhAtw9WIAasfEw.png)

In the logs that we found that the function executed finished without error. But how about the BigQuery table result? If would like to write less code, then just in the GCP portal BigQuery component, just query that table.

![dataset result](https://miro.medium.com/max/2188/1*VRA3zbJu4t2AcRB6WIXt4g.png)

Good news!

Final words, this tutorial is based on the cloud function to load files in bucket into Bigquery, as this is a less expensive and event-driven process, so this is just a demonstration, for real project we just need to focus on the logic that we need to implement.

If you think this has helped you a bit, that would be enough!!! You could find this full blog [here](https://github.com/lugq1990/sharing_blogs/blob/main/cloud_function_with_cloud_storage.md)

If you would love to learn more about GCP hands-on in Python, please check my [Github repo](https://github.com/lugq1990/GCP_tutorial). Hope Python coding.