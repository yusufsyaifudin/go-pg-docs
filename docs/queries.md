---
template: main.html
---

# Writing queries

## Select

Select book by primary key:

```go
book := new(Book)
err := db.Model(book).Where("id = ?", 1).Select()
```

```sql
SELECT "book"."id", "book"."title", "book"."text"
FROM "books" WHERE id = 1
```

Select only book title and text:

```go
err := db.Model(book).
    Column("title", "text").
    Where("id = ?", 1).
    Select()
```

```sql
SELECT "title", "text" FROM "books" WHERE id = 1
```

Select only book title and text into variables:

```go
var title, text string
err := db.Model((*Book)(nil)).
    Column("title", "text").
    Where("id = ?", 1).
    Select(&title, &text)
```

```sql
SELECT "title", "text"
FROM "books" WHERE id = 1
```

Select book using `WHERE ... AND ...`:

```go
err := db.Model(book).
    Where("id > ?", 100).
    Where("title LIKE ?", "my%").
    Limit(1).
    Select()
```

```sql
SELECT "book"."id", "book"."title", "book"."text"
FROM "books"
WHERE (id > 100) AND (title LIKE 'my%')
LIMIT 1
```

Select book using `WHERE ... OR ...`:

```go
err := db.Model(book).
    Where("id > ?", 100).
    WhereOr("title LIKE ?", "my%").
    Limit(1).
    Select()
```

```sql
SELECT "book"."id", "book"."title", "book"."text"
FROM "books"
WHERE (id > 100) OR (title LIKE 'my%')
LIMIT 1
```

Select book user `WHERE ... AND (... OR ...)`:

```go
err := db.Model(book).
    Where("title LIKE ?", "my%").
    WhereGroup(func(q *orm.Query) (*orm.Query, error) {
        q = q.WhereOr("id = 1").
            WhereOr("id = 2")
        return q, nil
    }).
    Limit(1).
    Select()
```

```sql
SELECT "book"."id", "book"."title", "book"."text"
FROM "books"
WHERE (title LIKE 'my%') AND (id = 1 OR id = 2)
LIMIT 1
```

Select first 20 books:

```go
var books []Book
err := db.Model(&books).Order("id ASC").Limit(20).Select()
```

```sql
SELECT "book"."id", "book"."title", "book"."text"
FROM "books"
ORDER BY id ASC LIMIT 20
```

Count books:

```go
count, err := db.Model((*Book)(nil)).Count()
```

```sql
SELECT count(*) FROM "books"
```

Select 20 books and count all books:

```go
count, err := db.Model(&books).Limit(20).SelectAndCount()
```

```sql
SELECT "book"."id", "book"."title", "book"."text"
FROM "books" LIMIT 20;

SELECT count(*) FROM "books";
```

Select 20 books and count estimated number of books:

```go
count, err := db.Model(&books).Limit(20).SelectAndCountEstimate(100000)
```

```sql
SELECT "book"."id", "book"."title", "book"."text"
FROM "books" LIMIT 20;

EXPLAIN SELECT '_go_pg_placeholder' FROM "books"
SELECT count(*) FROM "books";
```

Select author ID and number of books:

```go
var res []struct {
    AuthorId  int
    BookCount int
}
err := db.Model((*Book)(nil)).
    Column("author_id").
    ColumnExpr("count(*) AS book_count").
    Group("author_id").
    Order("book_count DESC").
    Select(&res)
```

```sql
SELECT "author_id", count(*) AS book_count
FROM "books" AS "book"
GROUP BY author_id
ORDER BY book_count DESC
```

Select book IDs as PostgreSQL array:

```go
var ids []int
err := db.Model((*Book)(nil)).ColumnExpr("array_agg(id)").Select(pg.Array(&ids))
```

```sql
SELECT array_agg(id) FROM "books"
```

Select by multiple ids:

```go
ids := []int{1, 2, 3}
err := db.Model((*Book)(nil)).
    Where("id in (?)", pg.In(ids)).
    Select()
```

```sql
SELECT * FROM books WHERE id IN (1, 2, 3)
```

Select books for update

```go
book := &Book{}
err := db.Model(book).
    Where("id = ?", 1).
    For("UPDATE").
    Select()
```

```sql
SELECT * FROM books WHERE id  = 1 FOR UPDATE
```

## CTE

Select books using WITH statement:

```go
authorBooks := db.Model((*Book)(nil)).Where("author_id = ?", 1)

err := db.Model().
    With("author_books", authorBooks).
    Table("author_books").
    Select(&books)
```

```sql
WITH "author_books" AS (
  SELECT "book"."id", "book"."title", "book"."text"
  FROM "books" AS "book" WHERE (author_id = 1)
)
SELECT * FROM "author_books"
```

Same query using WrapWith:

```go
err := db.Model(&books).
    Where("author_id = ?", 1).
    WrapWith("author_books").
    Table("author_books").
    Select(&books)
```

```sql
WITH "author_books" AS (
  SELECT "book"."id", "book"."title", "book"."text"
  FROM "books" AS "book" WHERE (author_id = 1)
)
SELECT * FROM "author_books"
```

## Subqueries

Subquery in FROM:

```go
authorBooks := db.Model((*Book)(nil)).Where("author_id = ?", 1)

err := db.Model(nil).TableExpr("(?) AS book", authorBooks).Select(&books)
```

```sql
SELECT * FROM (
  SELECT "book"."id", "book"."title", "book"."text"
  FROM "books" AS "book" WHERE (author_id = 1)
) AS book
```

Subquery in WHERE:

```go
authorBooks := db.Model((*Book)(nil)).ColumnExpr("id").Where("author_id = ?", 1)

err := db.Model(&books).Where("id IN (?)", authorBooks).Select()
```

```sql
SELECT * FROM "books" WHERE id IN (
  SELECT id FROM "books" AS "book" WHERE (author_id = 1)
)
```

### Column names

Select book and associated author:

```go
err := db.Model(book).Relation("Author").Select()
```

```sql
SELECT
  "book"."id", "book"."title", "book"."text",
  "author"."id" AS "author__id", "author"."name" AS "author__name"
FROM "books"
LEFT JOIN "users" AS "author" ON "author"."id" = "book"."author_id"
WHERE id = 1
```

Select book ID and associated author id:

```go
err := db.Model(book).Column("book.id").Relation("Author.id").Select()
```

```sql
SELECT "book"."id", "author"."id" AS "author__id"
FROM "books"
LEFT JOIN "users" AS "author" ON "author"."id" = "book"."author_id"
WHERE id = 1
```

Select book and join author without selecting it:

```go
err := db.Model(book).Relation("Author._").Select()
```

```sql
SELECT "book"."id"
FROM "books"
LEFT JOIN "users" AS "author" ON "author"."id" = "book"."author_id"
WHERE id = 1
```

Join and select book author without selecting book:

```go
err := db.Model(book).
    Column("_").
    Relation("Author").
    Where("id = ?", 1).
    Select()
```

```
SELECT "author"."id" AS "author__id", "author"."name" AS "author__name"
FROM "books"
LEFT JOIN "users" AS "author" ON "author"."id" = "book"."author_id"
WHERE id = 1
```

## Insert

### Insert struct

Insert new book returning primary keys:

```go
err := db.Model(book).Insert()
```

```sql
INSERT INTO "books" (title, text) VALUES ('my title', 'my text') RETURNING "id"
```

Insert new book returning all columns:

```go
err := db.Model(book).Returning("*").Insert()
```

```sql
INSERT INTO "books" (title, text) VALUES ('my title', 'my text') RETURNING *
```

Insert new book or update existing one:

```go
_, err := db.Model(book).
    OnConflict("(id) DO UPDATE").
    Set("title = EXCLUDED.title").
    Insert()
```

```sql
INSERT INTO "books" ("id", "title") VALUES (100, 'my title')
ON CONFLICT (id) DO UPDATE SET title = 'title version #1'
```

### Insert slice

Insert slice in a single query:

```go
books := []*Book{book1, book2}
_, err := db.Model(&books).Insert()
```

```sql
INSERT INTO "books" (title, text) VALUES ('title1', 'text2'), ('title2', 'text2') RETURNING "id"
```

Alternatively:

```go
_, err := db.Model(book1, book2).Insert()
```

```sql
INSERT INTO "books" (title, text) VALUES ('title1', 'text2'), ('title2', 'text2') RETURNING *
```

### Insert map

Insert `map[string]interface{}`:

```go
values := map[string]interface{}{
    "title": "title1",
    "text":  "text1",
}
_, err := db.Model(&values).TableExpr("books").Insert()
```

```sql
INSERT INTO books (title, text) VALUES ('title1', 'text2')
```

### Select or insert

Select existing book by name or create new book:

```go
_, err := db.Model(book).
    Where("title = ?title").
    OnConflict("DO NOTHING"). // optional
    SelectOrInsert()
```

```sql
SELECT * FROM "books" WHERE title = 'my title';

INSERT INTO "books" (title, text) VALUES ('my title', 'my text') RETURNING "id";
```

## Update

### Update struct

Update all columns except primary keys:

```go
err := db.Model(book).Update()
```

```sql
UPDATE books SET title = 'my title', text = 'my text' WHERE id = 1
```

Update only column "title":

```go
res, err := db.Model(book).Set("title = ?title").Where("id = ?id").Update()
```

```sql
UPDATE books SET title = 'my title' WHERE id = 1
```

Update only column "title":

```go
res, err := db.Model(book).
    Column("title").
    Where("id = ?", 1).
    Update()
```

```sql
UPDATE books SET title = 'my title' WHERE id = 1
```

Upper column "title" and scan it:

```go
var title string
res, err := db.Model(book).
    Set("title = upper(title)").
    Where("id = ?", 1).
    Returning("title").
    Update(&title)
```

```sql
UPDATE books SET title = upper(title) WHERE id = 1 RETURNING title
```

### Update slice

Update multiple books with single query:

```go
err := db.Model(book1, book2).Update()
```

```sql
UPDATE books AS book SET title = _data.title, text = _data.text
FROM (VALUES (1, 'title1', 'text1'), (2, 'title2', 'text2')) AS _data (id, title, text)
WHERE book.id = _data.id
```

### Update map

Update `map[string]interface{}`:

```go
values := map[string]interface{}{
    "title": "title1",
    "text":  "text1",
}
_, err := db.Model(&values).TableExpr("books").Where("id = ?", 1).Update()
```

```sql
UPDATE books SET title = 'title1', text = 'text2' WHERE id = 1
```

## Delete

Delete book by primary key:

```go
res, err := db.Model(book).Where("id = ?", 1).Delete()
```

```sql
DELETE FROM "books" WHERE id = 1
```

Delete book by title:

```go
res, err := db.Model(book).Where("title = ?title").Delete()
```

```sql
DELETE FROM "books" WHERE title = 'my title'
```

Delete multiple books using ids:

```go
res, err := db.Model((*Book)(nil)).
    Where("id IN (?)", pg.In([]int{1, 2})).
    Delete()
```

```sql
DELETE FROM "books" WHERE id IN (1, 2)
```

Delete multiple books using structs:

```go
books := []*Book{book1, book2} // slice of books with ids

res, err := db.Model(&books).WherePK().Delete()
```

```sql
DELETE FROM "books" WHERE id IN (1, 2)
```

## Joins

Select a book and manually join the book author:

```go
book := new(Book)
err := db.Model(book).
    ColumnExpr("book.*").
    ColumnExpr("a.id AS author__id, a.name AS author__name").
    Join("JOIN authors AS a ON a.id = book.author_id").
    First()
```

```
SELECT book.*, a.id AS author__id, a.name AS author__name
FROM books
JOIN authors AS a ON a.id = book.author_id
ORDER BY id LIMIT 1
```

Join conditions can be split using `JoinOn`:

```go
q.Join("LEFT JOIN authors AS a").
    JoinOn("a.id = book.author_id").
    JoinOn("a.active = ?", true)
```

You can achieve the same using ORM relations which are described below.

## ORM

### Has one

Following examples selects users joining their profiles:

```go
type Profile struct {
    Id   int
    Lang string
}

// User has one profile.
type User struct {
    Id        int
    Name      string
    ProfileId int
    Profile   *Profile `pg:"rel:has-one"`
}

db := connect()
defer db.Close()

qs := []string{
    "CREATE TEMP TABLE users (id int, name text, profile_id int)",
    "CREATE TEMP TABLE profiles (id int, lang text)",
    "INSERT INTO users VALUES (1, 'user 1', 1), (2, 'user 2', 2)",
    "INSERT INTO profiles VALUES (1, 'en'), (2, 'ru')",
}
for _, q := range qs {
    _, err := db.Exec(q)
    if err != nil {
        panic(err)
    }
}

// Select users joining their profiles with following query:
//
// SELECT
//   "user".*,
//   "profile"."id" AS "profile__id",
//   "profile"."lang" AS "profile__lang",
//   "profile"."user_id" AS "profile__user_id"
// FROM "users" AS "user"
// LEFT JOIN "profiles" AS "profile" ON "profile"."user_id" = "user"."id"

var users []User
err := db.Model(&users).
    Column("user.*").
    Relation("Profile").
    Select()
if err != nil {
    panic(err)
}

fmt.Println(len(users), "results")
fmt.Println(users[0].Id, users[0].Name, users[0].Profile)
fmt.Println(users[1].Id, users[1].Name, users[1].Profile)
// Output: 2 results
// 1 user 1 &{1 en}
// 2 user 2 &{2 ru}
```

### Belongs to

Following examples selects users joining their profiles:

```go
// Profile belongs to User.
type Profile struct {
    Id     int
    Lang   string
    UserId int
}

type User struct {
    Id      int
    Name    string
    Profile *Profile `pg:"rel:belongs-to"`
}

db := connect()
defer db.Close()

qs := []string{
    "CREATE TEMP TABLE users (id int, name text)",
    "CREATE TEMP TABLE profiles (id int, lang text, user_id int)",
    "INSERT INTO users VALUES (1, 'user 1'), (2, 'user 2')",
    "INSERT INTO profiles VALUES (1, 'en', 1), (2, 'ru', 2)",
}
for _, q := range qs {
    _, err := db.Exec(q)
    if err != nil {
        panic(err)
    }
}

// Select users joining their profiles with following query:
//
// SELECT
//   "user".*,
//   "profile"."id" AS "profile__id",
//   "profile"."lang" AS "profile__lang"
// FROM "users" AS "user"
// LEFT JOIN "profiles" AS "profile" ON "profile"."id" = "user"."profile_id"

var users []User
err := db.Model(&users).
    Column("user.*").
    Relation("Profile").
    Select()
if err != nil {
    panic(err)
}

fmt.Println(len(users), "results")
fmt.Println(users[0].Id, users[0].Name, users[0].Profile)
fmt.Println(users[1].Id, users[1].Name, users[1].Profile)
// Output: 2 results
// 1 user 1 &{1 en 1}
// 2 user 2 &{2 ru 2}
```

### Has many

Following example selects first user and all his active profiles:

```go
type Profile struct {
    Id     int
    Lang   string
    Active bool
    UserId int
}

// User has many profiles.
type User struct {
    Id       int
    Name     string
    Profiles []*Profile `pg:"rel:has-many"`
}

db := connect()
defer db.Close()

qs := []string{
    "CREATE TEMP TABLE users (id int, name text)",
    "CREATE TEMP TABLE profiles (id int, lang text, active bool, user_id int)",
    "INSERT INTO users VALUES (1, 'user 1')",
    "INSERT INTO profiles VALUES (1, 'en', TRUE, 1), (2, 'ru', TRUE, 1), (3, 'md', FALSE, 1)",
}
for _, q := range qs {
    _, err := db.Exec(q)
    if err != nil {
        panic(err)
    }
}

// Select user and all his active profiles with following queries:
//
// SELECT "user".* FROM "users" AS "user" ORDER BY "user"."id" LIMIT 1
//
// SELECT "profile".* FROM "profiles" AS "profile"
// WHERE (active IS TRUE) AND (("profile"."user_id") IN ((1)))

var user User
err := db.Model(&user).
    Column("user.*").
    Relation("Profiles", func(q *orm.Query) (*orm.Query, error) {
        return q.Where("active IS TRUE"), nil
    }).
    First()
if err != nil {
    panic(err)
}
fmt.Println(user.Id, user.Name, user.Profiles[0], user.Profiles[1])
// Output: 1 user 1 &{1 en true 1} &{2 ru true 1}
```

### Has many to many

Following example selects an order and all items in the order using intermediary `order_to_items`
table.

```go
package pg_test

import (
    "fmt"

    "github.com/go-pg/pg/v10"
    "github.com/go-pg/pg/v10/orm"
)

func init() {
    // Register many to many model so ORM can better recognize m2m relation.
    // This should be done before dependant models are used.
    orm.RegisterTable((*OrderToItem)(nil))
}

type Order struct {
    Id    int
    Items []Item `pg:"many2many:order_to_items"`
}

type Item struct {
    Id int
}

type OrderToItem struct {
    OrderId int
    ItemId  int
}

func ExampleDB_Model_manyToMany() {
    db := connect()
    defer db.Close()

    if err := createManyToManyTables(db); err != nil {
        panic(err)
    }

    values := []interface{}{
        &Item{Id: 1},
        &Item{Id: 2},
        &Order{Id: 1},
        &OrderToItem{OrderId: 1, ItemId: 1},
        &OrderToItem{OrderId: 1, ItemId: 2},
    }
    for _, v := range values {
        _, err := db.Model(v).Insert()
        if err != nil {
            panic(err)
        }
    }

    // Select order and all items with following queries:
    //
    // SELECT "order"."id" FROM "orders" AS "order" ORDER BY "order"."id" LIMIT 1
    //
    // SELECT order_to_items.*, "item"."id" FROM "items" AS "item"
    // JOIN order_to_items AS order_to_items ON (order_to_items."order_id") IN (1)
    // WHERE ("item"."id" = order_to_items."item_id")

    order := new(Order)
    err := db.Model(order).Relation("Items").First()
    if err != nil {
        panic(err)
    }
    fmt.Println("Order", order.Id, "Items", order.Items[0].Id, order.Items[1].Id)

    // Select order and all items sorted by id with following queries:
    //
    // SELECT "order"."id" FROM "orders" AS "order" ORDER BY "order"."id" LIMIT 1
    //
    // SELECT order_to_items.*, "item"."id" FROM "items" AS "item"
    // JOIN order_to_items AS order_to_items ON (order_to_items."order_id") IN (1)
    // WHERE ("item"."id" = order_to_items."item_id")
    // ORDER BY item.id DESC

    order = new(Order)
    err = db.Model(order).
        Relation("Items", func(q *orm.Query) (*orm.Query, error) {
            q = q.OrderExpr("item.id DESC")
            return q, nil
        }).
        First()
    if err != nil {
        panic(err)
    }
    fmt.Println("Order", order.Id, "Items", order.Items[0].Id, order.Items[1].Id)

    // Output: Order 1 Items 1 2
    // Order 1 Items 2 1
}

func createManyToManyTables(db *pg.DB) error {
    models := []interface{}{
        (*Order)(nil),
        (*Item)(nil),
        (*OrderToItem)(nil),
    }
    for _, model := range models {
        err := db.Model(model).CreateTable(&orm.CreateTableOptions{
            Temp: true,
        })
        if err != nil {
            return err
        }
    }
    return nil
}
```

## Executing custom queries

Create index for the table:

```go
_, err := db.Model((*Book)(nil)).Exec(`
    CREATE INDEX CONCURRENTLY books_author_id_idx
    ON ?TableName (author_id)
`)
```

Or:

```go
var count int
_, err := db.Model((*Book)(nil)).QueryOne(pg.Scan(&count), `
    SELECT count(*)
    FROM ?TableName AS ?TableAlias
`)
```
