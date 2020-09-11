# Airflow_forexExample

#### Airflow
#### Hadoop
- hadoop-base
- hadoop-datanode
- hadoop-historyserver
- haddop-namenode
- hadoop-nodemanager
- hadoop-resourcemanager
#### Hive
- hive-base
- hive-metastore
- hive-server
- hive-webhcat
#### Hue - an open-source SQL Cloud Editor, licensed under the Apache License 2.0
#### Livy - an open source REST interface for interacting with Apache Spark from anywhere
#### Postgres
#### Spark
- spark-base
- spark-master
- spark-submit
- spark-worker
#### Vagrant 
- To create and configure lightweight, reproducible and portable development environments
- Used for creating the Kubernetes clusterlocally

```
from airflow import DAG
from airflow.sensors.http_sensor import HttpSensor
from airflow.contrib.sensors.file_sensor import FileSensor
from airflow.operators.python_operator import PythonOperator
from airflow.operators.bash_operator import BashOperator
from airflow.operators.hive_operator import HiveOperator
from airflow.operators.email_operator import EmailOperator
from airflow.operators.slack_operator import SlackAPIPostOperator
from airflow.contrib.operators.spark_submit_operator import SparkSubmitOperator
from datetime import datetime, timedelta

import csv
import requests
import json

default_args = {
            "owner": "airflow",
            "start_date": datetime(2019, 1, 1),
            "depends_on_past": False,
            "email_on_failure": False,
            "email_on_retry": False,
            "email": "youremail@host.com",
            "retries": 1,
            "retry_delay": timedelta(minutes=5)
        }

# Download forex rates according to the currencies we want to watch
# described in the file forex_currencies.csv
def download_rates():
    with open('/usr/local/airflow/dags/files/forex_currencies.csv') as forex_currencies:
        reader = csv.DictReader(forex_currencies, delimiter=';')
        for row in reader:
            base = row['base']
            with_pairs = row['with_pairs'].split(' ')
            indata = requests.get('https://api.exchangeratesapi.io/latest?base=' + base).json()
            outdata = {'base': base, 'rates': {}, 'last_update': indata['date']}
            for pair in with_pairs:
                outdata['rates'][pair] = indata['rates'][pair]
            with open('/usr/local/airflow/dags/files/forex_rates.json', 'a') as outfile:
                json.dump(outdata, outfile)
                outfile.write('\n')
```
with DAG(dag_id="forex_data_pipeline_v_9", schedule_interval="@daily", default_args=default_args, catchup=False) as dag:

### Checking the forex API
```    
    is_forex_rates_available = HttpSensor(
        task_id="is_forex_rates_available",
        method="GET",
        http_conn_id="forex_api",
        endpoint="latest",
        response_check=lambda response: "rates" in response.text,
        poke_interval=5,
        timeout=20
    )
```

### Checking the file having the forex pairs to watch 
```
is_forex_currencies_file_available = FileSensor(
        task_id="is_forex_currencies_file_available",
        fs_conn_id="forex_path",
        filepath="forex_currencies.csv",
        poke_interval=5,
        timeout=20
    )
```
### Downloading the forex rates with Python
```
downloading_rates = PythonOperator(
            task_id="downloading_rates",
            python_callable=download_rates
    )
```
### Saving the rates in the HDFS
```
saving_rates = BashOperator(
        task_id="saving_rates",
        bash_command="""
            hdfs dfs -mkdir -p /forex && \
            hdfs dfs -put -f $AIRFLOW_HOME/dags/files/forex_rates.json /forex
            """
    )
```

### Creating a table for storing the rates with Hive
```
creating_forex_rates_table = HiveOperator(
        task_id="creating_forex_rates_table",
        hive_cli_conn_id="hive_conn",
        hql="""
            CREATE EXTERNAL TABLE IF NOT EXISTS forex_rates(
                base STRING,
                last_update DATE,
                eur DOUBLE,
                usd DOUBLE,
                nzd DOUBLE,
                gbp DOUBLE,
                jpy DOUBLE,
                cad DOUBLE
                )
            ROW FORMAT DELIMITED
            FIELDS TERMINATED BY ','
            STORED AS TEXTFILE
        """
    )
```
 ### Processing the raTes with Spark
```
    forex_processing = SparkSubmitOperator(
        task_id="forex_processing",
        conn_id="spark_conn",
        application="/usr/local/airflow/dags/scripts/forex_processing.py",
        verbose=False
    )
```

 ### Sending email to notify the data pipeline owner
```  sending_email_notification = EmailOperator(
            task_id="sending_email",
            to="airflow_course@yopmail.com",
            subject="forex_data_pipeline",
            html_content="""
                <h3>forex_data_pipeline succeeded</h3>
            """
            )

 ```
 ### Sending a Slack notification
 ```
 sending_slack_notification = SlackAPIPostOperator(
        task_id="sending_slack",
        slack_conn_id="slack_conn",
        username="airflow",
        text="DAG forex_data_pipeline: DONE",
        channel="#airflow-exploit"
    )
```
