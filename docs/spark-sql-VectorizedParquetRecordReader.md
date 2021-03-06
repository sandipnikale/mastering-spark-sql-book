# VectorizedParquetRecordReader

`VectorizedParquetRecordReader` is a concrete [SpecificParquetRecordReaderBase](spark-sql-SpecificParquetRecordReaderBase.md) for [parquet](spark-sql-ParquetFileFormat.md) file format for [Vectorized Parquet Decoding](spark-sql-vectorized-parquet-reader.md).

`VectorizedParquetRecordReader` is <<creating-instance, created>> exclusively when `ParquetFileFormat` is requested for a [data reader](spark-sql-ParquetFileFormat.md#buildReaderWithPartitionValues) (with [spark.sql.parquet.enableVectorizedReader](spark-sql-properties.md#spark.sql.parquet.enableVectorizedReader) property enabled and the read schema with [AtomicType](spark-sql-DataType.md#AtomicType) data types only).

[[creating-instance]]
`VectorizedParquetRecordReader` takes the following to be created:

* [[convertTz]] `TimeZone` (`null` when no timezone conversion is expected)
* [[useOffHeap]] `useOffHeap` flag (per <<spark-sql-properties.md#spark.sql.columnVector.offheap.enabled, spark.sql.columnVector.offheap.enabled>> property)
* [[capacity]] Capacity (per <<spark-sql-properties.md#spark.sql.parquet.columnarReaderBatchSize, spark.sql.parquet.columnarReaderBatchSize>> property)

`VectorizedParquetRecordReader` uses the <<capacity, capacity>> attribute for the following:

* Creating <<columnVectors, WritableColumnVectors>> when <<initBatch, initializing a columnar batch>>

* Controlling <<rowsReturned, number of rows>> when <<nextBatch, nextBatch>>

`VectorizedParquetRecordReader` uses <<OFF_HEAP, OFF_HEAP>> memory mode when spark-sql-properties.md#spark.sql.columnVector.offheap.enabled[spark.sql.columnVector.offheap.enabled] internal configuration property is enabled (`true`).

NOTE: spark-sql-properties.md#spark.sql.columnVector.offheap.enabled[spark.sql.columnVector.offheap.enabled] configuration property is disabled (`false`) by default.

[[internal-registries]]
.VectorizedParquetRecordReader's Internal Properties (e.g. Registries, Counters and Flags)
[cols="1m,3",options="header",width="100%"]
|===
| Name
| Description

| batchIdx
| [[batchIdx]] Current batch index that is the index of an `InternalRow` in the <<columnarBatch, ColumnarBatch>>. Used when `VectorizedParquetRecordReader` is requested to <<getCurrentValue, getCurrentValue>> with the <<returnColumnarBatch, returnColumnarBatch>> flag disabled

Starts at `0`

Increments every <<nextKeyValue, nextKeyValue>>

Reset to `0` when <<nextBatch, reading next rows into a columnar batch>>

| columnarBatch
| [[columnarBatch]] [ColumnarBatch](ColumnarBatch.md)

| columnReaders
| [[columnReaders]] <<spark-sql-VectorizedColumnReader.md#, VectorizedColumnReaders>> (one reader per column) to <<nextBatch, read rows as batches>>

Intialized when <<checkEndOfRowGroup, checkEndOfRowGroup>> (when requested to <<nextBatch, read next rows into a columnar batch>>)

| columnVectors
| [[columnVectors]] Allocated `WritableColumnVectors`

| MEMORY_MODE
a| [[MEMORY_MODE]] Memory mode of the <<columnarBatch, ColumnarBatch>>

* [[OFF_HEAP]] `OFF_HEAP` (when <<useOffHeap, useOffHeap>> is on as per spark-sql-properties.md#spark.sql.columnVector.offheap.enabled[spark.sql.columnVector.offheap.enabled] configuration property)
* [[ON_HEAP]] `ON_HEAP`

Used exclusively when `VectorizedParquetRecordReader` is requested to <<initBatch, initBatch>>.

| missingColumns
| [[missingColumns]] Bitmap of columns (per index) that are missing (or simply the ones that the reader should not read)

| numBatched
| [[numBatched]]

| returnColumnarBatch
| [[returnColumnarBatch]] Optimization flag to control whether `VectorizedParquetRecordReader` offers rows as the <<columnarBatch, ColumnarBatch>> or one row at a time only

Default: `false`

Enabled (`true`) when `VectorizedParquetRecordReader` is requested to <<enableReturningBatches, enable returning batches>>

Used in <<nextKeyValue, nextKeyValue>> (to <<nextBatch, read next rows into a columnar batch>>) and <<getCurrentValue, getCurrentValue>> (to return the internal <<columnarBatch, ColumnarBatch>> not a single `InternalRow`)

| rowsReturned
| [[rowsReturned]] Number of rows read already

| totalCountLoadedSoFar
| [[totalCountLoadedSoFar]]

| totalRowCount
| [[totalRowCount]] Total number of rows to be read

|===

=== [[nextKeyValue]] `nextKeyValue` Method

[source, java]
----
boolean nextKeyValue() throws IOException
----

NOTE: `nextKeyValue` is part of Hadoop's https://hadoop.apache.org/docs/r2.7.4/api/org/apache/hadoop/mapred/RecordReader.html[RecordReader] to read (key, value) pairs from a Hadoop https://hadoop.apache.org/docs/r2.7.4/api/org/apache/hadoop/mapred/InputSplit.html[InputSplit] to present a record-oriented view.

`nextKeyValue`...FIXME

[NOTE]
====
`nextKeyValue` is used when:

* `NewHadoopRDD` is requested to compute a partition (`compute`)

* `RecordReaderIterator` is requested to <<spark-sql-RecordReaderIterator.md#hasNext, check whether or not there are more internal rows>>
====

=== [[resultBatch]] `resultBatch` Method

[source, java]
----
ColumnarBatch resultBatch()
----

`resultBatch` gives <<columnarBatch, columnarBatch>> if available or does <<initBatch, initBatch>>.

NOTE: `resultBatch` is used exclusively when `VectorizedParquetRecordReader` is requested to <<nextKeyValue, nextKeyValue>>.

=== [[initialize]] Initializing -- `initialize` Method

[source, java]
----
void initialize(InputSplit inputSplit, TaskAttemptContext taskAttemptContext)
----

NOTE: `initialize` is part of spark-sql-SpecificParquetRecordReaderBase.md#initialize[SpecificParquetRecordReaderBase Contract] to...FIXME.

`initialize`...FIXME

=== [[enableReturningBatches]] `enableReturningBatches` Method

[source, java]
----
void enableReturningBatches()
----

`enableReturningBatches` simply turns <<returnColumnarBatch, returnColumnarBatch>> internal flag on.

`enableReturningBatches` is used when `ParquetFileFormat` is requested for a [data reader](spark-sql-ParquetFileFormat.md#buildReaderWithPartitionValues) (for [vectorized parquet decoding in whole-stage codegen](spark-sql-ParquetFileFormat.md#supportBatch)).

=== [[initBatch]] Initializing Columnar Batch -- `initBatch` Method

[source, java]
----
void initBatch(StructType partitionColumns, InternalRow partitionValues) // <1>
// private
private void initBatch() // <2>
private void initBatch(
  MemoryMode memMode,
  StructType partitionColumns,
  InternalRow partitionValues)
----
<1> Uses <<MEMORY_MODE, MEMORY_MODE>>
<2> Uses <<MEMORY_MODE, MEMORY_MODE>> and no `partitionColumns` and no `partitionValues`

`initBatch` creates the batch spark-sql-schema.md[schema] that is spark-sql-SpecificParquetRecordReaderBase.md#sparkSchema[sparkSchema] and the input `partitionColumns` schema.

`initBatch` requests spark-sql-OffHeapColumnVector.md#allocateColumns[OffHeapColumnVector] or spark-sql-OnHeapColumnVector.md#allocateColumns[OnHeapColumnVector] to allocate column vectors per the input `memMode`, i.e. <<OFF_HEAP, OFF_HEAP>> or <<ON_HEAP, ON_HEAP>> memory modes, respectively. `initBatch` records the allocated column vectors as the internal <<columnVectors, WritableColumnVectors>>.

[NOTE]
====
spark-sql-properties.md#spark.sql.columnVector.offheap.enabled[spark.sql.columnVector.offheap.enabled] configuration property controls <<OFF_HEAP, OFF_HEAP>> or <<ON_HEAP, ON_HEAP>> memory modes, i.e. `true` or `false`, respectively.

`spark.sql.columnVector.offheap.enabled` is disabled by default which means that spark-sql-OnHeapColumnVector.md[OnHeapColumnVector] is used.
====

`initBatch` creates a [ColumnarBatch](ColumnarBatch.md) (with the <<columnVectors, allocated WritableColumnVectors>>) and records it as the internal <<columnarBatch, ColumnarBatch>>.

`initBatch` creates new slots in the <<columnVectors, allocated WritableColumnVectors>> for the input `partitionColumns` and sets the input `partitionValues` as constants.

`initBatch` initializes <<missingColumns, missing columns>> with `nulls`.

`initBatch` is used when:

* `VectorizedParquetRecordReader` is requested for [resultBatch](#resultBatch)
* `ParquetFileFormat` is requested to [build a data reader with partition column values appended](spark-sql-ParquetFileFormat.md#buildReaderWithPartitionValues)

=== [[nextBatch]] Reading Next Rows Into Columnar Batch -- `nextBatch` Method

[source, java]
----
boolean nextBatch() throws IOException
----

`nextBatch` reads at least <<capacity, capacity>> rows and returns `true` when there are rows available. Otherwise, `nextBatch` returns `false` (to "announce" there are no rows available).

Internally, `nextBatch` firstly requests every <<spark-sql-WritableColumnVector.md#, WritableColumnVector>> (in the <<columnVectors, columnVectors>> internal registry) to <<spark-sql-WritableColumnVector.md#reset, reset itself>>.

`nextBatch` requests the <<columnarBatch, ColumnarBatch>> to [specify the number of rows (in batch)](ColumnarBatch.md#setNumRows) as `0` (effectively resetting the batch and making it available for reuse).

When the <<rowsReturned, rowsReturned>> is greater than the <<totalRowCount, totalRowCount>>, `nextBatch` finishes with (_returns_) `false` (to "announce" there are no rows available).

`nextBatch` <<checkEndOfRowGroup, checkEndOfRowGroup>>.

`nextBatch` calculates the number of rows left to be returned as a minimum of the <<capacity, capacity>> and the <<totalCountLoadedSoFar, totalCountLoadedSoFar>> reduced by the <<rowsReturned, rowsReturned>>.

`nextBatch` requests every <<columnReaders, VectorizedColumnReader>> to <<spark-sql-VectorizedColumnReader.md#readBatch, readBatch>> (with the number of rows left to be returned and associated <<columnVectors, WritableColumnVector>>).

NOTE: <<columnReaders, VectorizedColumnReaders>> use their own <<columnVectors, WritableColumnVectors>> for storing values read. The numbers of <<columnReaders, VectorizedColumnReaders>> and <<columnVectors, WritableColumnVector>> are equal.

NOTE: The number of rows in the internal <<columnarBatch, ColumnarBatch>> matches the number of rows that <<columnReaders, VectorizedColumnReaders>> decoded and stored in corresponding <<columnVectors, WritableColumnVectors>>.

In the end, `nextBatch` registers the progress as follows:

* The number of rows read is added to the <<rowsReturned, rowsReturned>> counter

* Requests the internal <<columnarBatch, ColumnarBatch>> to [set the number of rows (in batch)](ColumnarBatch.md#setNumRows) to be the number of rows read

* The <<numBatched, numBatched>> registry is exactly the number of rows read

* The <<batchIdx, batchIdx>> registry becomes `0`

`nextBatch` finishes with (_returns_) `true` (to "announce" there are rows available).

NOTE: `nextBatch` is used exclusively when `VectorizedParquetRecordReader` is requested to <<nextKeyValue, nextKeyValue>>.

=== [[checkEndOfRowGroup]] `checkEndOfRowGroup` Internal Method

[source, java]
----
void checkEndOfRowGroup() throws IOException
----

`checkEndOfRowGroup`...FIXME

NOTE: `checkEndOfRowGroup` is used exclusively when `VectorizedParquetRecordReader` is requested to <<nextBatch, read next rows into a columnar batch>>.

=== [[getCurrentValue]] Getting Current Value (as Columnar Batch or Single InternalRow) -- `getCurrentValue` Method

[source, java]
----
Object getCurrentValue()
----

NOTE: `getCurrentValue` is part of the Hadoop https://hadoop.apache.org/docs/r2.7.5/api/org/apache/hadoop/mapreduce/RecordReader.html[RecordReader] Contract to break the data into key/value pairs for input to a Hadoop `Mapper`.

`getCurrentValue` returns the entire <<columnarBatch, ColumnarBatch>> with the <<returnColumnarBatch, returnColumnarBatch>> flag enabled (`true`) or requests it for a [single row](ColumnarBatch.md#getRow) instead.

[NOTE]
====
`getCurrentValue` is used when:

* `NewHadoopRDD` is requested to compute a partition (`compute`)

* `RecordReaderIterator` is requested for the <<spark-sql-RecordReaderIterator.md#next, next internal row>>
====
