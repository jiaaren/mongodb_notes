## Reference
1. [Mongodb](https://www.mongodb.com/developer/article/mongodb-schema-design-best-practices/)
2. 6 Rules of thumb for mongodb schema design
	- [Part 1](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-1)
	- [Part 2](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-2)
	- [Part 3](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-3)

## Schema design
### Overview
- It depends
- Proper MongoDB schema design is the most critical part of deploying a scalable, fast, and affordable database
- Schema design most overlooked in MongoDB administration

### Migration from Relational/SQL databases
1. Most people using MongoDB come from traditional relational databases
	- which may not take advantage of all that MongoDB has to offer
2. When designing relational/sql
	- Split data into tables, i.e. normalizing (usually in [3rd normal form](https://en.wikipedia.org/wiki/Database_normalization#Satisfying_3NF))
	- Normalizing prevents duplication of data

<figure>
	<img src='relational-user-model.png'>
	<figcaption>User data is split into separate tables and it can be JOINED together using foreign keys in the user_id column of the Professions and Cars table.</figcaption>
</figure>

### Considerations
1. App **read** or **write** heavy
2. What data is **frequently accessed** together
3. What are the **query performance** considerations
4. How will data **grow and scale**?

In general, how data is **used** will impact schema design, two applications with same data but different uses may result in different schema designs.

### Designing MongoDB Schema
#### Vs relational
- No formal process
- No algorithms
- No rules

#### Embedding vs Referencing
1. Usually comes down to 2 choices for every piece of data, whether to **embed data directly** or **reference** another piece of data using the `$lookup` operator (which is similar to `JOIN` in a relational database).
2. Translating relational tables above, and embedding `Professions` and `Cars` within `Users`
	- instead of splitting data, we embed data in forms of `arrays` and `objects` within User object
	- Now only need to make a simple query to pull data for our application
```{attr.source='.numberLines'}
{
    "first_name": "Paul",
    "surname": "Miller",
    "cell": "447557505611",
    "city": "London",
    "location": [45.123, 47.232],
    "profession": ["banking", "finance", "trader"],
    "cars": [
        {
            "model": "Bentley",
            "year": 1973
        },
        {
            "model": "Rolls Royce",
            "year": 1965
        }
    ]
}
```
#### Embedding
<ins>Advantages</ins>
- You can **retrieve** all relevant information in a **single query**.
- **Avoid implementing joins** in application code or using $lookup.
- Update related information as a single atomic operation. By default, all CRUD operations on a single document are **ACID compliant**.
  - [ACID compliance](https://mariadb.com/resources/blog/acid-compliance-what-it-means-and-why-you-should-care/)
  - [ACID for MongoDB](https://www.mongodb.com/basics/acid-transactions)
- However, if you need a **transaction** across multiple operations, you can use the transaction operator. 
  - Though transactions are available starting 4.0, however, I should add that it's an anti-pattern to be overly reliant on using them in your application.

<ins>Limitations</ins>
- Large documents mean more overhead if most fields are not relevant. You can increase query performance by limiting the size of the documents that you are sending over the wire for each query.
- There is a 16-MB document size limit in MongoDB. If you are embedding too much data inside a single document, you could potentially hit this limit.
- Makes certain queries much more troublesome, e.g. need to access all `Arrays/Objects` of all `documents`

#### Referencing
- referencing another document using `unique Object ID`, and connecting using `$lookup` operator
- generally results in more efficient and scalable queries > after splitting up data

<ins>Advantages</ins>
- By splitting up data, you will have smaller documents.
- Less likely to reach 16-MB-per-document limit.
- Infrequently accessed information not needed on every query.
- Reduce the amount of duplication of data. 
  - However, it's important to note that **<ins>data duplication should not be avoided if it results in a better schema</ins>**.

<ins>Limitations</ins>
- In order to retrieve all the data in the referenced documents, a **minimum of two queries or $lookup required** to retrieve all the information.


### Types of relationships and example implementations
#### Considerations 
1. Cardinality of relationship
- Not just merely One to N, may be more specific, i.e.
  - One to Few
  - One to Many
  - One to Squillions
  
2. Standalone
- Will entities in 'N' side of One to N ever need to be standalone?
- Do you need to access the object on the “N” side separately, or only in the context of the parent object?

3. Ratio of read and writes

#### One to one
- E.g. 
  - a User table, **one user only has one name**
  - an employee can work in one and only one department.
- model one-to-one data as **key-value pairs** in database
```
{
    "_id": "ObjectId('AAA')",
    "name": "Joe Karlsson",
    "company": "MongoDB",
    "twitter": "@JoeKarlsson1",
    "twitch": "joe_karlsson",
    "tiktok": "joekarlsson",
    "website": "joekarlsson.com"
}
```

#### One to Few
- **Not to be confused with One to Many**
- Sequence of data associated to a document
  - e.g. store several addresses associated to user
- Usually **embed** using `Arrays` or `Objects/Dicts`
```
{
    "_id": "ObjectId('AAA')",
    "name": "Joe Karlsson",
    "company": "MongoDB",
    "twitter": "@JoeKarlsson1",
    "twitch": "joe_karlsson",
    "tiktok": "joekarlsson",
    "website": "joekarlsson.com",
    "addresses": [
        { "street": "123 Sesame St", "city": "Anytown", "cc": "USA" },
        { "street": "123 Avenue Q",  "city": "New York", "cc": "USA" }
    ]
}
```

#### One to Many
- E.g. product page for e-commerce website
  - In website, save information about all the many parts that make up each product for <ins>repair services</ins>.
  - With a schema that could potentially be saving thousands of sub parts, we probably **do not need to have all of the data for the parts on every single request**.
  - But still **important that relationship is maintained**
- Keep `Array` of `Object IDs` of parts linked to the Product
- Can be saved in same or separate collection
- Below: Parts ObjectIDs are in Products as an array

##### **Products**
```
{
    "name": "left-handed smoke shifter",
    "manufacturer": "Acme Corp",
    "catalog_number": "1234",
    "parts": ["ObjectID('AAAA')", "ObjectID('BBBB')", "ObjectID('CCCC')"]
}
```

##### **Parts**
```
{
    "_id" : "ObjectID('AAAA')",
    "partno" : "123-aff-456",
    "name" : "#4 grommet",
    "qty": "94",
    "cost": "0.94",
    "price":" 3.99"
}
```

#### One to [Squillions](https://www.dictionary.com/browse/squillion)
- Potentially millions of subdocuments, or more?
- E.g. server logging application > each server could potentially save massive amounts of data
  - with MongoDB, tracking data within an unbounded array is **dangerous**
  - could potentially hit that **16-MB-per-document limit**.
    - even if only ObjectIDs are stored in an Array
- Tracking using `Arrays` or `Objects` is a **no go** 
- Instead of **tracking relationship** in the host document (parent referencing), track in the log message document. Then, we don't need to worry about unbounded arrays hitting the 16MB limit.


##### **Hosts**
```
{
    "_id": ObjectID("AAAB"),
    "name": "goofy.example.com",
    "ipaddr": "127.66.66.66"
}
```

##### **Log Message**
```
{
    "time": ISODate("2014-03-28T09:42:41.382Z"),
    "message": "cpu is on fire!",
    "host": ObjectID("AAAB")
}
```

### Many to Many
- E.g. To-do application
  - user may have many tasks
  - tasks may have many users assigned to it
- To **preserve relationship**
  - need references from one user to many tasks
  - need references from one task to many users
- Below: each user has a sub-array of linked tasks, and each task has a sub-array of owners for each item in our to-do app.

##### **Users**
```
{
    "_id": ObjectID("AAF1"),
    "name": "Kate Monster",
    "tasks": [ObjectID("ADF9"), ObjectID("AE02"), ObjectID("AE73")]
}
```

##### **Tasks**
```
{
    "_id": ObjectID("ADF9"),
    "description": "Write blog post about MongoDB schema design",
    "due_date": ISODate("2014-04-01"),
    "owners": [ObjectID("AAF1"), ObjectID("BB3G")]
}
```

### Vocabulary

### Important
- using transactions for multiple operations/documents, if editing multiple

#### Rules
- Rule 1: Favor embedding unless there is a compelling reason not to. (One to Few)
- Rule 2: Needing to **access an object on its own** is a compelling reason not to embed it. (One to Many)
- Rule 3: Avoid joins/lookups if possible, but don't be afraid if they can provide a better schema design. (One to Many)
  - if you index correctly and use the projection specifier then application-level joins are barely more expensive than server-side joins in a relational database.
- Rule 4: Arrays should not grow without bound. If there are more than a couple of hundred documents on the "many" side, don't embed them; if there are more than a few thousand documents on the "many" side, don't use an array of ObjectID references. High-cardinality arrays are a compelling reason not to embed. (One to Squillion)
- Rule 5: As always, with MongoDB, how you model your data depends – entirely – on your particular application's data access patterns. You want to structure your data to match the ways that your application queries and updates it.



### Denormalising
Consider the write/read ratio when denormalizing. A field that will mostly be read and only seldom updated is a good candidate for denormalization: if you denormalize a field that is updated frequently then the extra work of finding and updating all the instances is likely to overwhelm the savings that you get from denormalizing.

### Steps to decide on database design
1. Main criterias
	- Cardinality of relationship
	- Need to access 'N' side separately, or only in context of parent object
	- Ratio of read and write (may not be as important first time round)

2. Choices for structuring data
   - One-to-One - Prefer **key value pairs** within the document
   - One-to-Few - Prefer **embedding**
   - One-to-Many - Prefer **embedding** and **referencing** (child referencing) or two-way referencing (depending on how we want data to be shown)
   - One-to-Squillions - Prefer **referencing** (parent referencing)
   - Many-to-Many - Prefer **referencing** (both child and parent referencing)

3. Consider denormalizing data
   1. removing data from One side to N side or N side to One side
		- impacted by frequency of read and write
	2. Updating denormalized data is slower, more expensive and not atomic