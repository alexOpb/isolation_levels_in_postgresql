# Purpose: to test isolation levels in posgresql.

# TLDR:

Implemented four concurrency issues in databases (lost update, dirty read, repeatable read, phantom read). Checked which of the four PostgreSQL’s isolation levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable) can address them.

Findings:

1. In PostgreSQL, dirty reads are impossible, Read Uncommitted exists but functions like Read Committed.
2. In ‘read committed’ mode, a ‘lost update’ can occur. When T1 updates a cell, T2 waits for T1 to commit before T2 updates the same cell, but T2 will change the cell eventually causing T1’s update to be lost. BUT T1 only blocks the row, not the entire table, T2 can update other rows immediately even before T1 commits.

|                       | Lost Update (LU) | Dirty Read (DR)    | Repeatable Read (rr) | Phantom Read (PHR) |
| --------------------- | ---------------- | ------------------ | -------------------- | ------------------ |
| Read Uncommitted (RU) |                  | ✅ (in PostgreSQL) |                      |                    |
| Read Committed (RC)   |                  | ✅                 |                      |                    |
| Repeatable Read (RR)  | ✅               | ✅                 | ✅                   | ✅                 |
| Serializable (S)      | ✅               | ✅                 | ✅                   | ✅                 |

# LONGREAD:

## SETUP

1. On Windows, Start Docker desktop.
2. Open terminal in the folder with the Dockerfile
3. Run `docker build -t my-postgres .`
4. Run `docker run -p 5432:5432 my-postgres`
5. In Docker Desktop, find the container and `open in external terminal` two times. These two terminals are two connections to the db. In them, transaction 1 and transaction 2 will run. (T1 and T2)
6. In both terminals connect to the test_db: `psql postgresql://postgres:postgres@localhost:5432/testdb`

## TESTING GENERAL

- There are five tests:
  - 1\. Dirty read
  - 2\. Lost update
  - 3\. Repeatable read
  - 4\. Phontom read
  - 5.1 What does T1 blocks (a cell, a row, or table)?
  - 5.2 What does T1 blocks (a cell, a row, or table)?

## TESTING DETAILS

1. Run each line separately in a respective terminal.
2. After each test run this:
   After each test run this:
   `sql
    drop table users;
    CREATE TABLE users (id SERIAL PRIMARY KEY,name VARCHAR(50),email VARCHAR(50));
    INSERT INTO users (name, email) VALUES ('User1', 'user1@example.com');
    INSERT INTO users (name, email) VALUES ('User2', 'user2@example.com');
    INSERT INTO users (name, email) VALUES ('User3', 'user3@example.com');
    `

### Test 1: Dirty read

```sql
-- 1. T1
select * from users;
begin transaction isolation level read uncommitted;
update users set name = 'User1*' where id = 1;
select * from users;
-- 2. T2
begin transaction isolation level read uncommitted;
select * from users;
-- 3. ANSWER: Can T2 read uncommited changes made by T1, i.e. can it make a dirty read?
-- NO. WHY?
-- EXPLANATION:
-- In PostgreSQL, READ UNCOMMITTED and READ COMMITTED behave identically.
-- PostgreSQL treats READ UNCOMMITTED as READ COMMITTED.
-- PostgreSQL doesn’t allow for dirty reads (reading uncommitted data), which are permitted in the READ UNCOMMITTED level in some other database systems.
-- 4. T1
commit;
-- 5. T2
commit;
```

## Test 2: Lost update

```sql
-- 1. T1
select * from users;
-- Check for:
begin transaction isolation level read committed;
-- begin transaction isolation level repeatable read;
-- begin transaction isolation level serializable;
update users set name = 'User1*' where id = 1;
select * from users;
-- 2. T2
begin transaction isolation level read committed;
select * from users;
update users set name = 'User1**' where id = 1;
-- 3. T1
commit;
-- 4. T2
commit;
select * from users;
-- Observe: Did PostgreSQL allow T2 to commit and override the change made by T1?
-- read committed: Yes. Lost Update happens.
-- repeatable read: No. ERROR: could not serialize access due to concurrent update. Nothin can be done in the T2 transaction, one needs to commit; and rollback will happen.
-- serializable: No. same as repeatable read.
```

## Test 3: Repeatable read

```sql
-- 1. T1
select * from users;
-- Check for:
begin transaction isolation level read committed;
-- begin transaction isolation level repeatable read;
-- begin transaction isolation level serializable;
select * from users;
-- 2. T2
-- Check for:
begin transaction isolation level read committed;
-- begin transaction isolation level repeatable read;
-- begin transaction isolation level serializable;
update users set name = 'User1**' where id = 1;
select * from users;
commit;
-- 3. T1
select * from users;
-- 4. Can T1 see the commited changes by T2? If so, then repeatable read happened.
-- read committed: Yes. T1 can see commited changes by T2.
-- repeatable read: NO. Although T2 committed changes. T1 can't see them. Repetable read doesn't happen.
-- serializable: No. same as repeatable read.
-- 5. T1
commit;
select * from users;
```

## Test 4: Phantom read

```sql
-- 1. T1
select * from users;
-- Check for:
begin transaction isolation level read committed;
-- begin transaction isolation level repeatable read;
-- begin transaction isolation level serializable;
select * from users;
-- 2. T2
-- Check for:
begin transaction isolation level read committed;
-- begin transaction isolation level repeatable read;
-- begin transaction isolation level serializable;
INSERT INTO users (name, email) VALUES ('**User4**', 'user4@example.com');
select * from users;
commit;
-- 3. T1
select * from users;
-- 4. CAN T1 see the commited changes by T2 (insertion|deletion)? If so, then -- phantom read happened.
-- read committed: T1 can see committed insertion by T2. Phantom read happened.
-- repeatable read: NO. T1 can't see committed insertion by T2. Phantom read DID'T happen.
-- serializable: same as repeatable read
-- 5. T1
commit;
select * from users;
```

## Test 5.1: What does T1 blocks (a cell, a row, or table)?

```sql
-- 1. T1
select * from users;
begin transaction isolation level read committed;
update users set name = 'User1*' where id = 1;
select * from users;
-- 2. T2
begin transaction isolation level read committed;
update users set email = 'user1@example.com**' where id = 1;
select * from users;
commit;
-- 3.
-- What we know:
-- a) 'read committed' doesn't solve LOST UPDATE.
-- b) When T2 tries to update the same row and cell it can't do that until T1 -- commits, but after T1 commits it performs the update making LOST UPDATE problem.
-- c) The question is: what T1 really blocks? a cell? a row? a table?
-- d) Here we check if it's the cell that T1 is blocking. If so, we can immediately -- change another cell in the same row without waiting.
-- ACTUAL RESULT: T1 blocks not just a cell, but the whole row. T2 can't update the -- email value in the same row immediately.
-- 4. T1
commit; -- at that moment T2 completes its update
-- 5. T2
commit;
```

## Test 5.2: What does T1 blocks (a cell, a row, or table)?

```sql
-- 1. T1
select * from users;
begin transaction isolation level read committed;
update users set name = 'User1*' where id = 1;
select * from users;
-- 2. T2
begin transaction isolation level read committed;
update users set name = 'User2**' where id = 2;
select * from users;
commit;
-- 3. ACTUAL RESULT: T1 block a specific row, not the whole table. T2 can immediately change another row, not waiting for T1 to commit.
4. T1
commit;
5. T2
commit;
```

```
Based on 5.1, 5.2 what isolation level blocks?
1. Not just a cell
2. It blocks a row.
3. It doesn't block the whole table for other Transactions.
```
