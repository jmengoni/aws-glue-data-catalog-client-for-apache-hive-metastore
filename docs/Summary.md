# Summay of repo based on AI code Analysis (Antigravity IDE, Claude Sonnet 4.6 model)

## What This Repo Does
This is the AWS Glue Data Catalog Client for Apache Hive Metastore — an open-source Java library that lets Apache Hive and Apache Spark use AWS Glue Data Catalog as their external Hive Metastore, instead of a traditional local Hive Metastore server.

## Core Purpose
When you run Hive or Spark on Amazon Elastic Map Reduce (EMR) (or another Hadoop cluster), they normally need a Hive Metastore to track databases, tables, partitions, and schemas. This library replaces that local metastore with AWS Glue's fully-managed, serverless Data Catalog, using the same IMetaStoreClient Java interface that Hive/Spark already expect.

## Maven Module Structure
| Module                             | Role                                                         |
|------------------------------------|--------------------------------------------------------------|
| aws-glue-datacatalog-client-common | Shared core logic — the bulk of the implementation           |
| aws-glue-datacatalog-hive3-client  | Hive 3.x-specific client adapter                             |
| aws-glue-datacatalog-spark-client  | Spark (Hive 2.3 fork) client adapter                         |
| shims                              | Compatibility shims for differences between Hive 2.x and 3.x APIs |

## Key Components (in client-common)

GlueMetastoreClientDelegate
— The heart of the library. Implements all metastore operations (CRUD for databases, tables, partitions, user-defined functions, column stats, etc.) by translating Hive metastore API calls into AWS Glue API calls.

DefaultAWSGlueMetastore — Wraps the actual AWS Glue SDK client, making raw calls to the Glue service.


AWSGlueMetastoreCacheDecorator
— An optional caching layer (using Guava Cache) that can cache GetTable and GetDatabase responses locally to reduce Glue API calls. Configurable via:

aws.glue.cache.table.enable / aws.glue.cache.db.enable
Cache size and TTL properties.

AWSGlueMultipleCatalogDecorator
— Allows a single Glue client to work across multiple Glue catalogs by parsing a catalog separator (e.g. catalogId:databaseName) from the database name and routing requests to the correct catalog.

Converters (converters/ package) — Bidirectional conversion between Glue's data model (e.g. com.amazonaws.services.glue.model.Table) and Hive's Thrift model (e.g. org.apache.hadoop.hive.metastore.api.Table).

Batch Helpers (BatchCreatePartitionsHelper, BatchDeletePartitionsHelper) — Handle Glue's 100-partition-per-request limit by splitting large operations into parallel batches using a thread pool.

## How It Plugs In
You configure Hive/Spark to use this client via hive-site.xml:
```xml
<property>
  <name>hive.metastore.client.factory.class</name>
  <value>com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory</value>
</property>

```
That's all it takes — the rest of Hive/Spark thinks it's talking to a regular Hive Metastore, but all metadata actually lives in AWS Glue.

## Notable Design Details
- Requires a Hive patch (HIVE-12679) to allow pluggable client factories — this patch is already applied in Amazon EMR.
- Thread pool with 5 threads handles parallel partition batch operations.
- Table rename is explicitly unsupported (throws UnsupportedOperationException).
- Built for Java 8, targets Spark 3 / Hive 3 (version 3.4.0-SNAPSHOT).
