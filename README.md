
# Data Pipeline with Airflow, EMR, S3, and Snowflake

## **Workflow**
![Workflow Diagram](workflow.png) 
![AirflowPipline](pipeline.png) 
![DataOnSnowflake](output.png)


## **1. Project Overview**
This project sets up a data pipeline using Apache Airflow to automate ingestion, transformation, and loading workflows. The pipeline leverages AWS EMR for data processing with PySpark, Amazon S3 for storage, and Snowflake for analytics.

---

## **2. Architecture Diagram** 
The workflow is as follows:
- **Ingestion**: Fetch raw data with `wget` and upload to the landing layer in S3.
- **Transformation**: Process data using PySpark in EMR and store it in the curated layer.
- **Loading**: Create and refresh an external table in Snowflake for analytics.

**Technologies Used**:  
Apache Airflow, AWS EMR, S3, Snowflake, PySpark.

---

## **3. Setup Instructions**

### **3.1 Install Dependencies**
```bash
# Update system and install dependencies
sudo apt update
sudo apt install -y python3-pip sqlite3 libpq-dev postgresql postgresql-contrib

# Install Airflow and necessary Python libraries
pip install boto3 pandas apache-airflow[postgres]==2.5.0
pip install apache-airflow-providers-snowflake snowflake-connector-python
pip install apache-airflow-providers-amazon
```

### **3.2 Configure Airflow**
1. Initialize the Airflow database:
   ```bash
   airflow db init
   ```
2. Configure Airflow to use PostgreSQL as the backend:
   ```bash
   # Edit airflow.cfg file
   sed -i 's#sqlite:////home/ubuntu/airflow/airflow.db#postgresql+psycopg2://airflow:airflow@localhost/airflow#g' airflow.cfg
   sed -i 's#SequentialExecutor#LocalExecutor#g' airflow.cfg
   ```
3. Start the Airflow webserver and scheduler:
   ```bash
   airflow webserver &
   airflow scheduler &
   ```

### **3.3 Configure Snowflake**
Run the following Snowflake queries to set up the database, stages, and tables:
```sql
-- Create database
CREATE DATABASE IF NOT EXISTS s3_to_snowflake;

-- Use the database
USE s3_to_snowflake;

-- Create external stage
CREATE OR REPLACE STAGE s3_to_snowflake.PUBLIC.snow_simple
URL='s3://irisseta/output_folder/'
CREDENTIALS=(AWS_KEY_ID='' AWS_SECRET_KEY='');

-- Create file format for Parquet
CREATE OR REPLACE FILE FORMAT my_parquet_format
TYPE = parquet;

-- Create external table
CREATE OR REPLACE EXTERNAL TABLE s3_to_snowflake.PUBLIC.Iris_dataset (
    CLASS_NAME VARCHAR(20) AS (VALUE:CLASS_NAME::VARCHAR),
    Count_Value NUMBER AS (VALUE:count::NUMBER)
)
WITH LOCATION = @s3_to_snowflake.PUBLIC.snow_simple
FILE_FORMAT = 'my_parquet_format';

-- Verify the data
SELECT * FROM s3_to_snowflake.PUBLIC.Iris_dataset;
```

---

## **4. Airflow DAG**

### **4.1 DAG Script**
```python
# Airflow DAG code provided in the project
import boto3
import logging
import airflow
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.contrib.operators.snowflake_operator import SnowflakeOperator
from airflow.contrib.hooks.snowflake_hook import SnowflakeHook
from airflow.operators.bash_operator import BashOperator
from datetime import datetime, timedelta
from time import sleep
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

args = {"owner": "Airflow", "start_date": airflow.utils.dates.days_ago(2)}

dag = DAG(
    dag_id="snowflake_automation_dag", default_args=args, schedule_interval=None
)



client = boto3.client('emr', region_name='us-east-1',aws_access_key_id='{}',aws_secret_access_key='{}')

def create_emr_cluster():
  cluster_id = client.run_job_flow(
    Name="transient_demo_testing",
    Instances={
    'InstanceGroups': [
    {
    'Name': "Master",
    'Market': 'ON_DEMAND',
    'InstanceRole': 'MASTER',
    'InstanceType': 'm1.xlarge',
    'InstanceCount': 1,
    },
    {
    'Name': "Slave",
    'Market': 'ON_DEMAND',
    'InstanceRole': 'CORE',
    'InstanceType': 'm1.xlarge',
    'InstanceCount': 2,
    }
    ],
    'Ec2KeyName': 'helloairflow',
    'KeepJobFlowAliveWhenNoSteps': True,
    'TerminationProtected': False,
    'Ec2SubnetId': 'subnet-03a4055dd5d656279',
    },
    LogUri="s3://{}/",
    ReleaseLabel= 'emr-5.33.0',
    BootstrapActions=[],
    VisibleToAllUsers=True,
    JobFlowRole="EMR_EC2_DefaultRole",
    ServiceRole="EMR_DefaultRole",
    Applications = [ {'Name': 'Spark'},{'Name':'Hive'}])
  print("The cluster started with cluster id : {}".format(cluster_id))
  return cluster_id


  
def add_step_emr(cluster_id,jar_file,step_args):
  print("The cluster id : {}".format(cluster_id))
  print("The step to be added : {}".format(step_args))
  response = client.add_job_flow_steps(
  JobFlowId=cluster_id,
  Steps=[
  {
    'Name': 'test12',
    'ActionOnFailure':'CONTINUE',
    'HadoopJarStep': {
  'Jar': jar_file,
  'Args': step_args
  }
  },
  ]
  )
  print("The emr step is added")
  return response['StepIds'][0]
  
def get_status_of_step(cluster_id,step_id):
  response = client.describe_step(
    ClusterId=cluster_id,
    StepId=step_id
  )
  return response['Step']['Status']['State']
  
  
def wait_for_step_to_complete(cluster_id,step_id):
  print("The cluster id : {}".format(cluster_id))
  print("The emr step id : {}".format(step_id))
  while True:
    try:
      status=get_status_of_step(cluster_id,step_id)
      if status =='COMPLETED':
        break
      else:
        print("The step is {}".format(status))
        sleep(40)

    except Exception as e:
      logging.info(e)
	  

def terminate_cluster(cluster_id):
    try:
        client.terminate_job_flows(JobFlowIds=[cluster_id])
        logger.info("Terminated cluster %s.", cluster_id)
    except ClientError:
        logger.exception("Couldn't terminate cluster %s.", cluster_id)
        raise
		
with dag:
  create_emr_cluster = PythonOperator(
  task_id='create_emr_cluster',
  python_callable=create_emr_cluster,
  dag=dag, 
  )
  ingest_layer = PythonOperator(
  task_id='ingest_layer',
  python_callable=add_step_emr,
  op_args=['{{ ti.xcom_pull("create_emr_cluster")["JobFlowId"]}}','s3://us-east-1.elasticmapreduce/libs/script-runner/script-runner.jar',[ 's3://irisseta/scripts/ingest.sh']],
  dag=dag, 
  )
  poll_step_layer = PythonOperator(
  task_id='poll_step_layer',
  python_callable=wait_for_step_to_complete,
  op_args=['{{ ti.xcom_pull("create_emr_cluster")["JobFlowId"]}}','{{ ti.xcom_pull("ingest_layer")}}'],
  dag=dag, 
  )
  transform_layer = PythonOperator(
  task_id='transform_layer',
  python_callable=add_step_emr,
  op_args=['{{ ti.xcom_pull("create_emr_cluster")["JobFlowId"]}}','command-runner.jar',[ 'spark-submit',
            '--master', 'yarn',
            '--deploy-mode', 'cluster',
            's3://irisseta/scripts/transform.py']],
  dag=dag, 
  )
  poll_step_layer2 = PythonOperator(
  task_id='poll_step_layer2',
  python_callable=wait_for_step_to_complete,
  op_args=['{{ ti.xcom_pull("create_emr_cluster")["JobFlowId"]}}','{{ ti.xcom_pull("transform_layer")}}'],
  dag=dag, 
  )
  terminate_emr_cluster = PythonOperator(
  task_id='terminate_emr_cluster',
  python_callable=terminate_cluster,
  op_args=['{{ ti.xcom_pull("create_emr_cluster")["JobFlowId"]}}'],
  dag=dag, 
  )
  snowflake_load=SnowflakeOperator(
		task_id="snowflake_load",
		sql="""ALTER EXTERNAL TABLE s3_to_snowflake.PUBLIC.Iris_dataset REFRESH""" ,
		snowflake_conn_id="snowflake_conn"
	)

create_emr_cluster >> ingest_layer >> poll_step_layer >> transform_layer >> poll_step_layer2 >> terminate_emr_cluster >> snowflake_load
```

---

## **5. Shell Script for Ingestion**

File: **ingest.sh**
```bash
wget -O - https://archive.ics.uci.edu/ml/machine-learning-databases/iris/iris.data | aws s3 cp - s3://irisseta/input_folder/hello_world.csv
```

---

## **6. Execution Process**
1. Start the Airflow scheduler and webserver.
2. Trigger the DAG from the Airflow UI.
3. Monitor task status on the Airflow UI.
4. Validate the Snowflake external table with:
   ```sql
   SELECT * FROM s3_to_snowflake.PUBLIC.Iris_dataset;
   ```

---

## **Outputs**
- **Pipeline**: Ingestion and Transformation complete with PySpark and EMR.
- **Snowflake External Table**: Refreshed external table for analytics.

---

Would you like to add troubleshooting tips or logs?
