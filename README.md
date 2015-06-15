# SQLiteGlue

Low-level Android &amp; other Java native glue interface to sqlite using Gluegen.

Unlicense (public domain).

## About

SQLiteGlue provides the basic low-level functions necessary to use sqlite from an Android or other
Java application over JNI (Java native interface). This is accomplished by using
[GlueGen](http://jogamp.org/gluegen/www/) around a simple wrapper C module.

TBD API & some internal details

**NOTE:** This project references the `gluegentools` & `sqlite-amalgamation` subproject, which are resolved by: $ `make init`

# Building

## Normal build

Initialize with subprojects:

$ `make init`

Then to build:

$ `make`

## Regenerage Java & C glue code

$ `make regen`

# Basics

## Structure

There is a single `org.sqlg.SQLiteGlue` object that provides the sqlite accessor functions as static native functions.

## Usage

To open a database:

```Java
long dbhandle = org.sqlg.SQLiteGlue.sqlg_db_open(fullFilePath, openFlags);
```
where `fullFilePath` is the _string_ path to the file and `openFlags` is the combination of flags used to open the file (using `sqlite3_open_v2()`). The result in `dbhandle` is the `long` handle that can be used to access the database if positive, or the negative value of the sqlite error code returned if negative. Here is an example (from an Android Activity function):

```Java
java.io.File dbfile = new java.io.File(this.getFilesDir(), "DB.db");

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

To prepare a statement with no parameters (on Android):

```Java
long sthandle = org.sqlg.SQLiteGlue.sqlg_db_prepare_st(dbhandle, "select upper('How about some ascii text?') as caps");

if (sthandle < 0) {
    android.util.Log.e("MySQLiteApp", "prepare statement error: " + -sthandle);
    org.sqlg.SQLiteGlue.sqlg_db_close(dbhandle);
    return;
}
```

**WARNING:** Again, if you use a negative statement handle in other `SQLiteGlue` functions the behavior is undefined.

TBD ...


# Adaptations & extensions

TBD

