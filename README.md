# GoFig
GoFig is a tool for managing Firestore database migrations. This project aims to achieve the following:
1. The end user will have visibility into all pending migration changes and effects in relation to the active state of their database. The user can then decide to push or cancel the migration.
2. When a migration is pushed to the database, all the changes presented to the end user are implemented and no other changes/effects are implemented by the migrator.
3. All changes made by the migrator in a migration job can be completely reversed by staging and pushing the generated `_rollback` migration file.

## Install
```
go get github.com/aaronhough/GoFig
```
## Initialize Migrator
The migrator is initialized with a few configuration parameters. If your storage path starts with `[firestore]/`, migrations will be stored on your database (suffix path must be a collection). The `Name` value will be the file/doc name so it should only contain alphanumeric digits and underscores/dashes `A-Z,a-z,0-9,_,-`.
```go
import fig "github.com/aaronhough/GoFig"

config := fig.Config{
    // The location of your admin key file.
    KeyPath: "~/project/.keys/my-admin-key.json",
    // The location to load/store migration files.
    StoragePath: "~/project/storage",
    // A unique name for this migration.
    Name: "my-migration",
}

fg, err := fig.New(config)

defer fg.Close()
```

## Stage a New Migration
Stage each change into the migrator using the `Stage` utility.
```go
// Sample data.
data := map[string]any{
    "foo": "bar",
    "fiz": false,
    "buz": map[string][]int{
        "a": { 2, 2, 3 },
    },
}

// Stage options are Add, Update, Set, and Delete.
fg.Stage().Add("fig/fog", data)
fg.Stage().Update("foo/bar", map[string]string{ "hello": "world" })
```

## Save a Migration To Storage
Save a staged migration to storage then load and run it at a later time.
```Go
// A new file is saved to `StoragePath` folder.
fg.StoreMigration()
```

## Load an Existing Migration
The migrator will search for an existing migration file/doc in the `StoragePath` folder that matches the configured `Name`.
```go
// If a file matching the migration `Name` exists, it will be loaded.
fg.LoadFromStorage()
```
## Run a Migration
The interactive migration shell will present the changes and prompt for confirmation. On confirmation, the changes will be executed against the database and a rollback migration file will be saved to the `StoragePath` folder/collection.
```go
// Launch the interactive shell.
fg.ManageStagedMigration()
```

## Rollback
Locate the `_rollback` file/doc generated by the target migration job. Ensure the migration config matches the name of the rollback file. Load and run the migration.
```go
config := fig.Config{
    KeyPath: "~/project/.keys/my-admin-key.json",
    StoragePath: "~/project/storage",
    // File expected at ~/project/storage/my-migration_rollback.json
    Name: "my-migration_rollback",
}

fg, err := fig.New(config)
defer fg.Close()

fg.LoadFromStorage()
fg.ManageStagedMigration()
```

## Complex types
Note some examples of supported complex types.
```go
data := map[string]any{
    // Represent a document reference object.
    "ref": fg.RefField("fig/fog"),
    // Time objects are handled normally.
    "time": time.Now(),
    // Safely marks a key for deletion.
    "prev": fg.DeleteField(),
}
```

## Expansions
Since the migrator can load any migration file, feel free to use your own languages/scripts to build migrations then load them by path/name with GoFig. You will need to serialize complex types as follows:
- Timestamp: `"<time>2023-05-13T13:44:40.522Z<time>"`
- Document reference: `"<ref>fig/fog<ref>"`
- Delete: `"<delete>!delete<delete>"`

The actual migration file simply needs to host an array of serialized changeUnits. Each change contains a docPath, a patch, and a numeric command. Commands are `0`, `1`, `2`, `3`, `4` which represent `MigratorUnknown`, `MigratorUpdate`, `MigratorSet`, `MigratorAdd`, and `MigratorDelete` respectively.

## To Do
- Abbreviate large diffs in terminal