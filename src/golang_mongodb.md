title: Go MongoDB
---

# Golang MongoDB
---

## Installation

```
$ go get -u github.com/globalsign/mgo
```

## Connect

```go
import (
    "github.com/globalsign/mgo"
    "github.com/globalsign/mgo/bson"
)

var mgoSession *mgo.Session
var mgoDB      *mgo.Database


mgoSession, err = mgo.Dial("mongodb://userid:passwd@example.mongodbsvr.com:27017/dbname")
if err != nil {
    log.Printf("MongoDB Dial failed: %v\n", err)
    return false
}

mgoDB = mgoSession.DB("")
```

## Close
```go
if mgoSession != nil {
    mgoSession.Close()
    mgoSession = nil
}
```

## Query One

### By ID
```go
type userT struct {
    Id        bson.ObjectId `bson:"_id"`
    CreatedAt time.Time     `bson:"created_at"`
    Account   string        `bson:"account"`
    IsActive  bool          `bson:"active"`
    Name      string        `bson:"name"`
}

sessionCopy := mgoSession.Copy()
defer sessionCopy.Close()
coll := mgoDB.With(sessionCopy).C("collection_name")

var user userT
err := coll.FindId(userId).One(&user) // userId type: bson.ObjectId
if err != nil {
    log.Printf("Find(%s) failed: %v", userId.Hex(), err)
    return false
}
```

### By condition
```go
coll.Find(bson.M{"account": acntId}).One(&user)
```

### Specific fields only
Document의 내용 전체를 읽을때 I/O 부하가 있으면 일부 필드만 선택적으로 읽어올 수 있다.
```go
coll.FindId(userId).Select(bson.M{"account": 1, "name": 1}).One(&user)
```

## Query All
```go
var filter = make(bson.M)
filter["active"] = true
filter["name"] = bson.RegEx{Pattern: "John"}})
...
selector := bson.M{"_id": 1}
...
var users []userT
err = coll.Find(filter).Select(selector).Limit(limit).All(&users)
```

## Update
SQL 생각하고 UpdateId 두번째 파라메터에 변경할 필드를 직접 쓰면 다 지워지고 그것으로 셋팅되어 버린다. 반드시 $set, $unset 등을 써야 한다.
```go
err = collAlarm.UpdateId(dataId, bson.M{
    "$set":  bson.M{"modifiedAt": time.Now()},
    "$unset": bson.M{"toBeRemoved": nil},
    "$pull": bson.M{"arrToPop": data1}, // 배열 필드에 값 빼기
    "$addToSet": bson.M{"arrToPush": data2}, // 배열 필드에서 값 넣기
})
```

## $lookup
TBD
```go
sessionCopy := mgoSession.Copy()
defer sessionCopy.Close()
coll := mgoDB.With(sessionCopy).C("coll_a")

query := []bson.M{
    bson.M{"$match": bson.M{"_id": userId}},
    bson.M{"$lookup": bson.M{
        "from":         "coll_b",
        "foreignField": "_id",  // field of <from>
        "localField":   "pics", // field of <coll>
        "as":           "imgs", // field name to generate
    }},
    bson.M{"$unwind": "$imgs"},
    bson.M{"$project": bson.M{"_id": 0, "imgs.baseUri": 1}},
    bson.M{"$replaceRoot": bson.M{"newRoot": "$imgs"}},
}

var imgs []myImageT
err := coll.Pipe(query).All(&imgs)
if err != nil {
    log.Printf("Pipe.All failed: %v", err)
}
```

## Generate sequence number
```go
name := "my_seq_name"

sessionCopy := mgoSession.Copy()
coll := mgoDB.With(sessCopy).C("my_seqs_coll")
result := bson.M{}
_, err := coll.FindId(name).Apply(mgo.Change{
    Update:    bson.M{"$set": bson.M{"_id": name}, "$inc": bson.M{"val": 1}},
    Upsert:    true,
    ReturnNew: true,
}, &result)

if err != nil {
    log.Printf("nextSeq(%s) failed: %v", name, err)
    return 0
}

seq, _ := result["val"].(int)
return seq
```