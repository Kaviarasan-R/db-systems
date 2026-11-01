# ACID

## Principles

### Atomicity
* It guarantees that a transaction—which might contain multiple SQL steps—is treated as a single, indivisible unit of work. Either every single step succeeds and is saved (Committed), or if even one-step fails, the entire transaction is canceled and undone (Rolled Back).

### Isolation
* A **dirty read** is a database anomaly that occurs when Transaction A reads data that has been modified by Transaction B, but Transaction B has not yet committed the change 
* A **non-repeatable read** is a database concurrency anomaly that occurs when a transaction reads the same row of data twice but gets different values each time because another concurrent transaction modified and committed that row in the interim 
* A **phantom read** occurs when a transaction inserts a query returning a range of rows, but when it re-runs that exact same query later, it finds newly created "phantom" rows inserted by another transaction that committed in the meantime. 
* A non-repeatable read affects existing rows, while a Phantom read introduces entirely new rows. 
* PostgreSQL uses a versioning system called MVCC (Multi-Version Concurrency Control). The Repeatable Read isolation level in Postgres, both non-repeatable reads and phantom reads are completely blocked, meaning your queries will return the exact same rows and values even if another transaction has committed changes in the meantime. 
* A **lost update** is when two transactions read the same data at the same time, modify it, and then write it back, causing one transaction to overwrite and completely erase the other's changes.

#### Isolation Levels:
* Each DBMS implements Isolation level differently.
* **Read uncommitted:** No Isolation, any change from the outside is visible to the transaction, committed or not. Example: Dirty Reads, but faster.
* **Read committed:** Each query in a transaction only sees committed changes by other transactions, it is default isolation level by dbs.
* **Repeatable Read:** The transaction will make sure that when a query reads a row, that row will remain unchanged while its running. Example: Never returns a different value unlike non-repeatable read but not phantom read.
* **Snapshot:** Each query in a transaction only sees changes that have been committed up to the start of the transaction. It's like a snapshot version of the database at that moment. It is very expensive to use. It completely removes phantom read completely read phenomenas and by default postgres uses this.
* **Serializable:** No concurrency transactions, only 1 transaction run at a time.

![Isolation Levels](../assets/isolation_levels.png)

#### Implementation of Isolation:
* **Row level locks:** When Transaction A accesses a specific row with intent to update, it places an exclusive lock. Transaction B attempting to modify or read-for-update that same row is forced to wait until Transaction A commits or rolls back.
* **Page level locks:** A Table is a Logical Concept; It represents your schema, columns, and relations (e.g., customers). A Page is a Physical Container; It is a fixed-size chunk of disk/memory (e.g., 8 KB or 16 KB). Rows are Stored Inside Pages; Rows are placed side-by-side inside pages until the page runs out of space. A single table usually spans hundreds, thousands, or millions of pages. In modern relational engines, a data page belongs to only one table. If Transaction A updates Row 1 and Transaction B wants to update Row 50, but both happen to reside on the same physical page, Transaction B must wait even though the two transactions operate on completely unrelated business entities.
* **Table level locks:** Transaction A acquires an exclusive table-level lock. No other transaction can read or write to any record in that table until Transaction A finishes.

### Consistency
