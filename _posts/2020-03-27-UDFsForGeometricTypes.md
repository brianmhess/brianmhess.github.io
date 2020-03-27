---
title: UDFs for Geometric Types
layout: post
tags:
 - Cassandra
---

DataStax Enterprise (DSE) contains support for geometric data types - PointType, LineStringPoint, PolygonType - 
which can be really useful.

One thing that is not included in DSE is some functions to operate on those data types. 
This blog post will demonstrate some UDFs we can write to provide some basic functions.
---

## PointType Functions 

The PointType represents a 2-dimensional point, such as a latitude and longitude. 
You can create a table with a PointType column like so:
```
CREATE TABLE test.pts(pkey INT, geopt 'PointType', PRIMARY KEY ((pkey)));
```
We can then insert some data into that table using the following CQL:
```
INSERT INTO test.pts(pkey, geopt) VALUES (1, 'POINT(1.0 1.0)');
INSERT INTO test.pts(pkey, geopt) VALUES (2, 'POINT(2.0 4.0)');
INSERT INTO test.pts(pkey, geopt) VALUES (3, 'POINT(3.0 6.0)');
```

If we then select this data, we see that the geopt column is represented as a string in the output in cqlsh:

```
cqlsh:test> SELECT * FROM test.pts;

 pkey | geopt
------+-----------------
    1 | POINT (1.0 1.0)
    2 | POINT (2.0 4.0)
    3 | POINT (3.0 6.0)

(3 rows)
```

There is a problem in creating a UDF that takes a `'PointType'` as an input. The work around we’ll explore 
is to process that data as well-known binary.

WKB is a common data format for geometric types. The format for the Point data type is:
* first byte determines endian-ness (0 for big endian, 1 for small endian)
* four bytes for the geometry type
* 8 bytes for the x coordinate as a double-precision floating point
* 8 bytes for the y coordinate as a double-precision floating point

In our case, the data is in small-endian, so we need to flip around the bytes of the x and y 
double-precision floating point values.

So, to extract the x coordinate the plan is to grab bytes 5-12 (0-counting…), reverse those bytes, 
put them into a long value by shifting bits and OR’ing them, and using the 
`Double#longBitsToDouble(long)` method.

Here is the DDL to create this method:

```
CREATE OR REPLACE FUNCTION test.point_x(pt 'PointType')
 RETURNS NULL ON NULL INPUT
 RETURNS double
 LANGUAGE java
 AS $$
  byte[] bytes = pt.array();
  long longbits = 0;
  for (int i = 0; i < 8; i++)
   longbits = (longbits<<8)|(bytes[12-i]&0xFF);
  return Double.longBitsToDouble(longbits);
$$;
```

Now, we can test our function on the data we inserted above:

```
cqlsh:test> SELECT pkey, geopt, point_x(geopt) AS x FROM test.pts;

 pkey | geopt           | x
------+-----------------+---
    1 | POINT (1.0 1.0) | 1
    2 | POINT (2.0 4.0) | 2
    3 | POINT (3.0 6.0) | 3

(3 rows)
```

We can now do the same for the y coordinate:

```
CREATE OR REPLACE FUNCTION test.point_y(pt 'PointType')
 RETURNS NULL ON NULL INPUT
 RETURNS double
 LANGUAGE java
 AS $$
  byte[] bytes = pt.array();
  long longbits = 0;
  for (int i = 0; i < 8; i++)
   longbits = (longbits<<8)|(bytes[20-i]&0xFF);
  return Double.longBitsToDouble(longbits);
$$;
```

And we can test it as follows:

```
cqlsh:test> SELECT pkey, geopt, point_x(geopt) AS x, point_y(geopt) AS y FROM test.pts;

 pkey | geopt           | x | y
------+-----------------+---+---
    1 | POINT (1.0 1.0) | 1 | 1
    2 | POINT (2.0 4.0) | 2 | 4
    3 | POINT (3.0 6.0) | 3 | 6

(3 rows)
```

Another approach

We could tweak this approach slightly and convert the `'PointType'` into a user-defined type 
that would allow us to reference the internal x and y coordinates directly. To do that, we 
first create a user-defined type:

```
CREATE TYPE xy (x DOUBLE, y DOUBLE);
```

Now, we can create a function to convert from a `'PointType'` to an `xy`:

```
cqlsh:test> SELECT geopt, point_to_xy(geopt) FROM test.pts;

 geopt           | test.point_to_xy(geopt)
-----------------+-------------------------
 POINT (1.0 1.0) |            {x: 1, y: 1}
 POINT (2.0 4.0) |            {x: 2, y: 4}
 POINT (3.0 6.0) |            {x: 3, y: 6}

(3 rows)
```

We can also now reference the x and y coordinates via CQL:

```
cqlsh:test> SELECT geopt, point_to_xy(geopt).x AS x, point_to_xy(geopt).y AS y FROM test.pts;

 geopt           | x | y
-----------------+---+---
 POINT (1.0 1.0) | 1 | 1
 POINT (2.0 4.0) | 2 | 4
 POINT (3.0 6.0) | 3 | 6

(3 rows)
```

## LineString Functions

The Line String construct is essentially an ordered list of points. You can create a table of 
line strings like so:

```
CREATE TABLE lines(pkey INT, geoline 'LineStringType', PRIMARY KEY ((pkey)));
```

And we can insert some data like so:

```
INSERT INTO test.lines(pkey,geoline) VALUES (0, 'LINESTRING(1.0 2.0, 3.0 4.0, 5.0 6.0)');
INSERT INTO test.lines(pkey,geoline) VALUES (1, 'LINESTRING(1.1 2.1, 3.1 4.1, 5.1 6.1)');
INSERT INTO test.lines(pkey,geoline) VALUES (2, 'LINESTRING(1.2 2.2, 3.2 4.2, 5.2 6.2)');
```

We can also view the data in the table:

```
cqlsh:test> SELECT * FROM test.lines;

 pkey | geoline
------+----------------------------------------
    1 | LINESTRING (1.1 2.1, 3.1 4.1, 5.1 6.1)
    0 | LINESTRING (1.0 2.0, 3.0 4.0, 5.0 6.0)
    2 | LINESTRING (1.2 2.2, 3.2 4.2, 5.2 6.2)

(3 rows)
```

We want to write a UDF that will return the n-th point in the line string. 
Unfortunately, there is a current limitation that we cannot return a `'PointType'`, 
so we can use the simple UDT we created above.

We can now create a UDF to extract out the n-th point (0-based counting):

```
CREATE FUNCTION test.line_point(line 'LineStringType', idx INT)
 RETURNS NULL ON NULL INPUT
 RETURNS xy
 LANGUAGE java
 AS $$
  byte[] bytes = line.array();
  long longbitsx = 0;
  long longbitsy = 0;
  for (int i = 0; i < 8; i++) {
   longbitsx = (longbitsx<<8)|(bytes[9+(16*idx)+8-1-i]&0xFF);
   longbitsy = (longbitsy<<8)|(bytes[9+(16*idx)+16-1-i]&0xFF);
  }
  Double x = Double.longBitsToDouble(longbitsx);
  Double y = Double.longBitsToDouble(longbitsy);
  UDTValue retVal = udfContext.newReturnUDTValue();
  retVal.setDouble("x", x);
  retVal.setDouble("y", y);
  return retVal;
 $$;
```

We can use this function as follows:

```
cqlsh:test> SELECT geoline, line_point(geoline, 0) as pt_0 FROM test.lines;

 geoline                                | pt_0
----------------------------------------+------------------
 LINESTRING (1.1 2.1, 3.1 4.1, 5.1 6.1) | {x: 1.1, y: 2.1}
 LINESTRING (1.0 2.0, 3.0 4.0, 5.0 6.0) |     {x: 1, y: 2}
 LINESTRING (1.2 2.2, 3.2 4.2, 5.2 6.2) | {x: 1.2, y: 2.2}

(3 rows)
```

Since this returns a UDT, we can actually just extract the x coordinate if we want, too:

```
cqlsh:test> SELECT geoline, line_point(geoline, 0).x as pt_0_x FROM test.lines;

 geoline                                | pt_0_x
----------------------------------------+--------
 LINESTRING (1.1 2.1, 3.1 4.1, 5.1 6.1) |    1.1
 LINESTRING (1.0 2.0, 3.0 4.0, 5.0 6.0) |      1
 LINESTRING (1.2 2.2, 3.2 4.2, 5.2 6.2) |    1.2

(3 rows)
```

## Wrapping Up

This is just a simple few UDFs that help play around with the geometric types `'PointType'`
 and `'LineStringType'`. Enjoy!

## Appendix - DDL for UDFs:

```
CREATE OR REPLACE FUNCTION point_x(pt 'PointType')
 RETURNS NULL ON NULL INPUT
 RETURNS double
 LANGUAGE java
 AS $$
  byte[] bytes = pt.array();
  long longbits = 0;
  for (int i = 0; i < 8; i++)
   longbits = (longbits<<8)|(bytes[12-i]&0xFF);
  return Double.longBitsToDouble(longbits);
$$;

CREATE OR REPLACE FUNCTION point_y(pt 'PointType')
 RETURNS NULL ON NULL INPUT
 RETURNS double
 LANGUAGE java
 AS $$
  byte[] bytes = pt.array();
  long longbits = 0;
  for (int i = 0; i < 8; i++)
   longbits = (longbits<<8)|(bytes[20-i]&0xFF);
  return Double.longBitsToDouble(longbits);
$$;
 
CREATE TYPE xy (x DOUBLE, y DOUBLE);

CREATE OR REPLACE FUNCTION point_to_xy(pt 'PointType')
 RETURNS NULL ON NULL INPUT
 RETURNS xy
 LANGUAGE java
 AS $$
  byte[] bytes = pt.array();
  long longbitsx = 0;
  long longbitsy = 0;
  for (int i = 0; i < 8; i++) {
   longbitsx = (longbitsx<<8)|(bytes[12-i]&0xFF);
   longbitsy = (longbitsy<<8)|(bytes[20-i]&0xFF);
  }
  Double x = Double.longBitsToDouble(longbitsx);
  Double y = Double.longBitsToDouble(longbitsy);
  UDTValue retVal = udfContext.newReturnUDTValue();
  retVal.setDouble("x", x);
  retVal.setDouble("y", y);
  return retVal;
$$;

CREATE FUNCTION line_point(line 'LineStringType', idx INT)
 RETURNS NULL ON NULL INPUT
 RETURNS xy
 LANGUAGE java
 AS $$
  byte[] bytes = line.array();
  long longbitsx = 0;
  long longbitsy = 0;
  for (int i = 0; i < 8; i++) {
   longbitsx = (longbitsx<<8)|(bytes[9+(16*idx)+8-1-i]&0xFF);
   longbitsy = (longbitsy<<8)|(bytes[9+(16*idx)+16-1-i]&0xFF);
  }
  Double x = Double.longBitsToDouble(longbitsx);
  Double y = Double.longBitsToDouble(longbitsy);
  UDTValue retVal = udfContext.newReturnUDTValue();
  retVal.setDouble("x", x);
  retVal.setDouble("y", y);
  return retVal;
 $$;
```
