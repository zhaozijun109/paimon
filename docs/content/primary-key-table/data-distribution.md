---
title: "Data Distribution"
weight: 2
type: docs
aliases:
- /primary-key-table/data-distribution.html
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Data Distribution

A bucket is the smallest storage unit for reads and writes, each bucket directory contains an [LSM tree]({{< ref "primary-key-table/overview#lsm-trees" >}}).

## Fixed Bucket

Configure a bucket greater than 0, using Fixed Bucket mode, according to `Math.abs(key_hashcode % numBuckets)` to compute
the bucket of record.

Rescaling buckets can only be done through offline processes, see [Rescale Bucket]({{< ref "/maintenance/rescale-bucket" >}}).
A too large number of buckets leads to too many small files, and a too small number of buckets leads to poor write performance.

## Dynamic Bucket

Default mode for primary key table, or configure `'bucket' = '-1'`.

The keys that arrive first will fall into the old buckets, and the new keys will fall into the new buckets, the
distribution of buckets and keys depends on the order in which the data arrives. Paimon maintains an index to determine
which key corresponds to which bucket.

Paimon will automatically expand the number of buckets.

- Option1: `'dynamic-bucket.target-row-num'`: controls the target row number for one bucket.
- Option2: `'dynamic-bucket.initial-buckets'`: controls the number of initialized bucket.
- Option3: `'dynamic-bucket.max-buckets'`: controls the number of max buckets.

{{< hint info >}}
Dynamic Bucket only support single write job. Please do not start multiple jobs to write to the same partition
(this can lead to duplicate data). Even if you enable `'write-only'` and start a dedicated compaction job, it won't work.
{{< /hint >}}

### Normal Dynamic Bucket Mode

When your updates do not cross partitions (no partitions, or primary keys contain all partition fields), Dynamic
Bucket mode uses HASH index to maintain mapping from key to bucket, it requires more memory than fixed bucket mode.

Performance:

1. Generally speaking, there is no performance loss, but there will be some additional memory consumption, **100 million**
   entries in a partition takes up **1 GB** more memory, partitions that are no longer active do not take up memory.
2. For tables with low update rates, this mode is recommended to significantly improve performance.

`Normal Dynamic Bucket Mode` supports sort-compact to speed up queries. See [Sort Compact]({{< ref "maintenance/dedicated-compaction#sort-compact" >}}).

### Cross Partitions Upsert Dynamic Bucket Mode

When you need cross partition upsert (primary keys not contain all partition fields), Dynamic Bucket mode directly
maintains the mapping of keys to partition and bucket, uses local disks, and initializes indexes by reading all
existing keys in the table when starting stream write job. Different merge engines have different behaviors:

1. Deduplicate: Delete data from the old partition and insert new data into the new partition.
2. PartialUpdate & Aggregation: Insert new data into the old partition.
3. FirstRow: Ignore new data if there is old value.

Performance: For tables with a large amount of data, there will be a significant loss in performance. Moreover,
initialization takes a long time.

If your upsert does not rely on too old data, you can consider configuring index TTL to reduce Index and initialization time:
- `'cross-partition-upsert.index-ttl'`: The TTL in rocksdb index and initialization, this can avoid maintaining too many
  indexes and lead to worse and worse performance.

But please note that this may also cause data duplication.

## Postpone Bucket

Postpone bucket mode is configured by `'bucket' = '-2'`.
This mode aims to solve the difficulty to determine a fixed number of buckets
and support different buckets for different partitions.

When writing records into the table,
all records will first be stored in the `bucket-postpone` directory of each partition
and are not available to readers.

To move the records into the correct bucket and make them readable,
you need to run a compaction job.
See `compact` [procedure]({{< ref "flink/procedures" >}}).
The bucket number for the partitions compacted for the first time
is configured by the option `postpone.default-bucket-num`, whose default value is `4`.

Finally, when you feel that the bucket number of some partition is too small,
you can also run a rescale job.
See `rescale` [procedure]({{< ref "flink/procedures" >}}).

## Pick Partition Fields

The following three types of fields may be defined as partition fields in the warehouse:

- Creation Time (Recommended): The creation time is generally immutable, so you can confidently treat it as a partition field
  and add it to the primary key.
- Event Time: Event time is a field in the original table. For CDC data, such as tables synchronized from MySQL
  CDC or Changelogs generated by Paimon, they are all complete CDC data, including `UPDATE_BEFORE` records, even
  if you declare the primary key containing partition field, you can achieve the unique effect (require `'changelog-producer'='input'`).
- CDC op_ts: It cannot be defined as a partition field, unable to know previous record timestamp. So you need to use cross partition upsert, it will consume more resources.
