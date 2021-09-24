# Notes from implementing a Glue job with PySpark
Date: 2021-09-24

## Example script

This script fetches VehiclePositions files from the given S3 bucket/prefix
(`s3://mbta-gtfs-s3/2021/09/06`), flattens them into a table, and exports the
table back to S3 (`s3://mbta-ctd-tmp/pswartz/vehiclepositions-2021-09-06`) as
Parquet files.

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.dynamicframe import DynamicFrame
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

## @params: [JOB_NAME]
args = getResolvedOptions(sys.argv, ['JOB_NAME'])

# Glue boilerplate=
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

import fnmatch
import datetime
import json
import gzip
import boto3
from pyspark.sql import Row

def iter_bucket_glob(bucket, prefix, glob, client = None):
    """
    Iterate over all objects in an S3 bucket with a given prefix, matching a given glob.
    """
    if client is None:
        client = boto3.client('s3')
    response = client.list_objects_v2(Bucket=bucket, Prefix=prefix)
    while True:
        keys = [c['Key'] for c in response['Contents']]
        yield from fnmatch.filter(keys, glob)
        # break
        if response['IsTruncated']:
            token = response['NextContinuationToken']
            response = client.list_objects_v2(Bucket=bucket, Prefix=prefix, ContinuationToken=token)
        else:
            break

def load_from_s3(bucket, key, client=None):
    """
    Parse a given S3 object as a VehiclePositions JSON file.
    """
    if client is None:
        client = boto3.client('s3')
    response = client.get_object(Bucket=bucket, Key=key)
    if response['ResponseMetadata']['HTTPHeaders'].get('content-encoding') == 'gzip':
        body = json.loads(gzip.decompress(response['Body'].read()))
    else:
        body = json.load(response['Body'])
    # some support in Spark/Parquet for Python data types such as datetime
    feed_timestamp = datetime.datetime.utcfromtimestamp(body['header']['timestamp'])
    entities = body['entity']
    for e in entities:
        v = e['vehicle']
        trip = v.get('trip', {})
        yield Row(
            feed_timestamp=feed_timestamp,
            vehicle_id=v['vehicle']['id'],
            current_status=v['current_status'],
            current_stop_sequence=v.get('current_stop_sequence'),
            occupancy_status=v.get('occupancy_status'),
            bearing=v['position'].get('bearing'),
            latitude=v['position']['latitude'],
            longitude=v['position']['longitude'],
            stop_id=v.get('stop_id'),
            timestamp=datetime.datetime.utcfromtimestamp(v['timestamp']),
            direction_id=trip.get('direction_id'),
            route_id=trip['route_id'],
            schedule_relationship=trip['schedule_relationship'],
            start_date=trip.get('start_date'),
            start_time=trip.get('start_time'),
            trip_id=trip['trip_id'],
            label=v['vehicle'].get('label')
        )


# normally we could use Glue/PySpark's built-in S3 handling, but our S3 files have a colon in them.
# this breaks the built-in S3 handling: see https://issues.apache.org/jira/browse/HADOOP-14217
# instead, we use Boto3 to generate an iterable of all the relevant files
client = boto3.client('s3')
i = iter_bucket_glob("mbta-gtfs-s3", "2021/09/06", "*realtime_VehiclePositions_enhanced.json*", client=client)

# then, we can use PySpark's `parallelize` to download the files in parallel
rdd = sc.parallelize(i, numSlices=1000)
rdd = rdd.flatMap(lambda key: load_from_s3("mbta-gtfs-s3", key))
# make a DataFrame, keeping only one GPS ping per vehicle/time, even if it's present in multiple feeds
df = rdd.toDF().dropDuplicates(["vehicle_id", "timestamp"])
# DynamicFrame is like a DataFrame, but has some Glue-specific functionality
df_dyf = DynamicFrame.fromDF(df, glueContext, "vehiclepositions-2021-09-06")
glueContext.write_dynamic_frame.from_options(
       frame = df_dyf,
       connection_type = "s3",
       connection_options = {"path": "s3://mbta-ctd-tmp/pswartz/vehiclepositions-2021-09-06"},
       format = "parquet")

# PySpark also has writing-Parquet-to-S3 support, but we don't use it here
#df.write.parquet("s3a://mbta-ctd-tmp/pswartz/vehiclepositions-2021-09-06")

job.commit()
```

## Glue Crawler / Tables
To pull these Parquet files into Glue for later querying, we need a Database
and a Crawler. The Database is a name for a collection of tables: I used
`mbta-gtfs-s3` as that's also the name of the S3 bucket which has the source
data. The temporary crawler generated here
(https://console.aws.amazon.com/glue/home?region=us-east-1#crawler:name=tmp-vehiclepositions-2021-09-06)
looks at the S3 prefix which contains our Parquet files, examines them for
the schema, and then generates the metadata for the table in Glue
(https://console.aws.amazon.com/glue/home?region=us-east-1#table:catalog=434035161053;name=vehiclepositions_2021_09_06;namespace=mbta-gtfs-s3)


## Queries
Once the table is present in Glue, we can query it with Athena, like:

```sql
SELECT * FROM "mbta-gtfs-s3"."vehiclepositions_2021_09_06" limit 10;
```

## Open Questions
- when using `dropDuplicates` which row is kept?
- how can we partition the output, so it's in a directory like `vehicle_positions/year=2021/month=9/day=6` ?
- how can we integrate this with Tableau, or QuickSight?
