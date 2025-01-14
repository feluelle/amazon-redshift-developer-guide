# COPY Examples<a name="r_COPY_command_examples"></a>

**Note**  
These examples contain line breaks for readability\. Do not include line breaks or spaces in your *credentials\-args* string\.

**Topics**
+ [Load FAVORITEMOVIES from an DynamoDB Table](#r_COPY_command_examples-load-favoritemovies-from-an-amazon-dynamodb-table)
+ [Load LISTING from an Amazon S3 Bucket](#r_COPY_command_examples-load-listing-from-an-amazon-s3-bucket)
+ [Load LISTING from an Amazon EMR Cluster](#copy-command-examples-emr)
+ [Example: COPY from Amazon S3 using a manifest](#copy-command-examples-manifest)
+ [Load LISTING from a Pipe\-Delimited File \(Default Delimiter\)](#r_COPY_command_examples-load-listing-from-a-pipe-delimited-file-default-delimiter)
+ [Load LISTING Using Columnar Data in Parquet Format](#r_COPY_command_examples-load-listing-from-parquet)
+ [Load LISTING Using Temporary Credentials](#sub-example-load-favorite-movies)
+ [Load EVENT with Options](#r_COPY_command_examples-load-event-with-options)
+ [Load VENUE from a Fixed\-Width Data File](#r_COPY_command_examples-load-venue-from-a-fixed-width-data-file)
+ [Load CATEGORY from a CSV File](#load-from-csv)
+ [Load VENUE with Explicit Values for an IDENTITY Column](#r_COPY_command_examples-load-venue-with-explicit-values-for-an-identity-column)
+ [Load TIME from a Pipe\-Delimited GZIP File](#r_COPY_command_examples-load-time-from-a-pipe-delimited-gzip-file)
+ [Load a Timestamp or Datestamp](#r_COPY_command_examples-load-a-time-datestamp)
+ [Load Data from a File with Default Values](#r_COPY_command_examples-load-data-from-a-file-with-default-values)
+ [COPY Data with the ESCAPE Option](#r_COPY_command_examples-copy-data-with-the-escape-option)
+ [Copy from JSON Examples](#r_COPY_command_examples-copy-from-json)
+ [Copy from Avro Examples](#r_COPY_command_examples-copy-from-avro)
+ [Preparing Files for COPY with the ESCAPE Option](#r_COPY_preparing_data)

## Load FAVORITEMOVIES from an DynamoDB Table<a name="r_COPY_command_examples-load-favoritemovies-from-an-amazon-dynamodb-table"></a>

The AWS SDKs include a simple example of creating a DynamoDB table called *Movies*\. \(For this example, see [Getting Started with DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStarted.html)\.\) The following example loads the Amazon Redshift MOVIES table with data from the DynamoDB table\. The Amazon Redshift table must already exist in the database\.

```
copy favoritemovies from 'dynamodb://Movies'
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole' 
readratio 50;
```

## Load LISTING from an Amazon S3 Bucket<a name="r_COPY_command_examples-load-listing-from-an-amazon-s3-bucket"></a>

The following example loads LISTING from an Amazon S3 bucket\. The COPY command loads all of the files in the `/data/listing/` folder\.

```
copy listing
from 's3://mybucket/data/listing/' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole';
```

## Load LISTING from an Amazon EMR Cluster<a name="copy-command-examples-emr"></a>

The following example loads the SALES table with tab\-delimited data from lzop\-compressed files in an Amazon EMR cluster\. COPY loads every file in the `myoutput/` folder that begins with `part-`\.

```
copy sales
from 'emr://j-SAMPLE2B500FC/myoutput/part-*' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
delimiter '\t' lzop;
```

The following example loads the SALES table with JSON formatted data in an Amazon EMR cluster\. COPY loads every file in the `myoutput/json/` folder\.

```
copy sales
from 'emr://j-SAMPLE2B500FC/myoutput/json/' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
JSON 's3://mybucket/jsonpaths.txt';
```

## Using a Manifest to Specify Data Files<a name="copy-command-examples-manifest"></a>

You can use a manifest to ensure that your COPY command loads all of the required files, and only the required files, from Amazon S3\. You can also use a manifest when you need to load multiple files from different buckets or files that do not share the same prefix\. 

For example, suppose that you need to load the following three files: `custdata1.txt`, `custdata2.txt`, and `custdata3.txt`\. You could use the following command to load all of the files in `mybucket` that begin with `custdata` by specifying a prefix: 

```
copy category
from 's3://mybucket/custdata' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole';
```

If only two of the files exist because of an error, COPY loads only those two files and finish successfully, resulting in an incomplete data load\. If the bucket also contains an unwanted file that happens to use the same prefix, such as a file named `custdata.backup` for example, COPY loads that file as well, resulting in unwanted data being loaded\.

To ensure that all of the required files are loaded and to prevent unwanted files from being loaded, you can use a manifest file\. The manifest is a JSON\-formatted text file that lists the files to be processed by the COPY command\. For example, the following manifest loads the three files in the previous example\.

```
{  
   "entries":[  
      {  
         "url":"s3://mybucket/custdata.1",
         "mandatory":true
      },
      {  
         "url":"s3://mybucket/custdata.2",
         "mandatory":true
      },
      {  
         "url":"s3://mybucket/custdata.3",
         "mandatory":true
      }
   ]
}
```

The optional `mandatory` flag indicates whether COPY should terminate if the file does not exist\. The default is `false`\. Regardless of any mandatory settings, COPY terminates if no files are found\. In this example, COPY returns an error if any of the files is not found\. Unwanted files that might have been picked up if you specified only a key prefix, such as `custdata.backup`, are ignored, because they are not on the manifest\. 

When loading from data files in ORC or Parquet format, a `meta` field is required, as shown in the following example\.

```
{  
   "entries":[  
      {  
         "url":"s3://mybucket-alpha/orc/2013-10-04-custdata",
         "mandatory":true,
         "meta":{  
            "content_length":99
         }
      },
      {  
         "url":"s3://mybucket-beta/orc/2013-10-05-custdata",
         "mandatory":true,
         "meta":{  
            "content_length":99
         }
      }
   ]
}
```

The following example uses a manifest named `cust.manifest`\. 

```
copy customer
from 's3://mybucket/cust.manifest' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
manifest;
```

You can use a manifest to load files from different buckets or files that do not share the same prefix\. The following example shows the JSON to load data with files whose names begin with a date stamp\.

```
{
  "entries": [
    {"url":”s3://mybucket/2013-10-04-custdata.txt","mandatory":true},
    {"url":”s3://mybucket/2013-10-05-custdata.txt”,"mandatory":true},
    {"url":”s3://mybucket/2013-10-06-custdata.txt”,"mandatory":true},
    {"url":”s3://mybucket/2013-10-07-custdata.txt”,"mandatory":true}
  ]
}
```

The manifest can list files that are in different buckets, as long as the buckets are in the same Region as the cluster\. 

```
{
  "entries": [
    {"url":"s3://mybucket-alpha/custdata1.txt","mandatory":false},
    {"url":"s3://mybucket-beta/custdata1.txt","mandatory":false},
    {"url":"s3://mybucket-beta/custdata2.txt","mandatory":false}
  ]
}
```

## Load LISTING from a Pipe\-Delimited File \(Default Delimiter\)<a name="r_COPY_command_examples-load-listing-from-a-pipe-delimited-file-default-delimiter"></a>

The following example is a very simple case in which no options are specified and the input file contains the default delimiter, a pipe character \('\|'\)\. 

```
copy listing 
from 's3://mybucket/data/listings_pipe.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole';
```

## Load LISTING Using Columnar Data in Parquet Format<a name="r_COPY_command_examples-load-listing-from-parquet"></a>

The following example loads data from a folder on Amazon S3 named parquet\. 

```
copy listing 
from 's3://mybucket/data/listings/parquet/' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
format as parquet;
```

## Load LISTING Using Temporary Credentials<a name="sub-example-load-favorite-movies"></a>

The following example uses the SESSION\_TOKEN parameter to specify temporary session credentials:

```
copy listing
from 's3://mybucket/data/listings_pipe.txt'
access_key_id '<access-key-id>'
secret_access_key '<secret-access-key'
session_token '<temporary-token>';
```

## Load EVENT with Options<a name="r_COPY_command_examples-load-event-with-options"></a>

The following example loads pipe\-delimited data into the EVENT table and applies the following rules: 
+ If pairs of quotation marks are used to surround any character strings, they are removed\.
+ Both empty strings and strings that contain blanks are loaded as NULL values\.
+ The load fails if more than 5 errors are returned\.
+ Timestamp values must comply with the specified format; for example, a valid timestamp is `2008-09-26 05:43:12`\.

```
copy event
from 's3://mybucket/data/allevents_pipe.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole' 
removequotes
emptyasnull
blanksasnull
maxerror 5
delimiter '|'
timeformat 'YYYY-MM-DD HH:MI:SS';
```

## Load VENUE from a Fixed\-Width Data File<a name="r_COPY_command_examples-load-venue-from-a-fixed-width-data-file"></a>

```
copy venue
from 's3://mybucket/data/venue_fw.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
fixedwidth 'venueid:3,venuename:25,venuecity:12,venuestate:2,venueseats:6';
```

The preceding example assumes a data file formatted in the same way as the sample data shown\. In the sample following, spaces act as placeholders so that all of the columns are the same width as noted in the specification: 

```
1  Toyota Park              Bridgeview  IL0
2  Columbus Crew Stadium    Columbus    OH0
3  RFK Stadium              Washington  DC0
4  CommunityAmerica BallparkKansas City KS0
5  Gillette Stadium         Foxborough  MA68756
```

## Load CATEGORY from a CSV File<a name="load-from-csv"></a>

Suppose you want to load the CATEGORY with the values shown in the following table\.

[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/redshift/latest/dg/r_COPY_command_examples.html)

The following example shows the contents of a text file with the field values separated by commas\.

```
12,Shows,Musicals,Musical theatre
13,Shows,Plays,All "non-musical" theatre  
14,Shows,Opera,All opera, light, and "rock" opera
15,Concerts,Classical,All symphony, concerto, and choir concerts
```

If you load the file using the DELIMITER parameter to specify comma\-delimited input, the COPY command fails because some input fields contain commas\. You can avoid that problem by using the CSV parameter and enclosing the fields that contain commas in quote characters\. If the quote character appears within a quoted string, you need to escape it by doubling the quote character\. The default quote character is a double quotation mark, so you need to escape each double quotation mark with an additional double quotation mark\. Your new input file looks something like this\. 

```
12,Shows,Musicals,Musical theatre
13,Shows,Plays,"All ""non-musical"" theatre"
14,Shows,Opera,"All opera, light, and ""rock"" opera"
15,Concerts,Classical,"All symphony, concerto, and choir concerts"
```

Assuming the file name is `category_csv.txt`, you can load the file by using the following COPY command:

```
copy category
from 's3://mybucket/data/category_csv.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole' 
csv;
```

Alternatively, to avoid the need to escape the double quotation marks in your input, you can specify a different quote character by using the QUOTE AS parameter\. For example, the following version of `category_csv.txt` uses '`%`' as the quote character:

```
12,Shows,Musicals,Musical theatre
13,Shows,Plays,%All "non-musical" theatre%
14,Shows,Opera,%All opera, light, and "rock" opera%
15,Concerts,Classical,%All symphony, concerto, and choir concerts%
```

The following COPY command uses QUOTE AS to load `category_csv.txt`:

```
copy category
from 's3://mybucket/data/category_csv.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole' 
csv quote as '%';
```

## Load VENUE with Explicit Values for an IDENTITY Column<a name="r_COPY_command_examples-load-venue-with-explicit-values-for-an-identity-column"></a>

The following example assumes that when the VENUE table was created that at least one column \(such as the `venueid` column\) was specified to be an IDENTITY column\. This command overrides the default IDENTITY behavior of auto\-generating values for an IDENTITY column and instead loads the explicit values from the venue\.txt file\.

```
copy venue
from 's3://mybucket/data/venue.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
explicit_ids;
```

## Load TIME from a Pipe\-Delimited GZIP File<a name="r_COPY_command_examples-load-time-from-a-pipe-delimited-gzip-file"></a>

The following example loads the TIME table from a pipe\-delimited GZIP file:

```
copy time
from 's3://mybucket/data/timerows.gz' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
gzip
delimiter '|';
```

## Load a Timestamp or Datestamp<a name="r_COPY_command_examples-load-a-time-datestamp"></a>

The following example loads data with a formatted timestamp\.

**Note**  
The TIMEFORMAT of `HH:MI:SS` can also support fractional seconds beyond the `SS` to a microsecond level of detail\. The file `time.txt` used in this example contains one row, `2009-01-12 14:15:57.119568`\.

```
copy timestamp1 
from 's3://mybucket/data/time.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
timeformat 'YYYY-MM-DD HH:MI:SS';
```

The result of this copy is as follows: 

```
select * from timestamp1;
c1
----------------------------
2009-01-12 14:15:57.119568
(1 row)
```

## Load Data from a File with Default Values<a name="r_COPY_command_examples-load-data-from-a-file-with-default-values"></a>

The following example uses a variation of the VENUE table in the TICKIT database\. Consider a VENUE\_NEW table defined with the following statement: 

```
create table venue_new(
venueid smallint not null,
venuename varchar(100) not null,
venuecity varchar(30),
venuestate char(2),
venueseats integer not null default '1000');
```

Consider a venue\_noseats\.txt data file that contains no values for the VENUESEATS column, as shown in the following example: 

```
1|Toyota Park|Bridgeview|IL|
2|Columbus Crew Stadium|Columbus|OH|
3|RFK Stadium|Washington|DC|
4|CommunityAmerica Ballpark|Kansas City|KS|
5|Gillette Stadium|Foxborough|MA|
6|New York Giants Stadium|East Rutherford|NJ|
7|BMO Field|Toronto|ON|
8|The Home Depot Center|Carson|CA|
9|Dick's Sporting Goods Park|Commerce City|CO|
10|Pizza Hut Park|Frisco|TX|
```

The following COPY statement will successfully load the table from the file and apply the DEFAULT value \('1000'\) to the omitted column: 

```
copy venue_new(venueid, venuename, venuecity, venuestate) 
from 's3://mybucket/data/venue_noseats.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
delimiter '|';
```

Now view the loaded table: 

```
select * from venue_new order by venueid;
venueid |         venuename          |    venuecity    | venuestate | venueseats
---------+----------------------------+-----------------+------------+------------
1 | Toyota Park                | Bridgeview      | IL         |       1000
2 | Columbus Crew Stadium      | Columbus        | OH         |       1000
3 | RFK Stadium                | Washington      | DC         |       1000
4 | CommunityAmerica Ballpark  | Kansas City     | KS         |       1000
5 | Gillette Stadium           | Foxborough      | MA         |       1000
6 | New York Giants Stadium    | East Rutherford | NJ         |       1000
7 | BMO Field                  | Toronto         | ON         |       1000
8 | The Home Depot Center      | Carson          | CA         |       1000
9 | Dick's Sporting Goods Park | Commerce City   | CO         |       1000
10 | Pizza Hut Park             | Frisco          | TX         |       1000
(10 rows)
```

For the following example, in addition to assuming that no VENUESEATS data is included in the file, also assume that no VENUENAME data is included: 

```
1||Bridgeview|IL|
2||Columbus|OH|
3||Washington|DC|
4||Kansas City|KS|
5||Foxborough|MA|
6||East Rutherford|NJ|
7||Toronto|ON|
8||Carson|CA|
9||Commerce City|CO|
10||Frisco|TX|
```

 Using the same table definition, the following COPY statement fails because no DEFAULT value was specified for VENUENAME, and VENUENAME is a NOT NULL column: 

```
copy venue(venueid, venuecity, venuestate) 
from 's3://mybucket/data/venue_pipe.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
delimiter '|';
```

Now consider a variation of the VENUE table that uses an IDENTITY column: 

```
create table venue_identity(
venueid int identity(1,1),
venuename varchar(100) not null,
venuecity varchar(30),
venuestate char(2),
venueseats integer not null default '1000');
```

As with the previous example, assume that the VENUESEATS column has no corresponding values in the source file\. The following COPY statement successfully loads the table, including the predefined IDENTITY data values instead of autogenerating those values: 

```
copy venue(venueid, venuename, venuecity, venuestate) 
from 's3://mybucket/data/venue_pipe.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
delimiter '|' explicit_ids;
```

This statement fails because it does not include the IDENTITY column \(VENUEID is missing from the column list\) yet includes an EXPLICIT\_IDS parameter: 

```
copy venue(venuename, venuecity, venuestate) 
from 's3://mybucket/data/venue_pipe.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
delimiter '|' explicit_ids;
```

This statement fails because it does not include an EXPLICIT\_IDS parameter: 

```
copy venue(venueid, venuename, venuecity, venuestate)
from 's3://mybucket/data/venue_pipe.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
delimiter '|';
```

## COPY Data with the ESCAPE Option<a name="r_COPY_command_examples-copy-data-with-the-escape-option"></a>

The following example shows how to load characters that match the delimiter character \(in this case, the pipe character\)\. In the input file, make sure that all of the pipe characters \(\|\) that you want to load are escaped with the backslash character \(\\\)\. Then load the file with the ESCAPE parameter\. 

```
$ more redshiftinfo.txt
1|public\|event\|dwuser
2|public\|sales\|dwuser

create table redshiftinfo(infoid int,tableinfo varchar(50));

copy redshiftinfo from 's3://mybucket/data/redshiftinfo.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole' 
delimiter '|' escape;

select * from redshiftinfo order by 1;
infoid |       tableinfo
-------+--------------------
1      | public|event|dwuser
2      | public|sales|dwuser
(2 rows)
```

Without the ESCAPE parameter, this COPY command fails with an `Extra column(s) found` error\.

**Important**  
If you load your data using a COPY with the ESCAPE parameter, you must also specify the ESCAPE parameter with your UNLOAD command to generate the reciprocal output file\. Similarly, if you UNLOAD using the ESCAPE parameter, you need to use ESCAPE when you COPY the same data\.

## Copy from JSON Examples<a name="r_COPY_command_examples-copy-from-json"></a>

In the following examples, you load the CATEGORY table with the following data\. 

[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/redshift/latest/dg/r_COPY_command_examples.html)

**Topics**
+ [Load from JSON Data Using the 'auto' Option](#copy-from-json-examples-using-auto)
+ [Load from JSON Data Using a JSONPaths file](#copy-from-json-examples-using-jsonpaths)
+ [Load from JSON Arrays Using a JSONPaths file](#copy-from-json-examples-using-jsonpaths-arrays)

### Load from JSON Data Using the 'auto' Option<a name="copy-from-json-examples-using-auto"></a>

To load from JSON data using the `'auto'` argument, the JSON data must consist of a set of objects\. The key names must match the column names, but in this case, order does not matter\. The following shows the contents of a file named `category_object_auto.json`\.

```
{
    "catdesc": "Major League Baseball",
    "catid": 1,
    "catgroup": "Sports",
    "catname": "MLB"
}
{
    "catgroup": "Sports",
    "catid": 2,
    "catname": "NHL",
    "catdesc": "National Hockey League"
}{
    "catid": 3,
    "catname": "NFL",
    "catgroup": "Sports",
    "catdesc": "National Football League"
}
{
    "bogus": "Bogus Sports LLC",
    "catid": 4,
    "catgroup": "Sports",
    "catname": "NBA",
    "catdesc": "National Basketball Association"
}
{
    "catid": 5,
    "catgroup": "Shows",
    "catname": "Musicals",
    "catdesc": "All symphony, concerto, and choir concerts"
}
```

To load from the JSON data file in the previous example, execute the following COPY command\.

```
copy category
from 's3://mybucket/category_object_auto.json'
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole' 
json 'auto';
```

### Load from JSON Data Using a JSONPaths file<a name="copy-from-json-examples-using-jsonpaths"></a>

If the JSON data objects don't correspond directly to column names, you can use a JSONPaths file to map the JSON elements to columns\. Again, the order does not matter in the JSON source data, but the order of the JSONPaths file expressions must match the column order\. Suppose that you have the following data file, named `category_object_paths.json`\.

```
{
    "one": 1,
    "two": "Sports",
    "three": "MLB",
    "four": "Major League Baseball"
}
{
    "three": "NHL",
    "four": "National Hockey League",
    "one": 2,
    "two": "Sports"
}
{
    "two": "Sports",
    "three": "NFL",
    "one": 3,
    "four": "National Football League"
}
{
    "one": 4,
    "two": "Sports",
    "three": "NBA",
    "four": "National Basketball Association"
}
{
    "one": 6,
    "two": "Shows",
    "three": "Musicals",
    "four": "All symphony, concerto, and choir concerts"
}
```

The following JSONPaths file, named `category_jsonpath.json`, maps the source data to the table columns\.

```
{
    "jsonpaths": [
        "$['one']",
        "$['two']",
        "$['three']",
        "$['four']"
    ]
}
```

To load from the JSON data file in the previous example, execute the following COPY command\.

```
copy category
from 's3://mybucket/category_object_paths.json'
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole' 
json 's3://mybucket/category_jsonpath.json';
```

### Load from JSON Arrays Using a JSONPaths file<a name="copy-from-json-examples-using-jsonpaths-arrays"></a>

To load from JSON data that consists of a set of arrays, you must use a JSONPaths file to map the array elements to columns\. Suppose that you have the following data file, named `category_array_data.json`\.

```
[1,"Sports","MLB","Major League Baseball"]
[2,"Sports","NHL","National Hockey League"]
[3,"Sports","NFL","National Football League"]
[4,"Sports","NBA","National Basketball Association"]
[5,"Concerts","Classical","All symphony, concerto, and choir concerts"]
```

The following JSONPaths file, named `category_array_jsonpath.json`, maps the source data to the table columns\.

```
{
    "jsonpaths": [
        "$[0]",
        "$[1]",
        "$[2]",
        "$[3]"
    ]
}
```

To load from the JSON data file in the previous example, execute the following COPY command\.

```
copy category
from 's3://mybucket/category_array_data.json'
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole' 
json 's3://mybucket/category_array_jsonpath.json';
```

## Copy from Avro Examples<a name="r_COPY_command_examples-copy-from-avro"></a>

In the following examples, you load the CATEGORY table with the following data\. 

[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/redshift/latest/dg/r_COPY_command_examples.html)

**Topics**
+ [Load from Avro Data Using the 'auto' Option](#copy-from-avro-examples-using-auto)
+ [Load from Avro Data Using a JSONPaths File](#copy-from-avro-examples-using-avropaths)

### Load from Avro Data Using the 'auto' Option<a name="copy-from-avro-examples-using-auto"></a>

To load from Avro data using the `'auto'` argument, field names in the Avro schema must match the column names\. However, when using the `'auto'` argument, order does not matter\. The following shows the schema for a file named `category_auto.avro`\.

```
{
    "name": "category",
    "type": "record",
    "fields": [
        {"name": "catid", "type": "int"},
        {"name": "catdesc", "type": "string"},
        {"name": "catname", "type": "string"},
        {"name": "catgroup", "type": "string"},
}
```

The data in an Avro file is in binary format, so it is not human\-readable\. The following shows a JSON representation of the data in the `category_auto.avro` file\. 

```
{
   "catid": 1,
   "catdesc": "Major League Baseball",
   "catname": "MLB",
   "catgroup": "Sports"
}
{
   "catid": 2,
   "catdesc": "National Hockey League",
   "catname": "NHL",
   "catgroup": "Sports"
}
{
   "catid": 3,
   "catdesc": "National Basketball Association",
   "catname": "NBA",
   "catgroup": "Sports"
}
{
   "catid": 4,
   "catdesc": "All symphony, concerto, and choir concerts",
   "catname": "Classical",
   "catgroup": "Concerts"
}
```

To load from the Avro data file in the previous example, execute the following COPY command\.

```
copy category
from 's3://mybucket/category_auto.avro'
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
format as avro 'auto';
```

### Load from Avro Data Using a JSONPaths File<a name="copy-from-avro-examples-using-avropaths"></a>

If the field names in the Avro schema don't correspond directly to column names, you can use a JSONPaths file to map the schema elements to columns\. The order of the JSONPaths file expressions must match the column order\. 

Suppose that you have a data file named `category_paths.avro` that contains the same data as in the previous example, but with the following schema\.

```
{
    "name": "category",
    "type": "record",
    "fields": [
        {"name": "id", "type": "int"},
        {"name": "desc", "type": "string"},
        {"name": "name", "type": "string"},
        {"name": "group", "type": "string"},
        {"name": "region", "type": "string"} 
     ]
}
```

The following JSONPaths file, named `category_path.avropath`, maps the source data to the table columns\.

```
{
    "jsonpaths": [
        "$['id']",
        "$['group']",
        "$['name']",
        "$['desc']"
    ]
}
```

To load from the Avro data file in the previous example, execute the following COPY command\.

```
copy category
from 's3://mybucket/category_object_paths.avro'
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole' 
format avro 's3://mybucket/category_path.avropath ';
```

## Preparing Files for COPY with the ESCAPE Option<a name="r_COPY_preparing_data"></a>

The following example describes how you might prepare data to "escape" newline characters before importing the data into an Amazon Redshift table using the COPY command with the ESCAPE parameter\. Without preparing the data to delimit the newline characters, Amazon Redshift returns load errors when you run the COPY command, because the newline character is normally used as a record separator\. 

For example, consider a file or a column in an external table that you want to copy into an Amazon Redshift table\. If the file or column contains XML\-formatted content or similar data, you need to make sure that all of the newline characters \(\\n\) that are part of the content are escaped with the backslash character \(\\\)\. 

A good thing about a file or table containing embedded newlines characters is that it provides a relatively easy pattern to match\. Each embedded newline character most likely always follows a `>` character with potentially some white space characters \(`' '` or tab\) in between, as you can see in the following example of a text file named `nlTest1.txt`\. 

```
$ cat nlTest1.txt
<xml start>
<newline characters provide>
<line breaks at the end of each>
<line in content>
</xml>|1000
<xml>
</xml>|2000
```

With the following example, you can run a text\-processing utility to pre\-process the source file and insert escape characters where needed\. \(The `|` character is intended to be used as delimiter to separate column data when copied into an Amazon Redshift table\.\) 

```
$ sed -e ':a;N;$!ba;s/>[[:space:]]*\n/>\\\n/g' nlTest1.txt > nlTest2.txt
```

Similarly, you can use Perl to perform a similar operation: 

```
cat nlTest1.txt | perl -p -e 's/>\s*\n/>\\\n/g' > nlTest2.txt
```

To accommodate loading the data from the `nlTest2.txt` file into Amazon Redshift, we created a two\-column table in Amazon Redshift\. The first column c1, is a character column that holds XML\-formatted content from the `nlTest2.txt` file\. The second column c2 holds integer values loaded from the same file\. 

After running the `sed` command, you can correctly load data from the `nlTest2.txt` file into an Amazon Redshift table using the ESCAPE parameter\. 

**Note**  
When you include the ESCAPE parameter with the COPY command, it escapes a number of special characters that include the backslash character \(including newline\)\. 

```
copy t2 from 's3://mybucket/data/nlTest2.txt' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'  
escape
delimiter as '|';

select * from t2 order by 2;

c1           |  c2
-------------+------
<xml start>
<newline characters provide>
<line breaks at the end of each>
<line in content>
</xml>
| 1000
<xml>
</xml>       | 2000
(2 rows)
```

You can prepare data files exported from external databases in a similar way\. For example, with an Oracle database, you can use the REPLACE function on each affected column in a table that you want to copy into Amazon Redshift\. 

```
SELECT c1, REPLACE(c2, \n',\\n' ) as c2 from my_table_with_xml
```

In addition, many database export and extract, transform, load \(ETL\) tools that routinely process large amounts of data provide options to specify escape and delimiter characters\. 