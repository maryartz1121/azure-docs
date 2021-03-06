---
title: Query Parquet files using SQL on-demand (preview)
description: In this article, you'll learn how to query Parquet files using SQL on-demand (preview).
services: synapse analytics
author: azaricstefan
ms.service: synapse-analytics
ms.topic: how-to
ms.subservice: sql
ms.date: 05/20/2020
ms.author: v-stazar
ms.reviewer: jrasnick 
---

# Query Parquet files using SQL on-demand (preview) in Azure Synapse Analytics

In this article, you'll learn how to write a query using SQL on-demand (preview) that will read Parquet files.

## Quickstart example

`OPENROWSET` function enables you to read the content of parquet file by providing the URL to your file.

### Read parquet file

The easiest way to see to the content of your `PARQUET` file is to provide file URL to `OPENROWSET` function and specify parquet `FORMAT`. If the file is publicly available or if your Azure AD identity can access this file, you should be able to see the content of the file using the query like the one shown in the following example:

```sql
select top 10 *
from openrowset(
    bulk 'https://pandemicdatalake.blob.core.windows.net/public/curated/covid-19/ecdc_cases/latest/ecdc_cases.parquet',
    format = 'parquet') as rows
```

Make sure that you access this file. If your file is protected with SAS key or custom Azure identity, your would need to setup [server level credential for sql login](develop-storage-files-storage-access-control.md?tabs=shared-access-signature#server-scoped-credential).

### Data source usage

Previous example uses full path to the file. As an alternative, you can create an external data source with the location that points to the root folder of the storage, and use that data source and the relative path to the file in `OPENROWSET` function:

```sql
create external data source covid
with ( location = 'https://pandemicdatalake.blob.core.windows.net/public/curated/covid-19/ecdc_cases' );
go
select top 10 *
from openrowset(
        bulk 'latest/ecdc_cases.parquet',
        data_source = 'covid',
        format = 'parquet'
    ) as rows
```

If a data source is protected with SAS key or custom identity you can configure [data source with database scoped credential](develop-storage-files-storage-access-control.md?tabs=shared-access-signature#database-scoped-credential).

### Explicitly specify schema

`OPENROWSET` enables you to explicitly specify what columns you want to read from the file using `WITH` clause:

```sql
select top 10 *
from openrowset(
        bulk 'latest/ecdc_cases.parquet',
        data_source = 'covid',
        format = 'parquet'
    ) with ( date_rep date, cases int, geo_id varchar(6) ) as rows
```

In the following sections you can see how to query various types of PARQUET files.

## Prerequisites

Your first step is to **create a database** with a datasource that references [NYC Yellow Taxi](https://azure.microsoft.com/services/open-datasets/catalog/nyc-taxi-limousine-commission-yellow-taxi-trip-records/) storage account. Then initialize the objects by executing [setup script](https://github.com/Azure-Samples/Synapse/blob/master/SQL/Samples/LdwSample/SampleDB.sql) on that database. This setup script will create the data sources, database scoped credentials, and external file formats that are used in these samples.

## Dataset

[NYC Yellow Taxi](https://azure.microsoft.com/services/open-datasets/catalog/nyc-taxi-limousine-commission-yellow-taxi-trip-records/) dataset is used in this sample. You can query Parquet files the same way you [read CSV files](query-parquet-files.md). The only difference is that the `FILEFORMAT` parameter should be set to `PARQUET`. Examples in this article show the specifics of reading Parquet files.

## Query set of parquet files

You can specify only the columns of interest when you query Parquet files.

```sql
SELECT
        YEAR(tpepPickupDateTime),
        passengerCount,
        COUNT(*) AS cnt
FROM  
    OPENROWSET(
        BULK 'puYear=2018/puMonth=*/*.snappy.parquet',
        DATA_SOURCE = 'YellowTaxi',
        FORMAT='PARQUET'
    ) WITH (
        tpepPickupDateTime DATETIME2,
        passengerCount INT
    ) AS nyc
GROUP BY
    passengerCount,
    YEAR(tpepPickupDateTime)
ORDER BY
    YEAR(tpepPickupDateTime),
    passengerCount;
```

## Automatic schema inference

You don't need to use the OPENROWSET WITH clause when reading Parquet files. Column names and data types are automatically read from Parquet files.

The sample below shows the automatic schema inference capabilities for Parquet files. It returns the number of rows in September 2017 without specifying a schema.

> [!NOTE]
> You don't have to specify columns in the OPENROWSET WITH clause when reading Parquet files. In that case, SQL on-demand Query service will utilize metadata in the Parquet file and bind columns by name.

```sql
SELECT TOP 10 *
FROM  
    OPENROWSET(
        BULK 'puYear=2018/puMonth=*/*.snappy.parquet',
        DATA_SOURCE = 'YellowTaxi',
        FORMAT='PARQUET'
    ) AS nyc
```

### Query partitioned data

The data set provided in this sample is divided (partitioned) into separate subfolders. You can target specific partitions using the filepath function. This example shows fare amounts by year, month, and payment_type for the first three months of 2017.

> [!NOTE]
> The SQL on-demand Query is compatible with Hive/Hadoop partitioning scheme.

```sql
SELECT
        YEAR(tpepPickupDateTime),
        passengerCount,
        COUNT(*) AS cnt
FROM  
    OPENROWSET(
        BULK 'puYear=*/puMonth=*/*.snappy.parquet',
        DATA_SOURCE = 'YellowTaxi',
        FORMAT='PARQUET'
    ) nyc
WHERE
    nyc.filepath(1) = 2017
    AND nyc.filepath(2) IN (1, 2, 3)
    AND tpepPickupDateTime BETWEEN CAST('1/1/2017' AS datetime) AND CAST('3/31/2017' AS datetime)
GROUP BY
    passengerCount,
    YEAR(tpepPickupDateTime)
ORDER BY
    YEAR(tpepPickupDateTime),
    passengerCount;
```

## Type mapping

Parquet files contain type descriptions for every column. The following table describes how Parquet types are mapped to SQL native types.

| Parquet type | Parquet logical type (annotation) | SQL data type |
| --- | --- | --- |
| BOOLEAN | | bit |
| BINARY / BYTE_ARRAY | | varbinary |
| DOUBLE | | float |
| FLOAT | | real |
| INT32 | | int |
| INT64 | | bigint |
| INT96 | |datetime2 |
| FIXED_LEN_BYTE_ARRAY | |binary |
| BINARY |UTF8 |varchar \*(UTF8 collation) |
| BINARY |STRING |varchar \*(UTF8 collation) |
| BINARY |ENUM|varchar \*(UTF8 collation) |
| BINARY |UUID |uniqueidentifier |
| BINARY |DECIMAL |decimal |
| BINARY |JSON |varchar(max) \*(UTF8 collation) |
| BINARY |BSON |varbinary(max) |
| FIXED_LEN_BYTE_ARRAY |DECIMAL |decimal |
| BYTE_ARRAY |INTERVAL |varchar(max), serialized into standardized format |
| INT32 |INT(8, true) |smallint |
| INT32 |INT(16, true) |smallint |
| INT32 |INT(32, true) |int |
| INT32 |INT(8, false) |tinyint |
| INT32 |INT(16, false) |int |
| INT32 |INT(32, false) |bigint |
| INT32 |DATE |date |
| INT32 |DECIMAL |decimal |
| INT32 |TIME (MILLIS )|time |
| INT64 |INT(64, true) |bigint |
| INT64 |INT(64, false ) |decimal(20,0) |
| INT64 |DECIMAL |decimal |
| INT64 |TIME (MICROS / NANOS) |time |
|INT64 |TIMESTAMP (MILLIS / MICROS / NANOS) |datetime2 |
|[Complex type](https://github.com/apache/parquet-format/blob/master/LogicalTypes.md#lists) |LIST |varchar(max), serialized into JSON |
|[Complex type](https://github.com/apache/parquet-format/blob/master/LogicalTypes.md#maps)|MAP|varchar(max), serialized into JSON |

## Next steps

Advance to the next article to learn how to [Query Parquet nested types](query-parquet-nested-types.md).
