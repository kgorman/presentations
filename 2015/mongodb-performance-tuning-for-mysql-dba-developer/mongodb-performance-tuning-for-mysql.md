title: MongoDB Performance Tuning for MySQL DBA's and Developers
author:
  name: Kenny Gorman
  twitter: kennygorman
  url: http://kennygorman.com
  email: kenny.gorman@rackspace.com
output: mongodb-performance-mysql.html
controls: true
theme: theme

--
# MongoDB<br>Performance<br>Tuning
**for MySQL DBA's and Developers**

*Kenny Gorman*<br>
*Chief Technologist; Data @ Rackspace*<br>
*Co-Founder @ ObjectRocket*<br>

--
### A small bit of my background

* Database Engineer, Developer, DBA, Architect, Founder

* Oracle, MySQL, PostgreSQL, MongoDB, Apache Spark

* Obsessed: [http://www.kennygorman.com](http://www.kennygorman.com)

* MongoDB 1.2. Wrote original mongostat tool

--
### MongoDB Performance Tuning at a glance

* Not unlike MySQL


* Some additional nuances


* DBA skills translate


* Mindset change



--

### Data Modeling

* I thought this talk was on tuning?

* Let's talk about Edgar.

* MongoDB is completely denormalized (!)

>A first normal form relational database is similar, but not equal to MongoDB's implementation of a document database.

* New datatypes

--

### Example of typical DB Schema

```sql
mysql> desc employees;
+------------+---------------+------+-----+---------+-------+
| Field      | Type          | Null | Key | Default | Extra |
+------------+---------------+------+-----+---------+-------+
| emp_no     | int(11)       | NO   | PRI | NULL    |       |
| birth_date | date          | NO   |     | NULL    |       |
| first_name | varchar(14)   | NO   |     | NULL    |       |
| last_name  | varchar(16)   | NO   |     | NULL    |       |
| gender     | enum('M','F') | NO   |     | NULL    |       |
| hire_date  | date          | NO   |     | NULL    |       |
+------------+---------------+------+-----+---------+-------+
mysql> desc dept_emp;
+-----------+---------+------+-----+---------+-------+
| Field     | Type    | Null | Key | Default | Extra |
+-----------+---------+------+-----+---------+-------+
| emp_no    | int(11) | NO   | PRI | NULL    |       |
| dept_no   | char(4) | NO   | PRI | NULL    |       |
| from_date | date    | NO   |     | NULL    |       |
| to_date   | date    | NO   |     | NULL    |       |
+-----------+---------+------+-----+---------+-------+
mysql> desc departments;
+-----------+-------------+------+-----+---------+-------+
| Field     | Type        | Null | Key | Default | Extra |
+-----------+-------------+------+-----+---------+-------+
| dept_no   | char(4)     | NO   | PRI | NULL    |       |
| dept_name | varchar(40) | NO   | UNI | NULL    |       |
+-----------+-------------+------+-----+---------+-------+
```

--

### Example of typical DB queries

```sql
 SELECT * FROM employees e, departments d, dept_emp de
 WHERE e.emp_no = de.emp_no AND de.dept_no = d.dept_no
 AND d.dept_name = 'Development';
```

- Yeah, Edgar would be stoked
- Holy shit this sux, no-one does this in the real world

```sql
SELECT *
FROM employees
WHERE department_name IN ('d005');
```

- Yeah, like that, because webscale


--

### Sample Schema in MongoDB

```

```

- See what I did there?

```javascript
> db.employees.findOne()
```
```json
{
	"_id" : 496277,
	"birth_date" : ISODate("2015-09-04T17:59:48.911Z"),
	"first_name" : "Georgi",
	"last_name" : "Facello",
	"gender" : "M",
	"hire_date" : ISODate("2015-09-04T17:59:48.911Z"),
	"Departments" : [
		"Marketing",
		"Development"
	]
}
```

--

### MongoDB presents new options, new challenges..

```json
{
    "_id": 100,
    "first_name": "John",
    "department": "Engineering"
}
{
    "_id": 101,
    "first_name": "Hilary",
    "Departments": ["Marketing", "Engineering", "Sales"]
}
```

- Are you freaking out?
- Someone is going to take some abstract object and call save() on it, and your world is going to end, serious, true story.


--
### Example: SFLY

- User account that owns images, images organized in folders. Goal is to minimize I/O.

Model A:

```json
    {"_id": 9999, "folder_id": 33, "photo_id": 10}
    {"_id": 9998, "folder_id": 122, "photo_id": 343}
    {"_id": 3331, "folder_id": 122, "photo_id": 321}
```

Model B:

```json
    {"_id": 9999, "photo_id": 10, "path": "vacation-london" }
    {"_id": 9998, "photo_id": 343, "path": "vacation-cabo"}
    {"_id": 3331, "photo_id": 321, "path": "vacation-cabo"}
```

```javascript
db.images.find({"path": / ^vacation / })        // all vacations
db.images.find({"path": / ^vacation-cabo /})    // just cabo
```


--

### Bottom line...

- Schema design still matters
- adding columns vs MongoDB
- phantom reads, isolation, and transactions


--

### Statement Tuning

- Similar to MySQL, look for bad statements
- Enter profiler


--

### MongoDB profiler

- Runs in the background capturing what has been run on the DB.
- Doesn't capture the plan it used to execute
- Does capture the time it spent running.

Turn it on: ```db.setProfilingLevel(level, slowms)```

- level 1 uses slowms
- level 2 is everything
- turn it on, leave it on (!)

--

### Profiler output

```javascript
{
  "ts" : ISODate("2012-09-14T16:34:00.010Z"),   // date it occurred
	"op" : "query",                            // the operation type
	"ns" : "game.players",                     // the db and collection
	"query" : { "total_games" : 1000 },        // query document
	"ntoreturn" : 0,                           // # docs returned with limit()
	"ntoskip" : 0,                             // # of docs to skip()
	"nscanned" : 959967,                       // number of docs scanned
	"keyUpdates" : 0,                          // updates of secondary indexes
	"numYield" : 1,                            // # of times yields took place
	"lockStats" : { ... },                     // subdoc of lock stats
	"nreturned" : 0,                           // # docs actually returned
	"responseLength" : 20,                     // size of doc
	"millis" : 859,                            // how long it took
	"client" : "127.0.0.1",                    // client asked for it
	"user" : ""                                // the user asking for it
}
```

--
### Explain the output

```javascript
db.players.find({ "total_games" : 1000 }).explain();
{
	"cursor" : "BtreeCursor total_games-1",                    // Btree
	"isMultiKey" : false,
	"n" : 1,
	"nscannedObjects" : 1,
	"nscanned" : 1,
	"nscannedObjectsAllPlans" : 1,
	"nscannedAllPlans" : 1,
	"scanAndOrder" : false,
	"indexOnly" : false,                                       // hint hint
	"nYields" : 0,
	"nChunkSkips" : 0,
	"millis" : 0,
	"indexBounds" : { "total_games" : [[1,1]]},                // w00t
	...
}

```

- Similar to ```SQL EXPLAIN FOR SELCT * FROM ... ```
- Pt-Visual-Explain is better.


--

### Query plans



--

### Locks

- F these locks.

| Version | Lock Granularity          | I am |
| ------------- | ----------- | -------- |
| 1.8   | Process | Pissed |
| 2.2      | Database | Pissed |
| 2.4     | Database     | Pissed |
| 2.6   | Database | Pissed |
| 3.0   | Collection | Still pissed, getting over it |
| 3.0 WT | Document | Whew |

--
### Locks, Con't

- [Readers-writer lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock)

>Locks are “writer greedy,” which means write locks have preference over reads. When both a read and write are waiting for a lock, MongoDB grants the lock to the write.

- Locks are adaptive.
- WiredTiger should help (!)
- Not really a notion of Shared or Exclusive locks like Inno

--

### Instance Tuning


--

### Random Gotcha's

- If sharded, check balancer
- TTL indexes
- Can't authoritatively kill queries, just ask nicely

```json
>db.killOp(444493);
```


--
### Value of testing; eBay/Paypal

- Abstract insert test

```perl
$sth = $dbh->prepare( "
            INSERT INTO mytable VALUES (?, ?, ?)
        " );
$sth->bind_param( 1, $randomstuff );
...

while ( $iterator < $max_iterations ){
    $sth->execute();
}
```

- Enqueue Hash Chain Latch
- Pressure on Test/Set Solaris
- Fujitsu FTFW
