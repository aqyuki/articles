---
title: "sqlxでJOINした結果を保存したい！"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "database"]
published: true
---

# はじめに

Database へのアクセスに[sqlx](https://github.com/jmoiron/sqlx)を使用した際に、JOIN の扱いに困ったので備忘録として残しておこうと思います。

# 準備

使用したテーブルは以下のとおりです。PostgreSQL を使用しました。

**users**

| title  | type | description     |
| :----- | :--- | :-------------- |
| `id`   | UUID | User identifier |
| `name` | Text | User name       |

**notes**

| title      | type | description                  |
| :--------- | :--- | :--------------------------- |
| `id`       | UUID | Note identifier              |
| `owner_id` | UUID | Identifier of the note owner |
| `title`    | Text | Note title                   |
| `body`     | Text | Note body                    |

:::details SQL

```sql
-- user table
create table users (
  id uuid primary key,
  name text not null unique
);

-- note table
create table notes (
  id uuid primary key,
  owner_id uuid not null,
  title text not null,
  body text not null,
  foreign key (owner_id) references users(id)
);

```

:::

# プログラム

```go
package main

import (
	"fmt"

	"github.com/google/uuid"
	_ "github.com/jackc/pgx/v5/stdlib"
	"github.com/jmoiron/sqlx"
)

const DatabaseURL = "postgres://postgres:postgres@localhost:5432/database?sslmode=disable"

type User struct {
	ID   string `db:"id"`
	Name string `db:"name"`
}

func (u User) String() string {
	return fmt.Sprintf(`User
  ID: %s
  Name: %s
`, u.ID, u.Name)
}

type Note struct {
	ID      string `db:"id"`
	OwnerID string `db:"owner_id"`
	Title   string `db:"title"`
	Body    string `db:"body"`
}

func (n Note) String() string {
	return fmt.Sprintf(`Note
  ID: %s
  OwnerID: %s
  Title: %s
  Body: %s
`, n.ID, n.OwnerID, n.Title, n.Body)
}

func main() {
	user := &User{
		ID:   uuid.New().String(),
		Name: "John Doe",
	}

	note := &Note{
		ID:      uuid.New().String(),
		OwnerID: user.ID,
		Title:   "Hello, World!",
		Body:    "This is a note.",
	}

	note2 := &Note{
		ID:      uuid.New().String(),
		OwnerID: user.ID,
		Title:   "Database testing",
		Body:    "Database testing",
	}

	db, err := sqlx.Connect("pgx", DatabaseURL)
	if err != nil {
		panic(err)
	}

	// Create new user.
	if _, err := db.NamedExec(`insert into users (id, name) values (:id, :name)`, user); err != nil {
		panic(err)
	}

	// Create new note.
	if _, err := db.NamedExec(`insert into notes (id, owner_id, title, body) values (:id, :owner_id, :title, :body)`, note); err != nil {
		panic(err)
	}
	if _, err := db.NamedExec(`insert into notes (id, owner_id, title, body) values (:id, :owner_id, :title, :body)`, note2); err != nil {
		panic(err)
	}

	// fetch notes
	var notes []struct {
		User *User `db:"user"`
		Note *Note `db:"note"`
	}

	// join notes and users
	if err := db.Select(&notes, `select notes.id "note.id", notes.owner_id "note.owner_id", notes.title "note.title", notes.body "note.body", users.id "user.id", users.name "user.name" from notes join users on notes.owner_id = users.id`); err != nil {
		panic(err)
	}

	for _, n := range notes {
		fmt.Println(n.User)
		fmt.Println(n.Note)
	}

	// remove notes
	if _, err := db.Exec(`delete from notes`); err != nil {
		panic(err)
	}

	// remove users
	if _, err := db.Exec(`delete from users`); err != nil {
		panic(err)
	}
}

```

# ポイント

sqlx を用いて JOIN した結果を構造体にマッピングするには、`Select()`メソッドを使用します。
第一引数に Select 結果を保存したい構造体(スライス)を指定し、第二引数には SQL を指定します。今回は使用していませんが第三引数に値を指定すると prepared statement を使用することが出来ます。
注目すべきは Select に渡している SQL 文です。

```sql
select notes.id "note.id", notes.owner_id "note.owner_id", notes.title "note.title", notes.body "note.body", users.id "user.id", users.name "user.name" from notes join users on notes.owner_id = users.id
```

Select するカラムを指定する際にマッピング先の構造体を指定しています。sqlx では`db`タグの付いたフィールドに対してマッピングを行っていきます。そのため、カラムに対して対応したフィールドを指定すれば JOIN で結合しても正常にマッピングすることが出来ます。

# 参考

https://github.com/jmoiron/sqlx
