id: getting-started-with-snowpipe-streaming-azure-eventhubs
categories: snowflake-site:taxonomy/solution-center/certification/quickstart, snowflake-site:taxonomy/product/platform
language: en
summary: This guide will walk you through how to ingest real-time data into Snowflake using the High Performance (HP) Kafka Connector (v4.x) with Snowpipe Streaming and Azure Event Hubs.
environments: web
status: Published
feedback link: https://github.com/Snowflake-Labs/sfguides/issues
authors: James Sun


# Getting Started with Snowpipe Streaming and Azure Event Hubs
<!---------------------------->
## Overview

Snowflake's Snowpipe streaming capabilities are designed for rowsets with variable arrival frequency.
It focuses on lower latency and cost for smaller data sets. This helps data workers stream rows into Snowflake
without requiring files with a more attractive cost/latency profile.

Here are some of the use cases that can benefit from Snowpipe streaming:
- IoT time-series data ingestion
- CDC streams from OLTP systems
- Log ingestion from SIEM systems
- Ingestion into ML feature stores

In our demo, we will use real-time commercial flight data over the San Francisco Bay Area from the [Opensky Network](https://opensky-network.org) to illustrate the solution using 
Snowflake's Snowpipe streaming and [Azure Event Hubs](https://azure.microsoft.com/en-us/products/event-hubs).

The architecture diagram below shows the deployment. An Azure Event Hub and an [Azure Linux 
Virtual Machine](https://azure.microsoft.com/en-us/products/virtual-machines) (jumphost) will be provisioned in an [Azure Virtual Network](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview). 
The Linux jumphost will host the Kafka producer and Snowpipe streaming via [Kafka Connect](https://docs.snowflake.com/en/user-guide/kafka-connector-overview.html).

The Kafka producer calls the data sources' REST API and receives time-series data in JSON format. This data is then ingested into the Kafka cluster before being picked up by the [Snowflake High Performance (HP) Kafka Connector](https://docs.snowflake.com/en/connectors/kafkahp/setup-kafka) (v4.x) and delivered to a Snowflake table. The HP connector uses a server-side architecture with a PIPE object in Snowflake that manages data processing and buffering, delivering up to 10 GB/s throughput per table with 5-10 second latency.

> **Note:** This quickstart uses the Snowflake High Performance (HP) Kafka Connector (v4.x), which is currently in **Public Preview**.

The data in Snowflake table can be visualized in real-time with [Azure Managed Grafana](https://azure.microsoft.com/en-us/products/managed-grafana) and [Streamlit](https://streamlit.io)
The historical data can also be analyzed by BI tools like [Microsoft Power BI on Azure](https://azure.microsoft.com/en-us/products/power-bi).

![Architecture diagram for the Demo](assets/Overview-2-flight-arch.png)

![Data visualization](assets/Overview-2-dashboarding.png)

### Prerequisites

- Familiarity with Snowflake, basic SQL knowledge, Snowsight UI and Snowflake objects
- Familiarity with Azure Services, Networking and the Management Console
- Basic knowledge of Python and Linux shell scripting

### What You'll Need Before the Lab

To participate in the virtual hands-on lab, attendees need the following resources.

- A [Snowflake Enterprise Account on Azure](https://signup.snowflake.com/?utm_source=snowflake-devrel&utm_medium=developer-guides&utm_cta=developer-guides) with `ACCOUNTADMIN` access
- An [Azure Account](https://learn.microsoft.com/en-us/dotnet/azure/create-azure-account) with administrator privileges.

### What You'll Learn

- Using [Azure Event Hubs](https://azure.microsoft.com/en-us/products/event-hubs).
- Using the [Snowflake High Performance (HP) Kafka Connector](https://docs.snowflake.com/en/connectors/kafkahp/setup-kafka) for low-latency data ingestion
- Using Snowflake to query tables populated with time-series data

### What You'll Build

- [Create an Azure Event Hub](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-create)
- Create Kafka producers and connectors
- Create an Event hub (Kafka topic) for data streaming 
- A Snowflake database to receive real-time flight data

<!---------------------------->
## Create an Event Hub and a VM in Azure

#### 1. Create a resource group

Follow this [Azure doc](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal) to create a resource group, say `Streaming`.

![](assets/rg.png)

#### 2. Create an Event Hub in the resource group

Go to the newly created resource group, and click `+ Create` tab to create an event hub by
following this [Azure doc](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-create). Select the `Standard` [pricing tier](https://azure.microsoft.com/en-us/pricing/details/event-hubs/) to use Apache Kafka. Make sure that you select public access to the Event Hub in the networking setting.

See below sample screen capture for reference, here we have created a namespace called `SnowflakeTest`.

![](assets/eventhubs.png)

#### 3. Create a Linux VM

In the same resource group, create a Linux(Red Hat enterprise) virtual machine by
following this [doc](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal?tabs=ubuntu). Choose `Redhat Enterprise` as the image. Note that this quickstart guide was written using scripts based on the RedHat syntax, optionally you can select Ubuntu or other Linux distributions but will need to modify the scripts.

Also make sure that you allow ssh access to the VM in the network rule setting.

Download and save the private key for use in the next step.

Once the VM is provisioned, we will then use it to run the Kafka connector with Snowpipe streaming SDK and the producer. We will be using the default VM user `azureuser` for this workshop.

Here is a screen capture of the VM overview for reference.

![](assets/vm-overview.png)

#### 4. Connect to the Linux VM console

From you local machine, either using a ssh application such as [Putty](https://www.putty.org/) if you have a Windows PC or simply the ssh CLI (`ssh -i <your private_key for the VM> <VM's public IP address> -l azureuser)` for Linux or Mac based systems.

#### 5. Create a key-pair to be used for authenticating with Snowflake

Create a key pair in the VM console by executing the following commands. You will be prompted to give an encryption password, remember 
this phrase, you will need it later.

```commandline
cd $HOME
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8
```
See below example screenshot:

![](assets/key-pair-sessionmgr-1.png)

Next we will create a public key by running following commands. You will be prompted to type in the phrase you used in above step.
```
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```
see below example screenshot:

![](assets/key-pair-sessionmgr-2.png)

Next we will print out the public key string in a correct format that we can use for Snowflake.
```
grep -v KEY rsa_key.pub | tr -d '\n' | awk '{print $1}' > pub.Key
cat pub.Key
```
see below example screenshot:

![](assets/key-pair-sessionmgr-3.png)


#### 6. Install the Kafka connector for Snowpipe streaming

Run the following command to install the Kafka connector and Snowpipe streaming SDK

```commandline
passwd=changeit  # Use the default password for the Java keystore, you should change it after finishing the lab
directory=/home/azureuser/snowpipe-streaming # Installation directory

cd $HOME
mkdir -p $directory
cd $directory
pwd=`pwd`
sudo yum clean all
sudo yum -y install openssl vim-common java-11-openjdk-devel gzip tar jq python3-pip
wget https://archive.apache.org/dist/kafka/3.7.2/kafka_2.13-3.7.2.tgz
tar xvfz kafka_2.13-3.7.2.tgz -C $pwd
rm -rf $pwd/kafka_2.13-3.7.2.tgz
cd /tmp && cp $(find /usr/lib/jvm -name cacerts 2>/dev/null | head -1) kafka.client.truststore.jks
cd /tmp && keytool -genkey -keystore kafka.client.keystore.jks -validity 300 -storepass $passwd -keypass $passwd -dname "CN=snowflake.com" -alias snowflake -storetype pkcs12

#Snowflake High Performance (HP) Kafka connector v4.x (Public Preview) — uses server-side Snowpipe Streaming architecture
#v4.x is an uber-jar that bundles snowflake-ingest-sdk and snowflake-jdbc internally
wget https://repo1.maven.org/maven2/com/snowflake/snowflake-kafka-connector/4.0.0-rc8/snowflake-kafka-connector-4.0.0-rc8.jar -O $pwd/kafka_2.13-3.7.2/libs/snowflake-kafka-connector-4.0.0-rc8.jar

wget https://repo1.maven.org/maven2/org/bouncycastle/bc-fips/2.1.0/bc-fips-2.1.0.jar -O $pwd/kafka_2.13-3.7.2/libs/bc-fips-2.1.0.jar
wget https://repo1.maven.org/maven2/org/bouncycastle/bcpkix-fips/2.1.8/bcpkix-fips-2.1.8.jar -O $pwd/kafka_2.13-3.7.2/libs/bcpkix-fips-2.1.8.jar

```
Note that the version numbers for Kafka, the Snowflake Kafka connector, and the Snowpipe Streaming SDK are dynamic, as new versions are continually published. We are using the version numbers that have been validated to work.

#### 7. Retrieve the connection string 

Go to the Event Hubs console, select the namespace you just created, then select `Settings` and `Shared access policies` on the left pane and click `RootManageSharedAccessKey` policy. Then copy the `Connection string-primary key`, see screenshot below.

See example screenshot below.

![](assets/bs-4.png)

Now switch back to the VM console and execute the following command by replacing `<connection string>` with 
the copied values. DO NOT forget to include the double quotes.

```commandline
export CS="<connection string>"
```
see the example screen capture below.
![](assets/bs-5.png)

We also need to extract the kafka broker string(BS) from the connection string by running this command:
```
export BS=`echo $CS | awk -F\/ '{print $3":9093"}'`
```

Now run the following command to add `CS` as an environment variable so it is recognized across the Linux sessions.

```
echo "export CS=\"$CS\"" >> ~/.bashrc
echo "export BS=$BS" >> ~/.bashrc

```

#### 8. Create a configuration file for the Kafka connector

Run the following commands to generate a configuration file `connect-standalone.properties` in directory `/home/azureuser/snowpipe-streaming/scripts` for the client to authenticate with the Event hubs namespace.

```commandline
dir=/home/azureuser/snowpipe-streaming/scripts
mkdir -p $dir && cd $dir
cat << EOF > $dir/connect-standalone.properties
#************CREATING SNOWFLAKE Connector****************
bootstrap.servers=$BS

#************SNOWFLAKE VALUE CONVERSION****************
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false
#************SNOWFLAKE ****************

offset.storage.file.filename=/tmp/connect.offsets
# Flush much faster than normal, which is useful for testing/debugging
offset.flush.interval.ms=10000

# required EH Kafka security settings
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="\$ConnectionString" password="$CS";
consumer.security.protocol=SASL_SSL
consumer.sasl.mechanism=PLAIN
consumer.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="\$ConnectionString" password="$CS";
EOF

```
A configuration file `connect-standalone.properties` is created in directory `/home/azureuser/snowpipe-streaming/scripts`

#### 9. Create a security configuration file for the producer

Run the following commands to create a security configuration file `client.properties`.

```commandline
dir=/home/azureuser/snowpipe-streaming/scripts
cat << EOF > $dir/client.properties
# required EH Kafka security settings
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="\$ConnectionString" password="$CS";
EOF

```

A configuration file `client.properties` is created in directory `/home/azureuser/snowpipe-streaming/scripts`

#### 10. Create an event hub in the namespace

Go to the Event Hubs namespace and clicke `+ Event Hub`.

![](assets/create-topic-1.png)

Give the event hub a name, say `streaming`, then click `Next: Capture`.

![](assets/create-topic-2.png)

Leave everything as default, and click `Next: Review + create`.

![](assets/create-topic-3.png)

Click `Create` to create the event hub.

![](assets/create-topic-4.png)

Scroll down to the list of event hubs, you will see the `streaming` event hub there.

![](assets/create-topic-5.png)


To describe the topic, run the following commands in the VM shell:

```commandline
$HOME/snowpipe-streaming/kafka_2.13-3.7.2/bin/kafka-topics.sh --bootstrap-server $BS --command-config $HOME/snowpipe-streaming/scripts/client.properties --describe --topic streaming
```

See below example screenshot:

![](assets/list-kafka-topic.png)

<!---------------------------->
## Prepare the Snowflake account for streaming

#### 1. Creating user, role, database, and schema
Login to your Snowflake account as a power user with ACCOUNTADMIN role. 
Then run the following SQL commands in a worksheet to create the dedicated streaming user, role, database, schema, and warehouse that we will use in the lab. The streaming user uses key-pair authentication only (no password).

```
-- Set default value for multiple variables
-- For purpose of this workshop, it is recommended to use these defaults during the exercise to avoid errors
-- You should change them after the workshop
SET USER = 'STREAMING_USER';
SET DB = 'AZ_STREAMING_DB';
SET SCHEMA = 'AZ_STREAMING_SCHEMA';
SET WH = 'AZ_STREAMING_WH';
SET ROLE = 'AZ_STREAMING_RL';

USE ROLE ACCOUNTADMIN;

-- CREATE ROLE
CREATE OR REPLACE ROLE IDENTIFIER($ROLE);

-- CREATE USER (key-pair auth only, no password)
-- Uses IF NOT EXISTS so it won't fail if the user already exists.
-- The ALTER USER below ensures correct settings even for a pre-existing user.
CREATE USER IF NOT EXISTS IDENTIFIER($USER)
  DEFAULT_ROLE = $ROLE
  DEFAULT_WAREHOUSE = $WH
  COMMENT = 'Streaming connector user - key-pair auth only';

ALTER USER IF EXISTS IDENTIFIER($USER) SET
  DEFAULT_ROLE = $ROLE
  DEFAULT_WAREHOUSE = $WH
  DISABLED = FALSE;

-- CREATE DATABASE, SCHEMA AND WAREHOUSE
CREATE DATABASE IF NOT EXISTS IDENTIFIER($DB);
CREATE OR REPLACE SCHEMA IDENTIFIER($DB).IDENTIFIER($SCHEMA);
CREATE OR REPLACE WAREHOUSE IDENTIFIER($WH) WITH WAREHOUSE_SIZE = 'SMALL';

-- GRANTS
GRANT ROLE IDENTIFIER($ROLE) TO USER IDENTIFIER($USER);
GRANT OWNERSHIP ON DATABASE IDENTIFIER($DB) TO ROLE IDENTIFIER($ROLE) COPY CURRENT GRANTS;
GRANT OWNERSHIP ON SCHEMA IDENTIFIER($DB).IDENTIFIER($SCHEMA) TO ROLE IDENTIFIER($ROLE) COPY CURRENT GRANTS;
GRANT USAGE ON WAREHOUSE IDENTIFIER($WH) TO ROLE IDENTIFIER($ROLE);
GRANT CREATE TABLE ON SCHEMA IDENTIFIER($DB).IDENTIFIER($SCHEMA) TO ROLE IDENTIFIER($ROLE);
GRANT CREATE PIPE ON SCHEMA IDENTIFIER($DB).IDENTIFIER($SCHEMA) TO ROLE IDENTIFIER($ROLE);

-- RUN FOLLOWING COMMANDS TO FIND YOUR ACCOUNT IDENTIFIER, COPY IT DOWN FOR USE LATER
-- IT WILL BE SOMETHING LIKE <organization_name>-<account_name>
-- e.g. ykmxgak-wyb52636

WITH HOSTLIST AS 
(SELECT * FROM TABLE(FLATTEN(INPUT => PARSE_JSON(SYSTEM$allowlist()))))
SELECT REPLACE(VALUE:host,'.snowflakecomputing.com','') AS ACCOUNT_IDENTIFIER
FROM HOSTLIST
WHERE VALUE:type = 'SNOWFLAKE_DEPLOYMENT_REGIONLESS';

```
Please write down the Account Identifier, we will need it later.
![](assets/account-identifier.png)

Next we need to configure the public key for the streaming user to access Snowflake programmatically.

In the Snowflake worksheet, replace `< pubKey >` with the content of the file `/home/azureuser/pub.Key` (see `step 5` by clicking on `section #2 Create an Event Hub and a Linux virtual machine in Azure cloud` in the left pane) in the following SQL command and execute.
```commandline
USE ROLE ACCOUNTADMIN;
ALTER USER STREAMING_USER SET RSA_PUBLIC_KEY='< pubKey >';
```
See below example screenshot:

![](assets/key-pair-snowflake.png)

At this point, the Snowflake setup is complete.

<!---------------------------->
## Configure the Kafka connector for Snowpipe Streaming

#### 1. Collect parameters for the Kafka connector

Run the following commands to collect various connection parameters for the Kafka connector.

```commandline
cd $HOME
outf=/tmp/params
cat << EOF > /tmp/get_params
a=''
until [ ! -z \$a ]
do
 read -p "Input Snowflake account identifier: e.g. ylmxgak-wyb53646 ==> " a
done

echo export clstr_url=\$a.snowflakecomputing.com > $outf
export clstr_url=\$a.snowflakecomputing.com

read -p "Snowflake cluster user name: default: streaming_user ==> " user
if [[ \$user == "" ]]
then
   user="streaming_user"
fi

echo export user=\$user >> $outf
export user=\$user

pass=''
until [ ! -z \$pass ]
do
  read -p "Private key passphrase ==> " pass
done

echo export key_pass=\$pass >> $outf
export key_pass=\$pass

read -p "Full path to your Snowflake private key file, default: /home/azureuser/rsa_key.p8 ==> " p8
if [[ \$p8 == "" ]]
then
   p8="/home/azureuser/rsa_key.p8"
fi

priv_key=\`cat \$p8 | grep -v PRIVATE | tr -d '\n'\`
echo export priv_key=\$priv_key  >> $outf
export priv_key=\$priv_key
cat $outf >> $HOME/.bashrc
EOF
. /tmp/get_params

```
See below example screen capture.

![](assets/get_params.png)

#### 2. Create a Snowflake Kafka connect configuration file

Note that the High Performance (HP) connector (v4.x) will auto-create a default PIPE object named `AZ_STREAMING_TBL-STREAMING` in the target schema when the connector starts. This is why the `GRANT CREATE PIPE` was added earlier.

Run the following commands to generate a configuration file for the Kafka connector.

```commandline
dir=/home/azureuser/snowpipe-streaming/scripts
cat << EOF > $dir/snowflakeconnectorAZ.properties
name=snowpipeStreamingHP
connector.class=com.snowflake.kafka.connector.SnowflakeStreamingSinkConnector
tasks.max=4
topics=streaming
snowflake.private.key.passphrase=$key_pass
snowflake.database.name=AZ_STREAMING_DB
snowflake.schema.name=AZ_STREAMING_SCHEMA
snowflake.topic2table.map=streaming:AZ_STREAMING_TBL
snowflake.url.name=$clstr_url
snowflake.user.name=$user
snowflake.private.key=$priv_key
snowflake.role.name=AZ_STREAMING_RL
snowflake.ingestion.method=SNOWPIPE_STREAMING
snowflake.streaming.v2.enabled=true
snowflake.enable.schematization=TRUE
buffer.count.records=10000
buffer.flush.time=10
buffer.size.bytes=20000000
value.converter.schemas.enable=false
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
errors.tolerance=all
errors.log.enable=true
EOF
```

<!---------------------------->
## Putting it all together

Finally, we are ready to start ingesting data into the Snowflake table.

#### 1. Start the Kafka Connector for Snowpipe streaming

> **Important:** If your Snowflake account has a [network policy](https://docs.snowflake.com/en/user-guide/network-policies) enabled, make sure the VM's public IP is included in the allowed IP list. Otherwise, the connector will fail to connect to Snowflake with an IP access denied error.

Go back to the Linux console and execute the following commands to start the Kafka connector.
```commandline
$HOME/snowpipe-streaming/kafka_2.13-3.7.2/bin/connect-standalone.sh $HOME/snowpipe-streaming/scripts/connect-standalone.properties $HOME/snowpipe-streaming/scripts/snowflakeconnectorAZ.properties
```

If everything goes well, you should see the connector start up and begin polling the topic for messages.

Leave this screen open and let the connector continue to run.

#### 2. Start the producer

Open up a new ssh session connection to the VM. In the shell, run the following command:

```commandline
curl --connect-timeout 5 http://ecs-alb-1504531980.us-west-2.elb.amazonaws.com:8502/opensky | \
jq -c '.[]' | \
$HOME/snowpipe-streaming/kafka_2.13-3.7.2/bin/kafka-console-producer.sh --broker-list $BS --producer.config $HOME/snowpipe-streaming/scripts/client.properties --topic streaming
```
You should see the curl output with a successful HTTP response.

Note that in the script above, the producer queries a [Rest API](http://ecs-alb-1504531980.us-west-2.elb.amazonaws.com:8502/opensky ) that provides real-time flight data over the San Francisco 
Bay Area in JSON format. The API returns a JSON array, so we use `jq -c '.[]'` to break it into individual JSON objects — one per Kafka message. This is required because the HP connector with schematization enabled expects each message to be a flat JSON object whose keys map to table columns.

The data includes information such as timestamps, [icao](https://icao.usmission.gov/mission/icao/#:~:text=Home%20%7C%20About%20the%20Mission%20%7C%20U.S.,civil%20aviation%20around%20the%20world.) numbers, flight IDs, destination airport, longitude, 
latitude, and altitude of the aircraft, etc. The data is ingested into the `streaming` topic on the event hub and 
then picked up by the Snowpipe streaming Kafka connector, which delivers it directly into a Snowflake 
table `az_streaming_db.az_streaming_schema.az_streaming_tbl`.

![](assets/flight-json.png)

<!---------------------------->
## Query ingested data in Snowflake

Now, switch back to the Snowflake console and switch to the role `AZ_STREAMING_RL`. 
The data should have been streamed into a table, ready for further processing.

#### 1. Query the raw data
To verify that data has been streamed into Snowflake, execute the following SQL commands.

```sh
use az_streaming_db;
use schema az_streaming_schema;

-- Check the default pipe status (auto-created by HP connector)
SELECT SYSTEM$PIPE_STATUS('AZ_STREAMING_DB.AZ_STREAMING_SCHEMA."AZ_STREAMING_TBL-STREAMING"');

-- Show all pipes in the schema
SHOW PIPES IN SCHEMA AZ_STREAMING_DB.AZ_STREAMING_SCHEMA;
```
You should see the auto-created pipe `AZ_STREAMING_TBL-STREAMING` with `executionState=RUNNING`.

Note that, at this point, you should only see one batch of rows in the table, as we have only ingested data once. We will see new rows being added later as we continue to ingest more data.

Now run the following query on the table.
```
select * from az_streaming_tbl;
```
With schematization enabled, the HP connector automatically creates columns from the JSON keys. You should see columns like `RECORD_METADATA`, `ID`, `ICAO`, `LAT`, `LON`, `ALT`, `ORIG`, `DEST`, and `UTC` — no manual flattening needed.

![](assets/raw_data.png)

#### 2. Create a view with derived columns
Now execute the following SQL commands to create a convenience view with timestamps and geohashes for visualization.

```sh
create or replace view flights_vw
  as select
    to_timestamp_ntz(utc) as ts_utc,
    CONVERT_TIMEZONE('UTC','America/Los_Angeles',ts_utc) as ts_pt,
    alt,
    dest,
    orig,
    id,
    icao,
    lat,
    lon,
    st_geohash(to_geography(ST_MAKEPOINT(lon, lat)),12) geohash,
    year(ts_pt) yr,
    month(ts_pt) mo,
    day(ts_pt) dd,
    hour(ts_pt) hr
FROM   az_streaming_tbl;
```

The SQL commands create a view that converts the epoch timestamp to a proper timestamp, applies time zone conversion, and uses Snowflake's [Geohash function](https://docs.snowflake.com/en/sql-reference/functions/st_geohash.html) to generate geohashes that can be used in time-series visualization tools like Grafana.

Let's query the view `flights_vw` now.
```sh
select * from flights_vw;
```

As a result, you will see a nicely structured output with derived timestamp and geohash columns.
![](assets/materialized_view.png)

#### 3. Stream real-time flight data continuously to Snowflake

We can now write a loop to stream the flight data continuously into Snowflake.

Go back to the Linux session and run the following script.

```sh
while true
do
  curl --connect-timeout 5 -k http://ecs-alb-1504531980.us-west-2.elb.amazonaws.com:8502/opensky | \
  jq -c '.[]' | \
  $HOME/snowpipe-streaming/kafka_2.13-3.7.2/bin/kafka-console-producer.sh --broker-list $BS --producer.config $HOME/snowpipe-streaming/scripts/client.properties --topic streaming
  sleep 10
done

```
You can now go back to the Snowflake worksheet to run a `select count(1) from flights_vw` query every 10 seconds to verify that the row counts is indeed increasing.

<!---------------------------->
## Cleanup

When you are done with the demo, to tear down the Azure resources, follow this [doc](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/delete-resource-group?tabs=azure-powershell) to dismantle the resource group and its associated resources. 

For Snowflake cleanup, execute the following SQL commands.

```commandline
USE ROLE ACCOUNTADMIN;

-- Drop the auto-created default pipe (HP connector)
DROP PIPE IF EXISTS AZ_STREAMING_DB.AZ_STREAMING_SCHEMA."AZ_STREAMING_TBL-STREAMING";

DROP DATABASE AZ_STREAMING_DB;
DROP WAREHOUSE AZ_STREAMING_WH;
DROP ROLE AZ_STREAMING_RL;

-- Drop the streaming user
DROP USER IF EXISTS STREAMING_USER;
```

<!---------------------------->
## Conclusion and Resources

In this lab, we built a demo to show how to ingest time-series data using the Snowflake High Performance (HP) Kafka Connector (v4.x) with Snowpipe Streaming and Azure Event Hubs with low latency. We demonstrated this using a self-managed Kafka connector on an Azure VM. You can also containerize the connector on the [Azure Kubernetes Services (AKS)](https://azure.microsoft.com/en-us/products/kubernetes-service) to leverage the benefits of scalability and manageability.

Related Resources

- [Snowflake High Performance (HP) Kafka Connector](https://docs.snowflake.com/en/connectors/kafkahp/setup-kafka)
- [Snowflake Kafka Connector Overview](https://docs.snowflake.com/en/user-guide/kafka-connector-overview)
- [Snowpipe Streaming Overview](https://docs.snowflake.com/en/user-guide/data-load-snowpipe-streaming-overview)
- [Getting started with Snowflake](/en/developers/guides/)
- [Snowflake Marketplace](/en/data-cloud/marketplace/)
- [Azure Event Hubs](https://azure.microsoft.com/en-us/products/event-hubs)
- [Snowflake on Azure Marketplace](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/snowflake.snowflake_contact_me?tab=overview)


