---
title: "Accessing and Managing Data in S3 with SageMaker Notebooks"
teaching: 20
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions 

- How can I load data from S3 into a SageMaker notebook?
- How do I monitor storage usage and costs for my S3 bucket?
- What steps are involved in pushing new data back to S3 from a notebook?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Read data directly from an S3 bucket into memory in a SageMaker notebook.
- Check storage usage and estimate costs for data in an S3 bucket.
- Upload new files from the SageMaker environment back to the S3 bucket.

::::::::::::::::::::::::::::::::::::::::::::::::

#### Directory setup
Let's make sure we're starting in the root directory of this instance, so that we all have our AWS_helpers.py file located in the same path (/test_AWS/scripts/AWS_helpers.py)


```python
%cd /home/ec2-user/SageMaker/
!pwd
```

    /home/ec2-user/SageMaker
    /home/ec2-user/SageMaker


## 1A. Read data from S3 into memory
Our data is stored on an S3 bucket called 'titanic-dataset-test'. We can use the following code to read data directly from S3 into memory in the Jupyter notebook environment, without actually downloading a copy of train.csv as a local file.



```python
import boto3
import pandas as pd
import sagemaker
from sagemaker import get_execution_role

# Initialize the SageMaker role and session
# Define the SageMaker role and session
role = sagemaker.get_execution_role()
session = sagemaker.Session()
s3 = boto3.client('s3')

# Define the S3 bucket and object key
bucket_name = 'titanic-dataset-test'  # replace with your S3 bucket name

# Read the train data from S3
key = 'data/titanic_train.csv'  # replace with your object key
response = s3.get_object(Bucket=bucket_name, Key=key)
train_data = pd.read_csv(response['Body'])

# Read the test data from S3
key = 'data/titanic_test.csv'  # replace with your object key
response = s3.get_object(Bucket=bucket_name, Key=key)
test_data = pd.read_csv(response['Body'])

# check shape
print(train_data.shape)
print(test_data.shape)

# Inspect the first few rows of the DataFrame
train_data.head()
```

    sagemaker.config INFO - Not applying SDK defaults from location: /etc/xdg/sagemaker/config.yaml
    sagemaker.config INFO - Not applying SDK defaults from location: /home/ec2-user/.config/sagemaker/config.yaml
    (712, 12)
    (179, 12)



## 1B. Download copy into notebook environment
If you have larger dataset (> 1GB), you may want to skip this step and always read directly into memory. However, for smaller datasets, it can be convenient to have a "local" copy (i.e., one stored in your notebook's instance).

Download data from S3 to notebook environment. You may need to hit refresh on the file explorer panel to the left to see this file. If you get any permission issues...

* check that you have selected the appropriate policy for this notebook
* check that your bucket has the appropriate policy permissions


```python
# Define the S3 bucket and file location
file_key = "data/titanic_train.csv"  # Path to your file in the S3 bucket
local_file_path = "./titanic_train.csv"  # Local path to save the file

# Initialize the S3 client and download the file
s3 = boto3.client("s3")
s3.download_file(bucket_name, file_key, local_file_path)
print("File downloaded:", local_file_path)
```

    File downloaded: ./titanic_train.csv


## 2. Check current size and storage costs of bucket

It's a good idea to periodically check how much storage you have used in your bucket. You can do this from a Jupyter notebook in SageMaker by using the **Boto3** library, which is the AWS SDK for Python. This will allow you to calculate the total size of objects within a specified bucket. Here’s how you can do it...

### Step 1: Set Up the S3 Client and Calculate Bucket Size


```python
# Initialize the total size counter
total_size_bytes = 0

# List and sum the size of all objects in the bucket
paginator = s3.get_paginator('list_objects_v2')
for page in paginator.paginate(Bucket=bucket_name):
    for obj in page.get('Contents', []):
        total_size_bytes += obj['Size']

# Convert the total size to gigabytes for cost estimation
total_size_gb = total_size_bytes / (1024 ** 3)
# print(f"Total size of bucket '{bucket_name}': {total_size_gb:.2f} GB") # can uncomment this if you want GB reported

# Convert the total size to megabytes for readability
total_size_mb = total_size_bytes / (1024 ** 2)
print(f"Total size of bucket '{bucket_name}': {total_size_mb:.2f} MB")
```

    Total size of bucket 'titanic-dataset-test': 41.04 MB


We have added this code to a helper called `get_s3_bucket_size(bucket_name)` for your convenience. You can call this function via the below code.


```python
import test_AWS.scripts.AWS_helpers as helpers # test_AWS.scripts.AWS_helpers reflects path leading up to AWS_helpers.py

helpers.get_s3_bucket_size(bucket_name)
```




    {'size_mb': 41.043779373168945, 'size_gb': 0.0400818157941103}



### Explanation

1. **Paginator**: Since S3 buckets can contain many objects, we use a paginator to handle large listings.
2. **Size Calculation**: We sum the `Size` attribute of each object in the bucket.
3. **Unit Conversion**: The size is given in bytes, so dividing by `1024 ** 2` converts it to megabytes (MB).

> **Note**: If your bucket has very large objects or you want to check specific folders within a bucket, you may want to refine this code to only fetch certain objects or folders.

## 3: Check storage costs of bucket
To estimate the storage cost of your Amazon S3 bucket directly from a Jupyter notebook in SageMaker, you can use the following approach. This method calculates the total size of the bucket and estimates the monthly storage cost based on AWS S3 pricing.

**Note**: AWS S3 pricing varies by region and storage class. The example below uses the S3 Standard storage class pricing for the US East (N. Virginia) region as of November 1, 2024. Please verify the current pricing for your specific region and storage class on the [AWS S3 Pricing page](https://aws.amazon.com/s3/pricing/).




```python
# AWS S3 Standard Storage pricing for US East (N. Virginia) region
# Pricing tiers as of November 1, 2024
first_50_tb_price_per_gb = 0.023  # per GB for the first 50 TB
next_450_tb_price_per_gb = 0.022  # per GB for the next 450 TB
over_500_tb_price_per_gb = 0.021  # per GB for storage over 500 TB

# Calculate the cost based on the size
if total_size_gb <= 50 * 1024:
    # Total size is within the first 50 TB
    cost = total_size_gb * first_50_tb_price_per_gb
elif total_size_gb <= 500 * 1024:
    # Total size is within the next 450 TB
    cost = (50 * 1024 * first_50_tb_price_per_gb) + \
           ((total_size_gb - 50 * 1024) * next_450_tb_price_per_gb)
else:
    # Total size is over 500 TB
    cost = (50 * 1024 * first_50_tb_price_per_gb) + \
           (450 * 1024 * next_450_tb_price_per_gb) + \
           ((total_size_gb - 500 * 1024) * over_500_tb_price_per_gb)

print(f"Estimated monthly storage cost: ${cost:.4f}")


```

    Estimated monthly storage cost: $0.0009


For your convenience, we have also added this code to a helper function.


```python
import test_AWS.scripts.AWS_helpers as helpers # test_AWS.scripts.AWS_helpers reflects path leading up to AWS_helpers.py

cost = helpers.calculate_s3_storage_cost(bucket_name)

print(f"Estimated monthly storage cost for bucket '{bucket_name}': ${cost:.4f}")

```

    Estimated monthly storage cost for bucket 'titanic-dataset-test': $0.0009


**Important Considerations**:

- **Pricing Tiers**: AWS S3 pricing is tiered. The first 50 TB per month is priced at `$0.023 per GB`, the next 450 TB at `$0.022 per GB`, and storage over 500 TB at `$0.021 per GB`. Ensure you apply the correct pricing tier based on your total storage size.
- **Region and Storage Class**: Pricing varies by AWS region and storage class. The example above uses the S3 Standard storage class pricing for the US East (N. Virginia) region. Adjust the pricing variables if your bucket is in a different region or uses a different storage class.
- **Additional Costs**: This estimation covers storage costs only. AWS S3 may have additional charges for requests, data retrievals, and data transfers. For a comprehensive cost analysis, consider these factors as well.

For detailed and up-to-date information on AWS S3 pricing, please refer to the [AWS S3 Pricing page](https://aws.amazon.com/s3/pricing/).



## 4. Pushing new files from notebook environment to bucket
As your analysis generates new files, you can upload to your bucket as demonstrated below. For this demo, you can create a blank `results.txt` file to upload to your bucket.


```python
# Define the S3 bucket name and the file paths
train_file_path = "results.txt"

# Upload the training file
s3.upload_file(train_file_path, bucket_name, "results/results.txt")

print("Files uploaded successfully.")

```

    Files uploaded successfully.


:::::::::::::::::::::::::::::::::::::: keypoints 

- Load data from S3 into memory for efficient storage and processing.
- Periodically check storage usage and costs to manage S3 budgets.
- Use SageMaker to upload analysis results and maintain an organized workflow.

::::::::::::::::::::::::::::::::::::::::::::::::