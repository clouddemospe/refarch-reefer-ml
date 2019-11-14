# Generate telemetry data

We are using the simulator to generate data. In the industry, when developing new manufactured product, the engineers do not have a lot of data so they also use a mix of real sensors with simulators to create fake but realistic data to develop and test their models.

The historical data need to represent failure, and represent the characteristics of a Reefer container. We have define some sensors to get interesting correlated or independant features.

As of now our telemetry event structure can be seen in this [avro schema](https://github.com/ibm-cloud-architecture/refarch-reefer-ml/tree/master/data_schema/reefer_telemetry_value.avsc).


For the machine learning environment we can use csv file as input data or postgresql database or kafka topic. The environment looks like in the figure below:

![](images/data-collect.png)

The simulator can run as a standalone tool to create training and test data to be saved in a remote postgresql database or can be used to save to csv file. We use postgresql as a service on IBM cloud. The service has credential with URL and SSL certificate.

![](images/postgres-credential.png)

## Start python env

```
 ./startPythonEnv IBMCLOUD or LOCAL

root@03721594782f: cd /home/simulator
```

In the Dockerfile we set the first PYTHONPATH to /home to specify where python should find the new modules.

```
root@03721594782f:/home # 
```

#### Generate power off metrics

When the reefer containers lose power at some time, then restart and reloose it, it may become an issue.

The simulator accepts different arguments as specified below 

```
usage reefer_simulator-tool 
     --stype [poweroff | co2sensor | o2sensor | normal]
	  --cid [C01 | C02 | C03 | C04]
	 --product_id [ P01 | P02 | P03 | P04 ]
	 --records <the number of records to generate>
	 --file <the filename to create>
	 --append
	 --db
```

* The `cid` is for the container id. As the simulator is taking some data from internal datasource you can use only one of those values: `[C01 | C02 | C03 | C04]`
* `product_id` is also one of the value `[ P01 | P02 | P03 | P04 ]`, as the simulator will derive the target temperature and humidity level from its internal datasource:
    ('P01','Carrots',1,4,0.4),
    ('P02','Banana',2,6,0.6),
    ('P03','Salad',1,4,0.4),
    ('P04','Avocado',2,6,0.4);
* `--db` is when you want to save in a postgresql DB. In this case be sure to have set the credentials and URL in the `scripts/setenv.sh` script (see the `scripts/setenv-tmp.sh` template file)
* `--file` is to specify a csv file to write the data
* `--append` is used to append the output of this run to an existing file: It permits to accumulate different simulation in the same dataset.

* (Re)create a new file. It is an important step to get the header as first row.
```
python simulator/reefer_simulator_tool.py --stype poweroff --cid C01 --records 1000 --product_id P02 --file telemetries.csv 
```
* append to existing file
```
python simulator/reefer_simulator_tool.py --cid C03 --product_id P02 --records 1000 --file telemetries.csv --stype poweroff --append
```

The results looks like:
```
    Generating  1000  poweroff metrics

    Timestamp   ID  Temperature(celsius) Target_Temperature(celsius)      Power  PowerConsumption ContentType  O2 CO2  Time_Door_Open Maintenance_Required Defrost_Cycle
    1.000000  2019-06-30 T15:43 Z  101              3.416766                           4  17.698034          6.662044           1  11   1        8.735273                    0             6
    1.001001  2019-06-30 T15:43 Z  101              4.973630                           4   3.701072          8.457314           1  13   3        5.699655                    0             6
    1.002002  2019-06-30 T15:43 Z  101              1.299275                           4   7.629094 
```     

From the two previous commands you should have 2001 rows (one gor the header which will be used in the model creation):
```
wc -l telemetries.csv 
2001 telemetries.csv
```

#### Generate Co2 sensor malfunction in same file

In the same way as above the simulator can generate data for Co2 sensor malfunction using the command:

```
python simulator/reefer_simulator_tool.py --cid C03 --product_id P02 --records 1000 --file basedata --stype co2sensor --append
```


!!! note
        The simulator is integrated in the event producer to send real time events to kafka, as if the Reefer container was loaded with fresh goods and is travelling oversea. A consumer code can call the predictive model to assess if maintenance is required and post new event on a `containers` topic (this consumer code is in the `scoring/eventConsumer` folder).

#### Generate O2 sensor malfunction in same file

```
python simulator/reefer_simulator_tool.py --cid C03 --product_id P02 --records 1000 --file basedata --stype o2sensor --append
```

#### Saving to Postgres

The same tool can be used to save to a postgresql database. First be sure to set at least the following environment variables in the setenv.sh file

```
POSTGRES_URL,  POSTGRES_DBNAME,
```

If you want to use `psql` then you need to set all POSGRES* environment variables.

If you use POSTGRESQL on IBM Cloud or a deployment using SSL, you need to get the SSL certificate and put it as `postgres.pem` under the `simulator` folder, or set `POSTGRES_SSL_PEM` to the path where to find this file.

The `postgres.pem` file needs to be in the simulator folder.

Run the ReeferRepository.py tool to create the database and to add the reference data:

```
./startPythonEnv.sh IBMCLOUD
> python simulator/infrastructure/ReeferRepository.py
```

You should see:
```

```
Connect remote with ssl
('PostgreSQL 10.10 on x86_64-pc-linux-gnu, compiled by gcc (Debian 6.3.0-18+deb9u1) 6.3.0 20170516, 64-bit',)
[
    {
        "container_id": "C01",
        "last_maintenance_date": null,
        "reefer_model": "20RF"
    },
    {
        "container_id": "C02",
        "last_maintenance_date": null,
        "reefer_model": "20RF"
    },
    {
        "container_id": "C03",
        "last_maintenance_date": null,
        "reefer_model": "40RH"
    },
    {
        "container_id": "C04",
        "last_maintenance_date": null,
        "reefer_model": "45RW"
    }
]
[
    {
        "content_type": 1,
        "description": "Carrots",
        "product_id": "P01",
        "target_humidity_level": 0.4,
        "target_temperature": 4.0
    },
    {
        "content_type": 2,
        "description": "Banana",
        "product_id": "P02",
        "target_humidity_level": 0.6,
        "target_temperature": 6.0
    },
    {
        "content_type": 1,
        "description": "Salad",
        "product_id": "P03",
        "target_humidity_level": 0.4,
        "target_temperature": 4.0
    },
    {
        "content_type": 2,
        "description": "Avocado",
        "product_id": "P04",
        "target_humidity_level": 0.4,
        "target_temperature": 6.0
    }
]
('public', 'reefers', 'ibm-cloud-base-user', None, True, False, True, False)
('public', 'products', 'ibm-cloud-base-user', None, True, False, True, False)
('public', 'reefer_telemetries', 'ibm-cloud-base-user', None, True, False, True, False)
```
```

Once done run the simulator with the --db argument like below:

```
python simulator/reefer_simulator_tool.py --cid C03 --product_id P02 --records 1000  --stype poweroff --db

python simulator/reefer_simulator_tool.py --cid C03 --product_id P02 --records 1000  --stype co2sensor --db

python simulator/reefer_simulator_tool.py --cid C03 --product_id P02 --records 1000  --stype o2sensor --db
```

Here is an example of output:

```
Generating records for co2 sensor issue
Generating records for normal behavior
Generate 2000 records for Banana
     container_id           measurement_time product_id  temperature  ...  vent_1  vent_2  vent_3  maintenance_required
0             C03 2019-09-27 05:16:32.015791        P02     5.023500  ...    True    True    True                     0
1             C03 2019-09-27 05:21:32.015791        P02     7.015356  ...    True    True    True                     0
2             C03 2019-09-27 05:26:32.015791        P02     6.106849  ...    True    True    True                     0
3             C03 2019-09-27 05:31:32.015791        P02     6.521214  ...    True    True    True                     0
4             C03 2019-09-27 05:36:32.015791        P02     6.704980  ...    True    True    True                     0
...           ...                        ...        ...          ...  ...     ...     ...     ...       

[2000 rows x 18 columns]
dc2537b....databases.appdomain.cloud:32347
ibmclouddb
Connect remote with ssl
Done uploading telemetry records !
```

To verify the data loaded into the database we use `psql` with the following script:
```

```

Then in the CLI:
```
# get the list of tables
ibmclouddb> \d

ibmclouddb> SELECT * FROM reefers;
ibmclouddb> SELECT * FROM products;
ibmclouddb> SELECT * FROM reefer_telemetries;
```

#### Delete records in database



Next step is to build the model... 

