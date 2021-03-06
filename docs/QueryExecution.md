# QueryExecution &mdash; Structured Query Execution Pipeline

`QueryExecution` is the [execution pipeline](#execution-pipeline) (_workflow_) of a [structured query](#logical).

`QueryExecution` is made up of **execution stages** (phases).

![Query Execution &mdash; From SQL through Dataset to RDD](images/QueryExecution-execution-pipeline.png)

`QueryExecution` is the result of [executing a LogicalPlan in a SparkSession](SessionState.md#executePlan) (and so you could create a `Dataset` from a [logical operator](logical-operators/LogicalPlan.md) or use the `QueryExecution` after executing a logical operator).

```text
val plan: LogicalPlan = ...
val qe = new QueryExecution(sparkSession, plan)
```

## Creating Instance

`QueryExecution` takes the following to be created:

* <span id="sparkSession"> [SparkSession](SparkSession.md)
* <span id="logical"> [Logical Query Plan](logical-operators/LogicalPlan.md)
* [QueryPlanningTracker](#tracker)

`QueryExecution` is created when:

* [Dataset.ofRows](Dataset.md#ofRows) and [Dataset.selectUntyped](Dataset.md#selectUntyped) are executed
* `KeyValueGroupedDataset` is requested to [aggUntyped](spark-sql-KeyValueGroupedDataset.md#aggUntyped)
* `CommandUtils` utility is requested to [computeColumnStats](spark-sql-CommandUtils.md#computeColumnStats) and [computePercentiles](spark-sql-CommandUtils.md#computePercentiles)
* `BaseSessionStateBuilder` is requested to [create a QueryExecution for a LogicalPlan](BaseSessionStateBuilder.md#createQueryExecution)

## <span id="tracker"> QueryPlanningTracker

`QueryExecution` can be given a [QueryPlanningTracker](QueryPlanningTracker.md) when [created](#creating-instance).

## Accessing QueryExecution

`QueryExecution` is part of `Dataset` using [queryExecution](Dataset.md#queryExecution) attribute.

```text
val ds: Dataset[Long] = ...
ds.queryExecution
```

## <span id="attributes"><span id="execution-pipeline"><span id="query-plan-lifecycle"> Execution Pipeline Phases

### <span id="analyzed"> Analyzed Logical Plan

Analyzed [logical plan](#logical) that has passed [Logical Analyzer](Analyzer.md).

!!! tip
    Beside `analyzed`, you can use [Dataset.explain](spark-sql-dataset-operators.md#explain) basic action (with `extended` flag enabled) or SQL's `EXPLAIN EXTENDED` to see the analyzed logical plan of a structured query.

### <span id="withCachedData"> Analyzed Logical Plan with Cached Data

[Analyzed](#analyzed) logical plan after `CacheManager` was requested to [replace logical query segments with cached query plans](CacheManager.md#useCachedData).

`withCachedData` makes sure that the logical plan was [analyzed](#assertAnalyzed) and [uses supported operations only](#assertSupported).

### <span id="optimizedPlan"> Optimized Logical Plan

Logical plan after executing the [logical query plan optimizer](SessionState.md#optimizer) on the [withCachedData](#withCachedData) logical plan.

### <span id="sparkPlan"> Physical Plan

[Physical plan](physical-operators/SparkPlan.md) (after [SparkPlanner](SparkPlanner.md) has planned the [optimized logical plan](#optimizedPlan)).

`sparkPlan` is the first physical plan from the collection of all possible physical plans.

!!! note
    It is guaranteed that Catalyst's `QueryPlanner` (which `SparkPlanner` extends) [will always generate at least one physical plan](catalyst/QueryPlanner.md#plan).

### <span id="executedPlan"> Optimized Physical Plan

Optimized physical plan that is in the final optimized "shape" and therefore ready for execution, i.e. the [physical sparkPlan](#sparkPlan) with [physical preparation rules applied](#prepareForExecution).

### <span id="toRdd"> RDD

```scala
toRdd: RDD[InternalRow]
```

Spark Core's execution graph of a distributed computation (`RDD` of [internal binary rows](spark-sql-InternalRow.md)) from the [executedPlan](#executedPlan) after [execution](physical-operators/SparkPlan.md#execute).

The `RDD` is the top-level RDD of the DAG of RDDs (that represent physical operators).

!!! note
    `toRdd` is a "boundary" between two Spark modules: Spark SQL and Spark Core.

    After you have executed `toRdd` (directly or not), you basically "leave" Spark SQL's Dataset world and "enter" Spark Core's RDD space.

`toRdd` triggers a structured query execution (i.e. physical planning, but not execution of the plan) using [SparkPlan.execute](physical-operators/SparkPlan.md#execute) that recursively triggers execution of every child physical operator in the physical plan tree.

!!! note
    [SparkSession.internalCreateDataFrame](SparkSession.md#internalCreateDataFrame) applies a [schema](StructType.md) to an `RDD[InternalRow]`.

!!! note
    [Dataset.rdd](spark-sql-dataset-operators.md#rdd) gives the `RDD[InternalRow]` with internal binary rows deserialized to a concrete Scala type.

You can access the lazy attributes as follows:

```text
val dataset: Dataset[Long] = ...
dataset.queryExecution.executedPlan
```

`QueryExecution` uses the [Logical Query Optimizer](catalyst/Optimizer.md) and [Tungsten](spark-sql-tungsten.md) for better structured query performance.

`QueryExecution` uses the input `SparkSession` to access the current [SparkPlanner](SparkPlanner.md) (through [SessionState](SessionState.md)) when <<creating-instance, it is created>>. It then computes a [SparkPlan](physical-operators/SparkPlan.md) (a `PhysicalPlan` exactly) using the planner. It is available as the <<sparkPlan, `sparkPlan` attribute>>.

!!! note
    A variant of `QueryExecution` that Spark Structured Streaming uses for query planning is `IncrementalExecution`.

    Refer to [IncrementalExecution — QueryExecution of Streaming Datasets](https://jaceklaskowski.gitbooks.io/spark-structured-streaming/spark-sql-streaming-IncrementalExecution.html) in the Spark Structured Streaming online gitbook.

## <span id="planner"> SparkPlanner

[SparkPlanner](SparkPlanner.md)

## <span id="stringWithStats"> Text Representation With Statistics

```scala
stringWithStats: String
```

`stringWithStats`...FIXME

`stringWithStats` is used when [ExplainCommand](logical-operators/ExplainCommand.md) logical command is executed (with `cost` flag enabled).

## <span id="preparations"> Physical Query Optimizations (Physical Plan Preparation Rules)

```scala
preparations: Seq[Rule[SparkPlan]]
```

`preparations` creates an [InsertAdaptiveSparkPlan](physical-optimizations/InsertAdaptiveSparkPlan.md) (with a new [AdaptiveExecutionContext](physical-optimizations/AdaptiveExecutionContext.md)) that is added to the [preparations rules](#preparations-internal-utility).

`preparations` is used when `QueryExecution` is requested for an [optimized physical query plan](#executedPlan).

### preparations Internal Utility

```scala
preparations(
  sparkSession: SparkSession,
  adaptiveExecutionRule: Option[InsertAdaptiveSparkPlan] = None): Seq[Rule[SparkPlan]]
```

`preparations` is the set of [Catalyst Rules](catalyst/Rule.md) for transforming [physical operators](physical-operators/SparkPlan.md) (to be more efficient and optimized for execution) in the following order:

1. [InsertAdaptiveSparkPlan](physical-optimizations/InsertAdaptiveSparkPlan.md) (if defined)
1. [PlanDynamicPruningFilters](physical-optimizations/PlanDynamicPruningFilters.md)
1. [PlanSubqueries](physical-optimizations/PlanSubqueries.md)
1. [EnsureRequirements](physical-optimizations/EnsureRequirements.md)
1. [ApplyColumnarRulesAndInsertTransitions](physical-optimizations/ApplyColumnarRulesAndInsertTransitions.md)
1. [CollapseCodegenStages](physical-optimizations/CollapseCodegenStages.md)
1. [ReuseExchange](physical-optimizations/ReuseExchange.md)
1. [ReuseSubquery](physical-optimizations/ReuseSubquery.md)

`preparations` is used when:

* `QueryExecution` is requested for [physical optimization rules (preparations)](#preparations)

* `QueryExecution` utility is requested to [prepareExecutedPlan](#prepareExecutedPlan)

## <span id="prepareExecutedPlan"><span id="prepareExecutedPlan-SparkPlan"> prepareExecutedPlan for Physical Operators

```scala
prepareExecutedPlan(
  spark: SparkSession,
  plan: SparkPlan): SparkPlan
```

`prepareExecutedPlan` is...FIXME

`prepareExecutedPlan` is used when:

* `QueryExecution` utility is requested to [prepareExecutedPlan for a logical operator](#prepareExecutedPlan-LogicalPlan)

* [PlanDynamicPruningFilters](physical-optimizations/PlanDynamicPruningFilters.md) physical optimization is executed

## <span id="prepareExecutedPlan-LogicalPlan"> prepareExecutedPlan for Logical Operators

```scala
prepareExecutedPlan(
  spark: SparkSession,
  plan: LogicalPlan): SparkPlan
```

`prepareExecutedPlan` is...FIXME

`prepareExecutedPlan` is used when [PlanSubqueries](physical-optimizations/PlanSubqueries.md) physical optimization is executed.

## <span id="prepareForExecution"> Applying preparations Physical Query Optimization Rules to Physical Plan

```scala
prepareForExecution(
  plan: SparkPlan): SparkPlan
```

`prepareForExecution` takes <<preparations, physical preparation rules>> and applies them one by one to the input physical `plan`.

`prepareForExecution` is used when `QueryExecution` is requested to <<executedPlan, prepare the physical plan for execution>>.

## <span id="assertSupported"> assertSupported Method

```scala
assertSupported(): Unit
```

`assertSupported` requests `UnsupportedOperationChecker` to [checkForBatch](spark-sql-UnsupportedOperationChecker.md#checkForBatch) when...FIXME

`assertSupported` is used when `QueryExecution` is requested for [withCachedData](#withCachedData) logical plan.

## <span id="assertAnalyzed"> Creating Analyzed Logical Plan and Checking Correctness

```scala
assertAnalyzed(): Unit
```

`assertAnalyzed` triggers initialization of [analyzed](#analyzed) (which is almost like executing it).

`assertAnalyzed` executes [analyzed](#analyzed) by accessing it and throwing the result away. Since `analyzed` is a lazy value in Scala, it will then get initialized for the first time and stays so forever.

`assertAnalyzed` then requests `Analyzer` to [validate analysis of the logical plan](spark-sql-Analyzer-CheckAnalysis.md#checkAnalysis) (i.e. `analyzed`).

!!! note
    `assertAnalyzed` uses [SparkSession](#sparkSession) to access the current [SessionState](SparkSession.md#sessionState) that it then uses to access the [Analyzer](SessionState.md#analyzer).

In case of any `AnalysisException`, `assertAnalyzed` creates a new `AnalysisException` to make sure that it holds [analyzed](#analyzed) and reports it.

## <span id="toStringWithStats"> Building Text Representation with Cost Stats

```scala
toStringWithStats: String
```

`toStringWithStats` is a mere alias for [completeString](#completeString) with `appendStats` flag enabled.

`toStringWithStats` is a custom [toString](#toString) with [cost statistics](logical-operators/Statistics.md).

```text
val dataset = spark.range(20).limit(2)

// toStringWithStats in action - note Optimized Logical Plan section with Statistics
scala> dataset.queryExecution.toStringWithStats
res6: String =
== Parsed Logical Plan ==
GlobalLimit 2
+- LocalLimit 2
   +- Range (0, 20, step=1, splits=Some(8))

== Analyzed Logical Plan ==
id: bigint
GlobalLimit 2
+- LocalLimit 2
   +- Range (0, 20, step=1, splits=Some(8))

== Optimized Logical Plan ==
GlobalLimit 2, Statistics(sizeInBytes=32.0 B, rowCount=2, isBroadcastable=false)
+- LocalLimit 2, Statistics(sizeInBytes=160.0 B, isBroadcastable=false)
   +- Range (0, 20, step=1, splits=Some(8)), Statistics(sizeInBytes=160.0 B, isBroadcastable=false)

== Physical Plan ==
CollectLimit 2
+- *Range (0, 20, step=1, splits=Some(8))
```

`toStringWithStats` is used when [ExplainCommand](logical-operators/ExplainCommand.md) logical command is executed (with `cost` attribute enabled).

## <span id="toString"> Extended Text Representation with Logical and Physical Plans

```scala
toString: String
```

`toString` is a mere alias for [completeString](#completeString) with `appendStats` flag disabled.

!!! note
    `toString` is on the "other" side of [toStringWithStats](#toStringWithStats) which has `appendStats` flag enabled.

`toString` is part of Java's `Object` abstraction.

## <span id="simpleString"> Simple (Basic) Text Representation

```scala
// Uses formatted `false`
simpleString: String
simpleString(
  formatted: Boolean): String
```

`simpleString` requests the <<executedPlan, optimized SparkPlan>> for the [text representation](catalyst/TreeNode.md#treeString) (of all nodes in the query tree) with `verbose` flag turned off.

In the end, `simpleString` adds *== Physical Plan ==* header to the text representation and <<withRedaction, redacts sensitive information>>.

```text
import org.apache.spark.sql.{functions => f}
val q = spark.range(10).withColumn("rand", f.rand())
val output = q.queryExecution.simpleString

scala> println(output)
== Physical Plan ==
*(1) Project [id#5L, rand(6017561978775952851) AS rand#7]
+- *(1) Range (0, 10, step=1, splits=8)
```

## <span id="withRedaction"> Redacting Sensitive Information

```scala
withRedaction(
  message: String): String
```

`withRedaction` takes the value of spark-sql-properties.md#spark.sql.redaction.string.regex[spark.sql.redaction.string.regex] configuration property (as the regular expression to point at sensitive information) and requests Spark Core's `Utils` to redact sensitive information in the input `message`.

NOTE: Internally, Spark Core's `Utils.redact` uses Java's `Regex.replaceAllIn` to replace all matches of a pattern with a string.

NOTE: `withRedaction` is used when `QueryExecution` is requested for the <<simpleString, simple>>, <<toString, extended>> and <<stringWithStats, with statistics>> text representations.

## <span id="writePlans"> writePlans

```scala
writePlans(
   append: String => Unit,
   maxFields: Int): Unit
```

`writePlans`...FIXME

`writePlans` is used when...FIXME
