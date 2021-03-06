# Artifact for "Automatic Detection of Performance Bugs in Database Systems using Equivalent Queries"

The artifact consists of four main components:

1. An implementation of `AMOEBA` to find all performance bugs reported in the paper.
2. A docker image for executing `AMOEBA` and inspecting its codebase.
3. A spreadsheet named `bugs-found` contains a list of bugs that we reported and additional meta information.
4. A spreadsheet named `mutation-rules` contains a list of mutation rules that `AMOEBA` uses.
5. A spreadsheet named `SQL-features` lists SQL features that `AMOEBA` covers.


## `AMOEBA` 
<img src="./pictures/architecture.jpg" width="3000">

`AMOEBA` is a tool that we created to automatically find performance bugs in Database Management Systems (DBMS). As stated in the `INSTALL.md` document, the easiest way to get started is the following:

```
docker pull icsesub2022/paper360:artifact1
docker run --net=host -it --cpus 2 -m 8G --user postgres -v </a/directory/in/your/host>:/home/postgres/exp icsesub2022/paper360:artifact1 bash
// after starting and attaching to the docker container, run the following commands: 
cd /workspace
eval "$(direnv hook bash)"
start_pg13.sh
start_cockroach.sh
```

At a high level, `AMOEBA` has three main components:
1. `GENERATOR` is a grammar-aware query generator that randomly generates base queries.
2. `MUTATOR` takes a base query as input and seeks to generate equivalent queries.
3. `VALIDATOR` takes a set of equivalent queries as input and generates a list of performance bug reports. It executes each pair
of equivalent queries on the target DBMS and observes whether any pair exhibits a significant difference in their runtime performance. If that is the case, it indicates the likely presence of a performance bug in the DBMS.

## High-level overview of `AMOEBA`'s code repository (i.e., /workspace) 
We implement three main components of `AMOEBA` in the following directories or files:
1. `GENERATOR` is implemented in `/workspace/sqlfuzz`. Specifically, you can find the logic of how does it construct query objects in the file `/workspace/sqlfuzz/src/sqlfuzz/mutator.py`.
2. `MUTATOR` is implemented in `/workspace/calcite-fuzzing`. In specific, you can find the high-level logic of how does it mutate a query in the file `/workspace/calcite-fuzzing/core/src/test/java/org/apache/calcite/test/Transformer.java.`
3. `VALIDATOR` is implemented in `/workspace/validator.py`, which provides the logic of how does it extract the cost of query execution plans and time the query execution.
In addition, in `/workspace/test_driver.py`, you can find how does `AMOEBA` coordinates the invocations of these three components.

There are three other directories that are worth mentioning here:
1. in `/workspace/amoeba_conf/`, you can find files for configuring and fine-tuning the `AMOEBA`.
2. in `/workspace/demo100`, you can find schemas and data contents we used to set up our example database instance. Since they are stored in `.csv` format, you can re-use them to set up other database instances.
3. in `/workspace/utils`, you can find scripts for starting and stopping the `PostgreSQL` and `CockroachDB`.

## How to access and inspect the container's directories and files:
There are two ways:
1. Use an IDE to attach to the running container of `AMOEBA`, which allows users to inspect its directories and files the same way as inspecting a project on a local machine. I recommend using `VSCODE` and its [`Remote-Containers`](https://code.visualstudio.com/docs/remote/attach-container) extension to achieve this.
2. Alternatively, you can also use the `docker cp` command to copy files of the container to a local directory. For example, in the host machine's terminal, you can run the following command to copy `AMOEBA`'s code repository to a local directory.
```
docker cp <containerId>:/workspace/ </a/directory/in/your/host>
```
Notably, you can run the `docker ps` command on the host machine to find the `<containerId>`.

## Run `AMOEBA` with Custom Parameters
`AMOEBA` is configurable, a launch command template looks like the following:
```
$timeout {total_timeout} ./test_driver.py --workers {num_workers} --output {outputfolder} --queries {num_queries_per_worker} --rewriter ./calcite-fuzzing --dbms={dbms_undertest} --validate --num_loops={num_feedbackloops} --feedback={conf_feedback} --dbconf=db_conf_demo.json --query_timeout {per_query_timeout}

```

You can customize the value of the following options:
- {total_timeout}: timeout for the entire run of `AMOEBA` (unit is seconds)
- {workers}: number of parallel workers to invoke `GENERATOR` and `MUTATOR`
- {output}: location to store the intermediate results and bug reports
- {queries}: number of base queries that are generated by each `GENERATOR` instance
- {dbms}: DBMS that `AMOEBA` will evaluate on (i.e., `postgresql` or `cockroachdb`) 
- {num_loops}: number of feedback loops
- {validate}: a boolean argument that decides whether to invoke the `VALIDATOR` after generating the equivalent query pairs
- {feedback}: what types of feedbacks to utilize (i.e., `both`, `none`, `mutator`, or `validator`)
- {query_timeout}: timeout for executing each query (unit is seconds)


For example, you can launch `AMOEBA` with the following command:

```
timeout 3600 ./test_driver.py --workers 1 --output /home/postgres/exp/1 --queries 100 --rewriter ./calcite-fuzzing --dbms=postgresql --validate --num_loops=100 --feedback=none --dbconf=db_conf_demo.json --query_timeout 30
```
If `AMOEBA` is working correctly, you should expect to see the following progress information is printed:
```
start query generator
['mutator.py --prob_table=/home/postgres/test/190156/prob_table_190156.json --db_info=/workspace/amoeba_conf/db_conf_demo.json -s seq --queries 100 1>/home/postgres/test/190156/0/log_sa0 2>/home/postgres/test/190156/0/input.sql']
finish query generator
start query rewriter
['java -cp calcite-core-1.22.0-SNAPSHOT-tests.jar:./*:. org.apache.calcite.test.Transformer /home/postgres/test/190156/0']
finish query rewriter
start validator
begin compare plan cost of equivalent queries
compare plan cost /home/postgres/test/190156/0/out/q20.sql
find plan diff /home/postgres/test/190156/0/out/q20.sql
compare plan cost /home/postgres/test/190156/0/out/q13.sql
find plan diff /home/postgres/test/190156/0/out/q13.sql
compare plan cost /home/postgres/test/190156/0/out/q18.sql
compare plan cost /home/postgres/test/190156/0/out/q23.sql
```
This example command should complete within 10 minutes. You can check the generated intermediate results in `/home/postgres/exp/1`. If `AMOEBA` discovers potential performance bugs, the generated bug report will live at `/home/postgres/exp/1/bugs.md`.

The shortcut CTRL+C can be used to terminate `AMOEBA` manually. Otherwise, `AMOEBA` will terminate either after a specified experiment timeout is reached or after a specified number of base queries have been examined. The option `total_timeout` controls the experiment timeout. The options `workers`,  `queries`, and `num_loops` ultimately determine the number of base queries that `AMOEBA` is going to examine.

## How to Interpret the Bug Report
An example bug report generated by `AMOEBA` looks like the following:

#### General information
* raw file: /home/postgres/exp/1/104107/0/out/q2.sql
* generatetime: 2022-01-28 10:49:40.153930
* different DB: postgresql
* error type: performance bug

#### Query
```sql

SELECT t.a from t group by t.c
```
```sql
select t.a from t
```

#### Summary
|           | First Query | Second Query |
|-----------|-------------|--------------|
| time      |  0.118798 | 0.054864   |
| idx      |  2 | 0   |

</p>
</details>

As shown above, the bug report includes a pair of equivalent queries and their corresponding execution time. In order to debug the potential performance bug, developers can utilize the `EXPLAIN` functionality in the DBMS to compare the query execution plans between equivalent queries. To this end, for the example queries, developers can run the `EXPLAIN ANALYZE SELECT t.a from t group by t.c` and `EXPLAIN ANALYZE select t.a from t` statements in Postgresql.

## List of Bugs

To provide evidence for the bugs we found, we provide a spreadsheet named `bugs-found` that contains detailed information for each bug we found, including bug characteristics reported in the paper. 

### Notable Columns in the spreadsheet

* `Classification`: This column denotes the status of the bug report, following the taxonomy that we propose in section 3 of the paper.
* `Link`: This column provides the link to the bug report that we submitted, which shows our conversations with the developers.
* `Brief Summary`: This column provides a brief summarization of the potential performance bug.
* `Time Difference`: This column denotes the runtime performance difference of equivalent queries.


## List of Mutation Rules

We provide a spreadsheet named `mutation-rules` that contains detailed information for each rule that `AMOEBA` leverages for mutating queries. 

### Notable Columns in the spreadsheet

* `Transformation`: This column briefly summarizes the transformation that each rule performs.
* `SourceCodeLocation`: This column denotes where the rule is implemented in `MUTATOR`.


## List of SQL features

We provide a spreadsheet named `SQL-features` that denotes the SQL features that `AMOEBA` covers. 


## Using `AMOEBA` for other DBMSs

The implementation can easily be extended to test new DBMSs using a database instance based on the [SCOTT schema](https://www.orafaq.com/wiki/SCOTT). To evaluate the tool on a new DBMS, there are three things to do:
1. Set up the test database instance in the new DBMS. The database instance has three tables: dept, emp, and bonus. We provide their schemas and data contents in `/workspace/demo100`.
2. Modify and rebuild the `MUTATOR` to generate equivalent query pairs in the dialect of the new DBMS. You need to update the `_rewrite` function in `{MUTATOR}/core/src/test/java/org/apache/calcite/test/RelOptTestBase.java` to provide the `RelToSqlConverter` with the correct dialect.
3. Modify the `test_driver.py` and `validator.py` to provide necessary commands for communicating with the new DBMS (i.e., start and stop the database, time the query execution, and estimate the query plan cost)



