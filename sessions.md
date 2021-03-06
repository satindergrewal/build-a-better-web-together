# Sessions

Iris provides a fast, fully featured and easy to use sessions manager.

Iris sessions manager lives on its own [kataras/iris/sessions](https://github.com/kataras/iris/tree/master/sessions) package.

Some trivial examples,

- [Overview](https://github.com/kataras/iris/blob/master/_examples/sessions/overview/main.go)
- [Standalone](https://github.com/kataras/iris/blob/master/_examples/sessions/standalone/main.go)
- [Secure Cookie](https://github.com/kataras/iris/blob/master/_examples/sessions/securecookie/main.go)
- [Flash Messages](https://github.com/kataras/iris/blob/master/_examples/sessions/flash-messages/main.go)
- [Databases](https://github.com/kataras/iris/tree/master/_examples/sessions/database)
    * [File](https://github.com/kataras/iris/blob/master/_examples/sessions/database/file/main.go)
    * [BoltDB](https://github.com/kataras/iris/blob/master/_examples/sessions/database/boltdb/main.go)
    * [LevelDB](https://github.com/kataras/iris/blob/master/_examples/sessions/database/leveldb/main.go)
    * [Redis](https://github.com/kataras/iris/blob/master/_examples/sessions/database/redis/main.go)

## Overview

```go
import "github.com/kataras/iris/sessions"

sess := sessions.Start(http.ResponseWriter, *http.Request)
sess.
  ID() string
  Get(string) interface{}
  HasFlash() bool
  GetFlash(string) interface{}
  GetFlashString(string) string
  GetString(key string) string
  GetInt(key string) (int, error)
  GetInt64(key string) (int64, error)
  GetFloat32(key string) (float32, error)
  GetFloat64(key string) (float64, error)
  GetBoolean(key string) (bool, error)
  GetAll() map[string]interface{}
  GetFlashes() map[string]interface{}
  VisitAll(cb func(k string, v interface{}))
  Set(string, interface{})
  SetImmutable(key string, value interface{})
  SetFlash(string, interface{})
  Delete(string)
  Clear()
  ClearFlashes()
```

This example will show how to store data from a session.

You don't need any third-party library except Iris, but if you want you can use anything, remember Iris is fully compatible with the standard library. You can find a more detailed examples by pressing [here](https://github.com/kataras/iris/tree/master/_examples/sessions).

In this example we will only allow authenticated users to view our secret message on the `/secret` age. To get access to it, the will first have to visit `/login` to get a valid session cookie, hich logs him in. Additionally he can visit `/logout` to revoke his access to our secret message.

```go
// sessions.go
package main

import (
    "github.com/kataras/iris"

    "github.com/kataras/iris/sessions"
)

var (
    cookieNameForSessionID = "mycookiesessionnameid"
    sess                   = sessions.New(sessions.Config{Cookie: cookieNameForSessionID})
)

func secret(ctx iris.Context) {
    // Check if user is authenticated
    if auth, _ := sess.Start(ctx).GetBoolean("authenticated"); !auth {
        ctx.StatusCode(iris.StatusForbidden)
        return
    }

    // Print secret message
    ctx.WriteString("The cake is a lie!")
}

func login(ctx iris.Context) {
    session := sess.Start(ctx)

    // Authentication goes here
    // ...

    // Set user as authenticated
    session.Set("authenticated", true)
}

func logout(ctx iris.Context) {
    session := sess.Start(ctx)

    // Revoke users authentication
    session.Set("authenticated", false)
}

func main() {
    app := iris.New()

    app.Get("/secret", secret)
    app.Get("/login", login)
    app.Get("/logout", logout)

    app.Run(iris.Addr(":8080"))
}

```

```bash
$ go run sessions.go

$ curl -s http://localhost:8080/secret
Forbidden

$ curl -s -I http://localhost:8080/login
Set-Cookie: mysessionid=MTQ4NzE5Mz...

$ curl -s --cookie "mysessionid=MTQ4NzE5Mz..." http://localhost:8080/secret
The cake is a lie!
```

## Backend storage

Sometimes you need a back-end storage, i.e a file storage or a redis one, which will keep your session data on server restarts.

**Registering a database is ridiculous easy by using a single call `.UseDatabase(database)`**.


Let's view a simple example using the fast key-value storage [bolt db](https://github.com/boltdb/bolt).

```go
package main

import (
    "time"

    "github.com/kataras/iris"

    "github.com/kataras/iris/sessions"
    "github.com/kataras/iris/sessions/sessiondb/boltdb"
)

func main() {
    db, _ := boltdb.New("./sessions/sessions.db", 0666, "users")
    // use different go routines to sync the database
    db.Async(true)

    // close and unlock the database when control+C/cmd+C pressed
    iris.RegisterOnInterrupt(func() {
        db.Close()
    })

    sess := sessions.New(sessions.Config{
        Cookie:  "sessionscookieid",
        Expires: 45 * time.Minute, // <=0 means unlimited life
    })

    //
    // IMPORTANT:
    //
    sess.UseDatabase(db)

    // the rest of the code stays the same.
    app := iris.New()

    app.Get("/", func(ctx iris.Context) {
        ctx.Writef("You should navigate to the /set, /get, /delete, /clear,/destroy instead")
    })
    app.Get("/set", func(ctx iris.Context) {
        s := sess.Start(ctx)
        //set session values
        s.Set("name", "iris")

        //test if setted here
        ctx.Writef("All ok session setted to: %s", s.GetString("name"))
    })

    app.Get("/set/{key}/{value}", func(ctx iris.Context) {
        key, value := ctx.Params().Get("key"), ctx.Params().Get("value")
        s := sess.Start(ctx)
        // set session values
        s.Set(key, value)

        // test if setted here
        ctx.Writef("All ok session setted to: %s", s.GetString(key))
    })

    app.Get("/get", func(ctx iris.Context) {
        // get a specific key, as string, if no found returns just an empty string
        name := sess.Start(ctx).GetString("name")

        ctx.Writef("The name on the /set was: %s", name)
    })

    app.Get("/get/{key}", func(ctx iris.Context) {
        // get a specific key, as string, if no found returns just an empty string
        name := sess.Start(ctx).GetString(ctx.Params().Get("key"))

        ctx.Writef("The name on the /set was: %s", name)
    })

    app.Get("/delete", func(ctx iris.Context) {
        // delete a specific key
        sess.Start(ctx).Delete("name")
    })

    app.Get("/clear", func(ctx iris.Context) {
        // removes all entries
        sess.Start(ctx).Clear()
    })

    app.Get("/destroy", func(ctx iris.Context) {
        //destroy, removes the entire session data and cookie
        sess.Destroy(ctx)
    })

    app.Get("/update", func(ctx iris.Context) {
        // updates expire date with a new date
        sess.ShiftExpiration(ctx)
    })

    app.Run(iris.Addr(":8080"))
}
```

### Creating a custom backend sessions storage

You can create your own backend storage by implementing the `Database interface`.

```go
type Database interface {
    Load(sid string) returns struct {
        // Values contains the whole memory store, this store
        // contains the current, updated from memory calls,
        // session data (keys and values). This way
        // the database has access to the whole session's data
        // every time.
        Values memstore.Store
        // on insert it contains the expiration datetime
        // on update it contains the new expiration datetime(if updated or the old one)
        // on delete it will be zero
        // on clear it will be zero
        // on destroy it will be zero
        Lifetime LifeTime
    }

    Sync(accepts struct {
        // Values contains the whole memory store, this store
        // contains the current, updated from memory calls,
        // session data (keys and values). This way
        // the database has access to the whole session's data
        // every time.
        Values memstore.Store
        // on insert it contains the expiration datetime
        // on update it contains the new expiration datetime(if updated or the old one)
        // on delete it will be zero
        // on clear it will be zero
        // on destroy it will be zero
        Lifetime LifeTime
    })
}
```

That's how the boltdb sessions database looks like

```go
package boltdb

import (
    "bytes"
    "os"
    "path/filepath"
    "runtime"
    "time"

    "github.com/boltdb/bolt"
    "github.com/kataras/golog"
    "github.com/kataras/iris/core/errors"
    "github.com/kataras/iris/sessions"
)

// DefaultFileMode used as the default database's "fileMode"
// for creating the sessions directory path, opening and write
// the session boltdb(file-based) storage.
var (
    DefaultFileMode = 0666
)

// Database the BoltDB(file-based) session storage.
type Database struct {
    table []byte
    // Service is the underline BoltDB database connection,
    // it's initialized at `New` or `NewFromDB`.
    // Can be used to get stats.
    Service *bolt.DB
    async   bool
}

var (
    // ErrOptionsMissing returned on `New` when path or tableName are empty.
    ErrOptionsMissing = errors.New("required options are missing")
)

// New creates and returns a new BoltDB(file-based) storage
// instance based on the "path".
// Path should include the filename and the directory(aka fullpath), i.e sessions/store.db.
//
// It will remove any old session files.
func New(path string, fileMode os.FileMode, bucketName string) (*Database, error) {
    if path == "" || bucketName == "" {
        return nil, ErrOptionsMissing
    }

    if fileMode <= 0 {
        fileMode = os.FileMode(DefaultFileMode)
    }

    // create directories if necessary
    if err := os.MkdirAll(filepath.Dir(path), fileMode); err != nil {
        golog.Errorf("error while trying to create the necessary directories for %s: %v", path, err)
        return nil, err
    }

    service, err := bolt.Open(path, 0600,
        &bolt.Options{Timeout: 15 * time.Second},
    )

    if err != nil {
        golog.Errorf("unable to initialize the BoltDB-based session database: %v", err)
        return nil, err
    }

    return NewFromDB(service, bucketName)
}

// NewFromDB same as `New` but accepts an already-created custom boltdb connection instead.
func NewFromDB(service *bolt.DB, bucketName string) (*Database, error) {
    if bucketName == "" {
        return nil, ErrOptionsMissing
    }
    bucket := []byte(bucketName)

    service.Update(func(tx *bolt.Tx) (err error) {
        _, err = tx.CreateBucketIfNotExists(bucket)
        return
    })

    db := &Database{table: bucket, Service: service}

    runtime.SetFinalizer(db, closeDB)
    return db, db.Cleanup()
}

// Cleanup removes any invalid(have expired) session entries,
// it's being called automatically on `New` as well.
func (db *Database) Cleanup() error {
    err := db.Service.Update(func(tx *bolt.Tx) error {
        b := db.getBucket(tx)
        c := b.Cursor()
        for k, v := c.First(); k != nil; k, v = c.Next() {
            if len(k) == 0 { // empty key, continue to the next pair
                continue
            }

            storeDB, err := sessions.DecodeRemoteStore(v)
            if err != nil {
                continue
            }

            if storeDB.Lifetime.HasExpired() {
                if err := c.Delete(); err != nil {
                    golog.Warnf("troubles when cleanup a session remote store from BoltDB: %v", err)
                }
            }
        }

        return nil
    })

    return err
}

// Async if true passed then it will use different
// go routines to update the BoltDB(file-based) storage.
func (db *Database) Async(useGoRoutines bool) *Database {
    db.async = useGoRoutines
    return db
}

// Load loads the sessions from the BoltDB(file-based) session storage.
func (db *Database) Load(sid string) (storeDB sessions.RemoteStore) {
    bsid := []byte(sid)
    err := db.Service.View(func(tx *bolt.Tx) (err error) {
        // db.getSessBucket(tx, sid)
        b := db.getBucket(tx)
        c := b.Cursor()
        for k, v := c.First(); k != nil; k, v = c.Next() {
            if len(k) == 0 { // empty key, continue to the next pair
                continue
            }

            if bytes.Equal(k, bsid) { // session id should be the name of the key-value pair
                storeDB, err = sessions.DecodeRemoteStore(v) // decode the whole value, as a remote store
                break
            }
        }
        return
    })

    if err != nil {
        golog.Errorf("error while trying to load from the remote store: %v", err)
    }

    return
}

// Sync syncs the database with the session's (memory) store.
func (db *Database) Sync(p sessions.SyncPayload) {
    if db.async {
        go db.sync(p)
    } else {
        db.sync(p)
    }
}

func (db *Database) sync(p sessions.SyncPayload) {
    bsid := []byte(p.SessionID)

    if p.Action == sessions.ActionDestroy {
        if err := db.destroy(bsid); err != nil {
            golog.Errorf("error while destroying a session(%s) from boltdb: %v",
                p.SessionID, err)
        }
        return
    }

    s, err := p.Store.Serialize()
    if err != nil {
        golog.Errorf("error while serializing the remote store: %v", err)
    }

    err = db.Service.Update(func(tx *bolt.Tx) error {
        return db.getBucket(tx).Put(bsid, s)
    })
    if err != nil {
        golog.Errorf("error while writing the session bucket: %v", err)
    }
}

func (db *Database) destroy(bsid []byte) error {
    return db.Service.Update(func(tx *bolt.Tx) error {
        return db.getBucket(tx).Delete(bsid)
    })
}

func (db *Database) getBucket(tx *bolt.Tx) *bolt.Bucket {
    return tx.Bucket(db.table)
}

// Len reports the number of sessions that are stored to the this BoltDB table.
func (db *Database) Len() (num int) {
    db.Service.View(func(tx *bolt.Tx) error {
        // Assume bucket exists and has keys
        b := db.getBucket(tx)
        if b == nil {
            return nil
        }

        b.ForEach(func([]byte, []byte) error {
            num++
            return nil
        })
        return nil
    })
    return
}

// Close shutdowns the BoltDB connection.
func (db *Database) Close() error {
    return closeDB(db)
}

func closeDB(db *Database) error {
    err := db.Service.Close()
    if err != nil {
        golog.Warnf("closing the BoltDB connection: %v", err)
    }

    return err
}
```

