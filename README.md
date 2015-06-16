# SQLiteGlue-core

Low-level Android &amp; other Java native glue interface to sqlite using Gluegen.

Unlicense (public domain).

## About

SQLiteGlue provides the basic low-level functions necessary to use sqlite from an Android or other
Java application over JNI (Java native interface). This is accomplished by using
[GlueGen](http://jogamp.org/gluegen/www/) around a simple wrapper C module.

This project is meant to help build a higher-level sqlite interface library, with the JNI layer completely isolated.

## How

All constants and sqlite accessor functions are defined in `native/sqlg.h`. The `make regen` rule uses [GlueGen](http://jogamp.org/gluegen/www/) (referenced in a subproject) to (re)generate the following files which _are_ committed and maintained in this repository:
- `java/org/sqlg/SQLiteGlue.java` - provides the single `org.sqlg.SQLiteGlue` object that exports the static native sqlite accessor functions and some constants that may be helpful
- ...

TBD how to install and use.

# Building

## Normal build

*First* initialize with the `gluegentools` & `sqlite-amalgamation` subprojects:

$ `make init`

Then to build:

$ `make`

## Regenerage Java & C glue code

$ `make regen`

# Usage

## Java API Structure

There is a single `org.sqlg.SQLiteGlue` object that provides the sqlite accessor functions as static native functions.

## Open and close a database

To open a database:

```Java
long dbhandle = org.sqlg.SQLiteGlue.sqlg_db_open(fullFilePath, openFlags);
```
where `fullFilePath` is the _string_ path to the file and `openFlags` is the combination of flags used to open the file (using `sqlite3_open_v2()`). The result in `dbhandle` is the `long` handle that can be used to access the database if positive, or the negative value of the sqlite error code returned if negative. Here is an example (from an Android Activity function):

```Java
java.io.File dbfile = new java.io.File(this.getFilesDir(), "my.db");

long dbhandle = org.sqlg.SQLiteGlue.sqlg_db_open(dbfile.getAbsolutePath(),
    org.sqlg.SQLiteGlue.SQLG_OPEN_READWRITE | org.sqlg.SQLiteGlue.SQLG_OPEN_CREATE);

if (dbhandle < 0) {
    android.util.Log.e("MySQLiteApp", "Open failed with sqlite error: " + -dbhandle);
    return;
}
```

**WARNING:** the behavior is undefined if you call any other `SQLiteGlue` functions with a negative database handle.

To close the database handle:

```Java
org.sqlg.SQLiteGlue.sqlg_db_close(dbhandle);
```

## Prepare and run statements

To prepare a statement with no parameters (on Android):

```Java
long sthandle = org.sqlg.SQLiteGlue.sqlg_db_prepare_st(dbhandle,
    "CREATE TABLE mytable (text1 TEXT, num1 INTEGER, num2 INTEGER, real1 REAL)");

if (sthandle < 0) {
    android.util.Log.e("MySQLiteApp", "prepare statement error: " + -sthandle);
    org.sqlg.SQLiteGlue.sqlg_db_close(dbhandle);
    return;
}
```

**WARNING:** Again, if you use a negative statement handle in other `SQLiteGlue` functions the behavior is undefined.

To run the statement with no results expected:

```Java
int stresult = org.sqlg.SQLiteGlue.sqlg_st_step(sthandle);
if (stresult != org.sqlg.SQLiteGlue.SQLG_RESULT_DONE) {
    android.util.Log.e("MySQLiteApp", "unexpected step result: " + stresult);
    org.sqlg.SQLiteGlue.sqlg_st_finish(sthandle);
    org.sqlg.SQLiteGlue.sqlg_db_close(dbhandle);
    return;
}
```

**NOTE:** In the case of operations on existing sqlite statements the returned sqlite result is a positive value.

And to cleanup the statement handle:


```Java
org.sqlg.SQLiteGlue.sqlg_st_finish(sthandle);
```

To prepare an INSERT or UPDATE statement and programmaticallly bind some parameter values:

```Java
long sthandle = org.sqlg.SQLiteGlue.sqlg_db_prepare_st(dbhandle,
    "INSERT INTO mytable (text1, num1, num2, real1) VALUES (?,?,?,?)");

if (sthandle < 0) {
    android.util.Log.e("MySQLiteApp", "prepare statement error: " + -sthandle);
    org.sqlg.SQLiteGlue.sqlg_db_close(dbhandle);
    return;
}

org.sqlg.SQLiteGlue.sqlg_st_bind_text_native(sthandle, 1, "test");
org.sqlg.SQLiteGlue.sqlg_st_bind_int(sthandle, 2, 10100);
org.sqlg.SQLiteGlue.sqlg_st_bind_long(sthandle, 3, 0x1230000abcdL);
org.sqlg.SQLiteGlue.sqlg_st_bind_double(sthandle, 4, 123456.789);
```

Then run the statement and clean it up as described above.

## Get row results

First, prepare the query statement:

```Java

long sthandle = org.sqlg.SQLiteGlue.sqlg_db_prepare_st(dbhandle, "SELECT text1, num1, num2, real1 FROM mytable;");
if (sthandle < 0) {
    android.util.Log.e("MySQLiteApp", "prepare statement error: " + -sthandle);
    org.sqlg.SQLiteGlue.sqlg_db_close(dbhandle);
    return;
}
```

Then to step through the result rows:

```Java
int stresult = org.sqlg.SQLiteGlue.sqlg_st_step(sthandle);

while (stresult == org.sqlg.SQLiteGlue.SQLG_RESULT_ROW) {
    int colcount = org.sqlg.SQLiteGlue.sqlg_st_column_count(sthandle);
    android.util.Log.e("MySQLiteApp", "Row with " + colcount + " columns");

    for (int i=0; i<colcount; ++i) {
        int coltype = SQLiteGlue.sqlg_st_column_type(sthandle, i);
        switch(coltype) {
        case org.sqlg.SQLiteGlue.SQLG_INTEGER:
            android.util.Log.e("MySQLiteApp",
                "Col " + i + " type: INTEGER (long) value: 0x" +
                    java.lang.Long.toHexString(org.sqlg.SQLiteGlue.sqlg_st_column_long(sthandle, i)));
            break;

        case org.sqlg.SQLiteGlue.SQLG_FLOAT:
            android.util.Log.e("MySQLiteApp",
                "Col " + i + " type: FLOAT value: " + org.sqlg.SQLiteGlue.sqlg_st_column_double(sthandle, i));
            break;

        case org.sqlg.SQLiteGlue.SQLG_NULL:
            android.util.Log.e("MySQLiteApp", "Col " + i + " type: NULL (no value)");
            break;

        default:
            android.util.Log.e("MySQLiteApp",
                "Col " + i + " type: " + ((coltype == org.sqlg.SQLiteGlue.SQLG_BLOB) ? "BLOB" : "TEXT") +
                    " value: " + org.sqlg.SQLiteGlue.sqlg_st_column_text_native(sthandle, i));
            break;
        }
    }

    stresult = org.sqlg.SQLiteGlue.sqlg_st_step(sthandle);
}
```

Yes, this code was actually tested.

# Under the hood

## Defining the API

All constants and sqlite accessor functions are defined in `native/sqlg.h`. The `make regen` rule uses [GlueGen](http://jogamp.org/gluegen/www/) (referenced in a subproject) to regenerate `java/org/sqlg/SQLiteGlue.java`

