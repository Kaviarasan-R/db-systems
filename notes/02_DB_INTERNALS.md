# DB INTERNALS

## Storage Concepts

### Table & Row ID

* Row ID is internal & system maintained.
* In certain databases like MySQL it is same as primary key `emp_id` but other databases like Postgres have a system maintained column (not visible logically) row_id (tuple_id) for querying purpose. 

<img alt="table row id" src="../assets/table_row_id.png" height="412" width="768"/>

### Page

* The table rows are stored & read in logical pages.
* The database doesn't read a single row, it reads a page or more in a single IO and we get a lot of rows in that IO.
* Each page has a size of (e.g. 8KB in Postgres & 16KB in MySQL)
* Assume each page holds 3 rows in example, with 1001 rows we will have ~333 pages.
* A single page cannot have different table rows (e.g. postgreSQL)

<img alt="page" src="../assets/page.png" height="768" width="400"/>

### IO

* IO operation (input/output) is a read request to a disk. A common misconception, it is not a sql query.
* We try to minimize this as much as possible.
* An IO can fetch 1 page or more depending on the disk partitions and other factors.
* An IO cannot read a single row, its a page with many rows in them.
* We want to minimize the number of IOs as they are expensive.
* Some IOs in operating systems goes to the operating system cache and not disk. (e.g. postgres heavily rely on os cache)

### Heap

* The Heap is data structure where the table is stored with all its pages one after another.
* This is where the actual data is stored including everything.
* A single heap can have multiple pages.
* Traversing the heap is expensive as we need to read so may data to find what we want.
* That is why we need indexes that help tell us exactly what part of the heap we need to read. 
* What page(s) of the heap we need to pull.
* Each row stores as tuple in heap pages especially in postgres.

### Index

* An index is another different data structure that has “pointers”.
* It has part of the data and used to quickly search for something.
* We can index (quick reference) on one column or more.
* Index pages do not live inside heap pages. They are stored in entirely separate files on disk, each formatted as 8 KB blocks specifically designed for that access method (e.g., B-Tree internal/leaf layout vs. Heap tuple layout).
* So, basically 2 or more separate IO's performed. 1 I/O to read the B-Tree root page. 1 I/O to jump to the heap page and fetch the actual tuple.
* Index tells us EXACTLY which page to fetch in the heap instead of taking the hit to scan every page in the heap.
* NOTE: The index is also stored as pages and cost IO to pull the entries of the index.
* The smaller the index, the more it can fit in memory the faster the search. So, avoid indexing every column in table.
* Popular data structure for index is b-trees.

![Index & Heap Pages](../assets/index_and_heap_pages.png)

The WHERE clause is still effective for filtering rows out of our result set, but it cannot skip reading pages without an index. PostgreSQL must perform a sequential scan across every heap page from disk to inspect each tuple against the condition.

![Scan pages](../assets/scan_pages.png)

If index is used, then we can skip reading pages sequential and search exactly the pages in heap with help of indexing using heap pointers. 

![Search using index 1](../assets/index_search_1.png)
![Search using index 2](../assets/index_search_2.png)

### Clustered Index vs. Heap-Organized Tables

* **Clustered Index (The Telephone Directory):** The book itself is physically ordered by the index key (e.g., alphabetically by name). Finding the entry means we are already holding the full row data on that exact page no extra lookup required. Because physical disk blocks can only be ordered one way, a table can only have **one** clustered index.
* **Heap + Secondary Index (The Textbook with an Index at the Back):** The rows are written in whatever order they were created (unordered heap). An index at the back simply maps a search key to a specific page number (`ctid` / physical pointer), requiring an extra hop to fetch the full row.
* A table's Primary Key is typically the clustered index by default in systems that support it (e.g., MySQL InnoDB, MS SQL Server).
* In non-clustered index, all user-defined indexes (including Primary Key indexes) are **secondary indexes**.
* Sometimes the heap table can be organized around a single index. This is called a clustered index or an Index Organized Table.
* Clustered Index = There is NO Heap pages. The B-Tree is the table. There is no separate "heap file" where rows live. (e.g. By default MySQL InnoDB uses clustered index, meaning only 1 type of page is available that is index pages & no heap pages)
* Non-Clustered (Secondary) Index = Index + Separate Heap. The table data lives in a completely separate, unordered collection of pages called the Heap. (e.g. Postgres, we have 2 different types of pages)

## Row vs Column Based Databases

### Row Oriented Databases
* Tables are stored as rows in disk.
* A single page/block will hold one or more rows.
* A single page/block IO can fetch multiple rows with all their columns.
* **Assuming No Indexes Used:** More IOs are required to find a particular row in a table scan because it starts from first block, but once we find the row then get all columns for that row as free of cost.
* Vertical partitioning, splits a table by columns into separate smaller tables. (e.g. Users table with 20 columns, split it into Users_Core [ID, Name, Email] and Users_Profile [ID, Bio, Profile_Picture_Blob]). Row-oriented databases (OLTP) like PostgreSQL or MySQL to isolate heavy or rarely accessed columns.
* Row oriented databases concept called OLTP (Online Transaction Processing)

![row db 1](../assets/row-db-1.png)

### Column Oriented Databases
* Tables are stored as columns first in disk.
* Working with multiple columns require more IO's.
* Column oriented databases concept called OLAP (Online Analytical Processing) (e.g. snowflake, hadoop big data)
* A single table column can have multiple blocks due to max block size. Every page/block has metadata where it stores the other parts of blocks that is split for same column.
* Each value in every block have mentioned with row_id "xxx:001, yyy:002". Once we find the row_id, then using that row_id can get the exact value from other block.

![1](../assets/col-db-1.png)

* It is very inefficient when it queries for multi-columns.

![2](../assets/col-db-2.png)

* It is very efficient with aggregation queries.

![3](../assets/col-db-3.png)

At last, When in need of every column frequently then row oriented is good, if sometimes only few columns are in need then column oriented is good.

## Primary vs Secondary Keys

### Primary Key 
* Clustered Index = Sorted Table: The database physically stores table rows sorted by the Primary Key inside a B-Tree (there is no separate heap). MySQL (InnoDB) is a prime example. PostgreSQL, however, separates storage into a Heap + B-Tree, meaning it has no true clustered index—as soon as we insert or update rows, Postgres places them wherever free space exists in the heap, breaking any physical order. By only taking this into argument, InnoDB wins. But, In real world postgres uses other techniques to overcome this.
* Sequential keys (1, 2, 3...) neatly append new rows to the end of the tree. Random UUIDs force the database to insert rows into the middle, causing page splits
* In MySQL (InnoDB), rows are physically stored sorted by the Primary Key by default. In Postgres, pages are never stored sorted by default. They are simply stored in insertion/free-space order inside the heap, regardless of Primary Key. Then comes the secondary key useful for postgres.
* Index Organised Table for MySQL.

### Secondary Key
* Every secondary index creates its own separate B-Tree sorted by the indexed column. At the leaf level, it stores the indexed value paired with a pointer to the full row—pointing directly to the Heap in PostgreSQL, or to the Primary Key value in MySQL (InnoDB). Because each index is an independent B-Tree, adding more indexes speeds up reads on those columns but slows down writes, as every tree must be updated on insert/update/delete.
* A Foreign Key is a logical constraint, not an index by definition. MySQL (InnoDB) automatically creates a secondary B-Tree index on every Foreign Key column, whereas PostgreSQL does not.
* MySQL (InnoDB), Secondary Indexes do not store physical disk pointers—they store the Primary Key value, meaning a secondary lookup must traverse two B-Trees (Secondary B-Tree -> Primary Key B-Tree) to fetch the row.
* In Postgres, Secondary key b-tree checks for pointers in Heap Organised Table. Whereas In MySQL, mentioned in previous point.
