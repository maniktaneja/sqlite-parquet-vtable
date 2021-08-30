# sqlite-parquet-vtable

This is a fork of  cldellow/sqlite-parquet-vtable. I have update this code to
build with the latest release of libarrow and libparquet. Steps to install as
follows :

- Install and build libarrow and libparquet cpp. See : https://arrow.apache.org/install/

- Make sure you build and install parquet-cpp, by enabling the parquet build
: cmake . -DARROW_PARQUET=ON

- The build the parquet vtable

```
gcc -std=c++11  -g -I. -I<path to sqlite> -I /usr/local/include/arrow/  -fPIC -shared parquet.cc parquet_cursor.cc parquet_filter.cc parquet_table.cc -lz -lcrypto -lssl -lparquet -larrow -o libparquet.so
```

### Note that these steps are for Centos7, I haven't yet tested with any other platform

To test :

```
sqlite> .load parquet/libparquet
sqlite> CREATE VIRTUAL TABLE census USING parquet('./userdata1.parquet');
select * from census limit 1;
1454486129000|1|Amanda|Jordan|ajordan0@com.com|Female|1.197.201.2|6759521864920116|Indonesia|3/8/1971|49756.53|Internal Auditor|1E+02
```


## Supported features

### Row group filtering

Row group filtering is supported for strings and numerics so long as the SQLite
type matches the Parquet type.

e.g. if you have a column `foo` that is an INT32, this query will skip row groups whose
statistics prove that it does not contain relevant rows:

```
SELECT * FROM tbl WHERE foo = 123;
```

but this query will devolve to a table scan:

```
SELECT * FROM tbl WHERE foo = '123';
```

This is laziness on my part and could be fixed without too much effort.

### Row filtering

For common constraints, the row is checked to see if it satisfies the query's
constraints before returning control to SQLite's virtual machine. This minimizes
the number of allocations performed when many rows are filtered out by
the user's criteria.

### Memoized slices

Individual clauses are mapped to the row groups they match.

eg going on row group statistics, which store minimum and maximum values, a clause
like `WHERE city = 'Dawson Creek'` may match 80% of row groups.

In reality, it may only be present in one or two row groups.

This is recorded in a shadow table so future queries that contain that clause
can read only the necessary row groups.

### Types

These Parquet types are supported:

* INT96 timestamps (exposed as milliseconds since the epoch)
* INT8/INT16/INT32/INT64
* UTF8 strings
* BOOLEAN
* FLOAT
* DOUBLE
* Variable- and fixed-length byte arrays

These are not currently supported:

* UINT8/UINT16/UINT32/UINT64
* DECIMAL
