# DBMS — 50 MCQs for NatWest SDE Assessment Prep

Organized by subtopic. Format: Question → 4 options → **Answer** → short explanation.

---

## ER Model & Keys (Q1–10)

**Q1.** Which key uniquely identifies a record in a table and cannot contain NULL values?
A) Candidate Key  B) Primary Key  C) Foreign Key  D) Alternate Key
**Answer: B** — A Primary Key must be unique and NOT NULL by definition; a table can have only one.

**Q2.** A key that is a candidate key but not chosen as the primary key is called:
A) Super Key  B) Alternate Key  C) Composite Key  D) Foreign Key
**Answer: B**

**Q3.** A foreign key in a table refers to:
A) The primary key of the same table  B) The primary key of another table  C) Any column of another table  D) An index of another table
**Answer: B** — It establishes referential integrity by referencing another table's primary/unique key.

**Q4.** Which of these is a super key but NOT necessarily a candidate key?
A) A minimal set of attributes uniquely identifying a row  B) Any set of attributes containing a candidate key  C) A foreign key  D) A non-unique attribute
**Answer: B** — Super key = candidate key + possibly extra (non-minimal) attributes.

**Q5.** In an ER diagram, a diamond shape represents:
A) Entity  B) Attribute  C) Relationship  D) Weak Entity
**Answer: C**

**Q6.** A weak entity depends on:
A) Its own primary key only  B) A strong (owner) entity for its existence  C) No other entity  D) Only foreign keys
**Answer: B** — Weak entities lack a full primary key of their own and are identified via a partial key + owner entity's key.

**Q7.** Multi-valued attributes are represented in ER diagrams using:
A) Single oval  B) Double oval  C) Rectangle  D) Double rectangle
**Answer: B**

**Q8.** A composite key consists of:
A) A single column  B) Two or more columns combined to uniquely identify a row  C) Only foreign key columns  D) A derived attribute
**Answer: B**

**Q9.** Which cardinality describes "one student enrolls in many courses, and one course has many students"?
A) One-to-One  B) One-to-Many  C) Many-to-One  D) Many-to-Many
**Answer: D**

**Q10.** A derived attribute is one that:
A) Is stored directly in the database  B) Can be calculated from other attributes (e.g., age from date of birth)  C) Has multiple values  D) Is always a key
**Answer: B**

---

## Normalization (Q11–20)

**Q11.** A relation is in 1NF if:
A) It has no partial dependency  B) All attributes contain only atomic (indivisible) values  C) It has no transitive dependency  D) Every determinant is a candidate key
**Answer: B**

**Q12.** A relation is in 2NF if it is in 1NF and:
A) Has no transitive dependency  B) Has no partial dependency of non-key attributes on the primary key  C) Every non-key attribute is atomic  D) All keys are foreign keys
**Answer: B** — Partial dependency = a non-key attribute depends on only part of a composite primary key.

**Q13.** A relation is in 3NF if it is in 2NF and:
A) It has no multi-valued dependency  B) It has no transitive dependency (non-key attribute depending on another non-key attribute)  C) It has no derived attributes  D) All attributes are keys
**Answer: B**

**Q14.** BCNF (Boyce-Codd Normal Form) is stricter than 3NF because:
A) It removes all NULL values  B) Every determinant must be a candidate key  C) It requires atomic values  D) It eliminates foreign keys
**Answer: B**

**Q15.** The main goal of normalization is to:
A) Increase data redundancy  B) Reduce data redundancy and avoid update/insert/delete anomalies  C) Increase query speed always  D) Merge all tables into one
**Answer: B**

**Q16.** Which anomaly occurs when deleting a row unintentionally removes other useful information?
A) Insertion anomaly  B) Update anomaly  C) Deletion anomaly  D) Redundancy anomaly
**Answer: C**

**Q17.** A functional dependency X → Y means:
A) Y determines X  B) X uniquely determines Y  C) X and Y are independent  D) X and Y must be primary keys
**Answer: B**

**Q18.** Denormalization is typically done to:
A) Reduce redundancy  B) Improve read/query performance at the cost of some redundancy  C) Enforce more constraints  D) Remove all joins permanently
**Answer: B**

**Q19.** 4NF deals primarily with eliminating:
A) Partial dependency  B) Transitive dependency  C) Multi-valued dependency  D) Functional dependency
**Answer: C**

**Q20.** A table storing (StudentID, CourseID, InstructorName) where InstructorName depends only on CourseID (not the full key) violates:
A) 1NF  B) 2NF  C) 3NF  D) BCNF
**Answer: B** — Classic partial dependency example (composite key StudentID+CourseID, but InstructorName depends on CourseID alone).

---

## SQL Queries & Joins (Q21–32)

**Q21.** Which SQL clause is used to filter groups after a GROUP BY?
A) WHERE  B) HAVING  C) FILTER  D) ORDER BY
**Answer: B** — WHERE filters rows before grouping; HAVING filters after aggregation.

**Q22.** What does `SELECT COUNT(*) FROM Employees WHERE DeptID IS NULL;` return?
A) Error  B) Count of rows where DeptID is NULL  C) Count of all rows  D) 0 always
**Answer: B**

**Q23.** Which JOIN returns all rows from both tables, with NULLs where there's no match?
A) INNER JOIN  B) LEFT JOIN  C) RIGHT JOIN  D) FULL OUTER JOIN
**Answer: D**

**Q24.** A LEFT JOIN between Table A and Table B returns:
A) Only matching rows  B) All rows from A, matched rows from B (NULL if no match)  C) All rows from B only  D) Cartesian product
**Answer: B**

**Q25.** Which query finds duplicate emails in a table `Person(id, email)`?
A) `SELECT email FROM Person GROUP BY email HAVING COUNT(*) > 1;`
B) `SELECT email FROM Person WHERE COUNT(*) > 1;`
C) `SELECT DISTINCT email FROM Person;`
D) `SELECT email FROM Person ORDER BY email;`
**Answer: A** — WHERE can't use aggregate functions; HAVING is required after GROUP BY.

**Q26.** What is the output of `SELECT 1 UNION SELECT 1 UNION ALL SELECT 1;`?
A) 1 row with value 1  B) 2 rows with value 1  C) 3 rows with value 1  D) Error
**Answer: B** — UNION removes duplicates for the first pair (giving one row of `1`), then UNION ALL adds another `1` without dedup, giving 2 rows total.

**Q27.** A CROSS JOIN between a table with 3 rows and a table with 4 rows produces:
A) 3 rows  B) 4 rows  C) 7 rows  D) 12 rows
**Answer: D** — Cartesian product = m × n rows.

**Q28.** Which SQL keyword removes duplicate rows from a result set?
A) UNIQUE  B) DISTINCT  C) FILTER  D) REMOVE
**Answer: B**

**Q29.** What does a self-join do?
A) Joins a table with itself using aliases  B) Joins two different tables  C) Deletes duplicate rows  D) Creates an index
**Answer: A** — Common use: finding employees who share the same manager, using `Employee e1 JOIN Employee e2`.

**Q30.** In SQL, `NULL = NULL` evaluates to:
A) TRUE  B) FALSE  C) NULL (unknown)  D) Error
**Answer: C** — This is why `IS NULL` must be used instead of `= NULL`.

**Q31.** Which clause limits the number of rows returned in MySQL?
A) TOP  B) LIMIT  C) ROWCOUNT  D) FETCH FIRST
**Answer: B** — (Note: SQL Server uses TOP, Oracle uses FETCH FIRST/ROWNUM — MySQL/PostgreSQL use LIMIT)

**Q32.** The query to find the 2nd highest salary using a subquery:
A) `SELECT MAX(salary) FROM Employee;`
B) `SELECT MAX(salary) FROM Employee WHERE salary < (SELECT MAX(salary) FROM Employee);`
C) `SELECT salary FROM Employee ORDER BY salary LIMIT 1;`
D) `SELECT MIN(salary) FROM Employee;`
**Answer: B**

---

## Transactions & ACID (Q33–40)

**Q33.** ACID stands for:
A) Atomicity, Consistency, Isolation, Durability  B) Accuracy, Concurrency, Integrity, Durability  C) Atomicity, Concurrency, Isolation, Data  D) Accuracy, Consistency, Isolation, Data
**Answer: A**

**Q34.** Which ACID property ensures a transaction is either fully completed or fully rolled back?
A) Consistency  B) Atomicity  C) Isolation  D) Durability
**Answer: B**

**Q35.** Which property ensures that once a transaction is committed, it remains so even after a system crash?
A) Atomicity  B) Consistency  C) Isolation  D) Durability
**Answer: D**

**Q36.** The "dirty read" problem occurs when:
A) A transaction reads data that was later rolled back by another transaction  B) A transaction reads committed data  C) Two transactions deadlock  D) A transaction reads the same row twice
**Answer: A**

**Q37.** Which isolation level prevents dirty reads but still allows non-repeatable reads?
A) Read Uncommitted  B) Read Committed  C) Repeatable Read  D) Serializable
**Answer: B**

**Q38.** The strictest SQL isolation level, preventing dirty reads, non-repeatable reads, and phantom reads, is:
A) Read Committed  B) Repeatable Read  C) Serializable  D) Snapshot
**Answer: C**

**Q39.** A deadlock in a database occurs when:
A) Two transactions wait indefinitely for each other's locks  B) A transaction fails to commit  C) A table has no primary key  D) An index is corrupted
**Answer: A**

**Q40.** Which command is used to permanently save changes made in a transaction?
A) SAVE  B) COMMIT  C) ROLLBACK  D) FLUSH
**Answer: B**

---

## Indexing & Storage (Q41–46)

**Q41.** An index in a database primarily improves:
A) INSERT speed  B) Read/SELECT query speed  C) Storage compression  D) Data integrity
**Answer: B** — Trade-off: indexes speed up reads but slow down writes (INSERT/UPDATE/DELETE) since the index must also be updated.

**Q42.** A clustered index determines:
A) Only the order of the index itself  B) The physical storage order of data rows in the table  C) Nothing about storage  D) Foreign key relationships
**Answer: B** — A table can have only ONE clustered index (since data can only be physically sorted one way) but many non-clustered indexes.

**Q43.** Which data structure is most commonly used internally for database indexes?
A) Linked List  B) B-Tree / B+ Tree  C) Hash Table only  D) Binary Search Tree only
**Answer: B** — B+ Trees are preferred because they keep data sorted and support efficient range queries, unlike plain hash tables.

**Q44.** A primary key automatically creates:
A) A non-clustered index always  B) A clustered index in most RDBMS (e.g., default in SQL Server/MySQL InnoDB)  C) No index  D) A foreign key
**Answer: B**

**Q45.** What is a major downside of having too many indexes on a table?
A) Faster SELECT queries  B) Slower INSERT/UPDATE/DELETE due to index maintenance overhead  C) Reduced storage usage  D) No downside
**Answer: B**

**Q46.** A composite index on columns (A, B) can be efficiently used for a query filtering on:
A) Only B  B) Only A, or A and B together  C) Neither column  D) Any column in the table
**Answer: B** — Left-prefix rule: a composite index is usable when the query includes the leftmost column(s) of the index.

---

## Views, Triggers & Constraints (Q47–50)

**Q47.** A VIEW in SQL is:
A) A physical copy of table data  B) A virtual table based on a stored query  C) An index  D) A type of trigger
**Answer: B**

**Q48.** A trigger in SQL is:
A) A stored procedure executed manually  B) A block of code automatically executed in response to an event (INSERT/UPDATE/DELETE)  C) A type of index  D) A constraint on a foreign key
**Answer: B**

**Q49.** Which constraint ensures a column's value must be unique across the table but allows one NULL (in most RDBMS)?
A) PRIMARY KEY  B) UNIQUE  C) CHECK  D) NOT NULL
**Answer: B**

**Q50.** The `CHECK` constraint is used to:
A) Enforce a condition that all values in a column must satisfy  B) Create a foreign key relationship  C) Index a column  D) Remove duplicate rows
**Answer: A** — e.g., `CHECK (age >= 18)`
