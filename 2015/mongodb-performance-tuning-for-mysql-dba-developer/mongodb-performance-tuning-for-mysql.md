title: MongoDB Performance Tuning for MySQL DBA's and Developers
author:
  name: Kenny Gorman
  twitter: kennygorman
  url: http://kennygorman.com
output: mongodb-performance-mysql.html
controls: true
theme: kgorman/cleaver-light

--

# MongoDB Performance Tuning
## for MySQL DBA's and Developers
Kenny Gorman

Chief Technologist; Data @ Rackspace

Co-Founder @ ObjectRocket

--
### A small bit of my background

* Database Engineer, Developer, DBA, Architect, Founder



* Oracle, MySQL, PostgreSQL, MongoDB, Apache Spark



* Obsessed

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

* Let's talk datatypes

--

### Example of typical DB schema

```sql
SELECT *
FROM customers c, departments d, customer_departments cd
WHERE c.customer_id = cd.customer_id
AND d.department_id = cd.department_id
AND cd.name = 'Engineering'
```

- Yeah, Edgar would be stoked
- Holy shit this sux, no-one does this in the real world

```sql
SELECT *
FROM customers
WHERE department_name IN ('Engineering');
```

- Yeah, like that


--

### Now let's look at it in MongoDB

```json
{
    "customer_id": 100,
    "customer_name": "John",
    "department": "Engineering"
}
{
    "customer_id": 101,
    "customer_name": "Hilary",
    "departments": ["Marketing", "Engineering", "Sales"]
}
```

- Are you freaking out?

--

## Tables, Indexing, etc

- adding indexes background
- adding columns vs MongoDB
- phantom reads


--

## Statement Tuning

- Similar to MySQL, look for bad statements
- Enter profiler


--

## MongoDB profiler






--

## Query plans



--

## Instance Tuning


--

## Random

- If sharded, look for balancer jacking you
- TTL indexes


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
