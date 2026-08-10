# Top 75 Database & SQL Interview Questions for Data Analyst / Data Science Roles

Compiled from patterns reported across Data Analyst and Data Scientist interview loops at **Amazon, Meta, Google, Microsoft, Uber, Airbnb, Netflix, and Stripe** — the companies that lean hardest on SQL as a primary technical filter for data roles. Questions go from foundational DBMS concepts through core SQL, up to advanced query-writing, optimization, and system-design-adjacent topics that show up in senior/onsite rounds.

Difficulty is labeled per question: **[F] Foundational**, **[M] Medium**, **[A] Advanced** — roughly mapping to phone-screen, onsite technical round, and senior/bar-raiser round difficulty respectively.

How to use this: SQL interviews for data roles are rarely pure trivia — most companies pair a conceptual question with a live query-writing task on a real or sample schema. For every question below, be ready to (1) give the concept, (2) write the actual SQL for it, and (3) explain a business scenario where you'd use it (funnel analysis, retention, revenue trends, experiment analysis — these are the recurring shapes at Meta, Amazon, and Uber specifically).

---

## 1. Database & DBMS Fundamentals (Q1–10)

**1. [F] What is a database, and what is a DBMS? What's the difference between a DBMS and an RDBMS?**

> A database is an organized collection of structured data stored electronically so it can be efficiently accessed, managed, and updated. A DBMS, Database Management System, is the software layer that lets users and applications interact with that data — creating, reading, updating, and deleting it — without having to manage the raw file storage directly. An RDBMS, Relational DBMS, is a specific type of DBMS that organizes data into tables with rows and columns, enforces relationships between those tables using keys, and supports SQL as the query language — MySQL, PostgreSQL, and SQL Server are all RDBMS examples. The distinction matters because not every DBMS is relational — MongoDB, for instance, is a DBMS but not an RDBMS, since it stores data as flexible JSON-like documents instead of rigid tables. In my own work, I mostly interact with RDBMS systems day to day, since most business reporting and analytics still sits on structured, relational data.

**2. [F] What is the difference between SQL and NoSQL databases? When would you choose one over the other?**

> SQL databases are relational, meaning data is stored in structured tables with a predefined schema, and they enforce strong consistency and relationships through keys — think PostgreSQL or MySQL. NoSQL databases are non-relational and come in several flavors — document stores like MongoDB, key-value stores like Redis, wide-column stores like Cassandra, and graph databases like Neo4j — and they typically trade strict schema enforcement for flexibility and horizontal scalability. I'd choose SQL when the data is highly structured, relationships between entities matter a lot, like orders linked to customers linked to products, and I need strong consistency guarantees, like in financial transactions. I'd choose NoSQL when the data is unstructured or semi-structured and the schema might evolve frequently, like storing raw JSON event logs from an app, or when I need to scale horizontally across many servers to handle massive write volume, like a social media platform's activity feed. In practice, many real companies use both side by side — a NoSQL store to ingest raw high-volume event data, and a SQL data warehouse downstream where that data gets cleaned and modeled for analysts to query.

**3. [F] What are the different types of SQL commands — DDL, DML, DQL, DCL, TCL? Give an example statement for each.**

> DDL, Data Definition Language, defines or modifies the structure of database objects — for example, `CREATE TABLE employees (...)` or `ALTER TABLE employees ADD COLUMN email VARCHAR(255)`. DML, Data Manipulation Language, is used to modify the actual data within tables — for example, `INSERT INTO employees VALUES (...)`, `UPDATE employees SET salary = 90000 WHERE id = 5`, or `DELETE FROM employees WHERE id = 5`. DQL, Data Query Language, is technically just `SELECT`, used purely to retrieve data — for example, `SELECT name, salary FROM employees WHERE department = 'Sales'`. DCL, Data Control Language, manages permissions — for example, `GRANT SELECT ON employees TO analyst_role` or `REVOKE INSERT ON employees FROM analyst_role`. TCL, Transaction Control Language, manages transactions — for example, `COMMIT`, `ROLLBACK`, and `SAVEPOINT`. As a data analyst, I spend the vast majority of my time in DQL and occasionally DML, but understanding all five categories helps me communicate clearly with data engineers about schema changes or access requests.

**4. [F] What is a primary key? What is a foreign key? How are they different from a unique key?**

> A primary key uniquely identifies each row in a table — it can't contain NULL values, and a table can have only one primary key, though that key can be composed of multiple columns. A foreign key is a column, or set of columns, in one table that references the primary key of another table, and it's what enforces relationships between tables — for example, an `orders` table would have a `customer_id` foreign key referencing the `customers` table's primary key, ensuring you can't have an order tied to a customer that doesn't exist. A unique key is similar to a primary key in that it also enforces uniqueness across a column, but a table can have multiple unique keys, and unlike a primary key, a unique key typically does allow a single NULL value, depending on the database. A concrete example: in an employees table, `employee_id` would typically be the primary key, `email` might be a unique key since no two employees should share an email but it's not what identifies the row, and `department_id` would be a foreign key pointing to a separate `departments` table.

**5. [F] What is a composite key, and when would you need one?**

> A composite key is a primary key made up of two or more columns combined, used when no single column alone is enough to uniquely identify a row. A classic real-world example is an `order_items` table that stores which products belong to which order — a single row might be identified by the combination of `order_id` and `product_id` together, since neither column alone is unique across the table, an order has multiple products, and a product appears in multiple orders, but that specific pairing is unique. I'd reach for a composite key specifically in these many-to-many relationship or junction tables, where the natural identity of a row really is the combination of two foreign keys rather than any single generated ID.

**6. [F] What is the difference between a candidate key, a super key, and an alternate key?**

> A super key is any combination of columns that can uniquely identify a row — it can include extra, unnecessary columns and still technically qualify. A candidate key is a minimal super key, meaning it uniquely identifies a row using the fewest columns possible, with no redundant columns that could be removed while still maintaining uniqueness. A table can have multiple candidate keys — for example, an employees table might have both `employee_id` and `email` as valid candidate keys, since either one alone uniquely identifies a row. Once you pick one candidate key to actually serve as the table's primary key, every other candidate key that wasn't chosen becomes an alternate key. So the relationship is hierarchical: super keys are the broadest category, candidate keys are the minimal subset of those, and the primary key and alternate keys are just labels for which candidate key you picked versus didn't pick.

**7. [M] What is referential integrity, and how do foreign key constraints enforce it?**

> Referential integrity means that relationships between tables stay logically consistent — specifically, that a foreign key value in one table must always correspond to an existing primary key value in the referenced table, or be NULL if that's allowed. A foreign key constraint enforces this at the database level: if I try to insert a row into `orders` with a `customer_id` that doesn't exist in the `customers` table, the database will reject that insert and throw an error, rather than silently allowing an orphaned reference. It also governs what happens on deletes and updates — for example, a constraint might be set to `ON DELETE CASCADE`, meaning if a customer is deleted, all their related orders are automatically deleted too, or `ON DELETE RESTRICT`, which would block deleting a customer if they still have existing orders. From an analyst's perspective, referential integrity is exactly what lets me trust that a JOIN between `orders` and `customers` won't silently drop or mismatch rows because of a dangling reference — though in practice I've still seen real-world systems where constraints were relaxed for performance reasons, which is exactly why validating join results is still a habit worth having.

**8. [F] What is the difference between CHAR and VARCHAR data types?**

> CHAR is a fixed-length string type — if you define a column as CHAR(10) and store a 4-character value, the database pads it with trailing spaces to fill all 10 characters, using the same fixed storage every time. VARCHAR is a variable-length string type — it only uses as much storage as the actual string requires, plus a small amount of overhead to track the length, up to the defined maximum. The practical trade-off: CHAR can be marginally faster for fixed-length data since the database doesn't need to calculate variable lengths, which is why it's sometimes used for things like fixed-format codes, such as a 2-character country code or a 3-character currency code. VARCHAR is what I'd use for almost everything else — names, addresses, descriptions — where the actual length genuinely varies row to row, since using CHAR there would waste storage padding shorter values unnecessarily.

**9. [F] What is NULL in SQL, and how is it different from zero or an empty string?**

> NULL represents the complete absence of a value — it means "unknown" or "not applicable," not "zero" or "blank." This is a subtle but important distinction: zero is a real, defined numeric value, and an empty string is a real, defined string with zero characters, but NULL means the data simply isn't there at all. This has real consequences in queries — `NULL = NULL` doesn't evaluate to true, it evaluates to unknown, which is why you have to use `IS NULL` instead of `= NULL` to actually check for it. It also affects aggregate functions — `AVG()`, `SUM()`, and `COUNT(column)` all ignore NULL values by default rather than treating them as zero, which can silently skew results if you're not careful. A real example: if a customer's `phone_number` column is NULL, that means we don't have their phone number at all; if it were an empty string instead, that might ambiguously mean either "we don't have it" or "they explicitly have no phone," which is exactly why treating NULL and empty string as genuinely different states matters for accurate reporting.

**10. [M] What is the difference between a relational database and a data warehouse? When would an analyst work with each?**

> A relational database, in the OLTP sense, is typically optimized for fast, transactional read/write operations on individual records — like an e-commerce application's live database handling one order being placed at a time. A data warehouse is optimized for analytical queries across huge volumes of historical data — it's designed to answer questions like "what was total revenue by region for the last 12 months" efficiently, often by using a denormalized or star-schema structure and columnar storage that's much faster for large aggregations than a typical row-based OLTP database. As an analyst, I'd rarely query the live production OLTP database directly, both because it could slow down the actual application for real users, and because its schema is optimized for transactions, not analysis. Instead, I'd work against a data warehouse, like Snowflake, Redshift, or BigQuery, where that same data has already been extracted, transformed, and reshaped specifically to make aggregate, historical, and cross-table analytical queries fast and straightforward.

## 2. Core SQL — Querying, Filtering, Sorting (Q11–20)

**11. [F] What is the difference between WHERE and HAVING? Why can't you use an aggregate function in a WHERE clause?**

> WHERE filters individual rows before any grouping happens, while HAVING filters groups after aggregation has already occurred. The reason you can't use an aggregate function like `SUM()` or `COUNT()` in a WHERE clause comes down to SQL's logical execution order — WHERE is evaluated before GROUP BY, so at that point aggregate values don't exist yet, since there are no groups formed to aggregate over. For example, if I wanted departments with more than 10 employees, I couldn't write `WHERE COUNT(*) > 10`, because at the row-filtering stage, there's no COUNT yet — I'd have to write it as `SELECT department, COUNT(*) FROM employees GROUP BY department HAVING COUNT(*) > 10`, since HAVING runs after the grouping and aggregation have already happened.

**12. [F] What is the correct logical order of execution of a SQL query (FROM, WHERE, GROUP BY, HAVING, SELECT, ORDER BY, LIMIT)?**

> Even though we write a query starting with SELECT, that's not the order it's actually processed in. The real logical order is: FROM, which identifies the source tables and processes any JOINs; WHERE, which filters individual rows; GROUP BY, which groups the remaining rows; HAVING, which filters those groups; SELECT, which picks the actual columns or expressions to return; ORDER BY, which sorts the final result set; and finally LIMIT, which restricts how many rows come back. Understanding this order actually explains several SQL quirks that used to confuse me — like why you can't reference a column alias defined in SELECT inside the WHERE clause, because WHERE executes before SELECT even runs, but you often can reference that same alias in ORDER BY, because ORDER BY runs after SELECT has already computed it.

**13. [F] What is the difference between DISTINCT and GROUP BY?**

> DISTINCT removes duplicate rows from the result set based on all selected columns, and it's purely about de-duplication. GROUP BY groups rows that share the same values in specified columns, and it's typically used together with aggregate functions to compute a value per group, like a count or sum per category. In some simple cases they can look interchangeable — `SELECT DISTINCT department FROM employees` and `SELECT department FROM employees GROUP BY department` would return the same result — but GROUP BY is the right tool the moment I actually want to compute something per group, like `SELECT department, COUNT(*) FROM employees GROUP BY department`, since DISTINCT alone has no concept of aggregation.

**14. [F] How do you handle NULL values in a query — explain IS NULL, IS NOT NULL, COALESCE, and IFNULL/ISNULL.**

> `IS NULL` and `IS NOT NULL` are the correct way to actually test for NULL, since as covered earlier, `= NULL` doesn't work the way people expect. `COALESCE(value1, value2, ...)` returns the first non-NULL value in the list, which is extremely useful for providing a fallback — for example, `COALESCE(phone_number, 'Not Provided')` would display an actual phone number if one exists, or fall back to that default text if it's NULL. `IFNULL` in MySQL and `ISNULL` in SQL Server are database-specific shortcuts that do the same thing as COALESCE but only accept exactly two arguments, whereas COALESCE is ANSI-standard and works across virtually every SQL dialect, which is why I'd generally default to COALESCE for portability unless I know I'm locked into one specific database.

**15. [M] Write a query to find duplicate records in a table.**

> I'd group by the columns that define what counts as a "duplicate" for that table, and then filter for groups appearing more than once using HAVING. For example, to find customers with duplicate email addresses:
> ```sql
> SELECT email, COUNT(*) AS occurrence_count
> FROM customers
> GROUP BY email
> HAVING COUNT(*) > 1;
> ```
> This tells me which values are duplicated and how many times, which is usually the first step before deciding how to actually clean them up — I'd typically run this exact kind of query early in any new dataset just to sanity-check for unexpected duplication before trusting downstream aggregates.

**16. [M] Write a query to remove duplicate rows from a table while keeping one copy.**

> A common, reliable approach is using a window function to rank duplicate rows and then deleting everything except the first occurrence per group:
> ```sql
> WITH ranked AS (
>   SELECT *,
>          ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at) AS rn
>   FROM customers
> )
> DELETE FROM customers
> WHERE customer_id IN (
>   SELECT customer_id FROM ranked WHERE rn > 1
> );
> ```
> Here I'm partitioning by whatever column defines a duplicate, `email` in this case, ordering by `created_at` so I consistently keep the earliest record, and then deleting every row where the row number is greater than 1, meaning every row after the first one in each duplicate group. I'd always run the `SELECT` version of this first to visually confirm which rows would be deleted before actually running the DELETE, since removing data is not something you want to get wrong.

**17. [F] What is the difference between DELETE, TRUNCATE, and DROP?**

> DELETE removes specific rows based on a WHERE condition, is logged row-by-row, can be rolled back within a transaction, and doesn't reset auto-increment counters — it's the slowest but most surgical option. TRUNCATE removes all rows from a table at once, is much faster since it doesn't log individual row deletions, typically resets auto-increment counters, but generally can't be selectively filtered with a WHERE clause and in most databases can't easily be rolled back once committed. DROP removes the entire table structure itself, not just the data — the table, its schema, indexes, and constraints all disappear entirely. A simple way I'd frame it: DELETE is "remove some rows," TRUNCATE is "empty the whole table but keep the structure," and DROP is "the table no longer exists at all." I'd be especially careful with TRUNCATE and DROP in any environment, since accidentally running either against a production table is a much bigger mistake than an over-broad DELETE, which at least you can often recover within a transaction.

**18. [M] What is the difference between UNION and UNION ALL? What about INTERSECT and EXCEPT/MINUS?**

> UNION combines the result sets of two queries and removes duplicate rows, while UNION ALL combines them and keeps every row, including duplicates. Because removing duplicates requires extra processing, UNION ALL is meaningfully faster, so I'd default to UNION ALL whenever I already know the two result sets won't overlap, or when I actually want to preserve duplicates intentionally, and only use plain UNION when de-duplication is actually needed. INTERSECT returns only the rows that appear in both query results, and EXCEPT, called MINUS in Oracle, returns rows that appear in the first query but not the second. A practical use case: if I have a table of users who signed up for a free trial and a table of users who made a purchase, `INTERSECT` would show me users who did both, while `EXCEPT` would show me users who signed up for the trial but never converted to a purchase — which is exactly the kind of funnel-drop-off question that comes up constantly in product analytics interviews.

**19. [F] How do LIKE and wildcard operators (%, _) work in SQL? Give an example use case.**

> LIKE is used for pattern matching within string columns. The `%` wildcard matches any sequence of zero or more characters, and `_` matches exactly one character. For example, `WHERE email LIKE '%@gmail.com'` would find every email ending in @gmail.com, `WHERE name LIKE 'A%'` would find every name starting with the letter A, and `WHERE product_code LIKE 'A_123'` would match a 5-character code starting with A, any single character, then 123. One thing I'd flag in an interview: `LIKE` with a leading wildcard, like `'%gmail.com'`, generally can't use a standard index efficiently since the database can't know where to start scanning from, so on very large tables it's worth being aware that a leading-wildcard search can be meaningfully slower than a trailing-wildcard one.

**20. [M] Write a query to fetch the nth highest salary from an employee table (without using LIMIT/OFFSET directly, and then with it).**

> Without LIMIT/OFFSET, I'd use a subquery counting how many distinct salaries are higher than each candidate value — this approach also works in databases that don't support LIMIT/OFFSET, like older SQL Server versions:
> ```sql
> SELECT salary
> FROM employees e1
> WHERE (
>   SELECT COUNT(DISTINCT salary)
>   FROM employees e2
>   WHERE e2.salary > e1.salary
> ) = N - 1;
> ```
> This says: "give me the salary where exactly N-1 distinct salaries are higher than it" — so for the 3rd highest salary, I'd look for exactly 2 distinct salaries above it. With LIMIT/OFFSET, it's much more straightforward:
> ```sql
> SELECT DISTINCT salary
> FROM employees
> ORDER BY salary DESC
> LIMIT 1 OFFSET (N - 1);
> ```
> I'd always use `DISTINCT` here, since without it, tied salaries would throw off the ranking — if three people are tied for 2nd highest, I want "3rd highest" to correctly mean the 3rd distinct value, not just skip past the tie by row count. In practice, I'd actually reach for `DENSE_RANK()` as the cleanest modern solution, which we'll get into in the window functions section.

## 3. Joins & Relationships (Q21–29)

**21. [F] Explain the different types of JOINs — INNER, LEFT, RIGHT, FULL OUTER, and CROSS JOIN — with a simple example of each.**

> An INNER JOIN returns only the rows that have a match in both tables — for example, `SELECT * FROM orders INNER JOIN customers ON orders.customer_id = customers.id` would return only orders that have a valid, matching customer. A LEFT JOIN returns all rows from the left table, plus matching rows from the right table, filling in NULLs where there's no match — so `orders LEFT JOIN customers` would return every order even if, hypothetically, its customer record was somehow missing, with NULL customer fields in that case. A RIGHT JOIN is the mirror image, returning all rows from the right table plus matches from the left — I rarely use RIGHT JOIN in practice since you can always rewrite it as a LEFT JOIN by swapping table order, and most teams standardize on LEFT for readability. A FULL OUTER JOIN returns all rows from both tables, matched where possible and NULL-filled where not, which is useful for finding mismatches in both directions at once, like customers with no orders and orders with no valid customer, in a single query. A CROSS JOIN returns the Cartesian product — every row from the first table paired with every row from the second table, with no matching condition at all — this is rarely what you actually want in analytics, but it does have legitimate uses, like generating every combination of date and store ID to build out a complete reporting calendar.

**22. [M] What is a self-join, and when would you use one? Give a real business example.**

> A self-join is when a table is joined to itself, treated as if it were two separate tables using aliases, typically used when rows in the same table have a relationship to each other. A classic example is an employees table where each row has a `manager_id` that references another employee's `employee_id` — to list each employee alongside their manager's name, I'd write:
> ```sql
> SELECT e.name AS employee_name, m.name AS manager_name
> FROM employees e
> LEFT JOIN employees m ON e.manager_id = m.employee_id;
> ```
> I use a LEFT JOIN here specifically because the CEO or top-level employee likely has a NULL `manager_id`, and I don't want them to disappear from the results entirely. Another common real-world use is comparing rows within the same table over time, like comparing each month's revenue against the previous month's revenue from the same sales table.

**23. [M] What's the difference between a JOIN and a subquery — when would you prefer one over the other?**

> A JOIN combines columns from two or more tables into a single result set, side by side, and is generally the go-to when you actually need columns from both tables in your final output. A subquery is a query nested inside another query, and is often used when you only need to check for existence, filter based on an aggregated value, or use the result as a single value or filtering list, without necessarily needing columns from the other table in your final output. For example, to list customers who have placed at least one order, I could either JOIN orders and customers and then DISTINCT the result, or use `WHERE customer_id IN (SELECT customer_id FROM orders)` — the subquery version is often cleaner here because I'm not actually pulling any columns from orders, just using it as a filter. In terms of performance, modern query optimizers often rewrite subqueries into equivalent joins internally anyway, so in many cases the choice comes down to readability and intent rather than raw speed, though very large or poorly structured correlated subqueries can sometimes perform worse than an equivalent join, which is worth checking with EXPLAIN if performance is a concern.

**24. [M] How would you find records that exist in one table but not another (LEFT JOIN with NULL check vs. NOT EXISTS vs. EXCEPT)?**

> There are three common approaches, and I'd generally reach for whichever is most readable for the situation, though performance can vary by database engine. First, a LEFT JOIN with a NULL check: `SELECT c.* FROM customers c LEFT JOIN orders o ON c.id = o.customer_id WHERE o.customer_id IS NULL` — this returns customers with no matching order, since unmatched rows get NULLs in the right table's columns. Second, `NOT EXISTS`: `SELECT * FROM customers c WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)` — this tends to be my preferred approach, since it clearly communicates intent and generally handles NULLs more safely than NOT IN. Third, `EXCEPT`: `SELECT id FROM customers EXCEPT SELECT customer_id FROM orders` — this is the most concise when I just need the ID column and not the full row. One thing I'd flag: I'd specifically avoid `NOT IN` with a subquery that might return NULL values, since if even one NULL sneaks into that subquery's result, `NOT IN` can unexpectedly return an empty result set entirely, which is a subtle bug that's bitten me before.

**25. [F] What happens if you JOIN two tables without an ON condition (Cartesian product)? Why is this dangerous in production queries?**

> Without a join condition, every row in the first table gets paired with every row in the second table, producing a result set with rows equal to the product of both table sizes — this is exactly what a CROSS JOIN produces, and it's what you accidentally get if you write an old-style comma join like `SELECT * FROM orders, customers` without a WHERE clause linking them. This is dangerous in production for a very practical reason: if `orders` has 1 million rows and `customers` has 100,000 rows, an accidental Cartesian product would generate 100 billion rows, which can silently consume massive amounts of memory and compute, potentially timing out a query, spiking a warehouse's compute costs, or even crashing a session. I've genuinely seen this happen from a simple typo, forgetting to include an ON clause, which is exactly why I always double-check my join conditions before running a query against a large table, especially in a shared production environment.

**26. [M] Write a query to find employees who earn more than their manager, given a self-referencing employee table.**

> Using the same self-join structure from before:
> ```sql
> SELECT e.name AS employee_name, e.salary AS employee_salary,
>        m.name AS manager_name, m.salary AS manager_salary
> FROM employees e
> JOIN employees m ON e.manager_id = m.employee_id
> WHERE e.salary > m.salary;
> ```
> Here I'm using an INNER JOIN rather than LEFT, since I specifically only care about employees who do have a manager to compare against — an employee with no manager can't meet this condition anyway. This is a classic interview question specifically because it tests whether you understand self-joins conceptually rather than just memorizing standard join syntax between two clearly different tables.

**27. [M] What is the difference between an equi-join and a non-equi-join? Give an example of when you'd use a non-equi-join.**

> An equi-join uses an equality condition to match rows between tables, which is what the vast majority of real-world joins are — `ON a.id = b.id`. A non-equi-join uses any other comparison operator, like `>`, `<`, `BETWEEN`, or `!=`, instead of equality. A practical example: joining a transactions table to a tax-bracket reference table where each bracket has a `min_income` and `max_income` range, using `ON transactions.income BETWEEN tax_brackets.min_income AND tax_brackets.max_income` — there's no single equal value to match on, since I'm matching a value against a range instead. Another common example is joining an events table to a promotions table using `ON events.event_date BETWEEN promotions.start_date AND promotions.end_date`, to figure out which promotion, if any, was active at the time of each event.

**28. [A] How would you join two very large tables efficiently, and what factors affect JOIN performance (indexing, join order, data types)?**

> A few things matter most for join performance at scale. First, indexing — the columns used in the JOIN condition should be indexed on both tables, ideally, since without an index the database may need to do a full table scan for every row being matched, which gets extremely expensive as table size grows. Second, data type consistency — if the join columns have mismatched types, like joining a VARCHAR column to an INT column, the database often has to implicitly cast one side, which can prevent it from using an index at all; making sure join keys share the exact same data type avoids this silent performance killer. Third, filtering as early as possible — applying WHERE conditions that reduce row count before the join happens, rather than joining the full unfiltered tables and filtering afterward, means the database has fewer rows to actually match in the first place. Fourth, in a distributed warehouse like Redshift, Snowflake, or BigQuery specifically, data distribution and clustering also matter a lot — if two large tables being joined aren't co-located on the same compute node for matching keys, the engine may need to shuffle massive amounts of data across the network, which is often the actual bottleneck, not the join logic itself. In practice, if I hit a genuinely slow join on a huge table, my first move is running `EXPLAIN` to see whether it's doing a full scan, a broadcast, or a shuffle, and work backward from there instead of guessing.

**29. [M] What are the risks of joining on a non-unique or nullable column, and how would you validate a join before trusting its output?**

> Joining on a non-unique column, meaning a column that isn't a primary or unique key on at least one side of the join, risks silently multiplying rows — often called "fan-out." For example, if I join an orders table to a promotions table on `store_id`, and a store has run multiple overlapping promotions, each order might get duplicated once per matching promotion, inflating my row count and any downstream SUM or COUNT without any obvious error being thrown. Joining on a nullable column is also risky, since NULL never equals NULL in standard join logic, meaning rows with a NULL join key on either side will simply be silently excluded from an INNER JOIN with no warning. To validate a join before trusting it, I'd typically compare row counts before and after the join against my expectation — if I joined a 10,000-row orders table to a dimension table expecting a one-to-one match, and I suddenly have 15,000 rows in the output, that's an immediate red flag telling me the join key isn't as unique as I assumed on the other side, and I'd go check for duplicates on that joining column before trusting any aggregate built on top of it.

## 4. Aggregation, Grouping & Subqueries (Q30–38)

**30. [F] What are the common aggregate functions in SQL (COUNT, SUM, AVG, MIN, MAX)? What's the difference between COUNT(*) and COUNT(column_name)?**

> The common aggregate functions are COUNT, which counts rows; SUM, which totals a numeric column; AVG, which averages a numeric column; MIN, which finds the smallest value; and MAX, which finds the largest value. The key difference between `COUNT(*)` and `COUNT(column_name)` is how they handle NULLs: `COUNT(*)` counts every row regardless of NULL values in any column, while `COUNT(column_name)` only counts rows where that specific column is not NULL. This matters practically — if I run `COUNT(*)` on an orders table I get the total number of orders, but if I run `COUNT(discount_code)` on the same table, I get only the number of orders that actually had a discount code applied, since NULL discount codes get excluded from that count. I've genuinely seen this distinction cause confusion in reporting before, where someone used `COUNT(column)` expecting total row count but silently got a smaller number because that column had NULLs.

**31. [M] What is the difference between a correlated subquery and a non-correlated subquery?**

> A non-correlated subquery runs independently of the outer query — it can execute entirely on its own, produce a result, and that result gets used by the outer query, only once. A correlated subquery references a column from the outer query, meaning it can't be run independently, and conceptually re-executes once for every row processed by the outer query. For example, `SELECT name FROM employees WHERE salary > (SELECT AVG(salary) FROM employees)` is non-correlated — the subquery calculating the average runs once, completely independently. But `SELECT name FROM employees e WHERE salary > (SELECT AVG(salary) FROM employees WHERE department = e.department)` is correlated, since the inner query references `e.department` from the outer query, meaning conceptually it recalculates the average salary separately for each employee's own department. Correlated subqueries are powerful but can be a lot slower on large tables since, depending on the query optimizer, they may effectively run once per outer row rather than once total — which is exactly the kind of thing a window function can often replace more efficiently.

**32. [M] What is the difference between a subquery and a CTE (Common Table Expression)? When would you prefer one over the other?**

> A CTE, defined using `WITH name AS (...)`, is a named, temporary result set that exists only for the duration of that one query, and can be referenced by name later in the same query, potentially multiple times. A subquery is a query nested directly inside another query, typically in the SELECT, FROM, or WHERE clause, without a name, and generally can't be reused elsewhere in the same query without repeating the whole subquery again. I'd prefer a CTE when the query has multiple logical steps I want to build up cleanly and readably, especially if I need to reference the same intermediate result more than once — which avoids duplicating the same subquery logic multiple times in one statement. I'd prefer a subquery for something quick and self-contained, like a single filtering condition that doesn't need a name or doesn't get reused. In practice, on any query I expect to hand off to a teammate or revisit later, I lean toward CTEs specifically because breaking a complex query into named, sequential steps — like `WITH monthly_revenue AS (...), cohort_users AS (...) SELECT ...` — makes the logic dramatically easier to follow than deeply nested subqueries.

**33. [A] What is a recursive CTE, and what kind of problem is it used to solve (e.g., organizational hierarchy, bill-of-materials)?**

> A recursive CTE is a CTE that references itself, used to work through hierarchical or graph-like data where the depth isn't known in advance. It has two parts: an anchor member, which is the base case, like the top-level row(s), and a recursive member, which repeatedly joins back to the CTE itself, building on the previous iteration's results, until no more matching rows are found. A classic example is an employee-manager hierarchy, where I want to find every person who reports up to a given executive, at any depth:
> ```sql
> WITH RECURSIVE org_chart AS (
>   SELECT employee_id, name, manager_id, 1 AS level
>   FROM employees
>   WHERE employee_id = 1  -- the starting executive
>   UNION ALL
>   SELECT e.employee_id, e.name, e.manager_id, oc.level + 1
>   FROM employees e
>   JOIN org_chart oc ON e.manager_id = oc.employee_id
> )
> SELECT * FROM org_chart;
> ```
> Here the anchor is the starting executive, and the recursive part keeps joining employees whose manager was found in the previous iteration, tracking depth with a `level` column. Another common real-world use is a bill-of-materials scenario, like figuring out every sub-component that makes up a manufactured product, where components can themselves be made of other components nested arbitrarily deep.

**34. [M] Write a query using GROUP BY and HAVING to find departments with more than 5 employees earning above the department's average salary.**

> This is a nice one because it actually needs two layers of aggregation logic. I'd approach it with a CTE to first calculate each department's average salary, then compare individual employees against their own department's average:
> ```sql
> WITH dept_avg AS (
>   SELECT department, AVG(salary) AS avg_salary
>   FROM employees
>   GROUP BY department
> )
> SELECT e.department, COUNT(*) AS above_avg_count
> FROM employees e
> JOIN dept_avg d ON e.department = d.department
> WHERE e.salary > d.avg_salary
> GROUP BY e.department
> HAVING COUNT(*) > 5;
> ```
> The logic breaks down into three clear steps: first, compute each department's average salary in the CTE; second, join that back to individual employees and filter to only those earning above their own department's average; third, group by department again and use HAVING to keep only departments where more than 5 employees clear that bar. I like walking through it in these named steps out loud in an interview, since it shows the interviewer I'm reasoning through the problem in stages rather than just pattern-matching to a memorized query.

**35. [M] What is the difference between EXISTS and IN? Which one performs better and why (short-circuiting, NULL handling)?**

> Both can be used to filter based on whether a value appears in another table's results, but they behave differently under the hood. `IN` evaluates the full subquery result first and then checks membership against that list. `EXISTS` checks row by row whether at least one matching row exists, and can short-circuit — stopping as soon as it finds a single match, without needing to evaluate the entire subquery result set. In terms of performance, `EXISTS` is generally the safer default on large tables, especially for correlated checks, since the short-circuiting behavior often makes it faster, and query optimizers frequently handle `EXISTS` more efficiently than a large `IN` list. There's also a NULL-handling trap worth mentioning: `NOT IN` behaves unexpectedly if the subquery's result contains even a single NULL — because comparing against NULL returns unknown rather than true or false, `NOT IN` can end up excluding every row entirely, silently returning zero results. `NOT EXISTS` doesn't have this problem, since it's checking row existence, not comparing against a literal NULL-containing list. Because of this, I specifically prefer `NOT EXISTS` over `NOT IN` whenever there's any chance the subquery's column could contain NULLs, which in real, messy production data, is honestly most of the time.

**36. [A] How does SQL handle NULLs in aggregate functions and in comparisons (e.g., why does `NULL = NULL` evaluate to unknown, not true)?**

> SQL uses three-valued logic — true, false, and unknown — specifically because NULL represents an unknown value, not a real value that could be compared. If I ask "is this unknown value equal to that unknown value," the honestly correct answer isn't "yes" or "no," it's "I don't know," which SQL represents as unknown. This is why `NULL = NULL` doesn't return true — logically, you can't confirm two unknowns are equal — and it's why you have to use `IS NULL` instead of `= NULL` to actually test for it, since `IS NULL` is a special existence check rather than a value comparison. In WHERE and HAVING clauses, only rows that evaluate to true are kept — both false and unknown are filtered out — which is a subtle but important detail, since a condition involving NULL doesn't error out, it just silently excludes that row. For aggregate functions, as covered earlier, SUM, AVG, MIN, MAX, and COUNT(column) all ignore NULL values entirely rather than treating them as zero — so `AVG(salary)` on a column with some NULLs computes the average only over the non-NULL rows, not by dividing by the total row count including NULLs, which is exactly the kind of subtlety that can quietly produce a misleading business metric if you're not paying attention to it.

**37. [M] How would you calculate a running total or cumulative sum using SQL?**

> The modern, clean way is with a window function:
> ```sql
> SELECT order_date, daily_revenue,
>        SUM(daily_revenue) OVER (ORDER BY order_date) AS running_total
> FROM daily_sales;
> ```
> The `SUM() OVER (ORDER BY order_date)` computes a cumulative sum up through each row's date, without collapsing the individual rows the way a plain GROUP BY aggregation would. Before window functions were widely supported, this used to require a correlated subquery instead, like `SELECT order_date, (SELECT SUM(daily_revenue) FROM daily_sales d2 WHERE d2.order_date <= d1.order_date) AS running_total FROM daily_sales d1`, which works but is noticeably less efficient and less readable on large datasets. I'd always reach for the window function version today unless I'm working in a genuinely legacy system that doesn't support them.

**38. [M] What is a derived table (a subquery in the FROM clause), and how is it different from a CTE?**

> A derived table is a subquery placed inside the FROM clause and given an alias, effectively treated as a temporary, inline table for the rest of that query — for example, `SELECT dept, avg_salary FROM (SELECT department AS dept, AVG(salary) AS avg_salary FROM employees GROUP BY department) AS dept_summary WHERE avg_salary > 70000`. Functionally, this is very similar to what a CTE accomplishes, but there are a few practical differences: a CTE is defined once at the top of the query using `WITH` and can be referenced multiple times later in that same query, while a derived table has to be rewritten in full each time you need to reuse that same logic elsewhere in the query. CTEs also tend to read more clearly top-to-bottom, especially when chaining multiple logical steps, whereas deeply nested derived tables can get visually harder to follow once you're several layers deep. I'd generally reach for a CTE for anything beyond a simple, single-use case, purely for readability, though under the hood many modern query optimizers treat them nearly identically in terms of execution.

## 5. Window Functions (Q39–47)

**39. [M] What are window functions, and how are they different from GROUP BY aggregations (they don't collapse rows)?**

> A window function performs a calculation across a set of related rows, called a "window," without collapsing those rows into a single output row the way a GROUP BY aggregation does. With GROUP BY, if I aggregate salary by department, I get back one row per department. With a window function, like `AVG(salary) OVER (PARTITION BY department)`, I still get back one row per employee, but each row now also shows that employee's department average alongside their own individual salary. This is the real practical difference: GROUP BY reduces the number of rows, window functions preserve every original row while adding calculated context from a group. This makes window functions perfect for things GROUP BY genuinely can't do cleanly, like comparing an individual employee's salary to their department's average in the same row, or ranking each row within a group, without losing the row-level detail.

**40. [M] What is the difference between RANK(), DENSE_RANK(), and ROW_NUMBER()? Give an example where they'd return different results.**

> All three assign a position to each row based on an ORDER BY within a window, but they handle ties differently. `ROW_NUMBER()` assigns a strictly unique, sequential number to every row regardless of ties, so two rows with identical values still get different numbers, like 1, 2, 3, 4. `RANK()` gives tied rows the same rank, but then skips the next rank number(s) to account for the tie, like 1, 2, 2, 4 — notice rank 3 is skipped because two rows tied for 2nd. `DENSE_RANK()` also gives tied rows the same rank, but doesn't skip any numbers afterward, like 1, 2, 2, 3. For example, imagine three salespeople tied for the highest sales this month, each with the exact same total: `ROW_NUMBER()` would arbitrarily assign them 1, 2, 3 even though they're tied, `RANK()` would give all three rank 1, and the 4th place person would be rank 4, and `DENSE_RANK()` would give all three rank 1, but the 4th place person would be rank 2. Picking the right one really depends on the business question — if I'm picking exactly "the top 3 salespeople" and ties genuinely shouldn't all count, `ROW_NUMBER()` forces a tiebreaker; if I want every tied person to legitimately share the top rank, `RANK()` or `DENSE_RANK()` is more honest to the data.

**41. [M] What does the OVER() clause do, and what is PARTITION BY used for within it?**

> `OVER()` is what turns a regular function into a window function — it defines the "window" of rows the function should operate across. Without any arguments, `OVER()` treats the entire result set as one single window. `PARTITION BY` inside the `OVER()` clause divides that window into smaller groups, similar conceptually to GROUP BY, but again without collapsing rows — the calculation restarts independently for each partition. For example, `RANK() OVER (PARTITION BY department ORDER BY salary DESC)` ranks employees by salary, but resets that ranking separately within each department, so every department gets its own independent #1, #2, #3, rather than one single company-wide ranking. I think of `PARTITION BY` as saying "run this calculation separately for each of these groups," while still returning results at the individual row level.

**42. [M] What are LAG() and LEAD() used for? Give a real use case (e.g., month-over-month growth, comparing a user's current and previous purchase).**

> `LAG()` retrieves a value from a previous row within the window, and `LEAD()` retrieves a value from a following row, both without needing a self-join. A very common real use case is calculating month-over-month growth:
> ```sql
> SELECT month, revenue,
>        LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
>        revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change
> FROM monthly_revenue
> ORDER BY month;
> ```
> Here, `LAG(revenue)` pulls in the previous row's revenue value directly alongside the current row, letting me compute the change without a self-join. Another practical use is comparing a user's current purchase to their previous purchase to calculate the time gap between orders, which is a common building block for churn or repeat-purchase analysis: `LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)` would give each customer's previous order date lined up right next to their current one, letting me subtract the two directly.

**43. [A] Write a query using a window function to calculate a 7-day moving average of daily sales.**

> ```sql
> SELECT sale_date, daily_sales,
>        AVG(daily_sales) OVER (
>          ORDER BY sale_date
>          ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
>        ) AS moving_avg_7day
> FROM daily_sales
> ORDER BY sale_date;
> ```
> The key part is the frame clause, `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`, which tells the window function to look at the current row plus the 6 rows immediately before it, giving exactly a 7-day window, and slide that window forward one row at a time as `sale_date` advances. One thing I'd flag as an edge case: for the first 6 days of data, this will average over fewer than 7 actual days, since there simply aren't 6 prior rows yet — depending on the business need, I might filter those early partial-window rows out, or explicitly note in the output that the average isn't a "true" 7-day figure until enough history has accumulated.

**44. [A] How would you find the top 3 highest-paid employees per department using a window function?**

> ```sql
> WITH ranked_employees AS (
>   SELECT name, department, salary,
>          DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
>   FROM employees
> )
> SELECT name, department, salary
> FROM ranked_employees
> WHERE salary_rank <= 3;
> ```
> I partition by department so the ranking restarts fresh for each one, order by salary descending so rank 1 is the highest earner, and then filter to rank 3 or below in an outer query, since window functions can't be filtered directly in the same SELECT they're defined in — that's exactly why I wrap it in a CTE first. I specifically chose `DENSE_RANK()` here over `ROW_NUMBER()` deliberately: if two employees in the same department are tied for 2nd highest salary, I'd want the business logic to genuinely reflect that both count as "top 3," rather than arbitrarily kicking one of them out just because `ROW_NUMBER()` forces a strict tiebreaker — though I'd double check with the interviewer or stakeholder which behavior they actually want, since that's a legitimate judgment call depending on the use case.

**45. [M] What is the difference between ROWS BETWEEN and RANGE BETWEEN in a window frame clause?**

> Both define the frame, or subset of rows, that a window function operates over, but they define that boundary differently. `ROWS BETWEEN` defines the frame based on physical row position, counting a fixed number of actual rows before or after the current one, regardless of whether their values are tied. `RANGE BETWEEN` defines the frame based on the logical value in the ORDER BY column, meaning if multiple rows share the same ordering value, they're all treated as part of the same "current" position and included together, rather than counted as separate row positions. This distinction mostly shows up with tied values — if I have `RANGE BETWEEN` on an `ORDER BY sale_date`, and three rows happen to share the exact same date, RANGE would treat all three of those tied rows as a single point in the frame, while ROWS would count them as three distinct row positions. In practice, `ROWS BETWEEN` is what I reach for by far most often, like in the moving average example, since I typically do want a fixed physical row count, but it's worth knowing RANGE exists for the cases where logical value grouping actually matters more than raw row position.

**46. [A] Write a query to identify the first and last purchase date for each customer using window functions, without using GROUP BY.**

> ```sql
> SELECT DISTINCT customer_id,
>        FIRST_VALUE(order_date) OVER (
>          PARTITION BY customer_id ORDER BY order_date
>        ) AS first_purchase_date,
>        FIRST_VALUE(order_date) OVER (
>          PARTITION BY customer_id ORDER BY order_date DESC
>        ) AS last_purchase_date
> FROM orders;
> ```
> Here I use `FIRST_VALUE()` twice with opposite sort orders within the window — ascending to get the earliest date, descending to get the latest — both partitioned by customer so each customer's calculation is independent. Since this returns one row per original order, not one row per customer, I wrap the whole thing in `DISTINCT` at the outer level to collapse it down to one row per customer with both dates attached. An alternative approach would be `MIN(order_date) OVER (PARTITION BY customer_id)` and `MAX(order_date) OVER (PARTITION BY customer_id)`, which honestly reads a bit more simply for this specific case — I'd probably actually prefer that version in real work, but `FIRST_VALUE`/`LAST_VALUE` are worth knowing since they generalize to cases where you want the first or last row's other columns too, not just a min/max of one column.

**47. [A] How would you use window functions to detect consecutive/streak behavior — e.g., users who logged in on 3 or more consecutive days?**

> This is a classic "gaps and islands" problem, and the elegant trick is combining `ROW_NUMBER()` with date arithmetic to identify unbroken streaks:
> ```sql
> WITH numbered_logins AS (
>   SELECT user_id, login_date,
>          ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS rn
>   FROM (SELECT DISTINCT user_id, login_date FROM logins) t
> ),
> streak_groups AS (
>   SELECT user_id, login_date,
>          DATE_SUB(login_date, INTERVAL rn DAY) AS streak_group
>   FROM numbered_logins
> )
> SELECT user_id, streak_group, COUNT(*) AS streak_length,
>        MIN(login_date) AS streak_start, MAX(login_date) AS streak_end
> FROM streak_groups
> GROUP BY user_id, streak_group
> HAVING COUNT(*) >= 3;
> ```
> The trick here is genuinely clever the first time you see it: for consecutive dates, subtracting a steadily increasing row number from each date produces the exact same resulting "anchor" date for every row within one unbroken streak — since both the date and the row number increase by 1 together on consecutive days, that difference stays constant. The moment there's a gap in login dates, the row number keeps incrementing normally but the date jumps ahead more, so the subtracted result changes, naturally creating a new group. I then just group by that computed `streak_group` value per user and filter for groups with 3 or more rows in them. This pattern comes up constantly in retention and engagement analysis — "streak" features on apps like Duolingo or Snapchat are built on exactly this kind of underlying query logic.

## 6. Database Design, Normalization & Modeling (Q48–56)

**48. [F] What is normalization? Explain 1NF, 2NF, and 3NF with a simple example.**

> Normalization is the process of organizing a database's tables and columns to reduce data redundancy and prevent update anomalies, by progressively splitting data into related tables based on a set of rules called normal forms. First Normal Form, 1NF, requires that each column holds only atomic, indivisible values, and there are no repeating groups — for example, instead of a `phone_numbers` column storing "555-1234, 555-5678" as one comma-separated string, each phone number should be its own row, or in its own properly structured column. Second Normal Form, 2NF, requires 1NF plus that every non-key column depends on the entire primary key, not just part of it — this only really matters when you have a composite key; for example, in an `order_items` table with a composite key of `order_id` and `product_id`, a column like `product_name` shouldn't live there, since it only depends on `product_id`, not the full composite key, so it belongs in a separate `products` table instead. Third Normal Form, 3NF, requires 2NF plus that non-key columns don't depend on other non-key columns — for example, if an `orders` table stores both `customer_id` and `customer_email`, that's a problem, since `customer_email` actually depends on `customer_id`, not directly on the order itself; it should live in the `customers` table instead. The overall goal across all three is the same: each piece of information should be stored in exactly one place, so updating it doesn't require hunting down and fixing multiple duplicated copies.

**49. [M] What is denormalization, and why would a data warehouse intentionally denormalize data that an OLTP system wouldn't?**

> Denormalization is the deliberate process of combining data back together, sometimes duplicating it, to reduce the number of joins needed for a query, trading some redundancy for read performance. An OLTP system avoids this because it prioritizes fast, consistent writes on individual transactions and wants to avoid update anomalies — normalization keeps that data clean and consistent. A data warehouse, on the other hand, is optimized almost entirely for read-heavy analytical queries across huge volumes of historical data, and joins across many normalized tables can get slow at that scale, so warehouses often intentionally denormalize — for example, storing a customer's region and segment directly on a sales fact table, rather than requiring analysts to join out to a separate customers table every single time they want to slice revenue by region. The trade-off is accepted specifically because warehouse data is typically not being updated in real time the way transactional data is — it's usually loaded in batches, so the redundancy risk that normalization protects against in OLTP systems matters much less here, while the read-speed benefit matters a lot more.

**50. [M] What is the difference between OLTP and OLAP systems? Give a real example of each.**

> OLTP, Online Transaction Processing, systems are designed for fast, real-time, small transactions — lots of individual reads and writes, like a single customer placing an order or updating their profile. A real example is the live database behind an e-commerce checkout page, handling one order at a time as customers actively shop. OLAP, Online Analytical Processing, systems are designed for complex analytical queries across large volumes of historical data — fewer, heavier queries rather than many small ones, typically read-only or batch-loaded rather than continuously transactional. A real example is a company's Snowflake or BigQuery data warehouse, where an analyst runs a query aggregating a full year of sales data across millions of rows to build a quarterly revenue dashboard. As an analyst, I almost always work against OLAP systems day to day, since OLTP databases are typically shielded from direct analytical querying both for performance reasons and because their schema is normalized for transactions rather than shaped for easy reporting.

**51. [M] What is a star schema vs. a snowflake schema? Which is more common in analytics/BI systems and why?**

> A star schema has one central fact table, containing measurable, quantitative data like sales amount or quantity, surrounded by denormalized dimension tables, like customer, product, and date, each directly connected to the fact table — visually it looks like a star, with the fact table in the middle. A snowflake schema takes those same dimension tables and further normalizes them into sub-dimensions — for example, splitting a `product` dimension into `product` and a separate `product_category` table, instead of keeping category as a flat column directly on `product`. Star schemas are generally more common in analytics and BI systems, mainly because they require fewer joins to answer a typical business question, which makes queries simpler to write and often faster to run, and BI tools like Tableau or Power BI tend to work more intuitively against a flatter, star-shaped structure. Snowflake schemas save some storage space through normalization and can better represent complex hierarchical relationships, but that comes at the cost of extra joins, so I'd generally lean toward a star schema by default for an analytics-focused warehouse unless there's a specific, strong reason for the added normalization.

**52. [F] What is the difference between a fact table and a dimension table?**

> A fact table stores the quantitative, measurable events or transactions — things like sales amount, quantity sold, or number of clicks — and typically consists mostly of foreign keys pointing to dimension tables, plus numeric measure columns, and tends to be the largest, fastest-growing table in a warehouse since a new row gets added for every new transaction or event. A dimension table stores the descriptive, contextual attributes that give meaning to those facts — things like customer name, product category, or store location — and is typically much smaller and more slowly changing than the fact table. A concrete example: in a retail sales schema, the fact table would have one row per sale with columns like `product_id`, `customer_id`, `date_id`, `quantity`, and `revenue`, while separate `products`, `customers`, and `date` dimension tables would each hold the full descriptive details that the fact table's foreign keys point back to.

**53. [M] What is a surrogate key vs. a natural key, and why do data warehouses often prefer surrogate keys?**

> A natural key is an identifier that already exists in the real-world business data itself, like a social security number, an email address, or a product SKU. A surrogate key is an artificially generated identifier, usually a simple auto-incrementing integer, that has no business meaning and exists purely to uniquely identify a row within the database. Data warehouses often prefer surrogate keys for a few practical reasons: natural keys can occasionally change over time, like a customer updating their email address, or can be reused or reassigned by a source system in ways that would break historical consistency in the warehouse; natural keys are sometimes composite or inconsistently formatted across different source systems being merged together, while a surrogate key is always a clean, simple, single-column integer; and surrogate keys join and index more efficiently than long text-based natural keys. This becomes especially important with slowly changing dimensions, which we're about to get into — a surrogate key lets the warehouse track multiple historical versions of the same real-world entity, like a customer who changed addresses, as separate rows, something a natural key alone can't cleanly support.

**54. [A] What is a slowly changing dimension (SCD), and what's the difference between Type 1, Type 2, and Type 3 SCDs?**

> A slowly changing dimension is a dimension whose attribute values change over time but not on every single load — like a customer's address, a product's category, or an employee's job title — and an SCD strategy defines how the warehouse should handle and preserve, or not preserve, that history. Type 1 simply overwrites the old value with the new one, keeping no history at all — this is the simplest approach, appropriate when historical accuracy genuinely doesn't matter, like correcting a typo in a customer's name. Type 2 preserves full history by inserting a new row every time an attribute changes, typically with `effective_date`, `end_date`, and a `current_flag` column, so past facts still correctly link back to the dimension values that were true at that point in time — this is the most common approach in real analytics warehouses, since it lets you accurately answer "what was this customer's segment when they made this specific purchase," not just what it is today. Type 3 keeps limited history by adding a new column to track just the previous value alongside the current one, like `current_region` and `previous_region` — it's a middle ground, but it only preserves one prior state, not a full unlimited history, so it's used less often than Type 2 in practice. For most real-world analytics needs, I'd default to Type 2, since losing historical accuracy on dimension changes can quietly corrupt trend and cohort analysis without anyone immediately noticing.

**55. [M] How would you design a schema to track user event data (e.g., clicks, purchases) for a product analytics team?**

> I'd design this around a fairly standard event-tracking pattern: a central `events` fact table with one row per event, containing columns like `event_id`, `user_id`, `event_type` (click, purchase, page_view, etc.), `event_timestamp`, and a flexible `properties` column, often stored as JSON, to capture event-specific details without needing a rigid, ever-expanding schema for every possible event type. I'd keep a separate `users` dimension table with relatively stable user attributes like signup date and plan type, joined in by `user_id` when needed for analysis. For high-volume event data specifically, I'd also strongly consider partitioning the events table by date, since most product analytics queries filter by a specific time range, and that partitioning lets the warehouse skip scanning irrelevant historical partitions entirely. I'd also think early about deduplication — client-side event tracking is notoriously prone to sending duplicate events on retries or app reloads, so I'd want a reliable event ID and a deduplication step in the pipeline before this data ever reaches analysts, rather than trying to catch duplicates after the fact in every downstream query.

**56. [A] What is data partitioning, and how would you decide what column to partition a large events table on (e.g., date-based partitioning)?**

> Partitioning splits a large table into smaller, physically separate chunks based on the value of one or more columns, so queries that filter on that partitioning column only need to scan the relevant partitions instead of the entire table — this can dramatically speed up queries and reduce compute cost on very large tables. For an events table specifically, date-based partitioning, like partitioning by day or month, is almost always the right default, because the overwhelming majority of analytics queries against event data filter by a time range — "show me last week's activity" — so the query engine can skip scanning months or years of irrelevant historical partitions entirely. I'd choose the partition column based on the most common filter pattern in actual queries against that table, not just what seems logically important — if 90% of queries filter by date but almost none filter by, say, `event_type`, partitioning by date delivers far more real-world performance benefit. I'd also think about partition granularity — daily partitions work well for a table with heavy daily query volume and steady ingestion, but if the table is smaller or queries typically span multi-month ranges anyway, monthly partitions might actually be more efficient by avoiding an excessive number of tiny partition files.

## 7. Indexing, Query Optimization & Performance (Q57–65)

**57. [F] What is an index, and how does it speed up query performance?**

> An index is a separate data structure, most commonly a B-tree, built on one or more columns of a table, that lets the database find rows matching a condition without scanning every single row in the table. Without an index, a query filtering on a specific column has to perform a full table scan, checking every row one by one — on a table with millions of rows, that's slow. With an index, the database can navigate directly to the relevant rows much faster, similar to how a book's index lets you jump straight to the right page instead of reading the entire book to find a topic. A concrete example: if I frequently query `WHERE customer_id = 12345` on a huge orders table, adding an index on `customer_id` means the database can locate matching rows almost instantly, rather than scanning potentially millions of unrelated rows every single time that query runs.

**58. [M] What is the difference between a clustered index and a non-clustered index?**

> A clustered index actually determines the physical order in which data is stored on disk — the table's rows themselves are physically sorted according to that index's key, which is why a table can have only one clustered index, since data can only be physically arranged in one order at a time. A non-clustered index is a separate structure entirely from the actual data, containing the indexed column's values plus a pointer back to where the actual row lives — a table can have many non-clustered indexes. Because a clustered index directly determines the row order, retrieving data via a clustered index tends to be a bit faster for range queries and full-row lookups, since the data is already physically contiguous. A non-clustered index adds one extra lookup step, using that pointer to go find the full row, unless the query can be satisfied entirely from the index itself, which is called a "covering index." A practical example: the primary key often becomes the clustered index by default in many database systems, physically ordering the table by that key, while I might add a non-clustered index separately on, say, an `email` column, since I frequently search by email but it's not what physically orders the table.

**59. [M] What are the trade-offs of adding an index (read speed vs. write/insert speed, storage overhead)?**

> Indexes speed up reads, specifically queries that filter, join, or sort on the indexed column, but they come with real costs. Every INSERT, UPDATE, or DELETE on the table also has to update every index that includes the affected column, which means write operations get slower as you add more indexes — the database isn't just writing to the table anymore, it's also maintaining every index structure tied to it. Indexes also consume additional storage space, sometimes significant amounts on very large tables, since they're essentially a duplicated, reorganized copy of the indexed column's data plus pointers. Because of this trade-off, I wouldn't just index every column defensively — I'd specifically add indexes on columns that are frequently used in WHERE clauses, JOIN conditions, or ORDER BY clauses on large tables, and I'd be more cautious about over-indexing tables that see heavy, frequent write traffic, like a high-volume transactional logging table, where the write-side cost of maintaining many indexes could genuinely outweigh the read-side benefit.

**60. [A] How would you use EXPLAIN or EXPLAIN ANALYZE to diagnose a slow query? What are you looking for in the output (full table scan, index usage, join order)?**

> `EXPLAIN` shows the query planner's intended execution plan without actually running the query, while `EXPLAIN ANALYZE` actually executes the query and shows the real, measured execution stats alongside the plan, which is more useful for genuinely diagnosing a slow query rather than just guessing at the plan. When I look at the output, I'm specifically checking a few things: whether the query is doing a "sequential scan" or "full table scan" on a large table where I'd expect an index to be used instead, which usually points to a missing index or a query written in a way that prevents the index from being used, like wrapping the indexed column in a function; the join order and join type being used, since a poorly ordered join, or the wrong join strategy for the data size, can massively inflate execution time; and the estimated versus actual row counts at each step, since a large mismatch between what the planner expected and what actually happened usually means the table's statistics are stale and need to be refreshed, which can mislead the optimizer into choosing a bad plan. In practice, if I see a full table scan on a huge table where I expected an indexed lookup, my first suspicion is either a genuinely missing index, or that I accidentally wrote the WHERE clause in a way that prevents the existing index from being used at all — like applying a function directly to the indexed column, such as `WHERE YEAR(order_date) = 2026` instead of a proper date range comparison.

**61. [M] What is a composite index, and does column order in the index matter? Why?**

> A composite index is an index built across multiple columns together, rather than just one. Column order absolutely matters, because a composite index is only efficiently usable, from the left, for queries that filter on a matching prefix of those columns, similar conceptually to how a phone book sorted by last name then first name lets you efficiently search by last name alone, but doesn't help much if you only know someone's first name. For example, a composite index on `(department, salary)` would efficiently speed up a query filtering on just `department`, or on `department` and `salary` together, but it generally wouldn't help much for a query that filters only on `salary` without also filtering on `department`, since the index is sorted primarily by department first. Because of this, when designing a composite index, I'd generally put the column that's most frequently filtered alone, or that has higher selectivity, meaning it narrows the result set down the most, first in the index, and add secondary columns after it based on how queries actually combine those filters in practice.

**62. [A] What are common causes of a slow SQL query, and how would you go about optimizing one (indexing, avoiding SELECT *, filtering early, avoiding functions on indexed columns)?**

> There are several common culprits I'd check, roughly in order of how often I actually run into them. Missing or ineffective indexing is often the first suspect — checking `EXPLAIN` for full table scans on large tables is usually my starting point. Using `SELECT *` instead of only the specific columns actually needed forces the database to retrieve more data than necessary, and also prevents the possibility of a fully "covering" index satisfying the query entirely from the index itself. Applying a function directly to an indexed column in the WHERE clause, like `WHERE UPPER(email) = 'JOHN@EMAIL.COM'`, typically prevents the database from using an index on that column at all, since it now has to compute that function for every single row before it can compare — rewriting it to filter on the raw, unmodified column when possible, or using a functional index specifically built for that transformation, usually solves this. Filtering as early as possible, meaning applying WHERE conditions before an expensive join or subquery rather than after, reduces the amount of data the database has to process at every subsequent step. And inefficient joins — joining on non-indexed or mismatched-type columns, or accidentally producing a much larger intermediate result than expected — are another very common cause. In practice, my actual optimization process is: run `EXPLAIN ANALYZE` first rather than guessing, identify the specific expensive step in the plan, and only then make a targeted fix rather than trying random changes and hoping something helps.

**63. [M] What is query caching, and how can materialized views help with expensive, frequently-run aggregate queries?**

> Query caching stores the result of a previously run query so that if the exact same query is run again, the database can return the cached result instantly instead of recomputing it from scratch — this works well for queries that run frequently and whose underlying data doesn't change often between runs. A materialized view is a related but distinct concept: it's a database object that stores the actual computed result of a query physically on disk, similar to a regular table, rather than recalculating it live every single time it's queried the way a standard view does. This is especially useful for expensive aggregate queries that get run repeatedly by many people, like a company-wide daily revenue summary that dozens of dashboards all reference — instead of every single dashboard load recomputing that expensive aggregation from raw transaction data, the materialized view computes it once, on a schedule, and every downstream query just reads that pre-computed result almost instantly. The trade-off is that materialized view data can become stale between refreshes, so I'd need to decide on an appropriate refresh schedule, like nightly or hourly, based on how fresh the business actually needs that specific data to be.

**64. [A] What is the N+1 query problem, and how does it typically show up in analytics/reporting pipelines?**

> The N+1 query problem happens when code runs one initial query to fetch a list of N items, and then runs a separate additional query for each of those N items individually to fetch related data, resulting in N+1 total queries instead of a single, efficient combined query. In an analytics or reporting context, this often shows up in application or dashboard code, or in poorly designed ETL scripts — for example, fetching a list of 500 customers with one query, and then looping through each customer in application code, individually querying their order history one customer at a time, resulting in 501 total database round trips instead of one single JOIN query that pulls customers and their orders together at once. This is a real performance killer specifically because each individual query has its own network round-trip overhead, and that overhead adds up multiplicatively as N grows — the fix is almost always to restructure the logic into a single JOIN, or a single query using `WHERE customer_id IN (...)` with a batch of IDs, rather than looping and querying one at a time.

**65. [A] How would you approach writing a query against a table with billions of rows without timing out or scanning the whole table?**

> A few strategies I'd combine here. First, I'd make sure I'm always filtering on a partitioned or indexed column, ideally date, if the table is partitioned by date, so the query engine can skip the vast majority of irrelevant data before doing any real work — querying "last 7 days" against a date-partitioned table should only scan 7 days worth of partitions, not the entire billion-row history. Second, I'd avoid `SELECT *` and only pull the specific columns actually needed, especially important in columnar warehouses like BigQuery or Redshift, where reading fewer columns directly reduces the amount of data scanned and, often, the literal compute cost billed. Third, I'd consider pre-aggregating with a materialized view or a scheduled summary table if the same expensive aggregation is going to be queried repeatedly, rather than recomputing it from raw billion-row data every single time. Fourth, for genuinely exploratory analysis where I don't need the full precision yet, I'd consider querying a sampled subset of the data first to validate my logic is correct before running the full, expensive version against the complete dataset. And finally, I'd run `EXPLAIN` before executing anything against a table that size, specifically to catch an accidental full scan or an unindexed join before it actually runs and potentially times out or burns a large amount of compute cost.

## 8. Transactions, ACID & Data Integrity (Q66–70)

**66. [F] What does ACID stand for, and why does it matter for a database?**

> ACID stands for Atomicity, Consistency, Isolation, and Durability, and it's the set of guarantees that make transactional databases trustworthy. Atomicity means a transaction either fully completes or fully fails — there's no partial state where only some of its changes went through, like a bank transfer where money leaves one account but somehow never arrives in the other. Consistency means a transaction takes the database from one valid state to another valid state, never violating defined rules like constraints or data types along the way. Isolation means concurrent transactions don't interfere with each other in ways that produce incorrect results, even if many transactions are running at the same time. Durability means once a transaction is committed, it's permanently saved, even if the system crashes immediately afterward. These properties matter because without them, a database couldn't be trusted for anything involving money, inventory, or any other data where partial or inconsistent updates would be genuinely unacceptable — a real-world example is a money transfer between two bank accounts, where atomicity specifically guarantees you never end up with money vanishing from one account without correctly appearing in the other, even if the system crashes mid-transaction.

**67. [M] What is a transaction, and what do COMMIT and ROLLBACK do?**

> A transaction is a sequence of one or more SQL statements that are executed together as a single logical unit of work, following the ACID guarantees just described. `COMMIT` permanently saves all the changes made during that transaction to the database, making them visible to other users and durable even through a crash. `ROLLBACK` undoes all the changes made during that transaction, reverting the database back to the state it was in before the transaction began, as if none of those statements had ever run. A practical example: if I'm updating inventory counts across multiple product rows as part of processing an order, I'd wrap those updates in a transaction — if something fails partway through, like an unexpected constraint violation on one of the rows, I can `ROLLBACK` the entire transaction, ensuring I never end up with some inventory counts updated and others not, which would leave the data in an inconsistent, partially-updated state.

**68. [M] What are the different transaction isolation levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable), and what problem does each solve?**

> Isolation levels control how much one transaction is allowed to see the effects of other concurrently running transactions, and there's a real trade-off between consistency and performance/concurrency as you move up each level. Read Uncommitted is the loosest level — a transaction can see uncommitted changes made by other transactions, which allows dirty reads and is rarely used in practice due to the risk of reading data that might get rolled back. Read Committed only allows a transaction to see changes that have already been committed by other transactions, preventing dirty reads, and is the default isolation level in many databases like PostgreSQL and SQL Server, offering a reasonable balance of consistency and performance for most everyday use. Repeatable Read guarantees that if a transaction reads the same row twice, it will see the same data both times, even if another transaction commits a change to that row in between, preventing non-repeatable reads. Serializable is the strictest level, guaranteeing transactions behave as if they ran one at a time in some serial order, preventing all anomalies including phantom reads, but at the cost of the most reduced concurrency and potential performance impact, since it may require more locking or blocking between transactions. I'd choose a stricter isolation level when data correctness is absolutely critical, like financial reporting, and accept a more relaxed level for less critical, high-concurrency workloads where a small amount of read inconsistency is an acceptable trade-off for better performance.

**69. [M] What is a dirty read, a non-repeatable read, and a phantom read?**

> These are the three classic concurrency anomalies that isolation levels are specifically designed to prevent, and I find it easiest to describe them chronologically as they'd actually happen. A dirty read happens when one transaction reads data that another transaction has changed but not yet committed — if that second transaction then rolls back, the first transaction has now acted on data that never actually, truly existed. A non-repeatable read happens when a transaction reads the same row twice, but gets different values each time, because another transaction updated and committed a change to that row in between the two reads — the data was valid both times, but it didn't stay "repeatable" within the same transaction. A phantom read happens when a transaction runs the same query twice, and a different set of rows is returned the second time, because another transaction inserted or deleted rows matching that query's condition in between — this is subtly different from a non-repeatable read, since it's about new or missing rows appearing, not an existing row's values changing. A concrete example that ties all three together: imagine I'm running a report that sums up "all pending orders" twice within the same transaction — a dirty read could show me an order that later gets cancelled and rolled back, a non-repeatable read could show me a different total the second time because an existing order's status was updated in between, and a phantom read could show me a different total because a brand new order was inserted in between my two queries.

**70. [A] In a data pipeline context, how do you ensure idempotency when a job might re-run and re-insert the same data (e.g., using UPSERT/MERGE or deduplication logic)?**

> Idempotency means running the same operation multiple times produces the same end result as running it once — which matters enormously in data pipelines, since jobs fail and get retried constantly, whether due to transient network issues, timeouts, or manual reruns, and I never want a retry to silently duplicate data. A few practical approaches I'd use: first, using `UPSERT` or `MERGE` statements instead of plain `INSERT`, which checks whether a row with a given key already exists and either updates it in place or inserts it fresh, rather than blindly inserting a new row every time regardless of whether it already exists. Second, designing the pipeline around a unique, stable identifier for each record, like an `event_id`, so that even if the exact same batch of data gets processed twice, a deduplication step, like `INSERT ... ON CONFLICT (event_id) DO NOTHING`, ensures the second run has no actual effect on the final data. Third, for full-table loads rather than incremental ones, sometimes the cleanest approach is fully replacing a specific partition, like truncating and reloading just that day's data, rather than appending, so a rerun for the same day always produces exactly the same final result regardless of how many times that job happens to execute. In practice, I always think about idempotency at design time for any pipeline, since retrofitting deduplication logic into a pipeline that's already been silently double-inserting data for months is a much more painful and error-prone cleanup than building it in correctly from the very start.

## 9. Advanced / Scenario-Based & Company-Style Questions (Q71–75)

**71. [A] Write a query to calculate month-over-month retention — e.g., what percentage of users active in month N were also active in month N+1.**

> ```sql
> WITH monthly_activity AS (
>   SELECT DISTINCT user_id,
>          DATE_TRUNC('month', activity_date) AS activity_month
>   FROM user_events
> ),
> retention_pairs AS (
>   SELECT curr.activity_month AS base_month,
>          COUNT(DISTINCT curr.user_id) AS active_users,
>          COUNT(DISTINCT next.user_id) AS retained_users
>   FROM monthly_activity curr
>   LEFT JOIN monthly_activity next
>     ON curr.user_id = next.user_id
>    AND next.activity_month = curr.activity_month + INTERVAL '1 month'
>   GROUP BY curr.activity_month
> )
> SELECT base_month, active_users, retained_users,
>        ROUND(100.0 * retained_users / active_users, 1) AS retention_rate_pct
> FROM retention_pairs
> ORDER BY base_month;
> ```
> The logic: first, I collapse raw events down to one row per user per month, since a user might have many events in the same month, but for retention I just care whether they were active at all that month. Then I self-join that monthly activity table to itself, matching each user's current month against exactly the following month, using a LEFT JOIN specifically so users who churned, meaning they don't appear in the next month, still get counted in the denominator rather than disappearing entirely. Finally, I compute the retention rate as retained users divided by that month's total active users. I'd always sanity-check the output makes sense — retention rates should generally be between 0 and 100%, and I'd double check the most recent month in the data, since it will artificially show low or zero retention just because the "next month" hasn't fully happened yet in the dataset.

**72. [A] Write a query to build a purchase funnel — e.g., number of users who viewed a product, added to cart, and completed checkout, and the drop-off rate at each stage. (Classic Meta/Amazon-style product analytics question.)**

> ```sql
> WITH funnel_stages AS (
>   SELECT
>     COUNT(DISTINCT CASE WHEN event_type = 'view' THEN user_id END) AS viewed,
>     COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS added_to_cart,
>     COUNT(DISTINCT CASE WHEN event_type = 'checkout_complete' THEN user_id END) AS purchased
>   FROM product_events
>   WHERE event_date BETWEEN '2026-01-01' AND '2026-01-31'
> )
> SELECT viewed, added_to_cart, purchased,
>        ROUND(100.0 * added_to_cart / NULLIF(viewed, 0), 1) AS view_to_cart_pct,
>        ROUND(100.0 * purchased / NULLIF(added_to_cart, 0), 1) AS cart_to_purchase_pct,
>        ROUND(100.0 * purchased / NULLIF(viewed, 0), 1) AS overall_conversion_pct
> FROM funnel_stages;
> ```
> This version answers "how many distinct users reached each stage at all," which is the most common framing of this question. A more rigorous version would enforce that each stage actually happened in the correct sequential order for that specific user — for example, someone could theoretically have an `add_to_cart` event without a preceding `view` event logged, due to a tracking gap, and I wouldn't want that to artificially inflate the funnel; I'd handle that with a window function checking that each stage's timestamp for a user comes after the previous stage's timestamp, using something like `MIN(view_time) < MIN(add_to_cart_time)` per user before counting them into that stage. I used `NULLIF(viewed, 0)` specifically to avoid a divide-by-zero error if a given time period happened to have zero views. I'd always mention to the interviewer that in a real production version of this, I'd also want to validate whether the drop-off numbers pass a basic sanity check, and think carefully about the exact business definition of each funnel stage before trusting the output, since ambiguous event definitions are a very common source of funnel analysis errors in real companies.

**73. [A] Given a table of user sign-ups and a table of transactions, write a query to calculate weekly cohort revenue — total revenue generated by each sign-up cohort in each subsequent week.**

> ```sql
> WITH cohorts AS (
>   SELECT user_id, DATE_TRUNC('week', signup_date) AS cohort_week
>   FROM signups
> ),
> weekly_revenue AS (
>   SELECT c.cohort_week,
>          DATE_TRUNC('week', t.transaction_date) AS revenue_week,
>          SUM(t.amount) AS revenue
>   FROM cohorts c
>   JOIN transactions t ON c.user_id = t.user_id
>   GROUP BY c.cohort_week, DATE_TRUNC('week', t.transaction_date)
> )
> SELECT cohort_week, revenue_week,
>        DATEDIFF('week', cohort_week, revenue_week) AS weeks_since_signup,
>        revenue
> FROM weekly_revenue
> ORDER BY cohort_week, revenue_week;
> ```
> The approach: first, I bucket every user into a weekly sign-up cohort based on when they joined. Then I join their transactions and bucket each transaction into its own week, and sum revenue by the combination of cohort and transaction week. The `weeks_since_signup` column, computed as the difference between the cohort week and each revenue week, is what actually makes this genuinely useful for cohort analysis, since it lets me compare, say, "week 4 revenue" across many different cohorts on equal footing, rather than comparing raw calendar weeks, which wouldn't be a fair comparison between an older cohort and a newer one. In a BI tool, this output would typically get pivoted so `weeks_since_signup` becomes columns and `cohort_week` becomes rows, producing the classic triangular cohort revenue table that product and growth teams use constantly to evaluate whether newer cohorts are monetizing better or worse than older ones over the same relative time period.

**74. [A] How would you detect and handle duplicate or fraudulent transactions in a raw events table before it feeds into a revenue dashboard?**

> I'd approach this in a few layers. First, for straightforward duplicate detection, I'd look for transactions with an identical `transaction_id` appearing more than once, which usually points to a pipeline retry or an upstream logging bug rather than genuinely separate transactions — a simple `GROUP BY transaction_id HAVING COUNT(*) > 1` check surfaces those immediately. Second, for less obvious duplicates that don't share an exact transaction ID, like the same user, same amount, and same product within an unreasonably short time window, say under a few seconds, I'd use a window function like `LAG()` partitioned by user, ordered by timestamp, to flag suspiciously close consecutive transactions with identical amounts, which often indicates a double-click or a client-side retry rather than two legitimately separate purchases. Third, for actual fraud detection specifically, I'd look at statistical outliers and known fraud patterns — an unusually high transaction count from a single user or IP address in a short window, transaction amounts far outside a user's normal historical range, or a burst of transactions from a new account with no prior history, all common signals worth flagging for a closer review rather than necessarily auto-excluding, since I wouldn't want to silently strip real revenue based on an imperfect heuristic without a human review step somewhere in that process. Before any of this touches an actual revenue dashboard, I'd build these checks as a validation layer in the pipeline itself, ideally flagging suspicious rows into a separate quarantine table rather than either blindly including them or silently deleting them, so there's a clear audit trail and a way to review edge cases rather than making an irreversible, automated decision on genuinely ambiguous data.

**75. [A] You're given a messy real-world dataset with inconsistent date formats, duplicate customer IDs, and some NULLs in a revenue column. Walk through how you'd approach cleaning and validating it in SQL before running any analysis.**

> I'd work through this systematically rather than jumping straight into analysis. First, I'd profile the data before touching anything — running basic checks like row counts, `COUNT(DISTINCT customer_id)` versus total row count to gauge the scale of duplication, and a NULL count per column, so I actually understand the scope of the mess before deciding how to fix it. Second, for inconsistent date formats, I'd identify all the distinct formats actually present, maybe using pattern matching or just eyeballing a sample, and standardize them using the appropriate parsing function for each format, like `TO_DATE()` or `STR_TO_DATE()` with explicit format strings, rather than assuming one format applies uniformly, since a silent parsing failure or misinterpretation, like flipping day and month, can badly corrupt downstream date-based analysis without any obvious error. Third, for duplicate customer IDs, I'd first investigate whether they're genuine exact duplicates, like the deduplication query from earlier in this guide, or whether they're actually different rows that legitimately share a customer ID for a valid reason, like multiple transactions from the same customer, since those two scenarios call for very different handling — one gets deduplicated, the other doesn't need any fix at all. Fourth, for NULLs in the revenue column, I'd specifically avoid silently treating them as zero without first understanding why they're NULL — a NULL could mean a genuinely free transaction, a data collection gap, or a failed data load, and each of those has a completely different, correct handling; I'd flag or separately quantify those NULL rows and make a deliberate, documented choice, whether that's excluding them, imputing them, or investigating further upstream, rather than letting `AVG()` or `SUM()` silently ignore them and quietly skew a metric. Finally, and I think this matters most in a real interview answer, I'd emphasize that I'd document every cleaning decision I made and roughly how many rows each step affected, since a stakeholder trusting a final number needs to know it didn't just come from arbitrarily dropping or altering a meaningful chunk of the original data without any visibility into that process.

---

**That completes all 75 questions with full candidate-style answers, spanning foundational DBMS concepts through advanced analytics SQL and messy real-world scenario handling.** For final prep, I'd suggest writing out 8–10 of the harder queries — the window function ones (Q43–47) and the scenario-based ones (Q71–74) especially — from scratch on a whiteboard or in a plain text editor without autocomplete, since that's the closest simulation of an actual live SQL interview round.

---

### Quick prep priorities if time is short
If you only have a few days, prioritize in this order — these are the patterns that repeat most consistently across Amazon, Meta, Google, and Uber data interviews specifically:
1. JOINs — all types, cold, plus being able to spot when a join will produce duplicates (Q21–29)
2. GROUP BY / HAVING / aggregate functions, and how NULLs interact with them (Q30, 36)
3. Window functions — RANK/DENSE_RANK/ROW_NUMBER and LAG/LEAD are asked constantly (Q40–42)
4. CTEs vs subqueries, and being able to write a multi-step CTE query cleanly (Q32)
5. One retention/funnel/cohort-style business scenario query, rehearsed and written out by hand (Q71–73)
6. Being able to explain WHERE vs HAVING, and query execution order, without hesitating (Q11–12)

### A note on company patterns
- **Amazon (Data Analyst / Data Scientist / BI Engineer)**: heavy emphasis on business-first framing — messy schemas, ambiguous requirements, and multi-layer aggregations rather than pure syntax trivia; phone screens start with basic DATE/GROUP BY/JOIN questions, onsite rounds escalate to CASE statements, subqueries, and query optimization.
- **Meta (Data Analyst / Data Scientist)**: window-function-heavy, often paired with Python/pandas questions in the same round; funnel, engagement, and experiment-analysis style scenario questions are common.
- **Google / Uber / Airbnb / Netflix**: strong focus on writing correct, edge-case-aware analytics SQL — handling duplicates, time ranges, and NULLs correctly is often what separates a passing from failing answer, more than raw query complexity.
