# EvalPythonExec Physical Operators

`EvalPythonExec` is an [extension](#contract) of the [UnaryExecNode](UnaryExecNode.md) abstraction for [unary physical operators](#implementations) that can [evaluate PythonUDFs](#evaluate).

## Contract

### <span id="evaluate"> Evaluating PythonUDFs

```scala
evaluate(
  funcs: Seq[ChainedPythonFunctions],
  argOffsets: Array[Array[Int]],
  iter: Iterator[InternalRow],
  schema: StructType,
  context: TaskContext): Iterator[InternalRow]
```

Evaluates [PythonUDFs](#udfs) (and produces [internal binary rows](../spark-sql-InternalRow.md))

Used when `EvalPythonExec` physical operator is [executed](#doExecute)

## Implementations

* ArrowEvalPythonExec
* BatchEvalPythonExec

## Creating Instance

`EvalPythonExec` takes the following to be created:

* <span id="udfs"> [PythonUDF](../pyspark/PythonUDF.md)s
* <span id="resultAttrs"> Result [Attributes](../expressions/Attribute.md)
* <span id="child"> Child [physical operator](SparkPlan.md)

!!! note "Abstract Class"
    `EvalPythonExec` is an abstract class and cannot be created directly. It is created indirectly for the [concrete EvalPythonExecs](#implementations).

## <span id="doExecute"> Executing Physical Operator

```scala
doExecute(): RDD[InternalRow]
```

`doExecute`...FIXME

`doExecute` is part of the [SparkPlan](SparkPlan.md#doExecute) abstraction.
