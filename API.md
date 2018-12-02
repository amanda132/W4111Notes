# 1. Introduction

## Lecture goals

- You can't actually create an application with SQL
- How do you actually access a database without SQL injections?
- How does the application talk to the database and do things?

## SQL != programming language!
**Not a general purpose programming language**
- Tailored for data access/manipulation
- Easy to optimize and parallelize 
- Can’t perform “business logic”

**Options**
- Extend SQL, make it Turing Complete
- Extend existing languages to understand SQL natively
- Provide an API between programming languages and DBMSes.

# 2. API
## 2.1 Many Database API Options
1. _Embedded SQL_
    - Design a programming language that understands SQL natively
    - **Embedded**: just as you can write a `x = y` statement in an application language, you can write `x = [a SQL query statement]` 

2. _Low-level library with core database calls (DBAPI)_
    - Use a library that's specific to your desired programming language to access the database from your application.

3. _Object-relational mapping (ORM)_
    - Each entity relates to an object via an automated, natural mapping. 
    - Don't have to interface directly with the database at all
    - Objects can save themselves to the database

## 2.2 Embedded SQL
Embedded SQL is a method of combining the computing power of a programming language and the database manipulation capabilities of SQL. Embedded SQL statements are SQL statements written inline with the program source code , of the host language. The embedded SQL statements are parsed by an embedded SQL preprocessor and replaced by host-language calls to a code library. The output from the preprocessor is then compiled by the host compiler. This way, SQL is compiled into program that interacts with DBMS directly.
- Was previously a really popular approach; not any more.

### Example 2.1 ###
Take the C (or C++, Fortran, Pascal, etc.) program that contains SQL, then compile it into parts that are just C code and parts that know how to talk with the DB.
See below Oracle Pro*C code:
```cpp
    int a;
    /* ... */
    EXEC SQL SELECT salary INTO :a
    FROM Employee
    WHERE SSN=876543210; 
    /* ... */
    printf("The salary is %d\n", a);
    /* ... */
```
- The `EXEC SQL` statement sets the salary into a variable `a`.
- **_BUT_**, what if the `SELECT` query returns multiple rows for `salary` which we declared as an integer? 
    - `a` would have to be a list or a set. And anything that references `a` would instead have to loop through `a`.
    - Queries that return a single row of data are handled with a singleton SELECT statement; this statement specifies both the query and the host variables in which to return data. 
    - Queries that return multiple rows of data are handled with cursors. A cursor keeps track of the current row within a result set. The DECLARE CURSOR statement defines the query, the OPEN statement begins the query processing, the FETCH statement retrieves successive rows of data, and the CLOSE statement ends query processing.

### Example 2.2 Embedded SQL Flow ###
<img src="https://github.com/Wangler/scribenotes/blob/master/embeddedsql.png" width="460">

1. **Preprocessor**: Run the code (with embedded SQL) through the preprocessor which identifies all the `EXEC SQL` statements and translates them into DBMS library calls. 
2. **Java Compiler**: Put the Java code into an executable that links to a DBMS library, to communicate with the database. 


### Aside: Embedded SQL History
Embedded SQL hasn't taken off in popularity. Some of the issues it has faced are:
- There could be runtime problems e.g. invalid SQL query
- SQL standard has changed over time. There are dependencies between SQL and other programming languages. e.g. If there's a fancy new SQL feature, the compilers of all these other languages need to be augmented to support this new feature
- SQL with enough extensions becomes Turing complete
- **_Making a library alone just sounds good enough_**


## 2.3 Libraries
### What does a library need to do?
- **_At minimum_**: write a function that takes in a query string and execute it
- It needs to take care of the engine, connection, and query execution.
- It still requires making a connection to the DBMS
- The library serves as an intermediary between your program and all the databases you want to talk to
- Given that there are so many different DBMS engines (i.e. Postgres, MySQL, Microsoft), ideally you'd be able to connect to them all with the same library
- e.g.
<img src="https://github.com/Wangler/scribenotes/blob/master/libraryflow.png" width="460">
- In a real life company application, you need to connect to multiple different databases (e.g. Ads, Users, Billing).
- Using a library allows for a common interface to all databases: 

### Engines
- Ideally you want a common interface to all database engines that hides all the syntactical differences between them
- e.g. `SQLAlchemy` is a popular library. See below example for establishing a database connection
    - You insert the path string to the db into the `create_engine()` function. 
    - The path looks like: `protocol_name` `://` `location` `:` `port` `/` `database_name`
        - **protocol_name**: the type of db, e.g. postgresql, mysql, sqlite
        - **host**: the location/ip address of the db
        - **port**: must specify the port that the db is on, as it can exist on many

``` python
    from sqlalchemy import create_engine 
    db1 = create_engine(“postgresql://localhost:5432/testdb” )
    db2 = create_engine(“sqlite:///testdb.db”)
    // note: sqllite has no host name (sqlite:///)
```

### Connections
- You establish an HTTP or TCP connection to a particular machine at that particular host and port. The engine then allocates the resources in the db to start running queries.
- **Issue**: can be expensive to set up
``` python
    conn1 = db1.connect()
    conn2 = db2.connect()
```
- You can open multiple connections to the same db for running queries in parallel.
- Should run `.close()` on your connections. otherwise resources leak.


### Query Execution
``` python
    conn1.execute(“update table test set a = 1”) 
    conn1.execute(“update table test set s = ‘wu’”)

    foo = conn1.execute(“select * from big_table”)
```
- Run `.execute()` with a string which is blindly sent to the database.
- Some issues:
    - Needs to do error handling
    - Want to be able to assign the result to a program variable.
- What should be the object type of what's returned in `.execute()` (`foo` in this e.g.)? 
    - You don't want `foo` to consist of 5TB of data in memory.
    - Lists? (not immutable)
    - Dictionaries? (take a lot of space with schema information)
    - Sets? (expensive to represent because of duplicate removal)
    - Answer is `cursor` (more on that later)

## Detour: SQL INJECTIONS
- **SQL Injection**: "an injection attack wherein an attacker can execute malicious SQL statements (also commonly referred to as a malicious payload) that control a web application’s database server" ([source](http://www.acunetix.com/websitesecurity/sql-injection/))
- A SQL injection can arise from inserting a string into your db that contains multiple sql statements separated by `;`

**If we insert the variable `name` defined as below:**
``` python
    name = " '); DELETE FROM bad_table; -- " 

    conn1.execute (“““
    INSERT INTO bad_table(name) 
    VALUES('{name}')”””.format(name=name))
```

**Query is:**
``` sql
    INSERT INTO bad_table(name) VALUES(''); DELETE FROM bad_table; -- ');
```

- The bad string adds a semicolon which finishes the insert statement without inserting anything. It then adds `--` which comments out the SQL code that comes after it.
- Must have some function which _sanitizes_ the input string to prevent certain types of sql text from existing
- Most libraries have a parameter in its `execute()` function that allows you to pass in a tuple of query arguments so that it can properly escape input values, thereby sanitizing everything **before** it hits the db.
``` python
    args = ('Dr Seuss', '40') 
    conn1.execute(
        “INSERT INTO users(name, age) VALUES(%s, %s)”, args)
``` 
- The above uses placeholders, into which variables are passed in
- **NEVER CONSTRUCT RAW SQL STRINGS**. You must always pass variables into template instead.

### Placeholders
Use for these query templates. Not standardized. Vary between languages and databases.
- Postgres in Python: Use `%s`
- Postgres in Go: Use `?`
- SQLite in Python: Use `?`

## Relational Impedance Mismatches
The way the db thinks about the world is not the same as how your programming language expresses and describes it
e.g. object(programming) != Row (dbms)

### By Type
Default Mappings: most programming languages have same set of fundamental data types
e.g. sql:
<img src=https://github.com/Wangler/scribenotes/blob/master/objecttypes.png width=300>

- This mapping is straightforward because types are simple
- What about complex objects? e.g. python dictionaries


### By Object
- Issue: in your programming language you want to be able to change an object's attribute, and have that update be reflected in the database
e.g.

<img src = https://github.com/Wangler/scribenotes/blob/master/objects.png width = 500>

- In the 80s there was a push for object-oriented-database system: it would use C executables which know how to change the DB automatically when you change them. (This didn't work for the same reason embedded sql didn't work. Real companies don't use a single programming language for all of their applications).
- ORMs are designed to solve this (!) by turning the object <-> db translation into a library ([see next section on ORM](https://github.com/w4111/scribenotes/wiki/apis#object-relational-mappers-orms))


### By Results
- Results can create a memory problem for your program depending on how they're stored
- An impedance mismatch occurs because SQL operates on _set_ of records, whereas host languages do not cleanly support a set-of-records abstraction. 
- We must provide a mechanism that allows us to retrieve rows one at a time from a relation.
- Solution: **cursors**! A cursor is like an iterator which points to the results in the db. It does not store all of the results at once.
- A cursor is **not** a list: there is no inherent order (_unless_ you use an `ORDER BY`)
```python
    cursor = execute(“SELECT * FROM bigtable”)
    cursor.next()
```
- Run `cursor.next()` to get next record
e.g. **Calling cursor.next()**

<img src = https://github.com/Wangler/scribenotes/blob/master/cursor1.png width = 300>

- **Calling cursor.next() again**

<img src = https://github.com/Wangler/scribenotes/blob/master/cursor2.png width = 300>

- **Benefits to using Cursors**
    - On program side: no need to hold onto all of the data in memory
    - On db side: only needs to process the records that you're explicitly requesting, otherwise not doing anything in the background
- Other cursor attributes/methods:
    - `.rowcount`: gives you number of records to be returned in query
    - `.keys()`: gives you the number of records to be returned in query
    - `.previous()`: works just like `.next()`
    - `.get(idx)`: call `.next()` enough times until you get given result specified by that index
- Cursors can be set to be read-only
- We usually need to open a cursor if the embedded statement is a SELECT query.

### By Functions
- How do you insert a function defined in python into the SQL string for execution?
- Solution: embed a language runtime into dbms. Convert it into a UDF.


### By Constraints
- Issue: Constraints are naturally expressed in dbms since there's only place where data is stored and represented. But in the program, it's scattered everywhere
- Issue: Constraints are duplicated between the programming language and the dbms

e.g. **Applying a Check Constraint in JS vs DBMS**

**JS**:

``` js
    age = get_age_input();
    if(age>100 or age<18)
        show_error(“age should be 18 – 100”);
```
**DBMS**:

``` sql
    CREATE TABLE Users (
    ...
    age int CHECK(age >= 18 and age <= 100)
    ... 
    )
```

- This JS error is purely for notifying - not for validating. The purpose is not to correct the data. 
- It won't guarantee that the constraint is upheld -- only doing it via DBMS will ensure that.
- ORM will have one place to define constraints ([see next section on ORM](https://github.com/w4111/scribenotes/wiki/apis#object-relational-mappers-orms)) by translating the code into a check constraint in DBMS.

#### Heavyweights in terms of libraries: 
- ODBC: Standard for Microsoft
- JDBC: Standard for Java. 2 different library implementations: java and javax
- The semantics and functions exposed are different

## Object-Relational Mappers (ORMS)
- Using an object directly in your program which represents the DBMS itself. Can change object directly while the object itself fully captures the structure of the DBMS object.
- Uses its own type of query syntax in the application's programming language
- A solution to object and constraint related relational impedance mismatches

### Solving Object-based Impedance Mismatch
e.g. **Defining Object in Python ORM vs SQL**
- Here we create a Base database object called User
- In the Python ORM, you are defining the DBMS schema and the object attributes at same time.

**In python**
``` python
    class User(Base): __tablename__ = 'users'
        id = Column(Integer, primary_key=True) 
        name = Column(String)
        fullname = Column(String)
        password = Column(String)
);
```
**In SQL**
``` sql
    CREATE TABLE users(
        id INT PRIMARY KEY, 
        name TEXT,
        fullname TEXT, 
        password TEXT
```

- Below is how you instantiate the object and assign it to the DBMS via `.add(object)`.

``` python
    >>> ed_user = User(
             name='ed', fullname='Ed Jones',
             password='edspassword ')
    >>> ed_user.name
    'ed'
    >>> ed_user.password
    'edspassword'
    >>> session.add(ed_user)
```
- **The database has not changed at this point**. The assignment has only been made in the program itself.
- Running `.save()` sends the update back into the dbms.
- The return value of an ORM query is a list of objects.

### Solving Constraint-based Impedance Mismatch
ORMs try to have one place to define constraints

**Python**:
```python
    class Person(models.Model):
        first_name = models.CharField(max_length = 30) 
        last_name = models.CharField (max_length = 30, null = True)
```

**SQL Equivalent**
``` sql
    CREATE TABLE myapp_person (
        "id" serial NOT NULL PRIMARY KEY, 
        "first_name" varchar(30) NOT NULL, 
        "last_name" varchar(30)
    );
```


### ORM Queries vs SQL Queries
e.g. **Query in ORM Python vs SQL**

**In ORM Python**
``` python
    session.query(User).filter(User.name.in_( ['Edwardo', 'fakeuser']).all()
```

**In SQL**
``` sql
    SELECT * FROM users
    WHERE name IN ('Edwardo', 'fakeuser')
```

### Issues with ORM:
- Inefficient queries
- Tricky when you want to change your application
- Queries are harder to write since you can't use raw SQL. Translating SQL -> ORM language is hard.
- If you have to create sql strings dynamically in your program, your program will have to deal with a lot of string manipulation
- **BUT** using ORM language instead of raw SQL protects you from SQL injections: you're not executing any SQL strings directly from your program.


## ORM Relationship Challenges (skip in lecture 15)

## Modern Database APIs
- Modern languages try to blur the lines between dbms query code and application code e.g. Linq, Scalding, SparkSQL
- e.g. Using Spark engine, written in Scala
``` scala
    val lines = spark.textFile(“logfile.log”)
    val errors = lines.filter(_ startswith “Error”) 
    val msgs = errors.map(_.split(“\t”)(2))

    msgs.filter(_ contains “foo”).count()
```

- Spark reads .log files in which each line is a single record in the form of a very large string
- `lines` is now a pointer to table with 1 attribute: which is the entire string. 
the app code, so all these objects know that theyre relations. so you can run `.filter` and just pass in a function, which it knows as a SQL query
- This is known as a distributed db system
    - You have a relation and can apply relational algebra
    - It has a built-in engine to execute the relational algebra statement.
    - All within the program, which has implemente a db engine, you can construct a SQL statement and execute it.


## References ##
[1]. https://en.wikipedia.org/wiki/Embedded_SQL

[2]. https://docs.microsoft.com/en-us/sql/odbc/reference/embedded-sql?view=sql-server-2017
