# Database Plugins

A database plugin adds a `Database` subclass backed by native code. Xojo code then uses it like any built-in
database: `Connect`, `ExecuteSQL`, `SelectSQL`, `RowSet`, `DatabaseRow`, transactions and `DatabaseException`.

> **Status:** derived from the Plugins SDK 2026r1 headers (`REALplugin.h`, `rb_plugin.h`) only; the SDK ships no
> database example. Nothing on this page is tested yet, so treat it all as *(unverified)*. After the first
> successful build and test, remove this note and mark only what's still unknown.

## When to write one

Only when the app must talk to a database engine that Xojo doesn't support (SQLite, PostgreSQL, MySQL/MariaDB,
ODBC and others are built in) **and** the engine has a free C/C++ client library. Check the decision ladder in
the [README](README.md#1-does-the-app-need-a-plugin) first: a REST API, a `Declare` into the client library or an
ODBC driver often does the job without a plugin.

## How it fits together

| Piece | You write | Registered with |
| ----- | --------- | --------------- |
| `struct dbDatabase` | your connection state (handle, last error); the SDK only declares the name | nothing |
| `struct dbCursor` | one result set: rows, current position | nothing |
| `REALdbEngineDefinition` | the functions Xojo calls on the database: run SQL, errors, transactions, close | `REALRegisterDBEngine` |
| `REALdbCursorDefinition` | the functions Xojo calls on a `RowSet`: columns, values, move, close | `REALRegisterDBCursor` |
| a `REALclassDefinition` with `superName` `"Database"` | the class the Xojo code uses, with a `Connect` method and any connection properties | `REALRegisterClass` |

The flow:

1. Xojo code calls `db.Connect`. Your method opens the native connection, allocates a `dbDatabase`, attaches it to
   the Xojo object with `REALConstructDBDatabase` and marks it connected with `REALSetDBIsConnected`.
2. `db.ExecuteSQL(sql, values…)` and `db.SelectSQL(sql, values…)` arrive in the engine's `ExecuteSQL` and `SelectSQL`
   with the `dbDatabase *` and the values as an array of Variants.
3. `SelectSQL` builds a `dbCursor` and returns it wrapped by `REALNewRowSetFromDBCursor`. Xojo then calls the
   cursor functions while the code walks the `RowSet`, and `closeCursor` when it's done.
4. `db.Close`, or the object being destroyed, calls the engine's `closeDatabase`, where you close the native
   connection and free the `dbDatabase`.

## REALdbEngineDefinition

Set `version` to `kCurrentREALDatabaseVersion`. Every function is optional; leave the ones you don't support as
`nullptr`. Zero-initialize the struct and assign fields by name in `PluginEntry` (see the example). It has 31
fields, and assigning by name is much safer than a positional initializer.

| Field | Called for | Signature |
| ----- | ---------- | --------- |
| `version` | | `kCurrentREALDatabaseVersion` |
| `forSystemUse` | | `0` |
| `flags1` | schema editor capabilities: `dbEnginePrimaryKeySupported`, `dbEngineAlterTableAddParens`, `dbEngineDontUseBrackets`, `dbEngineColumnDeleteSupported`, `dbEngineColumnModificationSupported`, `dbEngineColumnOptionsSupported` | `unsigned char` bit flags |
| `flags2`, `flags3` | | `0` |
| `closeDatabase` | `Close`, object destroyed | `void (dbDatabase *)` |
| `getTableSchemaCursor` | `Tables` | `REALdbCursor (dbDatabase *)` |
| `getFieldSchemaCursor` | `TableColumns(table)` | `REALdbCursor (dbDatabase *, REALstring table)` |
| `directSQLSelect` | older select without values | `REALdbCursor (dbDatabase *, REALstring sql)` |
| `directSQLExecute` | older execute without values | `void (dbDatabase *, REALstring sql)` |
| `createTable` | IDE schema editor | `void (dbDatabase *, REALstring name, REALnewColumn *columns, unsigned char *primaryKey, int primaryKeyCount)` |
| `addTableRecord` | `AddRow(table, row)` | `void (dbDatabase *, REALstring table, REALcolumnValue *values)` |
| `SelectSQL` | `SelectSQL(sql, values…)` | `REALdbCursor (dbDatabase *, REALstring sql, REALarray params)` |
| `updateFields` | legacy record editing | `void (dbDatabase *, REALfieldUpdate *fields)` |
| `addTableColumn` | IDE schema editor | `void (dbDatabase *, REALstring table, REALnewColumn *column)` |
| `getDatabaseIndexes` | `TableIndexes(table)` | `REALdbCursor (dbDatabase *, REALstring table)` |
| `getLastErrorCode` | error number | `long (dbDatabase *)` |
| `getLastErrorString` | error message | `REALstring (dbDatabase *)` |
| `commit` | `CommitTransaction` | `void (dbDatabase *)` |
| `rollback` | `RollbackTransaction` | `void (dbDatabase *)` |
| `BeginTransaction` | `BeginTransaction` | `void (dbDatabase *)` |
| `getSupportedTypes` | IDE schema editor type list | `void (int32_t **dataTypes, char **dataNames, size_t *count)` |
| `dropTable` | IDE schema editor | `void (dbDatabase *, REALstring table)` |
| `dropColumn` | IDE schema editor | `void (dbDatabase *, REALstring table, REALstring column)` |
| `alterTableName` | IDE schema editor | `void (dbDatabase *, REALstring oldName, REALstring newName)` |
| `alterColumnName` | IDE schema editor | `void (dbDatabase *, REALstring table, REALstring oldName, REALstring newName)` |
| `alterColumnType` | IDE schema editor | `void (dbDatabase *, REALstring table, REALstring column, REALstring oldType, REALstring newType)` |
| `alterColumnConstraint` | IDE schema editor | `void (dbDatabase *, REALstring table, REALstring column, REALstring which, REALstring oldValue, REALstring newValue)` |
| `PrepareStatement` | `Prepare(sql)` | `REALobject (dbDatabase *, REALstring sql)`, returns a `PreparedSQLStatement` object |
| `ExecuteSQL` | `ExecuteSQL(sql, values…)` | `void (dbDatabase *, REALstring sql, REALarray params)` |
| `AddRowWithReturnValue` | `AddRow(table, row, idColumn)` (2024r2+) | `RBInteger (dbDatabase *, REALstring table, REALcolumnValue *values, REALstring idColumnName)` |

A minimal engine sets `closeDatabase`, `SelectSQL`, `ExecuteSQL`, `getLastErrorCode` and `getLastErrorString`.
Set `directSQLSelect` and `directSQLExecute` too, pointing them to the same code with no values, in case a call
without values arrives there. Add the transaction functions when the engine supports them.

`params` is an array of Variants, one per value passed after the SQL. It can be empty or `nullptr`. Read it with
`REALGetArrayUBound` and `REALGetArrayValueObject`, then read each Variant with `REALGetPropValueString(v,
"StringValue", &s)`, `"Int64Value"`, `"DoubleValue"`, `"BooleanValue"` or `"Type"`. Bind them to the native
statement; never paste values into the SQL text.

`REALnewColumn` (linked list): `nextColumn`, `columnName`, `columnType` (a `dbFieldType`), `bAllowNULL`,
`columnTypeString` (e.g. `"Decimal(5,2)"`). `REALcolumnValue` (linked list): `nextColumn`, `columnName`,
`columnValue` (a string), `columnType`.

## REALdbCursorDefinition

Set `version` to `kCurrentREALDatabaseVersion`. The cursor starts **on the first row**, or at EOF when there are no
rows.

| Field | Called for | Signature |
| ----- | ---------- | --------- |
| `version` | | `kCurrentREALDatabaseVersion` |
| `forSystemUse` | | `0` |
| `closeCursor` | `RowSet.Close`, RowSet destroyed | `void (dbCursor *)`: free the cursor |
| `cursorColumnCount` | `ColumnCount` | `int (dbCursor *)` |
| `cursorColumnName` | `ColumnAt(i).Name`, `Column("name")` lookup | `REALstring (dbCursor *, int column)`, 0-based; return a new string |
| `cursorRowCount` | `RowCount` | `int (dbCursor *)`, or `-1` if unknown |
| `cursorColumnValue` | reading a column of the current row | `void (dbCursor *, int column, void **value, unsigned char *type, int *length)` |
| `cursorReleaseValue` | after Xojo copied a value | `void (dbCursor *)`: free a buffer you made in `cursorColumnValue` |
| `cursorNextRow` | `MoveToNextRow`, `For Each` | `RBBoolean (dbCursor *)`: move on; return true while a row is current |
| `cursorDelete` | `RemoveRow` | `void (dbCursor *)` |
| `cursorDeleteAll` | legacy | `void (dbCursor *)` |
| `cursorFieldKey` | legacy editing | `RBBoolean (dbCursor *, int, void **, int *, void **, int *)` |
| `cursorUpdate` | `SaveRow` after `EditRow` | `void (dbCursor *, REALcursorUpdate *fields)` |
| `cursorEdit` | `EditRow` | `void (dbCursor *)` |
| `cursorPrevRow` | `MoveToPreviousRow` | `void (dbCursor *)` |
| `cursorFirstRow` | `MoveToFirstRow` | `void (dbCursor *)` |
| `cursorLastRow` | `MoveToLastRow` | `void (dbCursor *)` |
| `cursorColumnType` | `ColumnType(i)` | `int (dbCursor *, int column)`: a `dbFieldType` |
| `CursorIsBOF` | `BeforeFirstRow` | `RBBoolean (dbCursor *)` |
| `CursorIsEOF` | `AfterLastRow` | `RBBoolean (dbCursor *)` |
| `dummy10` | | `nullptr` |

A forward-only cursor needs `closeCursor`, `cursorColumnCount`, `cursorColumnName`, `cursorColumnValue`,
`cursorNextRow`, `cursorColumnType` and `CursorIsEOF`. Leave the editing functions `nullptr` for a read-only
RowSet.

### Column values

`cursorColumnValue` sets three outputs for the current row:

- `*type`: a `dbFieldType` (`dbTypeNull`, `dbTypeText`, `dbTypeInt32`, `dbTypeInt64`, `dbTypeDouble`, `dbTypeBoolean`,
  `dbTypeDate`, `dbTypeTime`, `dbTypeTimeStamp`, `dbTypeBinary`, `dbTypeCurrency`, `dbTypeDecimal`…);
- `*value`: a pointer to the data. Text and binary point at the bytes (text is UTF-8); numbers point at a C value of
  that type (`int32_t`, `int64_t`, `double`); dates point at a `dbDate`, `dbTime` or `dbTimeStamp` struct (`short`
  fields `year`, `month`, `day`, `hour`, `minute`, `second`);
- `*length`: the byte count of the data.

For SQL `NULL`, set `*type = dbTypeNull`, `*value = nullptr` and `*length = 0`. The data must stay valid until
`cursorReleaseValue` or the next call on the cursor. Pointing into the cursor's own row storage meets that. Don't
use `dbTypeLongText`, `dbTypeLongBinary` or `dbTypeMacPICT`: they're deprecated.

## The Database subclass

```cpp
static REALmethodDefinition MyDBMethods[] = {
	{ (REALproc)MyDBConnect, REALnoImplementation, "Connect() As Boolean", REALconsoleSafe },
};

static REALproperty MyDBProperties[] = {   // connection settings the Xojo code fills in before Connect
	{ "", "FilePath", "String", REALconsoleSafe, REALstandardGetter, REALstandardSetter, FieldOffset(MyDBData, filePath) },
};
```

- `superName` is `"Database"`. Then `Close`, `ExecuteSQL`, `SelectSQL`, the transaction methods and `IsConnected`
  come from the framework, which calls your engine.
- `Database` already has `Host`, `DatabaseName`, `UserName` and `Password`. Read them in `Connect` with
  `REALGetDBHost`, `REALGetDBDatabaseName`, `REALGetDBUserName` and `REALGetDBPassword` (each returns a string you
  unlock). Add properties only for other settings.
- `Connect` opens the native connection. On success it calls `REALConstructDBDatabase((REALdbDatabase)instance,
  db, &gEngine)` and `REALSetDBIsConnected((REALdbDatabase)instance, true)`, then returns true. On failure it
  raises `DatabaseException` (see below).
- From a `REALdbDatabase` you get your struct back with `REALGetDBFromREALdbDatabase`, and from a RowSet's
  `REALdbCursor` with `REALGetCursorFromREALdbCursor`.
- If the compiler rejects `Connect() As Boolean` as clashing with the framework's `Connect`, ask the user for the
  exact error text, then match the framework signature it names.

## Errors

Keep the last error number and message in your `dbDatabase`, and return them from `getLastErrorCode` and
`getLastErrorString`. When a call fails, also raise a `DatabaseException` with `ErrorNumber` and `Message` set,
then return (`nullptr` from `SelectSQL`):

```cpp
static REALclassRef gDatabaseException;   // set in PluginEntry: REALGetClassRef("DatabaseException")

static void RaiseDBError(dbDatabase *db, RBInteger code, const std::string &message)
{
	db->errorCode = code;
	db->errorMessage = message;
	REALobject err = REALnewInstanceWithClass(gDatabaseException);
	REALstring msg = MakeString(message);
	REALSetPropValueString(err, "Message", msg);
	REALUnlockString(msg);
	REALSetPropValueInteger(err, "ErrorNumber", code);
	REALRaiseException(err);
}
```

Clear the error at the start of every call. Never let a native library's C++ exception escape into Xojo.

## Threads

Engine and cursor functions run on the calling Xojo thread. A long-running native query blocks every Xojo thread
until it returns (see [Writing.md](Writing.md#threads)). Use a native async API when there is one, polling it
with `REALYieldToRB`, or keep queries short.

## Linking the client library

Link the engine's client library statically when its license allows it. Otherwise ship it in `Libs/`. See
[Building.md](Building.md#dependency-libraries-libs). Xojo's own database plugins do both.

## Not covered here

- IDE integration: the connection dialog (`REALDBConnectionDialog…`), data sources (`REALRegisterDataSource`,
  `REALRegisterDataSourceInterface`, `REALDesignAddDataSource`) and `REALRegisterDatabaseConnection`.
- Prepared statements (`PrepareStatement` returns a `PreparedSQLStatement` subclass you define).
- `REALdbTableDefinition`, `REALRegisterDBTable` and the other legacy editing paths.

## Example: an in-memory key/value database

`MemoryDatabase` keeps key/value pairs in memory. It understands two commands instead of SQL, so the code shows the
plugin mechanics without a parser:

- `db.ExecuteSQL("PUT", key, value)` stores a pair;
- `db.SelectSQL("ALL")` returns every pair as a RowSet with the columns `key` and `value`, sorted by key;
- anything else raises `DatabaseException`;
- transactions work: `BeginTransaction` takes a snapshot, `RollbackTransaction` restores it.

### src/MemoryDB.cpp

```cpp
#include "rb_plugin.h"
#include <string.h>
#include <map>
#include <string>
#include <vector>

// ---------------------------------------------------------------------------
// Helpers

static REALclassRef gDatabaseException = nullptr;   // set in PluginEntry

static REALstring MakeString(const std::string &s)
{
	return REALBuildStringWithEncoding(s.data(), (int)s.size(), kREALTextEncodingUTF8);
}

static std::string ToStdString(REALstring s)
{
	std::string out;
	REALstringData data;
	if (s && REALGetStringData(s, kREALTextEncodingUTF8, &data)) {
		out.assign((const char *)data.data, data.length);
		REALDisposeStringData(&data);
	}
	return out;
}

// Value number `index` of a SelectSQL/ExecuteSQL call, as a string
static std::string ParamString(REALarray params, RBInteger index)
{
	std::string out;
	REALobject v = nullptr;
	REALGetArrayValueObject(params, index, &v);
	if (v) {
		REALstring s = nullptr;
		if (REALGetPropValueString(v, "StringValue", &s)) {
			out = ToStdString(s);
			REALUnlockString(s);
		}
		REALUnlockObject(v);
	}
	return out;
}

static RBInteger ParamCount(REALarray params)
{
	return params ? REALGetArrayUBound(params) + 1 : 0;
}

// ---------------------------------------------------------------------------
// Native state

struct dbDatabase {
	std::map<std::string, std::string> data;
	std::map<std::string, std::string> snapshot;   // taken by BeginTransaction
	bool inTransaction = false;
	RBInteger errorCode = 0;
	std::string errorMessage;
};

struct dbCursor {
	std::vector<std::pair<std::string, std::string>> rows;   // copied, so the RowSet outlives later changes
	size_t current = 0;
};

static void ClearError(dbDatabase *db)
{
	db->errorCode = 0;
	db->errorMessage.clear();
}

static void RaiseDBError(dbDatabase *db, RBInteger code, const std::string &message)
{
	db->errorCode = code;
	db->errorMessage = message;
	REALobject err = REALnewInstanceWithClass(gDatabaseException);
	REALstring msg = MakeString(message);
	REALSetPropValueString(err, "Message", msg);
	REALUnlockString(msg);
	REALSetPropValueInteger(err, "ErrorNumber", code);
	REALRaiseException(err);
}

// ---------------------------------------------------------------------------
// Cursor

static REALdbCursorDefinition gCursor = {};   // filled in PluginEntry

static void CloseCursor(dbCursor *cursor)
{
	delete cursor;
}

static int CursorColumnCount(dbCursor *)
{
	return 2;
}

static REALstring CursorColumnName(dbCursor *, int column)
{
	return MakeString(column == 0 ? "key" : "value");
}

static int CursorRowCount(dbCursor *cursor)
{
	return (int)cursor->rows.size();
}

static void CursorColumnValue(dbCursor *cursor, int column, void **value, unsigned char *type, int *length)
{
	if (cursor->current >= cursor->rows.size() || column < 0 || column > 1) {
		*value = nullptr;
		*type = dbTypeNull;
		*length = 0;
		return;
	}
	const std::string &s = column == 0 ? cursor->rows[cursor->current].first : cursor->rows[cursor->current].second;
	*value = (void *)s.data();     // stays valid: the cursor owns the row
	*type = dbTypeText;
	*length = (int)s.size();
}

static RBBoolean CursorNextRow(dbCursor *cursor)
{
	if (cursor->current < cursor->rows.size()) cursor->current++;
	return cursor->current < cursor->rows.size();
}

static int CursorColumnType(dbCursor *, int)
{
	return dbTypeText;
}

static RBBoolean CursorIsBOF(dbCursor *cursor)
{
	return cursor->rows.empty();
}

static RBBoolean CursorIsEOF(dbCursor *cursor)
{
	return cursor->current >= cursor->rows.size();
}

// ---------------------------------------------------------------------------
// Engine

static REALdbEngineDefinition gEngine = {};   // filled in PluginEntry

static void CloseDatabase(dbDatabase *db)
{
	delete db;
}

static REALdbCursor SelectSQL(dbDatabase *db, REALstring sql, REALarray)
{
	ClearError(db);
	if (ToStdString(sql) != "ALL") {
		RaiseDBError(db, 1, "unsupported command (use ALL)");
		return nullptr;
	}
	dbCursor *cursor = new dbCursor;
	for (const auto &pair : db->data) cursor->rows.push_back(pair);
	return REALNewRowSetFromDBCursor(cursor, &gCursor);
}

static REALdbCursor DirectSQLSelect(dbDatabase *db, REALstring sql)
{
	return SelectSQL(db, sql, nullptr);
}

static void ExecuteSQL(dbDatabase *db, REALstring sql, REALarray params)
{
	ClearError(db);
	if (ToStdString(sql) != "PUT") {
		RaiseDBError(db, 1, "unsupported command (use PUT)");
		return;
	}
	if (ParamCount(params) != 2) {
		RaiseDBError(db, 2, "PUT needs a key and a value");
		return;
	}
	db->data[ParamString(params, 0)] = ParamString(params, 1);
}

static void DirectSQLExecute(dbDatabase *db, REALstring sql)
{
	ExecuteSQL(db, sql, nullptr);
}

static long GetLastErrorCode(dbDatabase *db)
{
	return (long)db->errorCode;
}

static REALstring GetLastErrorString(dbDatabase *db)
{
	return MakeString(db->errorMessage);
}

static void BeginTransaction(dbDatabase *db)
{
	db->snapshot = db->data;
	db->inTransaction = true;
}

static void Commit(dbDatabase *db)
{
	db->snapshot.clear();
	db->inTransaction = false;
}

static void Rollback(dbDatabase *db)
{
	if (db->inTransaction) db->data = db->snapshot;
	db->snapshot.clear();
	db->inTransaction = false;
}

// ---------------------------------------------------------------------------
// Class MemoryDatabase (Super Database)

static RBBoolean MemoryConnect(REALobject instance)
{
	REALdbDatabase dbObject = (REALdbDatabase)instance;
	if (REALGetDBFromREALdbDatabase(dbObject)) return true;   // already connected
	REALConstructDBDatabase(dbObject, new dbDatabase, &gEngine);
	REALSetDBIsConnected(dbObject, true);
	return true;
}

static REALmethodDefinition MemoryDatabaseMethods[] = {
	{ (REALproc)MemoryConnect, REALnoImplementation, "Connect() As Boolean", REALconsoleSafe },
};

static REALclassDefinition MemoryDatabaseDefinition = {
	kCurrentREALControlVersion,
	"MemoryDatabase",
	"Database",                   // superName
	0,                            // dataSize: state lives in dbDatabase
	0,                            // forSystemUse
	nullptr,                      // constructor
	nullptr,                      // destructor
	nullptr, 0,                   // properties
	MemoryDatabaseMethods, _countof(MemoryDatabaseMethods),
};

// ---------------------------------------------------------------------------

void PluginEntry(void)
{
	gDatabaseException = REALGetClassRef("DatabaseException");

	gCursor.version = kCurrentREALDatabaseVersion;
	gCursor.closeCursor = CloseCursor;
	gCursor.cursorColumnCount = CursorColumnCount;
	gCursor.cursorColumnName = CursorColumnName;
	gCursor.cursorRowCount = CursorRowCount;
	gCursor.cursorColumnValue = CursorColumnValue;
	gCursor.cursorNextRow = CursorNextRow;
	gCursor.cursorColumnType = CursorColumnType;
	gCursor.CursorIsBOF = CursorIsBOF;
	gCursor.CursorIsEOF = CursorIsEOF;
	REALRegisterDBCursor(&gCursor);

	gEngine.version = kCurrentREALDatabaseVersion;
	gEngine.closeDatabase = CloseDatabase;
	gEngine.directSQLSelect = DirectSQLSelect;
	gEngine.directSQLExecute = DirectSQLExecute;
	gEngine.SelectSQL = SelectSQL;
	gEngine.ExecuteSQL = ExecuteSQL;
	gEngine.getLastErrorCode = GetLastErrorCode;
	gEngine.getLastErrorString = GetLastErrorString;
	gEngine.BeginTransaction = BeginTransaction;
	gEngine.commit = Commit;
	gEngine.rollback = Rollback;
	REALRegisterDBEngine(&gEngine);

	SetClassConsoleSafe(&MemoryDatabaseDefinition);
	REALRegisterClass(&MemoryDatabaseDefinition);
}
```

### Build and package

The same as [Example.md](Example.md#build-and-package-windows-x64), with `MemoryDB` in place of `TextTools`. It
needs no extra libraries.

### Using it from Xojo

Once the user confirms that `MemoryDatabase` autocompletes:

```xojo
Var db As New MemoryDatabase
If db.Connect Then
  db.ExecuteSQL("PUT", "name", "Ada")
  db.ExecuteSQL("PUT", "lang", "Xojo")

  db.BeginTransaction
  db.ExecuteSQL("PUT", "temp", "gone")
  db.RollbackTransaction                          // "temp" is removed again

  For Each row As DatabaseRow In db.SelectSQL("ALL")
    System.DebugLog(row.Column("key").StringValue + " = " + row.Column("value").StringValue)
  Next

  Try
    db.ExecuteSQL("DROP TABLE x")
  Catch err As DatabaseException
    System.DebugLog(err.ErrorNumber.ToString + ": " + err.Message)
  End Try

  db.Close
End If
```

Expected output in the Messages pane: `lang = Xojo`, `name = Ada`, then `1: unsupported command (use PUT)`.
