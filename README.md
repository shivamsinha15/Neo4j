# Neo4j Quick Reference
* [Neo4j PDF Manual](http://neo4j.com/docs/pdf/neo4j-manual-2.1.6.pdf)
* [Neo4j HTML Manual](http://neo4j.com/docs/2.1.6/)

The following is a quick guide to Neo4j.
######Neo4j is a property graph database. It consist of:
* Nodes - Which can have a number of labels
* Relationship - can have one direction and type.

Both nodes and relationships can have a number of properties.

######Neo4j has two running modes:
* A standalone server via REST ( Obviously a lot safer and a crash would not affect the server)
* An embedded database in Java Application (use when application is a standalone and no requirement for a separate Database Server.

######POM Dependency and Repo:
```sh
<dependencies>
		<dependency>
			<groupId>org.neo4j</groupId>
			<artifactId>neo4j</artifactId>
			<version>2.0.0</version>
		</dependency>
	</dependencies>
	<repositories>
		<repository>
			<id>neo4j</id>
			<url>http://m2.neo4j.org/content/repositories/releases/</url>
			<releases>
				<enabled>true</enabled>
			</releases>
		</repository>
	</repositories>
```

###### Creating an embedded database:
```java
        GraphDatabaseFactory graphDbFactory = new GraphDatabaseFactory();
        GraphDatabaseService db = graphDbFactory.newEmbeddedDatabase(DB_PATH);
        return db;
```
* With the **GraphDatabaseService** you can fully interact with the database: Create Nodes, Relationships, set properties and indexes.


###### Configuration:
Neo4j allows you to pass a set of configuration options for **performance tuning** , **caching**,**logging**,**file system use** and other low-level behaviours.

* **GraphDatabaseSettings.java** contains a list of properties which can be customized

```java
private static final String DB_PATH = "database/hr";

//Create with more memory
GraphDatabaseService db = graphDbFactory
                .newEmbeddedDatabaseBuilder(DB_PATH)
                .setConfig(GraphDatabaseSettings.nodestore_mapped_memory_size, "20M")
                .newGraphDatabase();
                
//Creating using props file
GraphDatabaseService db = graphDbFactory
                    .newEmbeddedDatabaseBuilder(DB_PATH)
                    .loadPropertiesFromURL(propertyFileUrl)
                    .newGraphDatabase();
                    
//neo4j.properties
neostore.nodestore.db.mapped_memory=20M
```


#### Chapter 1 - Querying with Pattern Matching
Chapter One contains a HR Management Tool consisting of 
* Nodes: Employee, CostCenter
* Relationships: (1) Employees that belong to cost centers, (2) Employees that report to other employees and (3) Employees that can be managers of cost centers

###### Defining Nodes
```java
public enum HrLables implements Label {
    Employee,CostCenter
}
```

###### Defining Relationships

```java
public enum EmployeeRelationship implements RelationshipType {
    REPORTS_TO, BELONGS_TO, MANAGER_OF;
    // from specifies the dates of validity
    public static final String PROPERTY_FROM = "from";
}
```
###### Transactions
Neo4J is a fully ACID compliant database. Similar to JPA and with **FlushMode.COMMIT** , the entity is persisted when the entity transaction is committed, a graph record is commited once the graph transaction is closed.

```java  
org.neo4j.graphdb.Transaction:
public interface Transaction extends AutoCloseable
```

Since Java SE7:

>The new syntax allows you to **declare resources** that are part of the **try** block. What this means is that you define the resources ahead of time and the runtime automatically closes those resources (if they are not already closed) after the execution of the try block. **Note: This only occurs fore resources that extend AutoCloseable** as shown above.

```java
// Java SE7 and Greater
 try (Transaction tx = graphDb.beginTx()) {
 //perform some work with graph ...
 tx.success();
 }
```

```java
// Java SE6 
Transaction tx = graphDb.beginTx();
 try {
 //perform some work with graph ...
  tx.success();
 } finally {
  tx.close();
 }
```

**tx.sucess()** - marks the transaction as sucessfull. Every change will be committed once the transaction is closed. If an exception is thrown from inside the try statement, the transaction automatically ends with a roll back.

###### Creating Nodes and relationships using Java API

Note in **com.packtpub.learningcypher.chapter1.DatabaseSetup.setup(GraphDatabaseService graphDb)**

Demonstrates creating Nodes with Labels and properties. And how to create Relationships from Nodes to Other Nodes E.g.:

```java
 Node cc1 = graphDb.createNode(HrLabels.CostCenter);
            cc1.setProperty(CostCenter.CODE, "CC1");
            
Node davies = graphDb.createNode(HrLabels.Employee);
            davies.setProperty(Employee.NAME, "Nathan");
            davies.setProperty(Employee.SURNAME, "Davies");     
            
 // davies belongs to CC1
 davies.createRelationshipTo(cc1, EmployeeRelationship.BELONGS_TO)
 .setProperty(EmployeeRelationship.FROM, new GregorianCalendar(2011, 1, 10).getTimeInMillis());

// .. and reports to Taylor
davies.createRelationshipTo(taylor, EmployeeRelationship.REPORTS_TO);            
```

* **createNode**:  This creates a node and then returns it as a result. The node will be created with a long, unique ID. **Note: Unlike relational databases, node IDs in Neo4j are not guaranteed to remain fixed forever. Infact IDs are recomputed upon node deletion, so don't trust IDs**

*  **createRelationshipTo**: This creates a relationship between two nodes and returns that relationship in a relationship instance. This one too will have a lon. unique ID
*  **setProperty**: This sets the value of a property of a node or relationship.

We put time in milliseconds in the property because Neo4j supports only the following types/array of types: **boolean,byte,short,int,long,float,double,String**

To store more complex types for arrays, we can:
* Code them into primitive types as seen in the list above.
    * E.g. To store a property such as the entire address fo a person, we can convert the address in JSON and store it as a string. This JSON format is the common way of storing data in document oriented DBs (Mongo DB), but since Neo4J isnt a document database, it wont build indexes on the properties of documents. This approach should only be used for raw data that wont be filtered or processed with Cypher.
    * The best approach usually to **create nodes** for complex types. Since they can be filtered and indexed.
    
#### Cypher Basics

**Cypher** - is Neo4j's declarative query language 

**CREATE NODES:**
```cypher
CREATE (ee:Person { name: "Emil", from: "Sweden", klout: 99 })
CREATE (ss:Person { name: "Shivam", from: "Sydney", klout: 99 })
```
* **CREATE** clause to create data
* **()** parenthesis to indicate a node
* **ee:Person** a variable 'ee' and label 'Person' for the new node
* **{}** brackets to add properties to the node

**FIND NODES**
```cypher
MATCH (ee:Person) WHERE ee.name = "Emil" RETURN ee;
```
* **MATCH** clause to specify a pattern of nodes and relationships
* **(ee:Person)** a single node pattern with label 'Person' which will assign matches to the variable 'ee'
* **WHERE** clause to constrain the results
* **ee.name = "Emil"** compares name property to the value "Emil"
* **RETURN** clause used to request particular results


**CREATE RELATIONSHIPS**
```cypher
Match (ee:Person {name:"Emil"}) Match (ss:User {name:"Shivam"})
CREATE (ee)-[:KNOWS {since: 2015}]->(ss)
```

**DELETE SINGLE NODE**
```cypher
MATCH (n:Useless) DELETE n
```

**Delete a node and connected relationships**
```cypher
MATCH (n { name: 'Andres' })-[r]-() DELETE n, r
```

**DELETE ALL NODES and Relationship**
```cypher
MATCH (n)
OPTIONAL MATCH (n)-[r]-()
DELETE n,r
```
###### ADDING NEW PROPERTIES TO EXISTING NODES/RELATIONSHIPS:

The following statement adds the property "RECOMMEND" to the WATCHED Relationship
```
MATCH (u1:User{name:"Shiv"})-[w:WATCHED]->(v1) SET w += {RECOMMEND:'yes'};
//The property can then be used on a WHERE CLAUSE
MATCH (u1:User{name:"Bijal"})-[:FRIENDS]-(f1)-[w:WATCHED]-(v1) WHERE w.RECOMMEND='yes' return v1;
```


###### MERGE
1. It ensures that a **pattern exists** if it does not exist then it creates it.
    *  ```REF#M1: MERGE (u1:User {name: "u1"})-[:FRIEND]-(u2:User {name:"u2"})```- In this case 
    since the pattern doesnt exist it will create one.

2. It **will NOT use partially existing (unbound) patterns**- it will **attempt to match the entire pattern** and **create** the entire **pattern if missing**
    * ```MERGE (u1:User {name:"u1"})-[:FRIEND]-(u2:User { name:"u2" })-[:LIVES_IN]->(c:Country { name:"Australia" }) ``` - Here MERGE does not consider the partial pattern from above and hence will create a new one as per below:
    ![](http://i.imgur.com/V9XUawh.png)
    
3. When unique constraints are defined, MERGE expects to find at most one node that matches the pattern
4. It also allows you to [define what should happen](http://neo4j.com/docs/stable/query-merge.html#_use_on_create_and_on_match) based on whether data was created or matched

**The key to understanding what part of the pattern is created if not matched is the concept of bound elements. So what is a bound element?**

* An element is bound if the identifier was used in an earlier clause of the cypher statement

In the above diagram (2) The entrie patterns has **no bound nodes**. Since merge won’t consider a partial pattern, it attempted to match the entire unbound pattern which does not exist, and created it.

To avoid recreating nodes which already exist, bind the parts of the pattern you dont want to recreated

```cypher
MERGE (u1:User {name:"u1"}) // <- This binds User "u1" to variable u1
MERGE (u2:User { name:"u2" }) // <- This binds User "u2" to variable u2

//Since u1 and u2 are bound variables in the statement below, the two user nodes are not recreated. Note if User "u1" and "u2" did not exist then MERGE would create them
MERGE (u1)-[:FRIEND]-(u2)-[:LIVES_IN]->(c:Country { name:"Australia" })
```

**Properties and nodes matching**

```MERGE (u1:User {name: "u1",age:20})-[:FRIEND]-(u2:User {name:"u2"})```

Assuming **REF#M1** has already executed and then the above statement is executed.
This will create two more nodes and a FRIEND relationship between them.

This is because MERGE could not find any User node that has both a name and age property that match and so it went ahead and created the entire pattern.

Every property does not need to be specified to find a matching node- a subset will do. If we had a User node with properties name=”u1” and age=20 and then attempted to MERGE (u:User {name:”u1”}) it would not create a new node because it could match a User node with name=”u1”.

Note that in case of **unique constraints** defined on User such as:

```CREATE CONSTRAINT ON (u:User) ASSERT u.name IS UNIQUE;```

In the case where a User node exists with name=”u1” (but no age)

```MERGE (u1:User {name:”u1”,age:20})```

will produce an exception:

```Node already exists with label User and property "name"=[u1]```

**This is expected because MERGE attempted to create a new node but the unique constraint was violated as a result of it.**





