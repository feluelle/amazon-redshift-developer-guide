# COPY from JSON Format<a name="copy-usage_notes-copy-from-json"></a>

The JSON data structure is made up of a set of *objects* or *arrays*\. A JSON *object* begins and ends with braces, and contains an unordered collection of name/value pairs\. Each name and value are separated by a colon, and the pairs are separated by commas\. The name is a string in double quotation marks\. The quote characters must be simple quotation marks \(0x22\), not slanted or "smart" quotes\. 

A JSON *array* begins and ends with brackets, and contains an ordered collection of values separated by commas\. A value can be a string in double quotation marks, a number, a Boolean true or false, null, a JSON object, or an array\. 

JSON objects and arrays can be nested, enabling a hierarchical data structure\. The following example shows a JSON data structure with two valid objects\. 

```
{
    "id": 1006410,
    "title": "Amazon Redshift Database Developer Guide"
}
{
    "id": 100540,
    "name": "Amazon Simple Storage Service Developer Guide"
}
```

The following shows the same data as two JSON arrays\.

```
[
    1006410,
    "Amazon Redshift Database Developer Guide"
]
[
    100540,
    "Amazon Simple Storage Service Developer Guide"
]
```

You can let COPY automatically load fields from the JSON file by specifying the 'auto' option, or you can specify a JSONPaths file that COPY uses to parse the JSON source data\. A *JSONPaths file* is a text file that contains a single JSON object with the name `"jsonpaths"` paired with an array of JSONPath expressions\. If the name is any string other than `"jsonpaths"`, COPY uses the `'auto'` argument instead of using the JSONPaths file\.

In the Amazon Redshift COPY syntax, a JSONPath expression specifies the explicit path to a single name element in a JSON hierarchical data structure, using either bracket notation or dot notation\. Amazon Redshift does not support any JSONPath elements, such as wildcard characters or filter expressions, that might resolve to an ambiguous path or multiple name elements\. As a result, Amazon Redshift can't parse complex, multi\-level data structures\.

The following is an example of a JSONPaths file with JSONPath expressions using bracket notation\. The dollar sign \($\) represents the root\-level structure\. 

```
{
    "jsonpaths": [
       "$['id']",
       "$['store']['book']['title']",
	"$['location'][0]" 
    ]
}
```

 In the previous example, `$['location'][0]` references the first element in an array\. JSON uses zero\-based array indexing\. Array indices must be positive integers \(greater than or equal to zero\)\.

The following example shows the previous JSONPaths file using dot notation\. 

```
{
    "jsonpaths": [
       "$.id",
       "$.store.book.title",
	"$.location[0]"
    ]
}
```

You cannot mix bracket notation and dot notation in the `jsonpaths` array\. Brackets can be used in both bracket notation and dot notation to reference an array element\. 

When using dot notation, the JSONPath expressions must not contain the following characters: 
+ Single straight quotation mark \( ' \) 
+ Period, or dot \( \. \) 
+ Brackets \( \[ \] \) unless used to reference an array element 

If the value in the name/value pair referenced by a JSONPath expression is an object or an array, the entire object or array is loaded as a string, including the braces or brackets\. For example, suppose your JSON data contains the following object\. 

```
{
    "id": 0,
    "guid": "84512477-fa49-456b-b407-581d0d851c3c",
    "isActive": true,
    "tags": [
        "nisi",
        "culpa",
        "ad",
        "amet",
        "voluptate",
        "reprehenderit",
        "veniam"
    ],
    "friends": [
        {
            "id": 0,
            "name": "Carmella Gonzales"
        },
        {
            "id": 1,
            "name": "Renaldo"
        }
    ]
}
```

The JSONPath expression `$['tags']` then returns the following value\. 

```
"["nisi","culpa","ad","amet","voluptate","reprehenderit","veniam"]" 
```

The JSONPath expression `$['friends'][1]` then returns the following value\. 

```
"{"id": 1,"name": "Renaldo"}" 
```

Each JSONPath expression in the `jsonpaths` array corresponds to one column in the Amazon Redshift target table\. The order of the `jsonpaths` array elements must match the order of the columns in the target table or the column list, if a column list is used\. 

For examples that show how to load data using either the `'auto'` argument or a JSONPaths file, and using either JSON objects or arrays, see [Copy from JSON Examples](r_COPY_command_examples.md#r_COPY_command_examples-copy-from-json)\. 

## Escape Characters in JSON<a name="copy-usage-json-escape-characters"></a>

COPY loads `\n` as a newline character and loads `\t` as a tab character\. To load a backslash, escape it with a backslash \( `\\` \)\.

For example, suppose you have the following JSON in a file named `escape.json` in the bucket `s3://mybucket/json/`\.

```
{
  "backslash": "This is a backslash: \\",
  "newline": "This sentence\n is on two lines.",
  "tab": "This sentence \t contains a tab."
}
```

Execute the following commands to create the ESCAPES table and load the JSON\.

```
create table escapes (backslash varchar(25), newline varchar(35), tab varchar(35));

copy escapes from 's3://mybucket/json/escape.json' 
iam_role 'arn:aws:iam::0123456789012:role/MyRedshiftRole'
format as json 'auto';
```

Query the ESCAPES table to view the results\.

```
select * from escapes;

       backslash        |      newline      |               tab
------------------------+-------------------+----------------------------------
 This is a backslash: \ | This sentence     | This sentence    contains a tab.
                        :  is on two lines.
(1 row)
```

## Loss of numeric precision<a name="copy-usage-json-rounding"></a>

You might lose precision when loading numbers from data files in JSON format to a column that is defined as a numeric data type\. Some floating point values are not represented exactly in computer systems\. As a result, data you copy from a JSON file might not be rounded as you expect\. To avoid a loss of precision, we recommend using one of the following alternatives:
+ Represent the number as a string by enclosing the value in double quotation characters\.
+ Use [ROUNDEC](copy-parameters-data-conversion.md#copy-roundec) to round the number instead of truncating\.
+ Instead of using JSON or Avro files, use CSV, character\-delimited, or fixed\-width text files\.