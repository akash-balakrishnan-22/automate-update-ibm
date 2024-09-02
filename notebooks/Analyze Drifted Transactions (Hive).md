<img src="https://github.com/pmservice/ai-openscale-tutorials/raw/master/notebooks/images/banner.png" align="left" alt="banner">

# Notebook for analyzing payload transactions causing drift

This notebook helps users of IBM Watson OpenScale to analyze payload transactions that are causing drift - both drop in accuracy and drop in data consistency. 

The notebook is designed to give users a jump start in their analysis of the payload transactions. It is by no means a comprehensive analysis. 

The user needs to provide the necessary inputs (where marked) to be able to proceed with the analysis. 

PS: This notebook is designed to analyse one drift monitor run at a time for a given subscription.

**Contents:**
1. [Pre-requisites](#Pre-requisites)
2. [Installing Dependencies](#Installing-Dependencies)
3. [User Inputs](#User-Inputs)
4. [Setting up Services](#Setting-up-Services)
5. [Measurement Summary](#Measurement-Summary)
6. [Counts from Drifted Transactions Table](#Counts-from-Drifted-Transactions-Table)
7. [Analyse Transactions Causing Drop in Accuracy](#Analyse-Transactions-Causing-Drop-in-Accuracy)
    * [Get all transactions causing drop in data accuracy](#Get-all-transactions-causing-drop-in-data-accuracy)
    * [Get all transactions causing drop in accuracy in given range of drift model confidence](#Get-all-transactions-causing-drop-in-accuracy-in-given-range-of-drift-model-confidence)
8. [Analyse Transactions Causing Drop in Accuracy and Drop in Data Consistency](#Analyse-Transactions-Causing-Drop-in-Accuracy-and-Drop-in-Data-Consistency)
    * [Get all transactions causing drop in accuracy and drop in data consistency](#Get-all-transactions-causing-drop-in-accuracy-and--drop-in-data-consistency)
    * [Get all transactions causing drop in accuracy and drop in data consistency in given range of drift model confidence](#Get-all-transactions-causing-drop-in-accuracy-and-drop-in-data-consistency-in-given-range-of-drift-model-confidence)
9. [Analyse Transactions Causing Drop in Data Consistency](#Analyse-Transactions-Causing-Drop-in-Data-Consistency)
    * [Get all transactions causing drop in data consistency](#Get-all-transactions-causing-drop-in-data-consistency)
    * [Get all transactions violating a data constraint](#Get-all-transactions-violating-a-data-constraint)
    * [Get all transactions where a column is causing drop in data consistency](#Get-all-transactions-where-a-column-is-causing-drop-in-data-consistency)
    * [Explain categorical distribution constraint violations](#Explain-categorical-distribution-constraint-violations)
    * [Explain numeric range constraint violations](#Explain-numeric-range-constraint-violations)
    * [Explain cat-numeric range constraint violations](#Explain-cat-numeric-range-constraint-violations)
    * [Explain cat-cat distribution constraint violations](#Explain-cat-cat-distribution-constraint-violations)

## Pre-requisites

1. The Jupyter Server on which this notebook is running should be able to access the HDFS cluster on which data is residing in Hive.
2. **Connecting to Non-Kerberised Hive:** The only property required for this is HIVE_METASTORE_URI in the [User Inputs](#User-Inputs) section.
3. **Connecting to Kerberised Hive:** 
    - Make sure you are able to obtain a Kerberos ticket using kinit for the cluster you are planning to connect to. This needs to be done before starting the jupyter server.
    - The **core-site.xml** file under **SPARK_HOME/conf** directory in the machine where jupyter is running should have the following property:
    ```
    <configuration>
        <property>
            <name>hadoop.security.authentication</name>
            <value>kerberos</value>
        </property>
    </configuration>

    ```
    - There are three other properties required in the [User Inputs](#User-Inputs) section besides HIVE_METASTORE_URI. 
    - PS: Currently it is not possible to connect to a Kerberised Hive from Watson Studio.

## Installing Dependencies


```python
# ----------------------------------------------------------------------------------------------------
# IBM Confidential
# OCO Source Materials
# 5737-H76
# Copyright IBM Corp. 2020, 2023
# The source code for this Notebook is not published or other-wise divested of its trade
# secrets, irrespective of what has been deposited with the U.S.Copyright Office.
# ----------------------------------------------------------------------------------------------------

VERSION = "hive-1.1.9"

# Changelog

# hive-1.1.9 : Upgrade ibm-wos-utils to 5.0.0
# hive-1.1.8 : Upgrade ibm-wos-utils to 4.8.0
# hive-1.1.7 : Upgrade ibm-wos-utils to 4.7.0
# hive-1.1.6 : Install pyspark as a pre-requisite
# hive-1.1.5 : Upgrade ibm-wos-utils to 4.5.0
# hive-1.1.4 : Upgrade ibm-wos-utils to 4.1.1 (scikit-learn has been upgraded to 1.0.2)
# hive-1.1.3 : Upgrade ibm-wos-utils to 4.0.34
# hive-1.1.2 : Upgrade ibm-wos-utils to 4.0.31
# hive-1.1.1 : Add comment about conda install for zLinux environments; Update wos-utils version to 4.0.25
# hive-1.1 : Update wos-utils version to 4.0.24
# 1.0 : First public release
```


```python
import warnings
warnings.filterwarnings('ignore')
%env PIP_DISABLE_PIP_VERSION_CHECK=1
```

    env: PIP_DISABLE_PIP_VERSION_CHECK=1



```python
import sys

PYTHON = sys.executable

!$PYTHON -m pip install --no-warn-conflicts --upgrade tabulate ibm-watson-openscale pyspark | tail -n 1   
```

    Requirement already satisfied: certifi>=2017.4.17 in /opt/conda/envs/Python-RT24.1-Premium/lib/python3.11/site-packages (from requests<3.0,>=2.0->ibm-watson-openscale) (2024.2.2)


**Note:** For IBM Watson OpenScale Cloud Pak for Data version 4.0.0 - 4.0.7, use the cell below:


```python
# When this notebook is to be run on a zLinux cluster,
# install scikit-learn==0.24.2 using conda before installing ibm-wos-utils
# !conda install scikit-learn=0.24.2

# !$PYTHON -m pip install --no-warn-conflicts "ibm-wos-utils==4.0.34" | tail -n 1
```

**Note:** For IBM Watson OpenScale Cloud Pak for Data version 4.0.8 - 4.0.x, use the cell below:


```python
# When this notebook is to be run on a zLinux cluster,
# install scikit-learn==1.0.2 using conda before installing ibm-wos-utils
# !conda install scikit-learn=1.0.2

# !$PYTHON -m pip install --no-warn-conflicts "ibm-wos-utils==4.1.1" | tail -n 1
```

**Note:** For IBM Watson OpenScale Cloud Pak for Data version 4.5.x, use the cell below:


```python
# When this notebook is to be run on a zLinux cluster,
# install scikit-learn==1.0.2 using conda before installing ibm-wos-utils
# !conda install scikit-learn=1.0.2

!$PYTHON -m pip install --no-warn-conflicts "ibm-wos-utils>=4.5.0" | tail -n 1
```

    Requirement already satisfied: s3transfer<0.8.0,>=0.7.0 in /opt/conda/envs/Python-RT24.1-Premium/lib/python3.11/site-packages (from boto3->ibm-wos-utils>=4.5.0) (0.7.0)


**Note:** For IBM Watson OpenScale Cloud Pak for Data version 4.6.x, use the cell below:


```python
# When this notebook is to be run on a zLinux cluster,
# install scikit-learn==1.0.2 using conda before installing ibm-wos-utils
# !conda install scikit-learn=1.0.2

!$PYTHON -m pip install --no-warn-conflicts "ibm-wos-utils>=4.6.0" | tail -n 1
```

    Requirement already satisfied: s3transfer<0.8.0,>=0.7.0 in /opt/conda/envs/Python-RT24.1-Premium/lib/python3.11/site-packages (from boto3->ibm-wos-utils>=4.6.0) (0.7.0)


**Note:** For IBM Watson OpenScale Cloud Pak for Data version 4.7.x, use the cell below:


```python
# When this notebook is to be run on a zLinux cluster,
# install scikit-learn==1.1.1 using conda before installing ibm-wos-utils
# !conda install scikit-learn=1.1.1

!$PYTHON -m pip install --no-warn-conflicts "ibm-wos-utils>=4.7.0" | tail -n 1
```

    Requirement already satisfied: s3transfer<0.8.0,>=0.7.0 in /opt/conda/envs/Python-RT24.1-Premium/lib/python3.11/site-packages (from boto3->ibm-wos-utils>=4.7.0) (0.7.0)


**Note:** For IBM Watson OpenScale Cloud Pak for Data version 4.8.x, use the cell below:


```python
# When this notebook is to be run on a zLinux cluster,
# install scikit-learn==1.1.1 using conda before installing ibm-wos-utils
# !conda install scikit-learn=1.1.1

!$PYTHON -m pip install --no-warn-conflicts "ibm-wos-utils>=4.8.0" | tail -n 1
```

    Requirement already satisfied: s3transfer<0.8.0,>=0.7.0 in /opt/conda/envs/Python-RT24.1-Premium/lib/python3.11/site-packages (from boto3->ibm-wos-utils>=4.8.0) (0.7.0)


**Note:** For IBM Watson OpenScale Cloud Pak for Data version 5.0.x, use the cell below:


```python
# When this notebook is to be run on a zLinux cluster,
# install scikit-learn==1.1.1 using conda before installing ibm-wos-utils
# !conda install scikit-learn=1.1.1

!$PYTHON -m pip install --no-warn-conflicts "ibm-wos-utils>=5.0.0" | tail -n 1
```

    Requirement already satisfied: s3transfer<0.8.0,>=0.7.0 in /opt/conda/envs/Python-RT24.1-Premium/lib/python3.11/site-packages (from boto3->ibm-wos-utils>=5.0.0) (0.7.0)


## User Inputs

The following inputs are required:

1. **IBM_CPD_ENDPOINT:** The URL representing the IBM Cloud Pak for Data service endpoint.
2. **IBM_CPD_USERNAME:** IBM Cloud Pak for Data username used to obtain a bearer token.
3. **IBM_CPD_PASSWORD:** IBM Cloud Pak for Data password used to obtain a bearer token.
4. **HIVE_METASTORE_URI:** Hive Metastore URI to connect to using this notebook
5. **KERBERISED_HIVE_YARN_RM_PRINCIPAL:** Yarn Resource Manager Principal (_required only if connecting to Kerberised Hive_)
6. **KERBERISED_HIVE_YARN_RM_KEYTAB:** Path to the Yarn Resource Manager KeyTab file on the cluster (_required only if connecting to Kerberised Hive_)
7. **KERBERISED_HIVE_METASTORE_PRINCIPAL:** Hive MetaStore Principal (_required only if connecting to Kerberised Hive_)
8. **ANALYSIS_INPUT_PARAMETERS:** Analysis Input Parameters to be copied from IBM Watson OpenScale UI


```python
# IBM Cloud Pak for Data credentials
IBM_CPD_ENDPOINT = "<The URL representing the IBM Cloud Pak for Data service endpoint.>"
IBM_CPD_USERNAME = "<IBM Cloud Pak for Data username used to obtain a bearer token.>"
IBM_CPD_PASSWORD = "<IBM Cloud Pak for Data password used to obtain a bearer token.>"

# Hive Metastore URI to connect to
HIVE_METASTORE_URI = "<Hive Metastore URI>"

# Additional Properties Required for connecting to KERBERISED Hive
KERBERISED_HIVE_YARN_RM_PRINCIPAL = "<Yarn Resource Manager Principal>"
KERBERISED_HIVE_YARN_RM_KEYTAB = "<Path to the Yarn Resource Manager KeyTab file on the cluster>"
KERBERISED_HIVE_METASTORE_PRINCIPAL = "<Hive MetaStore Principal>"


# Analysis Input Parameters to be copied from UI
# Please make sure that the quotes around the key-values 
# are correct after copying from UI
ANALYSIS_INPUT_PARAMETERS = {
    "data_mart_id": "<data_mart_id>",
    "subscription_id": "<subscription_id>",
    "monitor_instance_id": "<monitor_instance_id>",
    "measurement_id": "<measurement_id>"
}
```


```python
DATAMART_ID = ANALYSIS_INPUT_PARAMETERS.get("data_mart_id")
SUBSCRIPTION_ID = ANALYSIS_INPUT_PARAMETERS.get("subscription_id")
MONITOR_INSTANCE_ID = ANALYSIS_INPUT_PARAMETERS.get("monitor_instance_id")
MEASUREMENT_ID = ANALYSIS_INPUT_PARAMETERS.get("measurement_id")
```

## Setting up Services


```python
import pandas as pd
import pyspark.sql.functions as F
from pyspark import SparkConf
from pyspark.sql import SparkSession

from ibm_cloud_sdk_core.authenticators import CloudPakForDataAuthenticator
from ibm_watson_openscale import APIClient

from ibm_wos_utils.drift.batch.util.constants import ConstraintName
from ibm_wos_utils.joblib.utils.analyze_notebook_utils import (
    explain_catcat_distribution_constraint,
    explain_categorical_distribution_constraint,
    explain_catnum_range_constraint, explain_numeric_range_constraint,
    get_column_query, get_drift_archive_contents,
    get_table_details_from_subscription, show_constraints_by_column,
    show_dataframe, show_last_n_drift_measurements)
```


```python
conf = SparkConf()\
        .setAppName("Analyze Drifted Transactions")\
        .set("spark.hadoop.hive.metastore.uris", HIVE_METASTORE_URI)

# Uncomment the following block if connecting to a Kerberised Hive
"""
conf = conf\
        .set("spark.hadoop.yarn.resourcemanager.principal", KERBERISED_HIVE_YARN_RM_PRINCIPAL)\
        .set("spark.hadoop.yarn.resourcemanager.keytab", KERBERISED_HIVE_YARN_RM_KEYTAB)\
        .set("spark.hadoop.hive.metastore.kerberos.principal", KERBERISED_HIVE_METASTORE_PRINCIPAL)\
        .set("spark.hadoop.hive.metastore.sasl.enabled", "true")
"""

spark = SparkSession.builder.config(conf=conf).enableHiveSupport().getOrCreate()
```

    Setting default log level to "WARN".
    To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
    24/06/17 08:53:17 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    24/06/17 08:53:19 WARN Utils: Service 'SparkUI' could not bind on port 4040. Attempting port 4041.
    24/06/17 08:53:19 WARN Utils: Service 'SparkUI' could not bind on port 4041. Attempting port 4042.



```python
authenticator = CloudPakForDataAuthenticator(
        url=IBM_CPD_ENDPOINT,
        username=IBM_CPD_USERNAME,
        password=IBM_CPD_PASSWORD,
        disable_ssl_verification=True
    )
wos_client = APIClient(authenticator=authenticator, service_url=IBM_CPD_ENDPOINT)
```


```python
%%time

if not DATAMART_ID or not SUBSCRIPTION_ID:
    raise Exception("DATAMART_ID and SUBSCRIPTION_ID are required to proceed.")

subscription = wos_client.subscriptions.get(subscription_id=SUBSCRIPTION_ID).result
monitor_instance = wos_client.monitor_instances.list(data_mart_id=DATAMART_ID, target_target_id=SUBSCRIPTION_ID, monitor_definition_id="drift").result.monitor_instances[0]
model_drift_enabled = monitor_instance.entity.parameters.get("model_drift_enabled", False)
data_drift_enabled = monitor_instance.entity.parameters.get("data_drift_enabled", False)

if not MONITOR_INSTANCE_ID:
    MONITOR_INSTANCE_ID = monitor_instance.metadata.id
    
drift_archive = wos_client.monitor_instances.download_drift_model(monitor_instance_id=MONITOR_INSTANCE_ID).result.content
schema, ddm_properties, constraints_set = get_drift_archive_contents(drift_archive, model_drift_enabled, data_drift_enabled)
payload_database_name, _, payload_table_name = get_table_details_from_subscription(subscription, "payload")
drift_database_name, _, drift_table_name = get_table_details_from_subscription(subscription, "drift")
```

    CPU times: user 41.8 ms, sys: 10.9 ms, total: 52.6 ms
    Wall time: 648 ms


This notebook relies heavily on filtering transactions in the Drifted Transactions table based on three columns: `run_id`, `is_model_drift` and `is_data_drift`. 

It is, therefore, recommended that you create and build an index for these columns, if not done already as part of the common configuration notebook. You can use the following DDL to create and build the index.


```python
ddl_string = "CREATE INDEX {1}_index ON TABLE {0}.{1} (run_id, is_model_drift, is_data_drift) AS 'BITMAP' WITH DEFERRED REBUILD;\n"
ddl_string += "ALTER INDEX {1}_index ON {0}.{1} REBUILD;"
ddl_string.format(drift_database_name, drift_table_name)
print(ddl_string)
```

    CREATE INDEX {1}_index ON TABLE {0}.{1} (run_id, is_model_drift, is_data_drift) AS 'BITMAP' WITH DEFERRED REBUILD;
    ALTER INDEX {1}_index ON {0}.{1} REBUILD;



```python
if not MEASUREMENT_ID:
    print("Please pick a measurement to analyze from the following list:")
    
show_last_n_drift_measurements(10, wos_client, SUBSCRIPTION_ID)
```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Measurement ID</th>
      <th>Monitor Run ID</th>
      <th>Monitor Instance ID</th>
      <th>Subscription ID</th>
      <th>Timestamp</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>8fc89a1a-42c1-483c-9a9f-4eafec83e0b3</td>
      <td>105a8f06-1df5-415e-922a-6a1b20cbf45c</td>
      <td>20ed3082-4c24-4627-9542-866738d4fc67</td>
      <td>a74c08f9-e180-4347-b433-9d09cfcb15fd</td>
      <td>2024-06-17 07:19:36.170197+00:00</td>
    </tr>
  </tbody>
</table>
</div>



```python
# If you have not selected MEASUREMENT_ID so far, please enter a measurement ID
# from the above cell's output to analyze.

# MEASUREMENT_ID = None
```


```python
if not MEASUREMENT_ID:
    raise Exception("MEASUREMENT_ID is required to proceed.")

measurement = wos_client.monitor_instances.measurements.get(measurement_id=MEASUREMENT_ID, monitor_instance_id=MONITOR_INSTANCE_ID).result
measurement_data = measurement.entity.sources[0].data
MONITOR_RUN_ID = measurement.entity.run_id
MONITOR_RUN_ID
```




    '105a8f06-1df5-415e-922a-6a1b20cbf45c'



## Measurement Summary

### Counts of transactions causing drop in accuracy and drop in data consistency


```python
print("IBM Watson OpenScale analyzed {} transactions between {} and {} for drift. Here's a summary.".format(measurement_data["transactions_count"], measurement_data["start"], measurement_data["end"]))

if model_drift_enabled:
    print("  - Total {} transactions out of {} transactions are causing drop in accuracy.".format(measurement_data["drifted_transactions"]["count"], measurement_data["transactions_count"]))

if data_drift_enabled:
    print("  - Total {} transactions out of {} transactions are causing drop in data consistency.".format(measurement_data["data_drifted_transactions"]["count"], measurement_data["transactions_count"]))
    
if model_drift_enabled and data_drift_enabled:
    print("  - Total {} transactions out of {} transactions are causing both drop in accuracy and drop in data consistency.".format(measurement_data["model_data_drifted_transactions"]["count"], measurement_data["transactions_count"]))
```

    IBM Watson OpenScale analyzed 1460 transactions between 2021-05-13T09:14:16.183000 and 2021-05-20T09:11:00.785000 for drift. Here's a summary.
      - Total 0 transactions out of 1460 transactions are causing drop in data consistency.


### Counts of transactions causing drop in accuracy - percent bins


```python
if model_drift_enabled:
    rows_df = pd.DataFrame(measurement_data["drifted_transactions"]["drift_model_confidence_count"])
    rows_df = rows_df[["lower_limit", "upper_limit", "count"]]
    rows_df.columns = ["Drift Model Confidence - Lower Limit", "Drift Model Confidence - Upper Limit", "Violated Transactions Count"]
    display(rows_df)
```

### Counts of transactions causing drop in data consistency - feature columns


```python
if data_drift_enabled:
    rows_df = pd.Series(measurement_data["data_drifted_transactions"]["features_count"])\
                .sort_values(ascending=False).to_frame()
    rows_df.reset_index(inplace=True)
    rows_df.columns = ["Feature Column", "Violated Transactions Count"]
    display(rows_df)
```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Feature Column</th>
      <th>Violated Transactions Count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>electrical</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>garagecond</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>garagequal</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>lotshape</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>salecondition</td>
      <td>0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>yrsold</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>


### Counts of transactions causing drop in accuracy - constraints list


```python
if data_drift_enabled:
    rows_df = pd.Series(measurement_data["data_drifted_transactions"]["constraints_count"])\
                .sort_values(ascending=False).to_frame()
    rows_df.reset_index(inplace=True)
    rows_df.columns = ["Constraint Name", "Violated Transactions Count"]
    display(rows_df)

```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Constraint Name</th>
      <th>Violated Transactions Count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>catcat_distribution_constraint</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>categorical_distribution_constraint</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>catnum_range_constraint</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>numeric_range_constraint</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>


## Counts from Drifted Transactions Table


```python
drift_table_df = spark.sql("select * from {}.{} where run_id = '{}'".format(drift_database_name, drift_table_name, MONITOR_RUN_ID))
drift_table_df.printSchema()
```

    root
     |-- scoring_id: string (nullable = true)
     |-- scoring_timestamp: timestamp (nullable = true)
     |-- constraints_generation_id: string (nullable = true)
     |-- run_id: string (nullable = true)
     |-- is_model_drift: boolean (nullable = true)
     |-- drift_model_confidence: float (nullable = true)
     |-- is_data_drift: boolean (nullable = true)
     |-- categorical_distribution_constraint: string (nullable = true)
     |-- numeric_range_constraint: string (nullable = true)
     |-- catcat_distribution_constraint: string (nullable = true)
     |-- catnum_range_constraint: string (nullable = true)
    



```python
from ibm_wos_utils.joblib.utils.hive_utils import get_table_as_dataframe

payload_table_df = get_table_as_dataframe(spark=spark,
                                         database_name=payload_database_name,
                                         table_name=payload_table_name,
                                         columns_to_map=subscription.entity.asset_properties.feature_fields)
```

    root
     |-- garagequal: string (nullable = true)
     |-- garagecond: string (nullable = true)
     |-- electrical: string (nullable = true)
     |-- lotshape: string (nullable = true)
     |-- salecondition: string (nullable = true)
     |-- yrsold: integer (nullable = true)
     |-- prediction: double (nullable = true)
     |-- scoring_id: string (nullable = true)
     |-- scoring_timestamp: timestamp (nullable = true)
    


    [Stage 0:>                                                          (0 + 1) / 1]

    root
     |-- garagequal: string (nullable = true)
     |-- garagecond: string (nullable = true)
     |-- electrical: string (nullable = true)
     |-- lotshape: string (nullable = true)
     |-- salecondition: string (nullable = true)
     |-- yrsold: integer (nullable = true)
     |-- prediction: double (nullable = true)
     |-- scoring_id: string (nullable = true)
     |-- scoring_timestamp: timestamp (nullable = true)
    
    root
     |-- garagequal: string (nullable = true)
     |-- garagecond: string (nullable = true)
     |-- electrical: string (nullable = true)
     |-- lotshape: string (nullable = true)
     |-- salecondition: string (nullable = true)
     |-- yrsold: integer (nullable = true)
     |-- prediction: double (nullable = true)
     |-- scoring_id: string (nullable = true)
     |-- scoring_timestamp: timestamp (nullable = true)
    


                                                                                    


```python
%%time

print("Total number of drifted transactions: {}".format(drift_table_df.count()))
print("Total number of model drift transactions: {}".format(drift_table_df.where("is_model_drift").count()))
print("Total number of data drift transactions: {}".format(drift_table_df.where("is_data_drift").count()))
print("Total number of model + data drift transactions: {}".format(drift_table_df.where("is_model_drift").where("is_data_drift").count()))
print()
```

    24/06/17 08:53:34 WARN SessionState: METASTORE_FILTER_HOOK will be ignored, since hive.security.authorization.manager is set to instance of HiveAuthorizerFactory.
                                                                                    

    Total number of drifted transactions: 0


                                                                                    

    Total number of model drift transactions: 0


                                                                                    

    Total number of data drift transactions: 0


    [Stage 10:===================================================>  (124 + 1) / 129]

    Total number of model + data drift transactions: 0
    
    CPU times: user 54.5 ms, sys: 23 ms, total: 77.5 ms
    Wall time: 20.6 s


                                                                                    

## Analyse Transactions Causing Drop in Accuracy

### Get all transactions causing drop in data accuracy

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time

drifted_transactions_df = drift_table_df\
    .where("is_model_drift")\
    .select(["scoring_id","drift_model_confidence"])

count = drifted_transactions_df.count()

print("Total {} transactions are causing drop in accuracy.".format(count))

if count:
    num_rows = 10
    print("Showing {} such transactions in the order of drift_model_confidence".format(num_rows))

    drifted_transactions_df = payload_table_df\
        .join(drifted_transactions_df, ["scoring_id"], "leftsemi")\
        .join(drifted_transactions_df, ["scoring_id"], "left")\
        .sort(["drift_model_confidence"], ascending=False)

    show_dataframe(drifted_transactions_df, num_rows=num_rows, priority_columns=["drift_model_confidence"])

```

    [Stage 13:==============================================>       (112 + 1) / 129]

    Total 0 transactions are causing drop in accuracy.
    CPU times: user 10.2 ms, sys: 4.7 ms, total: 14.9 ms
    Wall time: 3.13 s


                                                                                    

### Get all transactions causing drop in accuracy in given range of drift model confidence

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time

# Drift Model Confidence Lower Limit
dm_conf_lower = 0.5
# Drift Model Confidence Upper Limit
dm_conf_upper = 1.0

drifted_transactions_df = drift_table_df\
    .where("is_model_drift")\
    .where(drift_table_df.drift_model_confidence.between(dm_conf_lower,dm_conf_upper))\
    .select(["scoring_id","drift_model_confidence"])

count = drifted_transactions_df.count()

print("Total {} transactions are causing drop in accuracy where drift model confidence is between {} and {}".format(count, dm_conf_lower, dm_conf_upper))

if count:
    num_rows = 10
    print("Showing {} such transactions in the order of drift_model_confidence".format(num_rows))

    drifted_transactions_df = payload_table_df\
        .join(drifted_transactions_df, ["scoring_id"], "leftsemi")\
        .join(drifted_transactions_df, ["scoring_id"], "left")\
        .sort(["drift_model_confidence"], ascending=False)

    show_dataframe(drifted_transactions_df, num_rows=num_rows, priority_columns=["drift_model_confidence"])

```

    [Stage 16:=================================================>    (119 + 1) / 129]

    Total 0 transactions are causing drop in accuracy where drift model confidence is between 0.5 and 1.0
    CPU times: user 9.77 ms, sys: 5.13 ms, total: 14.9 ms
    Wall time: 3.1 s


                                                                                    

## Analyse Transactions Causing Drop in Accuracy and Drop in Data Consistency

### Get all transactions causing drop in accuracy and  drop in data consistency

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time

drifted_transactions_df = drift_table_df\
    .where("is_model_drift")\
    .where("is_data_drift")\
    .select(["scoring_id","drift_model_confidence"])

count = drifted_transactions_df.count()

print("Total {} transactions are causing both drop in accuracy and drop in data consistency".format(count))

if count:
    num_rows = 10
    print("Showing {} such transactions in the order of drift_model_confidence".format(num_rows))

    drifted_transactions_df = payload_table_df\
        .join(drifted_transactions_df, ["scoring_id"], "leftsemi")\
        .join(drifted_transactions_df, ["scoring_id"], "left")\
        .sort(["drift_model_confidence"], ascending=False)

    show_dataframe(drifted_transactions_df, num_rows=num_rows, priority_columns=["drift_model_confidence"])

```

    [Stage 19:==================================================>   (121 + 1) / 129]

    Total 0 transactions are causing both drop in accuracy and drop in data consistency
    CPU times: user 8.62 ms, sys: 3.67 ms, total: 12.3 ms
    Wall time: 2.42 s


                                                                                    

### Get all transactions causing drop in accuracy and drop in data consistency in given range of drift model confidence

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time

# Drift Model Confidence Lower Limit
dm_conf_lower = 0.5
# Drift Model Confidence Upper Limit
dm_conf_upper = 1.0

drifted_transactions_df = drift_table_df\
    .where("is_model_drift")\
    .where("is_data_drift")\
    .where(drift_table_df.drift_model_confidence.between(dm_conf_lower,dm_conf_upper))\
    .select(["scoring_id","drift_model_confidence"])

count = drifted_transactions_df.count()

print("Total {} transactions are causing drop in accuracy and drop in data consistency where drift model confidence is between {} and {}".format(count, dm_conf_lower, dm_conf_upper))

if count:
    num_rows = 10
    print("Showing {} such transactions in the order of drift_model_confidence".format(num_rows))

    drifted_transactions_df = payload_table_df\
        .join(drifted_transactions_df, ["scoring_id"], "leftsemi")\
        .join(drifted_transactions_df, ["scoring_id"], "left")\
        .sort(["drift_model_confidence"], ascending=False)

    show_dataframe(drifted_transactions_df, num_rows=num_rows, priority_columns=["drift_model_confidence"])

```

    [Stage 22:===================================================>  (122 + 2) / 129]

    Total 0 transactions are causing drop in accuracy and drop in data consistency where drift model confidence is between 0.5 and 1.0
    CPU times: user 11.7 ms, sys: 4.86 ms, total: 16.5 ms
    Wall time: 2.94 s


                                                                                    

## Analyse Transactions Causing Drop in Data Consistency

### Get all transactions causing drop in data consistency

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time

drifted_transactions_df = drift_table_df\
    .where("is_data_drift")\
    .select(["scoring_id"])

count = drifted_transactions_df.count()

print("Total {} transactions are causing drop in data consistency".format(count))

if count:
    num_rows = 10
    print("Showing {} such transactions".format(num_rows))

    drifted_transactions_df = payload_table_df\
        .join(drifted_transactions_df, ["scoring_id"], "leftsemi")\
        .join(drifted_transactions_df, ["scoring_id"], "left")

    show_dataframe(drifted_transactions_df, num_rows=num_rows)

```

    [Stage 25:===============================================>      (114 + 1) / 129]

    Total 0 transactions are causing drop in data consistency
    CPU times: user 4.22 ms, sys: 5.82 ms, total: 10 ms
    Wall time: 2.14 s


                                                                                    

### Get all transactions violating a data constraint

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time

constraint_name = ConstraintName.CATEGORICAL_DISTRIBUTION_CONSTRAINT

drifted_transactions_df = drift_table_df\
        .where("is_data_drift")\
        .where(F.col(constraint_name.value).like("%1%"))\
        .select(["scoring_id"])

count = drifted_transactions_df.count()

print("Total {} transactions are violating {}.".format(count, constraint_name.value))

if count:
    num_rows = 10
    print("Showing {} such transactions.".format(num_rows))
    
    drifted_transactions_df = payload_table_df\
        .join(drifted_transactions_df, ["scoring_id"], "leftsemi")\
        .join(drifted_transactions_df, ["scoring_id"], "left")\

    show_dataframe(drifted_transactions_df, num_rows=num_rows)


```

    [Stage 28:===================================================>  (124 + 1) / 129]

    Total 0 transactions are violating categorical_distribution_constraint.
    CPU times: user 10.4 ms, sys: 1.31 ms, total: 11.7 ms
    Wall time: 2.4 s


                                                                                    

### Get all transactions where a column is causing drop in data consistency

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
filter_query = get_column_query(constraints_set, schema, column="electrical")

drifted_transactions_df = drift_table_df\
    .where("is_data_drift")\
    .where(filter_query)\
    .select(["scoring_id"])
count = drifted_transactions_df.count()

print("Total {} transactions are satisfying the given query.".format(count))

if count:
    num_rows = 10
    print("Showing {} such transactions.".format(num_rows))

    drifted_transactions_df = payload_table_df\
        .join(drifted_transactions_df, ["scoring_id"], "leftsemi")\
        .join(drifted_transactions_df, ["scoring_id"], "left")\

    show_dataframe(drifted_transactions_df, num_rows=num_rows)

```

    [Stage 31:===============================================>      (113 + 1) / 129]

    Total 0 transactions are satisfying the given query.


                                                                                    

### Query all the learnt constraints based on a column name

Use the `show_constraints_by_column` method to find all the constraints learnt for a particular column at training time. The constraint ids shown in the cell output can be used to explain the corresponding constraint in subsequent cells.


```python
show_constraints_by_column(constraints_set, "<column_name>")
```


<table>
<thead>
<tr><th>Constraint ID                                           </th><th>Constraint Name                    </th><th>Constraint Kind  </th><th>Constraint Columns  </th></tr>
</thead>
<tbody>
<tr><td>e6a7e3ab91371a63005aae582df063d42c11608be86a05fb5a8a9284</td><td>categorical_distribution_constraint</td><td>single_column    </td><td>[&#x27;yrsold&#x27;]          </td></tr>
</tbody>
</table>


### Explain categorical distribution constraint violations

Explains categorical distribution constraint violations given a constraint id. The constraint id can be gotten by running [this cell](#Query-all-the-learnt-constraints-based-on-a-column-name)

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time 

constraint_id = "<constraint_id>"

drifted_transactions_df = explain_categorical_distribution_constraint(drifted_transactions_df=drift_table_df,
                              payload_table_df=payload_table_df,
                              constraints_set=constraints_set,
                              schema=schema,
                              constraint_id=constraint_id)

if drifted_transactions_df:
    num_rows = 10
    print("Showing {} such transactions.".format(num_rows))

    show_dataframe(drifted_transactions_df, num_rows=num_rows)
```

    The frequency distribution of column 'electrical' at training time:



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>electrical</th>
      <th>count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>FuseA</td>
      <td>82</td>
    </tr>
    <tr>
      <th>1</th>
      <td>FuseF</td>
      <td>24</td>
    </tr>
    <tr>
      <th>2</th>
      <td>FuseP</td>
      <td>5</td>
    </tr>
    <tr>
      <th>3</th>
      <td>SBrkr</td>
      <td>1349</td>
    </tr>
  </tbody>
</table>
</div>


    [Stage 34:==============================================>       (112 + 1) / 129]

    Total 0 transactions are violating constraint with id 'fa550adb1291b5e7aea464f0ae742e25fcb5ed35734208f6e86e4517'.
    CPU times: user 14 ms, sys: 4.13 ms, total: 18.1 ms
    Wall time: 1.99 s


                                                                                    

### Explain numeric range constraint violations

Explains numeric range constraint violations given a constraint id. The constraint id can be gotten by running [this cell](#Query-all-the-learnt-constraints-based-on-a-column-name)

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time

constraint_id = "<constraint_id>"

drifted_transactions_df = explain_numeric_range_constraint(drifted_transactions_df=drift_table_df,
                              payload_table_df=payload_table_df,
                              constraints_set=constraints_set,
                              schema=schema,
                              constraint_id=constraint_id)


if drifted_transactions_df:
    num_rows = 10
    print("Showing {} such transactions.".format(num_rows))

    show_dataframe(drifted_transactions_df, num_rows=num_rows)
```

### Explain cat-numeric range constraint violations

Explains cat-numeric range constraint violations given a constraint id. The constraint id can be gotten by running [this cell](#Query-all-the-learnt-constraints-based-on-a-column-name)

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time

constraint_id = "<constraint_id>"

drifted_transactions_df = explain_catnum_range_constraint(drifted_transactions_df=drift_table_df,
                              payload_table_df=payload_table_df,
                              constraints_set=constraints_set,
                              schema=schema,
                              constraint_id=constraint_id)

if drifted_transactions_df:
    num_rows = 10
    print("Showing {} such transactions.".format(num_rows))

    show_dataframe(drifted_transactions_df, num_rows=num_rows)
```

### Explain cat-cat distribution constraint violations

Explains cat-cat distribution constraint violations given a constraint id. The constraint id can be gotten by running [this cell](#Query-all-the-learnt-constraints-based-on-a-column-name)

The `drifted_transactions_df` can be exported to a format of your choice for further analysis.


```python
%%time

constraint_id = "<constraint_id>"

drifted_transactions_df = explain_catcat_distribution_constraint(drifted_transactions_df=drift_table_df,
                              payload_table_df=payload_table_df,
                              constraints_set=constraints_set,
                              schema=schema,
                              constraint_id=constraint_id)


if drifted_transactions_df:
    num_rows = 10
    print("Showing {} such transactions.".format(num_rows))

    show_dataframe(drifted_transactions_df, num_rows=num_rows)
```

#### Authors
Developed by [Prem Piyush Goyal](mailto:prempiyush@in.ibm.com)
