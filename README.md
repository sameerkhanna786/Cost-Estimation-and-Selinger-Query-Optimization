# Cost Estimation and Query Optimization

## Background
A query optimizer attempts to find the optimal execution plan for a SQL statement. The optimizer selects the plan with the lowest estimated cost among all considered candidate plans. The optimizer uses available statistics to estimate cost. Because the database has many internal statistics and tools at its disposal, the optimizer is usually in a better position than the user to determine the optimal method of statement execution.

For a specific query in a given environment, the cost computation accounts for metrics of query execution such as I/O. For example, consider a query that selects all employees who are managers. If the statistics indicate that 80% of employees are managers, then the optimizer may decide that a full table scan is most efficient. However, if statistics indicate that very few employees are managers and there is an index on that key, then reading an index followed by a table access by rowid may be more efficient than a full table scan.

## Background: Query Interface
Databases are represented by `Database` objects that can be created as follows:
```java
Database db = new Database('myDBFolder');
```
This creates a database where all of the tables will be stored in the `myDBFolder` directory on your filesystem.
The next relevant class is the `Schema` class, which defines table schemas.
`Schema` objects can be created by providing a list of field names and field types:
```
List<String> names = Arrays.asList("boolAttr", "intAttr",
                                   "stringAttr", "floatAttr");

List<Type> types = Arrays.asList(Type.boolType(), Type.intType(),
                                  Type.stringType(5), Type.floatType());

Schema s = new Schema(names, types);
```
Tables can be created as follows:
```
//creates a table with with schema s
db.createTable(s, "myTableName");

//creates a table with with schema s and builds an index on the intAttr field
db.createTableWithIndices(s, "myTableName",
                           Arrays.asList("intAttr"));
```
The `QueryPlan` interface allows you to generate SQL-like queries without having to parse actual SQL queries:
```java

/**
* SELECT * FROM myTableName WHERE stringAttr = 'CS 186'
*/

// create a new transaction
Database.Transaction transaction = db.beginTransaction();


// add a select to the QueryPlan
QueryPlan query = transaction.query("myTableName");
query.select("stringAttr", PredicateOperator.EQUALS, "CS 186");

// execute the query and get the output
Iterator<Record> queryOutput = query.executeOptimal();
```
To consider a more complicated example:
```java

/**
* SELECT *
* FROM Students as S, Enrollment as E
* WHERE E.sid = S.sid AND
*       E.cid = 'CS 186'
*/

// create a new transaction
Database.Transaction transaction = this.database.beginTransaction();

// alias both the Students and Enrollments tables
transaction.queryAs("Students", "S");
transaction.queryAs("Enrollments", "E");

// add a join and a select to the QueryPlan
QueryPlan query = transaction.query("S");
query.join("E", "S.sid", "E.sid");
query.select("E.cid", PredicateOperator.EQUALS, "CS 186");

// execute the query and get the output
Iterator<Record> queryOutput = query.executeOptimal();
```
`query.join` specifies an equality join. Its first argument is one of the two relations to be joined; the remaining two arguments are a key from the left table and the right table to join.

Note the `executeOptimal()` method above, this returns and executes optimal query plan. To see what this query plan is, you can print the query operator object:
```
// assuming query.executeOptimal() has already been called as above
QueryOperator finalOperator = query.getFinalOperator();
System.out.println(finalOperator.toString());



type: BNLJ
leftColumn: S.sid
rightColumn: E.sid
    (left)
    type: WHERE
    column: E.cid
    predicate: EQUALS
    value: CS 186
        type: SEQSCAN
        table: E

    (right)
    type: SEQSCAN
    table: S
```

In summary, if you would like to run queries on the database, you can create a new `QueryPlan` by calling `Transaction#query` and passing the name of the base table for the query. You can then call the `QueryPlan#select`, `QueryPlan#join`, etc. methods in order to generate as simple or as complex a query as you would like. Finally, call `QueryPlan#executeOptimal` to run the query optimizer,  execute the query, and get a response of the form `Iterator<Record>`. You can also use the `Transaction#queryAs` methods to alias tables.


### Cost Estimation and Maintenance of Statistics
The first part of building the query optimizer is ensuring that each query operator has the appropriate IO cost estimates. In order to estimate IO costs for each query operator, you will need the table statistics for any input operators. This information is accessible from the `QueryOperator#getStats` method. The `TableStats` object returned represents estimated statistics of the operator's output, including information such as number of tuples and number of pages in the output among others. These statistics are generated whenever a `QueryOperator` is constructed.

**NOTE** that these statistics are meant to be approximate so please pay careful attention to how we define the quantities to track.

We used histograms to track table statistics. A histogram maintains approximate statistics about a (potentially large) set of values without explicitly storing the values.
A histogram is an ordered list of B "buckets", each of which defines a range \[low, high). For the first, B - 1 buckets, the low of the range is inclusive and the high of the range is exclusive. **Exception**: For the last Bucket the high of the range is inclusive as well.  Each bucket counts the number of values and distinct values that fall within its range:
```java
Bucket<Float> b = new Bucket(10.0, 100.0); //defines a bucket whose low value is 10 and high is 100
b.getStart(); //returns 10.0
b.getEnd(); //returns 100.0
b.increment(15);// adds the value 15 to the bucket
b.getCount();//returns the number of items added to the bucket
b.getDistinctCount();//returns the approximate number of distinct iterms added to the bucket
```

In our implementation, the `Histogram` class, you will work with a floating point histogram where low and high are defined by floats. All other data types are backed by this floating point histogram through a "quantization" function `Histogram#quantization`. The histogram tracks statistics for all the values in a column of a table; we also need to support filtering the histogram based on given predicates.  After implementing all the methods below, you should be passing all of the tests in `TestHistogram`.

### Selingger Query Optimization

To implement the single-table example in the previous part with a sequential scan:
```java
/**
* SELECT * FROM myTableName WHERE stringAttr = 'CS 186'
*/
QueryOperator source = SequentialScanOperator(transaction, myTableName);
QueryOperator select = SelectOperator(source, 'stringAttr', PredicateOperator.EQUALS, "CS 186");

select.iterator() //iterator over the results
```

To implement the join example in the previous part with a sequential scan and a block nested loop join:
```
/**
* SELECT *
* FROM Students as S, Enrollment as E
* WHERE E.sid = S.sid AND
*       E.cid = 'CS 186'
*/
QueryOperator s = SequentialScanOperator(transaction, 'Students');
QueryOperator e = SequentialScanOperator(transaction, 'Enrollment');

QueryOperator e186 = SelectOperator(e, 'cid', PredicateOperator.EQUALS, "CS 186");

BNLJOperator bjoin = BNLJOperator(s, e186, 'S.sid','E.sid', transaction);

bjoin.iterator() //iterator over the results
```
This defines a tree of `QueryOperator` objects, and `QueryPlan` finds such a tree to minimize I/O cost. Each `QueryOperator` has two relevant methods `estimateIOCost()` (which returns an estimated IO cost based on any stored statistics) and `iterator()` (which returns a iterator over the result tuples).
