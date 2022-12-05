# Working with database (Postgres + SQLC)

###### tags: `golang_backend`

## 1. install & use docker + porstgres +tableplus to create DB schemas

### what will we do

1. install docker desktop
2. run postgres container

```dockerfile
docker pull postgres:12-alpine
docker pull <image>:<tag>
```

**start a container**

```dockerfile=
docker run
--name <container_name>
-e <environment_variable>
        -d
    <image>:<tag>

docker run
--name some-postgres
-e POSTGRES_PASSWORD=mysecretpassword
        -d
    postgres


// our case
docker run --name postgres12 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres:12-alpine

// stop and start container
docker stop [container-name]
docker start [container-name]
```

Basically, a container is 1 instance of the application contained in the image.
we can start multiple containers from 1 single image.
**Run command in container**

```dockerfile=
docker exec -it
<container_name_or_id>
    <commnad> [args]
// in our case
docker exec -it postgres12 psql -U root
// log in postgres console using root as user


// go into containers bash run any linux command
docker exec -it postgres12 /bin/sh
```

**View container logs**

```dockerfile=
docker logs
<container_name_or_id>
```

3. install table plus

![](https://i.imgur.com/OC9zU4D.png)

4. create database schema

## 2. How to write & run database migration in Golang

1. install https://github.com/golang-migrate/migrate

```
brew install golang-migrate
```

useful command
![](https://i.imgur.com/zc1BMSf.png)

1. create folder /db/migration
   use code

```
migrate create -ext sql -dir db/migration -seq init_schema
```

2. go into docker postgres12's terminal

```
docker exec -it postgres12 createdb --username=root --owner=root simple_bank

//see if it work or not
docker exec -it postgres12 psql -U root simple_bank
```

in the Makefile

```
postgres:
	docker run --name postgres12 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres:12-alpine

createdb:
	docker exec -it postgres12 createdb --username=root --owner=root simple_bank

dropdb:
	docker exec -it postgres12 dropdb simple_bank

.PHONY: postgres createdb dropdb
```

In terminal run:

```
make postgres // to set up docker postgres
make createdb // to create db
...
```

3. migrate setup

To set up the database we created using the sql code in migrate up

```
migrate -path db/migration -database "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable" -verbose up
```

## 3. Generate CRUD Golang code from SQL

:::info

### Database / SQL

- Very fast & straightforward
- Manual mapping SQL fields to variables
- Easy to make mistakes, not caught until runtimes

### Gorm

- CRUD functions already implemented, very short production code
- Must learn to write queries using gorm's function
- Run slowly on high load

### SQLX

- Quite fast & easy to use.
- Fields mapping via query text & struct tags
- Failure won't occur until runtime.

### SQLC

- Very fast & easy to use
- Automatic code generation
- Catch SQL query errors before generating codes
- Full support Postgres. MySQL is experimental.
  :::

Makefile

```
postgres:
	docker run --name postgres12 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres:12-alpine

createdb:
	docker exec -it postgres12 createdb --username=root --owner=root simple_bank

dropdb:
	docker exec -it postgres12 dropdb simple_bank

migrateup:
	migrate -path db/migration -database "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable" -verbose up

migratedown:
	migrate -path db/migration -database "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable" -verbose down

sqlc:
	sqlc generate

test:
	go test -v -cover ./...
# -v verbose -cover -> measure code coverage ./... to run unit tests in all of them

.PHONY: postgres createdb dropdb migrateup migratedown sqlc test
```

1. 首先 sqlc.init 打在 teminal
   會產生 sqlc.yaml
2. 在 db 資料夾中創立 query 和 sqlc 兩個資料夾
3. 在 query 中按照資料庫寫 sql 語言，在此例中寫了 account, entry 和 transfer 三個 sql 檔
4. 回到 makefile 中打下 make sqlc 就會生成 go 檔，以此可以寫下測試檔來測試它們

## A clean way to implement database transaction in Golang

### What is a db transaction?

- A single unit of work
- Often made up of multiple db operations

Example:
Transfer 10 USD from bank account 1 to bank account 2

1. Create a transfer record with amount 10
2. Create an account entry for account 1 with amount = -10
3. Create an account entry for account 2 with amount = +10
4. Subtract 10 from the balance of account 1
5. Add 10 to the balance of account2

### Why do we need db transaction?

1. To provide a reliable and consistent unit of work, even in case of system failure
2. To provide isolation between programs that access the database concurrently.

### ACID PROPERTY

1. Atomicity(A): Either all operations complete successfully or the transaction fails and the db is unchanged.
2. Consistency(C): The db state must be valid after the transaction. All constranints must be satisfied.
3. Isolation(I): Concurrency transactions must not affect each other.
4. Durability(D): Data written by a successful transaction must be recorded in persistent storage.

### How to run sql TX?

```
BEGIN;
...
COMMIT
```

```
BEGIN
...
ROLLBACK;
```

## Deeply understand transaction isolation levels & read phenomena

### Read Phenomena

- DIRTY READ
  A transaction **reads** data written by other concurrent **uncommitted** transaction
- NON-REPEATABLE READ
  A transaction **reads** the **same row twice** and sees different value because it has been **modified** by other **committed** transaction
- PHANTOM READ
  A transaction **re-executes** a query to **find rows** that satisfy a condition and sees a **different set** of rows, due to changes by other **committed** transaction
- SERIALIZATION ANOMALY
  The result of a **group** of concurrent **committed transactions** is **impossible to achieve** if we try to run them **sequentially** in any order without overlapping.

### 4 Standard Isolation Levels(American National Standards Institure-ANSI)

- Low Level

1. READ UNCOMMITTED: Can see data written by uncommitted transaction
2. READ COMMITTED: Only see data written by committed transaction
3. REPEATABLE READ: Same read query always reutrns same result.
4. SERIALIZABLE: Can achieve same result if execute transactions serially in some order instead of concurrently.

### See 4 isolation Levels in MySQL & postgreSQL

- MySQL

1. READ UNCOMMITTED:
   ![](https://i.imgur.com/KuvBIPs.jpg)
2. READ COMMITTED
   ![](https://i.imgur.com/9lRWthS.jpg)
3. REPEATABLE COMMITTED
   看右邊底下，會有 serializable anomaly 的問題
   ![](https://i.imgur.com/FQWPyS0.jpg)
4. SERIALIZABLE
   MySQL 是靠鎖住來預防 serializable anomaly 的，因此右邊要 commit;左邊才會執行 update
   ![](https://i.imgur.com/5dZQG7n.jpg)

- postgreSQL
  因為 postgre 預設 read uncommitted 和 read committed 是一樣的，因此只有三個 isolation level.

1. READ UNCOMMITTED & READ COMMITTED:
   當左邊已經更新了，但右邊並不曉得，因此，會導致右邊相同的查詢有不同的值出現的情況，這就是 PHANTOM READ
   ![](https://i.imgur.com/HWQTbbs.jpg)
2. REPEATABLE READ
   同樣的情況，在 REAPEATABLE READ 中會提醒錯誤發生(could not serialize access due to concurrent update)
   ![](https://i.imgur.com/X8ZxFfc.jpg)
   這是在 REAPEATABLE READ 下發生的 SERIALIZATION ANOMALY(看右下角)
   ![](https://i.imgur.com/zEvZHQ1.jpg)

3. SERIALIZABLE
   右下角提醒了(CHECK)(HINT: The transaction might succeed if retried.)來避免 SERIALIZATION ANOMALY
   ![](https://i.imgur.com/6AcgIXz.jpg)

**Summary**
level 4 可以避免 全部
level 3 可以避免 SERIALIZATION ANOMALY 以外的問題
...

mysql default mode is REPEATABLE COMMITTED
postgreSql mode is READ COMMITTED

mysql 利用 Loking mechanism
postgresql 利用 dependencies detection
