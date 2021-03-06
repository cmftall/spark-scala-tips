# spark-scala-tips
A collection of Spark (Scala) tips or best practices based on my experience.

## Data skew
### Problem:
A data skew is a condition when few partitions contain much more data than on average other. This can happen when for example you process website visits and partitioning of data is done by the domain name.
Obviously, some sites can have multiple time more visitors than the others. As a result, you will have a few large partitions and many small ones.
And event thous Spark processes partitions in parallel you will get overall pure performance, since each stage will not be finished until large partitions are not processed.
To avoid this conditions, monitor your application in Spark Web UI. You will notice if there are tasks that are taking much longer than the others.
This is a sign of data skew. 
### Solution:
I would say that there is no one solution for all case. But being aware of data skew is the key.
If possible use another partitioning key which provides better distribution. This is not always the case since processing logic defines partitioning. 
Very often data skew may occur when you are joining  dataframes, and in this case, partitioning happens by joining the field.
 In this case, best approach is to broadcast smaller dataframe:
 ```
 val joinedDF = broadcast(df1).join(df2, "id")
 ```
 Also, if partitioning is not a result of processing logic like in case of join, you can repartition dataframes by a column which will provide more even distribution.
 For example, if we are talking about site visits then partitioning by domain name, date or even hour may result in data skew. But if you partition by 
 minute or second this will provide quite even distribution.
 
 
## Use schemas for JSONs
### Problem:
You probably know that Spark provides convenient methods to read and write JSON data. 
Let's focus on reading. If you have dataset consisting of JSONs and you want to load them as dataframe
than you can do this:
```
val df = spark.read.json("path/to/jsons")
```
It is pretty straight forward. And this is probably fine as long as you 100% percent sure that all JSONs have same structure so your data frame will have expected schema. 
But data is not always clean and consistent, so it may occur that when you process another batch of data there will be JSONs with different schema (some fields will miss for example). 
This will result in a wrong interpretation of schema by Spark and farther issues of processing.

### Solution
To avoid this type of issues, we should "help" Spark in schema detection, by providing exact schema:
```
val df = spark.read.schema(schema).json("path/to/jsons")
```
This way if there will be a field missing in JSON, Spark will fill his filed with null instead of omitting it at all. 
The schema is especially helpful if you have nested structures. 
Also, this schema can be used if you have stringified JSONs. You can use `from_json` method to extract JSON from a string.


## Working with Redshift
There is a very nice [library](https://github.com/databricks/spark-redshift) to load/write data from/to Redshift. The tip regarding loading data from Redshift
is quite short. **After you load data persist dataframe**:
```
val df = getDfFromRedshift(ss, config)
df.persist(StorageLevel.MEMORY_AND_DISK)
# do processing
df.unpersist()
```
As we know, dataframe does not contain data it is actually sequence of transformations which are performed only when action is triggered. 
This way, if Spark wails at some stage it can reconstruct the current state from an initial data source. In our case, an initial data source is Redshift.
And data is loaded to Spark via `UNLAOD` command. So if you do not persist (cache) dataframe, in case calculation fails and Spark resubmits it, it will trigger
new `UNLOAD` and therefore unnecessary load on Redshift (not to mention longer overall processing time).

Whereas, while writing data to redshift definitely use `CSV GZIP` as  `tempformat`. Here is a nice [benchmark](https://www.stitchdata.com/blog/redshift-database-benchmarks-copy-performance-of-csv-json-and-avro/) confirming that.

## Working with S3

While reading files from S3 bare in mind that depending on API (__s3a__ or __s3n__) number of partitions for files on S3 will be different ([source](https://hadoop.apache.org/docs/current/hadoop-aws/tools/hadoop-aws/index.html#Features)):

```
<property>
  <name>fs.s3a.block.size</name>
  <value>32M</value>
  <description>Block size to use when reading files using s3a: file system.
  </description>
</property>
``` 
and
```
<property>
  <name>fs.s3n.block.size</name>
  <value>67108864</value>
  <description>Block size to use when reading files using the native S3
  filesystem (s3n: URIs).</description>
</property>
```
Generally, I would suggest using s3a since it is more recent API. But knowing the number of partitions will help you better configure resource allocation. [Here](http://site.clairvoyantsoft.com/understanding-resource-allocation-configurations-spark-application/) you can find quite nice example of calcuation of recources for Spark application. 

### Problem:
Saving dataframe as Parquet files to S3 is a quite common use-case, however, it appears to be much slower than writing the same dataframe to HDFS. As I understand, the reason is that Spark creates `temporary` folder where it stores initial files and then after all tasks are finished it move them to a final destination (usually to the folder where `temporary` is located). In the case of HDFS it is achieved by renaming files, but in the case of S3 there is no such operation, and it should be done as `copy` and `delete`. Also, it seems this is done by one thread, and if the number of files is large the different between HDFS and S3 writes is bigger. 
#### Update
In [this](https://aws.amazon.com/blogs/big-data/improve-apache-spark-write-performance-on-apache-parquet-formats-with-the-emrfs-s3-optimized-committer/) blog post AWS describes above issue and annonces inprovement for Spark write performance of Parquet files to S3. From my experince , I still find bellow solution faster for large number of file. But, nevertheless, try to use EMR 5.20 and above anyway since EMR 5.20 also inroduced Spark 2.4. 

### Solution
Write dataframe to temporary HDFS folder and later copy it to s3 using [s3-dist-cp](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/UsingEMR_s3distcp.html). If you run Spark applications on EMR it will be available as EMR Step or just command line command. So you can use the following method to do that:
```
import scala.sys.process._

def s3distCp(src: String, dest: String): Unit = {
    s"s3-dist-cp --src $src --dest $dest".!
}
```
***Note***: 
To be able to use this method, you need Hadoop application to be added and you need to run Spark in client or local mode since s3-dist-cp is not available on slave nodes. If you want to run in cluster mode, then copy _s3-dist-cp_ command to slaves during bootstrap. 

(to be continued)


