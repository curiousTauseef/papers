## [DryadLINQ: A System for General-Purpose Distributed Data-Parallel Computing Using a High-Level Language (2008)](#dryadlinq-a-system-for-general-purpose-distributed-data-parallel-computing-using-a-high-level-language-2008)
DryadLINQ is a language extension and system which allows users to write
data-parallel code in a high-level language (i.e. LINQ supported languages like
C#, VB, F#, etc.) that operates over typed .NET objects. The code is then
automatically and transparently distributed by translating it to an optimized
Dryad job. MapReduce

- doesn't really have a type system;
- forces users to express complex programs as multiple MapReduce jobs; and
- doesn't support interstage optimizations.

Relation databases

- have very primitive type systems; and
- SQL is hard to use to express complex iterative workloads.

DryadLINQ doesn't suffer from these disadvantages.

**System Architecture.**
DryadLINQ compiles LINQ programs to Dryad jobs. To review, Dryad represents
computation as a graph where vertexes run C++ code and edges transport messages
between vertexes using temporary files, TCP connections, or in-memory pipes.
Dryad

1. instantiates dataflow graphs;
2. schedules tasks on workers;
3. handles faults by re-running failed vertexes;
4. monitors the progress of a job and exports it to users; and
5. facilitates runtime graph optimizations.

DryadLINQ execution follows the following the steps:

1. .NET programs lazily generate a DryadLINQ expression.
2. The program calls `ToDryadTable` which forces the execution of the
   expression.
3. The system compiles the expression into a Dryad graph.
4. A Job Manager (JM) is launched.
5. The JM spawns workers on the cluster.
6. Workers execute vertexes.
7. When the Dryad job finishes, the output is written into a table.
8. A `DryadTable` object is created.
9. Control is returned to the user.
10. Repeat.

**Programming with DryadLINQ.**

*LINQ.*
Users write programs using LINQ. The base type in LINQ is `IEnumerable<T>`
which represents an arbitrary iterable collection. There is also an
`IQueryable<T>` subclass of `IEnumerable<T>` which represents an unevaluated
LINQ expression which produces an `IEnumerable<T>`. `IQueryable`s are computed
lazily. LINQ supports a SQL-like declarative interface as well as a more
imperative object-oriented style.

*DryadLINQ.*
DryadLINQ adds a couple features and restrictions to LINQ. DryadLINQ datasets
are stored in `DryadTable<T>`s: a distributed collection partitioned across a
number of machines. A `DryadTable` is a subtype of `IQueryable` and can be
stored in a distributed file system, a relational database, etc. Input and
output `DryadTable`s are generated by calling `GetTable` and `ToDryadTable`.
DryadLINQ adds new operators to partition data. It also provides two escape
hatch operators: `Apply` and `Fork`. `Apply` calls arbitrary code with an
iterator over a `DryadTable`. `Fork` calls arbitrary code which generates
multiple different outputs. In the worst case, a `Fork` or `Apply` is executed
at a single node. DryadLINQ also allows users to specify optimization hints to
the compiler.

*Building on DryadLINQ.*
It's easy to build even higher level interfaces on top of DryadLINQ. For
example, it's easy to build MapReduce and ML libraries on top of DryadLINQ.

**System Implementation.**

*Execution Plan Graph.*
DryadLINQ converts LINQ expressions into Execution Plan Graphs (EPG) which are
like query plans in a traditional relational database query optimizer. The EPG
is optimized and acts as a skeleton for the final Dryad graph it is converted
to. Vertexes of the EPG are annotated with type information and the edges are
annotated with partitioning and ordering information.

*Static Optimizations.*
- *Pipelining.* If multiple vertexes are chained directly together, Dryad can
  run them in a single process.
- *Removing Redundancy.* DryadLINQ tries to avoid unnecessary or redundant
  partitions since they are expensive.
- *Eager Aggregation.* If an aggregation can be pushed before a partitioning,
  it is.
- *I/O Reduction.* DryadLINQ tries to make all edges TCP pipes or in-memory
  pipes. It also compresses data before a partition.

*Dynamic Optimizations.*
DryadLINQ performs tree aggregation, but the structure of the tree depends on
scheduling decisions that are made at runtime. It also can dynamically
determine the number of partitions or how a dataset is partitioned at runtime
based on the size of the dataset. Traditional databases would often have to
estimate this information and could be wildly wrong.

*Code Generation.*
When an EPG is converted into a Dryad graph, code is generated to (a) process
data on vertexes and (b) serialize and deserialize data. Also, any values that
the LINQ programs reference have to be serialized and shipped with the code.
Any libraries the code depends on also have to be shipped.

*Leveraging Other LINQ Providers.*
PLINQ is a system which runs LINQ programs in parallel on a single multicore
machine. DryadLINQ uses PLINQ within a vertex. It also uses a LINQ-to-SQL
converter which allows vertexes to run SQL queries constructed from LINQ
expressions.

*Debugging.*
DryadLINQ's typechecking helps catch a lot of bugs. An entire cluster can also
be run on a user's local machine to debug the execution of the program. If a
node fails while running, the system can also re-run the node to see what the
problem is. Performance debugging is doable but still challenging.
