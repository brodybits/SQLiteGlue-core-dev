# SQLiteGlue-core

Low-level Android &amp; other Java native glue interface to sqlite using Gluegen.

Unlicense (public domain).

## About

SQLiteGlue provides the basic low-level functions necessary to use sqlite from an Android or other
Java application over JNI (Java native interface). This is accomplished by using
[GlueGen](http://jogamp.org/gluegen/www/) around a simple wrapper C module.

This project is meant to help build a higher-level sqlite interface library, with the JNI layer completely isolated and may be embedded with the package name changed.

**WARNING:** The sqlite database and statement handles that are returned by the `SQLiteGlue` library functions are raw C pointer values (with `0x100000000` added). If someone uses a sqlite database or statement handle that is not valid, or no longer valid with the `SQLiteGlue` library the behavior is undefined (typically a crash that cannot be recovered). It is *NOT* recommended to use this API directly unless you really understand how this library works internally.

## How

All constants and sqlite accessor functions are defined in `native/sqlg.h`. The `make regen` rule uses a build of [GlueGen](http://jogamp.org/gluegen/www/) (referenced in a subproject) to (re)generate the following files which _are_ committed and maintained in this repository:
- `java/org/sqlg/SQLiteGlue.java` - provides the single `org.sqlg.SQLiteGlue` object that exports the static native sqlite accessor functions and some constants that may be helpful
- `native/SQLiteGlue_JNI.c` - C code that implements the native Java functions that call the functinos defined in `native/sqlg.h`

Other C modules:
- `native/sqlg.c` - implement the C functions defined in `native/sqlg.h`
- `sqlite-amalgamation/sqlite3.c` - sqlite amalgamation
- `native/sqlg_all.c` - includes the other C modules for easy build

# Building

## Normal build

*First* initialize with the `gluegentools` & `sqlite-amalgamation` subprojects:

$ `make init`

Then to build:

$ `make`

## Regenerage Java & C glue code

$ `make regen`

# Usage

## Installation

Incude `java/org/sqlg/SQLiteGlue.java` in your Java source code.

Include the contents of the `libs` subdirectory in your installation package. For Android, simply include the contents of `libs` in the `libs` subdirectory of your Android project.

## Java API Structure

There is a single `org.sqlg.SQLiteGlue` object that provides the sqlite accessor functions as static native functions.

## To load the native library

```Java
static
{
    System.loadLibrary("sqlg");
}
```

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

Yes, this sample code is actually tested.

# Automatic testing

Tested on Android in this project: https://github.com/sqlg/SQLiteGlue-test-Android

# Under the hood

## sqlite database and statement handles

The original design was for the C functions to return the sqlite database pointer (`sqlite3 *`) and statement pointer (`sqlite3_stmt *`) by first converting them to `ptrdiff_t` (by converting to `unsigned char *` then subtracting `(unsigned char *)NULL`). Since `ptrdiff_t` is both a _signed_ integer and is supposed to be big enough to store any valid pointer difference, the trick is that in case of an error, it would be possible to return the negated sqlite error code in case of a problem.

However, both the Android NDK code and Gluegen interpret `ptrdiff_t` as a 32-bit integer, and on newer versions of Android the addresses would exceed `0x7fffffff`, causing this design to break.

The sqlite database and statement pointers are _now_ returned as long handle values in order to solve this problem, with `0x100000000` added to be absolutely sure that these will be positive values.

## SQLiteGlue.cfg

`SQLiteGlue.cfg` defines the following:
- [GlueGen](http://jogamp.org/gluegen/www/) interface style: `AllStatic` - interface using static functions only
- Java package (`org.sqlg`)
- Generated Java class name (`SQLiteGlue`)
- String handling
- Output directories

See the [GlueGen documentation](http://jogamp.org/gluegen/www/) for more information.

