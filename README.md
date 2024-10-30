# sqltabler

`sqltabler` is a Go library that recursively modifies SQL statements, adding prefixes and suffixes to the names of tables, views, triggers, and indexes.

## Use Case

This library was created for [owobot](https://gitea.elara.ws/owobot/owobot), where several dynamic plugins, potentially from different authors, have to be able to access an SQLite database simultaneously without breaking the bot's internal data or that of another plugin.

Adding unique identifiers to the names within all SQL statements executed by the plugins before they're passed to the database ensures that all the names are unique, and therefore cannot conflict.

## Usage

### `Modify()`

The `Modify` function accepts an SQL string and adds a prefix and suffix to all relevant names.

```go
import "go.elara.ws/sqltabler"

const stmt = `
	CREATE TABLE users (
		id       INT  PRIMARY KEY,
		username TEXT NOT NULL,
		hash     BLOB NOT NULL
	);
	
	CREATE TABLE emails (
		user_id INT  NOT NULL UNIQUE,
		email   TEXT NOT NULL,
		FOREIGN KEY (user_id) REFERENCES users(id)
	);
	
	INSERT INTO users (username, hash) VALUES ('hello', 'world');
	INSERT INTO emails (user_id, email) VALUES (0, 'user@example.com');
`

func main() {
	modifiedStmt, err := sqltabler.Modify(stmt, "plugin_", "_1234")
	if err != nil {
	    panic(err)
	}
	fmt.Println(modifiedStmt)
}
```

**Output**:

```sql
CREATE TABLE "plugin_users_1234" (
  "id" INT PRIMARY KEY, "username" TEXT NOT NULL, 
  "hash" BLOB NOT NULL
);
CREATE TABLE "plugin_emails_1234" (
  "user_id" INT NOT NULL UNIQUE, 
  "email" TEXT NOT NULL, 
  FOREIGN KEY ("user_id") REFERENCES "plugin_users_1234" ("id")
);
INSERT INTO "plugin_users_1234" ("username", "hash") VALUES ('hello', 'world');
INSERT INTO "plugin_emails_1234" ("user_id", "email")  VALUES (0, 'user@example.com');
```

### ModifyStmt

Use `ModifyStmt` to work with [`sql.Statement`](https://pkg.go.dev/github.com/rqlite/sql#Statement) directly. It clones and modifies the statement, returning a new `sql.Statement`.

```go
import (
	"github.com/rqlite/sql"
	"go.elara.ws/sqltabler"
)

const stmt = `CREATE TABLE users ( username TEXT NOT NULL )`

func main() {
	ast, err := sqlparser.NewParser(strings.NewReader(stmt)).ParseStatement()
	if err != nil {
		panic(err)
	}
	ast = sqltabler.ModifyStmt(ast, "plugin_", "_5678")
	fmt.Println(ast.String())
}
```

**Output**:

```sql
CREATE TABLE "plugin_users_5678" ("username" TEXT NOT NULL);
```