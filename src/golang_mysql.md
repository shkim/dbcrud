title: Go MySQL
---

# Golang MySQL
---

## Installation

https://github.com/go-sql-driver/mysql
https://github.com/jmoiron/sqlx

```
$ go get -u github.com/go-sql-driver/mysql
$ go get -u github.com/jmoiron/sqlx
```
sqlx 는 "db:필드명" 메타정보를 이용해 struct 변수에 자동으로 필드값을 채워준다.
StructScan 메소드를 쓰려면 끝에 x로 끝나는 메소드를 써야 한다.
예: QueryRow().Scan() -> QueyrRowx().StructScan()
## Connect

```go
import (
    _ "github.com/go-sql-driver/mysql"
    "github.com/jmoiron/sqlx"
)

var sqlDB *sqlx.DB

connstr := "userid:passwd@tcp(example.mysqlsvr.com:3306)/dbname?parseTime=true&charset=utf8mb4&collation=utf8mb4_unicode_ci"
sqlDB, err = sqlx.Connect("mysql", connstr)
if err != nil {
    log.Printf("MySQL connect failed: %v\n", err)
    return
}

sqlDB.MustExec("SET NAMES utf8mb4")

// set timezone
sqlDB.MustExec("SET time_zone = '+9:00'")

// limit max connection (값을 줄여서 개발시에 에러를 경험해 보는 것이 좋다.)
sqlDB.SetMaxOpenConns(12)
```
* parseTime=true 옵션을 줘야 Scan()시에 DATETIME 컬럼이 time.Time 타입으로 파싱이 된다.
* 텍스트에 Emoji가 사용된다면 연결시 charset=utf8mb4 관련 옵션을 설정해줘야 한다.

## Close
```go
// on program termination
if sqlDB != nil {
    sqlDB.Close()
    sqlDB = nil
}
```

## Retain Connection
```go
func pingMysqlHeartbeat() {
    var ts string
    err := sqlDB.QueryRow("SELECT CURRENT_TIMESTAMP FROM DUAL").Scan(&ts)
    if err != nil {
        log.Printf("Heartbeat SQL failed: %v\n", err)
    } else {
        log.Println("Heartbeat OK:", ts)
    }
}

import "github.com/robfig/cron"

cr := cron.New()
cr.AddFunc("@hourly", pingMysqlHeartbeat)
cr.Start()
```

## Query One Row
```go
type userT struct {
    Id        int       `db:"id"`
    CreatedAt time.Time `db:"created_at"`
    Account   string    `db:"account"`
    IsActive  bool      `db:"active"`
    Name      string    `db:"name"`
}

var user userT
err := sqlDB.QueryRowx("SELECT id, name, created_at FROM user WHERE active=TRUE AND account=?", acnt)
    .StructScan(&user)
if err != nil {
    fmt.Printf("SeletUser(%s) failed: %v", acnt, err)
    return
}

sqlDB.QueryRow("SELECT id, name, created_at FROM user WHERE active=TRUE AND account=?", acnt)
    .Scan(&user.Id, &user.Name)

```

## Query Many Rows
```go
rows, err := sqlDB.Queryx("SELECT id, name FROM user")
if err != nil {
    fmt.Printf("Select failed: %v", err)
    return err
}

var users []*userT
for rows.Next() {
    var usr userT
    err = rows.StructScan(&usr)
    if err != nil {
        fmt.Printf("StructScan failed: %v", err)
        rows.Close()
        return err
    }

    users = append(users, &usr)
}
```
## Exec
```go
func isDupKeyError(err error) bool {
    me, ok := err.(*mysql.MySQLError)
    return (ok && me.Number == 1062)
}

res, err = sqlDB.Exec("INSERT INTO user(id, name) VALUES(?,?)", id, name)
if err != nil {
    if isDupKeyError(err) {
        return err
    }
    fmt.Printf("Insert(%d,%s) failed: %v\n", id, name, err)
    return err
}

lastId, err := res.LastInsertId()
if err != nil {
    log.Fatal(err)
}

ra, err := res.RowsAffected()
if err != nil {
    fmt.Printf("RowsAffected error: %v\n", affected)
    return err
}

fmt.Printf("RowsAffected: %d\n", ra)
```
## Stored Procedure
```sql
CREATE PROCEDURE `spGetUserId`
(
    IN _pid INT,
    IN _unq VARCHAR(45)
)
BEGIN
    DECLARE user_id INT;
    
    SELECT id INTO user_id FROM user_pk WHERE project_id=_pid AND uniq=_unq;
    IF FOUND_ROWS() = 0 THEN
        BEGIN
            INSERT INTO user_pk (project_id, uniq) VALUES(_pid,_unq);
            SET user_id = LAST_INSERT_ID();
        END;
    END IF;
    SELECT user_id;
END
```
```go
var userId int32
err := sqlDB.QueryRow("CALL spGetUserId(?,?)", projectId, keyval).Scan(&userId)
if err != nil {
    return 0, err
}
return userId, nil
```

## Prepared Statement
```go
stmt, err := sqlDB.Prepare("INSERT INTO user(id, name) VALUES(?,?)")
if err != nil {
    log.Fatal(err)
}
defer stmt.Close()

res, err = stmt.Exec(id, name)
if err != nil {
    log.Fatal(err)
}
```