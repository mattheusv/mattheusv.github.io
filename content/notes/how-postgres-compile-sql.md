# How PostgreSQL compile SQL?

A few days ago, [@eatonphil](https://twitter.com/eatonphil) organized a
Postgres Internals wehack group on Discord for people interested in
understanding and contributing to the PostgreSQL source code. I had previously
tried to understand more about how SQL expressions are compiled using LLVM but
without much success, with this encouragement I decided to go deeper and
understand how PostgreSQL compiles SQL code, and here I will describe my
understanding notes. 

Please let me know if something is not right, I'm just a curious guy trying to
understand Postgres source code.


## How Postgres decide if a SQL should be compiled or not?
Postgres decide if a SQL expression should be compiled or not based on the cost
of the query, if the estimated cost of the query is less than the setting
[jit_above_cost](https://www.postgresql.org/docs/current/runtime-config-query.html#GUC-JIT-ABOVE-COST), then the query will be
compiled using LLVM.

Lets see a practical example (make sure that the jit is enabled):

```sql
set jit_above_cost = 200;

create table t (a int, b int);

insert into t(a, b) select g, g+1 from generate_series(1, 10000) g(g);

explain(analyze) select a+b from t;
```

We can see that the JIT was not used to execute this query

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                             QUERY PLAN                                              │
├─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Seq Scan on t  (cost=0.00..170.00 rows=10000 width=4) (actual time=0.036..7.162 rows=10000 loops=1) │
│ Planning Time: 0.051 ms                                                                             │
│ Execution Time: 8.159 ms                                                                            │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

Lets now change the `jit_above_cost` to a low value:
```
set jit_above_cost = 10;

explain(analyze) select a+b from t;
```

We can see now that the query was compiled:
```
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                          QUERY PLAN                                                          │
├──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Seq Scan on t  (cost=0.00..170.00 rows=10000 width=4) (actual time=1.467..6.440 rows=10000 loops=1)                          │
│ Planning Time: 0.048 ms                                                                                                      │
│ JIT:                                                                                                                         │
│   Functions: 2                                                                                                               │
│   Options: Inlining false, Optimization false, Expressions true, Deforming true                                              │
│   Timing: Generation 0.120 ms (Deform 0.045 ms), Inlining 0.000 ms, Optimization 0.094 ms, Emission 1.334 ms, Total 1.548 ms │
│ Execution Time: 7.563 ms                                                                                                     │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## How a SQL expression is compiled
Postgres query plans are represented as a tree, [each Plan node on the tree may
have expression trees associated with it, to represent its target list,
qualification conditions,
etc.](https://github.com/postgres/postgres/blob/master/src/backend/executor/README#L68)
An expression tree is represented by the
[`ExprState`](https://github.com/postgres/postgres/blob/master/src/include/nodes/execnodes.h#L77)
struct which contains the instructions that the Postgres query executor should
run.

Consider the following SQL:
```sql
select a+b from t where a = 1;
```

The expression `a+b` is represented by an `ExprState` that contains the following list of steps that should be executed:

- EEOP_SCAN_FETCHSOME
    - [Fetch a specific number of attributes from a tuple](https://github.com/postgres/postgres/blob/master/src/backend/executor/execExprInterp.c#L554) (in this scenario 2 attributes, `a` and `b`)
- EEOP_SCAN_VAR
    - [Scan the `a` value from tuple values](https://github.com/postgres/postgres/blob/master/src/backend/executor/execExprInterp.c#L596)
- EEOP_SCAN_VAR
    - Scan the `b` value from tuple values
- EEOP_FUNCEXPR_STRICT
    - Execute the function that will sum `a` and `b` values
- EEOP_ASSIGN_TMP
    - Assign the result of the function to a temporary variable
- EEOP_DONE
    - Finish the expression execution

When an interpreted approach is used to execute a SQL expression, Postgres
will use the
[`ExecInterpExpr`](https://github.com/postgres/postgres/blob/master/src/backend/executor/execExprInterp.c#L395)
function to iterate over the list of steps (at `ExprState->steps[]`) and then
execute each step sequentially. On the other hand, when a SQL expression is
compiled, Postgres use the
[`ExecRunCompiledExpr`](https://github.com/postgres/postgres/blob/master/src/backend/jit/llvm/llvmjit_expr.c#L2539)
function that will compile the SQL expression into LLVM module and then load as
a shared library to execute it.

## How a LLVM module is created from SQL expressions
A LLVM module with all SQL expressions that should be compiled is build at
startup phase  of executor (when some preparation of execution is usually done)
via `ExecReadyExpr` function.  The
[`llvm_compile_expr`](https://github.com/postgres/postgres/blob/master/src/backend/jit/llvm/llvmjit_expr.c#L78)
is used internally to emit the LLVM IR code. Each `ExprState` generates a LLVM
module to be compiled and for each step on `ExprState->steps[]` array, Postgres
will emit a LLVM IR code to execute this step. 

When executing the SQL expression `a + b` interpreted, Postgres calls the
[`fetch_att`]() function to fetch each column value from a single tuple, and to
do this, a switch case on the length of data type is necessary to know how to
read the actual value from the page tuple. When compiling this same expression
Postgres will emit a LLVM code that will already know that the data type of
both columns is an int32, so the switch case statement will not be necessary,
which can reduce the query execution time when running for multiple tuples. 

For the `EEOP_SCAN_FETCHSOME` step for example, the [slot_compile_deform](https://github.com/postgres/postgres/blob/master/src/backend/jit/llvm/llvmjit_deform.c#L34)
function is used to generate the LLVM code that know exactly how to interpret
the datum values of a tuple without doing the switch case statements. The LLVM code
that would be compiled in this case will always work on datum values assuming
the data types is always an int32.

{{< details "If you are curious about how the LLVM code for this scenario looks like" >}}
```llvm
; Function Attrs: mustprogress nofree norecurse nosync nounwind uwtable willreturn writeonly
define internal void @deform_0_1(%struct.TupleTableSlot* nocapture noundef writeonly align 8 %0) #0 {
entry:
  %1 = getelementptr inbounds %struct.TupleTableSlot, %struct.TupleTableSlot* %0, i32 0, i32 5
  %tts_values = load i64*, i64** %1, align 8
  %2 = getelementptr inbounds %struct.TupleTableSlot, %struct.TupleTableSlot* %0, i32 0, i32 6
  %tts_ISNULL = load i8*, i8** %2, align 8
  %3 = getelementptr inbounds %struct.TupleTableSlot, %struct.TupleTableSlot* %0, i32 0, i32 1
  %4 = getelementptr inbounds %struct.TupleTableSlot, %struct.TupleTableSlot* %0, i32 0, i32 2
  %heapslot = bitcast %struct.TupleTableSlot* %0 to %struct.HeapTupleTableSlot*
  %5 = getelementptr inbounds %struct.HeapTupleTableSlot, %struct.HeapTupleTableSlot* %heapslot, i32 0, i32 2
  %6 = getelementptr inbounds %struct.HeapTupleTableSlot, %struct.HeapTupleTableSlot* %heapslot, i32 0, i32 1
  %tupleheader = load %struct.HeapTupleData*, %struct.HeapTupleData** %6, align 8
  %7 = getelementptr inbounds %struct.HeapTupleData, %struct.HeapTupleData* %tupleheader, i32 0, i32 3
  %tuple = load %struct.HeapTupleHeaderData*, %struct.HeapTupleHeaderData** %7, align 8
  %8 = getelementptr inbounds %struct.HeapTupleHeaderData, %struct.HeapTupleHeaderData* %tuple, i32 0, i32 5
  %t_bits = bitcast [0 x i8]* %8 to i8*
  %9 = getelementptr inbounds %struct.HeapTupleHeaderData, %struct.HeapTupleHeaderData* %tuple, i32 0, i32 3
  %infomask1 = load i16, i16* %9, align 2
  %10 = getelementptr inbounds %struct.HeapTupleHeaderData, %struct.HeapTupleHeaderData* %tuple, i32 0, i32 2
  %infomask2 = load i16, i16* %10, align 2
  %11 = and i16 1, %infomask1
  %hasnulls = icmp ne i16 %11, 0
  %maxatt = and i16 2047, %infomask2
  %12 = getelementptr inbounds %struct.HeapTupleHeaderData, %struct.HeapTupleHeaderData* %tuple, i32 0, i32 4
  %13 = load i8, i8* %12, align 1
  %t_hoff = zext i8 %13 to i32
  %14 = bitcast %struct.HeapTupleHeaderData* %tuple to i8*
  %v_tupdata_base = getelementptr i8, i8* %14, i32 %t_hoff
  %v_slot_off = load i32, i32* %5, align 4
  %15 = zext i32 %v_slot_off to i64
  %16 = icmp ult i16 %maxatt, 2
  br i1 %16, label %adjust_unavail_cols, label %find_startblock
```
{{< /details >}}

## Next steps
As far as I understood the overall flow of how SQL is actually compiled on
Postgres I wish now I could understand more about how LLVM APIs is used to
compile more complex expressions and then maybe try to play around with adding
support for jit to a new expression.




<!-- 
-- Some Other notes:
-------------------

The [`ExprState`](https://github.com/postgres/postgres/blob/master/src/include/nodes/execnodes.h#L77) struct represents a SQL expression that can be executed, e.g; 
`a+b` or `a = 1` from the following example:
```sql
select a+b from t where a = 1;
```

The steps of an `ExprState` that the executor should execute is stored at
[`ExprState->steps[]`](https://github.com/postgres/postgres/blob/master/src/include/nodes/execnodes.h#L77)
array. The
[`ExecReadyInterpretedExpr`](https://github.com/postgres/postgres/blob/master/src/backend/executor/execExprInterp.c#L235)
function take care of executing these steps when a query is interpreted
executed (via `ExecInterpExpr`). When the query is compiled using jit,
[`ExecRunCompiledExpr`](https://github.com/postgres/postgres/blob/master/src/backend/jit/llvm/llvmjit_expr.c#L2539)
will be called that will compile the llvm module created from the SQL
expression. The llvm module with all SQL expressions that should be compiled is
build at startup phase of executor via `ExecReadyExpr` using
[`llvm_compile_expr`](https://github.com/postgres/postgres/blob/master/src/backend/jit/llvm/llvmjit_expr.c#L78)
under the hood. 

When a query is interpreted executed `ExecInterpExpr` iterates over
`ExprState->steps[]` and execute each step sequentially, in the other hand when
the query is compiled, the `llvm_compile_expr` iterate over this array of steps
and emit llvm IR code that will later be compiled at `ExecRunCompiledExpr` as
shared library.

## How a LLVM module is created from SQL expressions
Each `ExprState` generates a LLVM module that will be compiled, the `a+b`
expression from the example above generates the following array of steps:

- EEOP_SCAN_FETCHSOME
    - [Fetch a specific number of attributes from a tuple](https://github.com/postgres/postgres/blob/master/src/backend/executor/execExprInterp.c#L554) (in this scenario 2 attributes, `a` and `b`)
- EEOP_SCAN_VAR
    - [Scan the `a` value from tuple values](https://github.com/postgres/postgres/blob/master/src/backend/executor/execExprInterp.c#L596)
- EEOP_SCAN_VAR
    - Scan the `b` value from tuple values
- EEOP_FUNCEXPR_STRICT
    - Execute the function that will sum `a` and `b` values
- EEOP_ASSIGN_TMP
    - Assign the result of the function to a temporary variable
- EEOP_DONE
    - Finish the execution

Postgres will emit specialized LLVM IR code separately for each step inside
`ExprState`. What I mean about specialized code is that for the
`EEOP_SCAN_FETCHSOME` step for example, Postgres will emit code that will know
exactly in how to read the table attributes from a tuple, and [will not need to
do switch case type checking at
runtime](https://github.com/postgres/postgres/blob/master/src/backend/executor/execTuples.c#L1006)
for each attribute from each tuple in a table. The PostgreSQL C function that
generate the tuple deforming LLVM function is
[slot_compile_deform](https://github.com/postgres/postgres/blob/master/src/backend/jit/llvm/llvmjit_deform.c#L34).
-->
