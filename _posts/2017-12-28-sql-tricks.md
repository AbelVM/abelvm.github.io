---
title: "Some little known PostgreSQL/PostGIS tips & tricks"
tags: PostgreSQL PostGIS
category: SQL
---

### 0. What's this and what's not

This is not an SQL course, not even a crash-course, not a introduction, not a master class... It's just a compilation of tips, tricks or unusual uses of PostgreSQL / PostGIS that I use a lot and may be helpful for anybody out there. Just try to keep your brain within your skull and buy me beers later.

This post is a revised version of an internal talk I gave at [CARTO](http://carto.com) on August 31st 2017.

### 1. Spatial operators

### **>>>>> [THIS](http://postgis.net/docs/reference.html#Operators) <<<<<**

You should use them as much as you can. Why? Because they are like **spatial indexes operators** !!! And the performance *may* be several orders higher than the equivalent ST_function for checking spatial relationships.

**CAVEATS:**

* Some of this operators use the indexes only in specific cases like when used within an `ORDER BY` clause of the query (like [<->](http://postgis.net/docs/geometry_distance_knn.html)). So, just <u>check the documentation</u> to be sure. But, even in the worst case, you get at least the same performance as it's equivalent ST_function, and it's shorter... so no worries.
* In latest versions of PostGIS, most these ST_functions are now index-aware and perform index conditions checks internally before running the function itself, so there's no actual gain in using this operators but for shorthand code sake.

### <a name="lateral"></a>2. Plain vs Lateral subqueries:  from BIG to small

You may have used the syntax described in the good 'ol [BULK NEAREST NEIGHBOR USING LATERAL JOINS](https://carto.com/blog/lateral-joins) blogpost by [Paul Ramsey](https://github.com/pramsey), without diving in the gears (and wrongly assuming that the keyword is `CROSS JOIN LATERAL` ). So, let's clarify some concepts.

You may be used to subqueries like
```sql
SELECT
    *
FROM  (SELECT * FROM my_dataset) _subquery
```
Or, even cleaner while developing, using the `WITH` clause
```sql
WITH
_subquery AS(
    SELECT * FROM my_dataset
)
SELECT
    *
FROM _subquery
```
The issue here is that those subqueries are fetching **all** the data without filtering. And here comes the [LATERAL](https://www.postgresql.org/docs/current/static/queries-table-expressions.html#QUERIES-LATERAL) keyword to allow you to contextually pre-filter the results of the subqueries.

Let's go with a bit twisted example. We have a dataset of people in Spain. We have two surnames here, the 1st comes from our dad 1st surname and the 2nd comes from our mom 1st surname. So let's say that we have a huge dataset like:

* id
* name
* surname_1
* surname_2
* pack_id
* gender

Being `pack_id` an ID that identify a group of brothers/sisters with the same tupla [dad, mom], let's find the potential biological parents tuplas for each person in the database. With the hypothesis:

* One can't be its own father or mother
* No incest allowed (I'm talking to you Cersei)
* We need a male and a female to born a child

The classical approach would be:

```sql
SELECT
    p.*,
    dads.id as dad,
    moms.id as mom
FROM
    people p,
    people dads,
    people moms
WHERE
    dads.gender = 'm' AND
    moms.gender = 'f' AND
    p.pack_id <> dads.pack_id AND
    p.pack_id <> moms.pack_id AND
    dads.pack_id <> moms.pack_id AND
    p.surname_1 = dads.surname_1 AND
    p.surname_2 = moms.surname_1
```
So we are fully scanning `people³`  with 7 conditions!. Let's try subqueries
```sql
WITH
dads AS(
    SELECT * FROM people where gender='m'
), -- potential dads
parents AS(
    SELECT
          *,
          dads.id as dad_id,
          dads.surname_1 as dad_surname,
          dads.pack_id as dad_pack
      FROM
          people
      LEFT JOIN dads ON pack_id <> dads.pack_id
      where gender='f'
), -- potential dad&mom tuplas
SELECT
    p.*
    dm.dad_id,
    dm.id as mom_id
FROM
    people p,
    parents dm
WHERE
    p.surname_1 = dm.dad_surname AND
    p.surname_2 = dm.surname_1 AND
    p.pack_id <> dm.dad_pack AND
    p.pack_id <> dm.pack_id
```
Have we gaining anything in terms of performance? Sure!.  Subqueries are still scanning the full dataset to get the right result, but we're funneling the results from the full dataset to a smallest dataset of potential tuplas ([male, female] with different pack_id). So it should be a bit faster than scanning all the combinations of `people³` .

So we have `LATERAL` subqueries. This special keyword enable the subquery that follows it to use the values of the precedent ones as parameters (**left to right**). Let's go on with the previous example:
```sql
SELECT
    p.*,
    dads.id as dad_id,
    moms.id as mom_id
FROM
    people p
LEFT JOIN
    LATERAL(
        SELECT
              *
          FROM people
          WHERE
              gender='m' AND
              surname_1 = p.surname_1 AND
              pack_id <> p.pack_id
    ) dads
ON 1=1
LEFT JOIN
    LATERAL(
        SELECT
              *
          FROM people
          WHERE
              gender='f' AND
              surname_1 = p.surname_2 AND
              pack_id <> p.pack_id AND
              pack_id <> dads.pack_id
    ) moms
ON 1=1
```
So, what's happening here? Both `dads` and `moms` subqueries scan **only** the rows that already comply with their `WHERE` condition, reducing the scope of the subsequent queries, so the performance improvement might be huge (not in this simple example). Let's break it down

* For each **person** in `people`
* We look for **potential dads** in the subset of `people` that
  * are male
  * share the same `surname_1`
  * are not brother of the person under study 
* So: #dads < #people
* For each of those [person, dad] tuplas, we look for **potential moms** in the subset of people that 
  * are female
  * not sister of either the person or the dad
  * her `surname_1`  is the same as the person `surname_2`
* So: #moms << #people

So, instead of scanning `people³`, we're scanning `people · dads(person) · moms(person, dad)`

The actual performance impact is, as always, mainly based on the right choice of indexes. But a good combination of indexes and `LATERAL` clauses can improve the performance of the query by several orders, as we're gonna learn in the next epigraph.

### 3. Geospatial index-aware operations

Let's go with a geospatial example of the above, one that you can test and check the performance improvement. Real life use case: aggregation of points in polygons (populated places in countries in this example). Classical query would be like the one below, but for the sake of the demo, I'm going to use here the `_ST_Within`  function instead of the common [ST_Within](https://postgis.net/docs/ST_Within.html) because the later one already uses spatial indexes and boundaries comparison in the latest versions of PostGIS (so, it's actually using the same technique I'm trying to showcase here)

```sql
SELECT
    polygons.*,
    agg_function(points.my_field) as my_agg_value
FROM
    polygons_dataset polygons
LEFT JOIN
    points_dataset points
ON _ST_within(points.geom, polygons.geom)
GROUP BY polygons.id
```
The query above performs quite crappy: above **47 ms** for  `populated_places` (649 rows) vs. `world_borders_hd` (254 rows). If the datasets are huge and/or the polygons geometry are complex... you're screwed up. You are checking the intersection of every point against every polygon, as you can see in the explain below

```sql
HashAggregate  (cost=8945.24..8946.13 rows=254 width=1290)
   Group Key: polygons.key_id
   ->  Nested Loop Left Join  (cost=0.00..8890.29 rows=54949 width=1290)
         Join Filter: _st_contains(polygons.the_geom, points.the_geom)
         ->  Seq Scan on world_borders_hd polygons  (cost=0.00..43.76 rows=254 width=1286)
         ->  Materialize  (cost=0.00..27.60 rows=649 width=36)
               ->  Seq Scan on populated_places points  (cost=0.00..26.95 rows=649 width=36)
(7 rows)
```

Using [ST_Within](https://postgis.net/docs/ST_Within.html) instead, you can we get an average query time below **12 ms** and you can see below how the indexes are actually used

```sql
 HashAggregate  (cost=185.86..186.75 rows=254 width=1290)
   Group Key: polygons.key_id
   ->  Nested Loop Left Join  (cost=0.03..185.61 rows=254 width=1290)
         ->  Seq Scan on world_borders_hd polygons  (cost=0.00..43.76 rows=254 width=1286)
         ->  Index Scan using ne_10m_populated_places_simple_the_geom_idx on populated_places points  (cost=0.03..0.56 rows=1 width=36)
               Index Cond: (polygons.the_geom ~ the_geom)
               Filter: _st_contains(polygons.the_geom, the_geom)
(7 rows)
```

Using spatial operators and lateral subqueries (but the still old `_ST_Within` to be fair) the query would be like
```sql
SELECT
    polygons.*,
    agg_function(points.my_field) as my_agg_value
FROM
    polygons_dataset polygons
LEFT JOIN
    LATERAL (
        SELECT * FROM points_dataset WHERE geom ~ polygons.geom
    ) points
ON _ST_within(points.geom, polygons.geom)
GROUP BY polygons.id
```
The query time is quite similar to the index-optimized `ST_Within`. Let's make an explain:

```sql
HashAggregate  (cost=185.86..186.75 rows=254 width=1290)
   Group Key: polygons.key_id
   ->  Nested Loop Left Join  (cost=0.03..185.61 rows=254 width=1290)
         ->  Seq Scan on world_borders_hd polygons  (cost=0.00..43.76 rows=254 width=1286)
         ->  Index Scan using ne_10m_populated_places_simple_the_geom_idx on populated_places  (cost=0.03..0.56 rows=1 width=36)
               Index Cond: (the_geom ~ polygons.the_geom)
               Filter: _st_contains(polygons.the_geom, the_geom)
(7 rows)
```

Oh! There are exactly the same gears inside both clocks! So, in this very specific case and PostGIS version, there's no advantage in using lateral queries if you use the proper ST_function. So: **always check the PostGIS docs of the version in production to avoid double work when building a heavy query or a simple one against a huge dataset**.

To check the PostGIS version you're running, you can ask the database this way:

```sql
select postgis_version()
```

Some ST_functions that needed some extra work to optimize in older PostGIS versions are now index-aware and there's no gain in double check the indexes. Let's explain the same query with `ST_Within` and `LATERAL` clause:

```sql
HashAggregate  (cost=185.99..186.88 rows=254 width=1290)
   Group Key: polygons.key_id
   ->  Nested Loop Left Join  (cost=0.03..185.73 rows=254 width=1290)
         ->  Seq Scan on world_borders_hd polygons  (cost=0.00..43.76 rows=254 width=1286)
         ->  Index Scan using ne_10m_populated_places_simple_the_geom_idx on populated_places  (cost=0.03..0.56 rows=1 width=36)
               Index Cond: ((the_geom ~ polygons.the_geom) AND (polygons.the_geom ~ the_geom))
               Filter: _st_contains(polygons.the_geom, the_geom)
(7 rows)
```

As you can see, it's applying the same condition twice. Bullshit. Other

Let's "explain the explain" of the index-aware query to understand the performance increase

* For each polygon
* Retrieve only the points which index fulfill the index condition `~` , which means that only the points which bounding box falls within the bounding box of the polygon under study will be taken into account.
* Now that we have reduced the number of points to the real candidates of being withing the polygon, we perform the actual check (using an index-unaware function) but with just a few of points per polygon!!


This combination of:

- `LATERAL` keyword
- bounding-box comparisons
- index-aware operations

led to an improvement factor of ~ **4x** in this specific and simple example, but can be way higher with more complex and heavier queries. This might be achieved with the common ST_function in latest versions of PostGIS.

 **So, lessons to be learn here:**

1. **Use index-aware functions where available**
2. **When checking spatial relations (intersect, within, contains, overlap, etc) , always reduce the scan scope by performing the same check with the bounding boxes of your features in a `LATERAL` clause (only if the doc of the ST_function to be used is not explicitly saying that it's performing this internally)**
3. **Always go big-to-small and light-to-heavy**

Regarding point #3 , let's add a final example. Let's say that you want to calc the percent of the areas of a dataset of plots that are going to be used to build a new road network. The right way to do so is:

```sql
SELECT
    p.id as plot_id,
    _r.id as road_id
    round(
          ST_Area(
              ST_Intersection(_r.the_geom, p.the_geom)
          )/ST_Area(p.the_geom)
     , 2) as percent_overlap
FROM
    plots_dataset p
CROSS JOIN
    LATERAL(
        SELECT
              *
          FROM
              roads_dataset
          WHERE ST_Intersects(the_geom, p.the_geom)
    ) _r
```

1. `ST_Intersects` internally uses **indexes** for...
2. **bounding box** checking
3. `_r` result is way **smaller** than `roads_dataset`
4. `ST_Intersects` is way **lighter** that `ST_Intersections`

So we complied with the 3 golden rules above.

### 4. Distance in "pseudometers"

Casting to geography can be costly if the dataset is huge, and distance calculations require 2 casts (being `the_geom` in EPSG:4326)

```sql
ST_Distance(a.the_geom::geography, b.the_geom::geography)
```

Distance in the_geom_webmercator (EPSG:3857) is measured in meters, but there's a distortion growing with the latitude, so we can use it as a 1st order approach for gross comparisons only.

![thread](https://upload.wikimedia.org/wikipedia/commons/a/a2/Mercator_scale_plot.svg)

But we can mildly correct it with a factor of `1/cos(latitude)`  for better results. For distance, let's use the middle point:

```sql
ST_Distance(a.the_geom_webmercator, b.the_geom_webmercator)
/
cos(
  radians(
      0.5*(ST_y(a.the_geom) + ST_y(b.the_geom))
  )
)
```

If wanted to make a [ST_DWithin](https://postgis.net/docs/ST_DWithin.html) (ref. [FINDING THE NEAREST NEIGHBOR](https://carto.com/blog/nearest-neighbor-joins))

```sql
SELECT
    s.*,
    a.key_id as lighthouse_id
FROM
    ships_dataset s,
    lighthouses_dataset a
WHERE ST_DWithin(
    s.the_geom_webmercator,
    a.the_geom_webmercator,
    my_radius / cos(radians(ST_y(a.the_geom)))
)
```

With latest versions of PostGIS, the casting to/from geography has been optimized a lot and these trick might be outdated. So, use it just in case you feel that the simple casting is taking eons.

### 5. Arrays as tables

Do you know about the power of arrays? You should. In PostgreSQL you can fake tables if you have arrays. Let's say that you have two arrays of the same size, geometries and values as `geom[ ]` and `values[ ]`, then your table would be like

```sql
SELECT
    t.index:bigint as key_id,
    t.geom as the_geom,
    t.value as my_value
FROM unnest(geom, values) WITH ORDINALITY t(geom, value, index)
```

`WITH ORDINALITY` generates an index of the elements of the array that you're unnesting, so we can using it as key_id. Cool!

### 6. Fun with Window Functions and aggregations

"_*Window functions* provide the ability to perform calculations across sets of rows that are related to the current query row_"

And this ^^^ is über-powerful!

So you have this list of [window functions](https://www.postgresql.org/docs/current/static/functions-window.html#FUNCTIONS-WINDOW-TABLE)  and a huge list of aggregation functions in [PostgreSQL](https://www.postgresql.org/docs/current/static/functions-aggregate.html) and [PostGIS](https://postgis.net/docs/manual-1.5/ch08.html#PostGIS_Aggregate_Functions) , some [hypothetical-set aggregates](https://www.postgresql.org/docs/current/static/functions-aggregate.html#FUNCTIONS-HYPOTHETICAL-TABLE), and you can even [build your own](https://www.postgresql.org/docs/current/static/xaggr.html). And you can find some [deep info](https://www.postgresql.org/docs/current/static/sql-expressions.html) about them and their uses and syntax.

The syntax for aggregations and window functions are quite similar:

* Aggregation (**IRL aggregation functions ARE window functions indeed!!** )

```sql
  SELECT
      aggregation(my_field ORDER BY other_field) FILTER (WHERE CLAUSE)
  FROM my_table
```

  Like counting categories

```sql
  SELECT
      event,
      SUM(value) FILTER (WHERE type='standard')  as standard,
      SUM(value) FILTER (WHERE type='vip') as vip
  FROM tickets
  GROUP BY event
```
  Or getting a sorted array

```sql
  SELECT array_agg(amount ORDER BY DATE) FROM expenses
```

* Window Functions, here comes the fun

```sql
  SELECT
      windowfunction(myfield)
          FILTER (WHERE CLAUSE)
          OVER (PARTITION BY other_field ORDER BY other_other_field [FRAME_DEFINITION])
  FROM my_table
```

The `OVER` clause defines the **window** where the function is going to be evaluated in, and it can be named if you want to reuse it:

```sql
  SELECT
      windowfunction(myfield) FILTER (WHERE CLAUSE) OVER mywindow,
      windowfunction_2(myfield_2) FILTER (WHERE CLAUSE_2) OVER mywindow
  FROM my_table
  WINDOW mywindow AS (PARTITION BY other_field ORDER BY other_other_field)
```

(check the [SELECT](https://www.postgresql.org/docs/current/static/sql-select.html) syntax to be sure where to put the `WINDOW` declaration when you also have conditions, sorting, grouping...)

Hypothetical-set aggregate functions require that you use `WITHIN GROUP` instead of `OVER` keyword.

And the `FILTER` clause can only be applied if the window function is an aggregated one.

A `WINDOW` is defined by (all optional)

* `PARTITION BY` clause. Let's say that we want to define a window with all the rows that have the same value for `field_1` as my current row (think on categories), then you go with `PARTITION BY field_1` . If there's no `PARTITION BY`, the window will cover the whole set of results of the `SELECT` query.
* `ORDER BY` that will order the rows in the partition by a field or number of fields.
* `ROWS` or `RANGE` frame, that defines a subset of the partition defined in relation to the current row, like `RANGE frame_start AND frame_end` or `ROWS frame_start`  .

Let's try to explain the frames definition in a graphic way, within the partition

| value | frame definition    | range | rows |
| ----- | ------------------- | ----- | ---- |
| A     | UNBOUNDED PRECEDING | x     | x    |
| ...   | ...                 |       | x    |
| H     | ...                 |       | x    |
| I     | ...                 |       | x    |
| J     | 2 PRECEDING         |       | x    |
| K     | 1 PRECEDING         |       | x    |
| **L** | **CURRENT ROW**     | x     | x    |
| M     | 1 FOLLOWING         |       | x    |
| N     | 2 FOLLOWING         |       | x    |
| O     | ...                 |       | x    |
| P     | ...                 |       | x    |
| ...   | ...                 |       | x    |
| Z     | UNBOUNDED FOLLOWING | x     | x    |

* The bounded `PRECEDING` and `FOLLOWING` cases are currently only allowed in `ROWS` mode.

* In `RANGE` mode , `CURRENT ROW` might not be the row you're looking for but the equivalent defined by the `ORDER BY`.

* With the `ROWS` option you define on a physical level how many rows are included in your window frame. With the `RANGE` option how many rows are included in the window frame depends on the ORDER BY values. Let's say you have a list like `[e, a, b, d, c]` , If you run an aggregation against a range, but using ID sorting or VALUE sorting, like in the next query

```sql
WITH
_a as(
    SELECT
        *
    FROM
        unnest(ARRAY['e','a','b','d','c']) WITH ORDINALITY as t(value, id)
)
SELECT
    id,
    value,
    array_agg(id)
        over(ORDER BY id RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
            as range_id,
    array_agg(id)
        over(ORDER BY value RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
            as range_value
FROM _a
ORDER BY id
```


  You'll get different values, because the window is different sized and sorted because of the sorting by `value`

| id   | value | range_id    | range_value |
| ---- | ----- | ----------- | ----------- |
| 1    | e     | [1]         | [2,3,5,4,1] |
| 2    | a     | [1,2]       | [2]         |
| 3    | b     | [1,2,3]     | [2,3]       |
| 4    | d     | [1,2,3,4]   | [2,3,5,4]   |
| 5    | c     | [1,2,3,4,5] | [2,3,5]     |

#### Defaults:

* If frame_start is not defined it defaults to `UNBOUND PRECEDING`
* If frame_end is not defined, it defaults to `CURRENT ROW`  (included)

Long story short: the frame defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` if not defined. So, the unique cases in which the use of `RANGE` makes sense are:

* `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` : from 1st to last row in the partition
* `RANGE BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`  : from current_row (included) to last row in the partition

#### Some common window functions, explained

A practical demo of the most common window functions of the list above, running

```sql
with _a as
(
    SELECT
          *
      FROM
          unnest(ARRAY['a','b','h','a','a','k','l','m','n','b','p','b','z'])
              WITH ORDINALITY as t(value, id)
)
SELECT
  id,
  value,
  row_number() over wn as row_number,
  rank() over wn as rank,
  dense_rank() over wn as dense_rank,
  percent_rank() over wn as percent_rank,
  cume_dist() over wn as cume_dist,
  lag(value, 1)  over wn as lag_1,
  lead(value, 1)  over wn as lead_1,
  first_value(value) over wn as first_value,
  last_value(value) over wn as last_value,
  nth_value(value, 3) over wn as nth_3_value
FROM _a
WINDOW wn as (ORDER BY value)
order by id
```

We get a result like

| id   | value | row_number | rank | dense_rank | percent_rank | cume_dist | lag_1 | lead_1 | first_value | last_value | nth_3_value |
| ---- | ----- | ---------- | ---- | ---------- | ------------ | --------- | ----- | ------ | ----------- | ---------- | ----------- |
| 1    | a     | 3          | 1    | 1          | 0            | 0.2308    | a     | b      | a           | a          | a           |
| 2    | b     | 5          | 4    | 2          | 0.25         | 0.4615    | b     | b      | a           | b          | a           |
| 3    | h     | 7          | 7    | 3          | 0.5          | 0.5385    | b     | k      | a           | h          | a           |
| 4    | a     | 1          | 1    | 1          | 0            | 0.2308    |       | a      | a           | a          | a           |
| 5    | a     | 2          | 1    | 1          | 0            | 0.2308    | a     | a      | a           | a          | a           |
| 6    | k     | 8          | 8    | 4          | 0.5833       | 0.6154    | h     | l      | a           | k          | a           |
| 7    | l     | 9          | 9    | 5          | 0.6667       | 0.6923    | k     | m      | a           | l          | a           |
| 8    | m     | 10         | 10   | 6          | 0.75         | 0.7692    | l     | n      | a           | m          | a           |
| 9    | n     | 11         | 11   | 7          | 0.8333       | 0.8466    | m     | p      | a           | n          | a           |
| 10   | b     | 6          | 4    | 2          | 0.25         | 0.4615    | b     | h      | a           | b          | a           |
| 11   | p     | 12         | 12   | 8          | 0.9167       | 0.9231    | n     | z      | a           | p          | a           |
| 12   | b     | 4          | 4    | 2          | 0.25         | 0.4615    | a     | b      | a           | b          | a           |
| 13   | z     | 13         | 13   | 9          | 1            | 1         | p     |        | a           | z          | a           |

Why?:

* **row_number**: the physical position of the row in the window
* **rank**: the ranking based on the sorting condition (`value` in this very case) where rows with the same value have the same rank, and there will be a gap in the numbering for next value. IE:
  * first value `a` has a rank=1, but there are 3 rows with this value (and same rank)
  * next value is `b` but the 3st rows were already numbered, so the rank for all the rows (3 of them) with this value will have rank = 4
  * next one is `h` but following the same logic, its rank = 7
  * next is `k` , but as the number of rows with value `h` was only one, there's no gap here and rank = 8

  So, graphically:

  * `a` : 1, 1, 1
  * `b` : 4, 4, 4
  * `h`:  7
  * `k`:  8
  * ...
* **dense_rank**: same as above, but without the gaps, so the numbering is like
  * `a` : 1, 1, 1
  * `b` : 2 , 2,  2
  * `h`:  3
  * `k`:  4
  * ...
* **percent_rank**: relative rank of the current row:
  * `(rank - 1) / (total rows - 1)`
* **cume_dist**: relative rank of the current row:
  * ` (number of rows preceding or peer with current row) / (total rows)`
* **lag(value, n)** : `value` evaluated at the row `n` positions before the current row
* **lead(value, n)** : `value` evaluated at the row `n` positions after the current row
* **first_value**: the first value in the window, in this case defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`  so it's always the same value: the first one
* **last_value**:  the last value in the window, in this case defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`  so it has the same value as the current row
* **nth_value(value, n)**:  the value evaluated at the row with `row_number=n` in the current window

#### HUGE CAVEAT

Window function calls are allowed in the `SELECT` list and the `ORDER BY` clause of the query only. So if you want to use them as a condition in a `WHERE` clause, you will need to build a subquery first with the results of your window function and then use them as parameter in your condition.

#### IRL examples of window functions:

* **Generate a valid key_id** for the result of your complex query

```sql
WITH
_a AS (...),
_b AS (... FROM _a)
SELECT
    ROW_NUMBER() OVER() as key_id,
    *
FROM _b
```

* Let's say that you want to **rank** the tax defrauders per municipality for the 2016 campaign, so you have a `hall of fame` of defaulters per council

```sql
SELECT
    council_name,
    citizen_id,
    debt,
    DENSE_RANK() OVER wn as fraud_rank_2016
FROM
    tax_debts 
WHERE campaign = 2016
WINDOW wn AS (PARTITION BY council_name ORDER BY debt desc)
```

* What about a **moving average**? Let's get the average temperature of the last 7 days for each date, including the current one:

```sql
SELECT
    day,
    AVG(temp) OVER week as weekly_avg_temp
FROM
    my_dataset
WINDOW week AS (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
```

* Other use of moving average is **smoothing** a curve. VG. the heading in a GPS track, to discard the errors, using 5 values in this case around the current one.

```sql
SELECT
    time,
    AVG(heading) OVER localheading as smoothed_heading
FROM
    my_dataset
WINDOW localheading AS (ORDER BY time ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING)
```

* As a side use, you can use this smoothing approach to **fill gaps** in your data series

```sql
UPDATE
    my_dataset
SET
    my_field = AVG(my_field)
        OVER(ORDER BY series_index ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING)
WHERE
    my_field is null
```

* **Running sum**? As simple as it can be

```sql
SELECT
    month,
    SUM(monthly_savings) OVER(ORDER BY MONTH ASC) as accumulated_savings
FROM
    my_dataset
```

Which is equivalent to

```sql
SELECT
    month,
    SUM(monthly_savings)
        OVER( ORDER BY MONTH ASC RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        as accumulated_savings
FROM
    my_dataset
```

* And if you want to know your diet **evolution on a weekly basis**? You may use  `lag`  to get the 'past' rows or `lead` for 'future' rows. Take into account that you may use them the other way around just changing the sorting order within the `OVER` clause

```sql
SELECT
    day,
    weight,
    weight - ( lag(weight, 7) OVER (ORDER BY day ASC) )
        as weekly_delta,
    AVG(weight) OVER(ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
        as weekly_average
FROM
    my_diet_log
```

* **Percent** of votes by party?

```sql
SELECT distinct on(party)
    party,
    round(
        100.0 * (
          ( count(1) OVER(PARTITION BY party) )::numeric
              /
          ( count(1) over() )::numeric
        )
    ,2) as percent_votes
FROM
    votes_dataset
```

* Yo can aggregate expressions, not only columns, so let's get the scores of a **test exam** in which:

  * right answer = +1
  * wrong answer = -0.5
  * blank answer = 0

  And there's one table `final_exam` with the expected results like `[id, question, answer]` and another one with the students answers like `final_exam_results` with `[student_id, question_id, answer]`

```sql
SELECT
    _r.student_id,
    SUM(
        CASE
          WHEN _r.answer = _a.answer THEN 1
          WHEN _r.answer is null THEN 0
          ELSE -0.5 END
    )  as final_score
FROM
    final_exam _a,
    final_exam_results _r
WHERE
    _r.question_id = _a.id
GROUP BY
    _r.student_id
```

### 7. Cancel all the pending queries

If you get into problems because of some stuck query, just run this **LITERALLY** . I mean, copy and paste this as is.
```sql
SELECT
    pg_terminate_backend(pid)
FROM
    pg_stat_activity
WHERE
    username=current_user
```
### 8. 24h  or 7 days histogram from a date field

Add a new field in your query
```sql
SELECT
    *,
    extract(hour from m.my_datetime) + round(extract(minute from m.my_datetime)/60) as my_hour
FROM
    my_dataset
```
And now, you can build a 24-bins histogram linked to `my_hour` field.

For days of week you have
```sql
SELECT
    *,
    extract(isodow from m.my_datetime) as my_day
FROM
    my_dataset
```
And then just build a 7-bins histogram linked to `my_day` .

If you want days of week starting on Sunday, just use `dow` instead of `isodow` and you're done!

Check point [#11](#histogram) for more info about how to easily build an histogram.

### 9. Select all the fields in a table **except** some of them

Let's say you want to select all the columns in a hundred-fields table, but if you also select 'field_1' or 'field_2', you may face a conflict later. So, you can't use the basic statement
```sql
SELECT * FROM my_dataset
```

Would you build the select query by hand? This weird query will build the needed query for you, and returns it as a string. You just need to replace the dataset and fields names with the ones you need
```sql
SELECT
    'SELECT '
    || array_to_string(
        ARRAY(
            SELECT
                'o' || '.' || c.column_name
            FROM
                information_schema.columns As c
            WHERE
                table_name = 'my_dataset' AND
                c.column_name NOT IN('field_1', 'field_2')
            ),
        ','
        )
    || ' FROM my_dataset As o' As sqlstmt
```

...so you just need to copy&paste the resulting string and use it as needed

### 10. Punched isolines (donuts)

If you have several isolines per **center** each one at a **isodistance** from center, separated by **delta**, you may want to punch them to avoid overlaps
```sql
SELECT
    center_id,
    isodistance,
    CASE isodistance
        WHEN delta THEN the_geom
        ELSE ST_Difference(
          the_geom,
          lag(the_geom) OVER (PARTITION BY center_id ORDER BY isodistance)
        )
    END AS the_geom
FROM
    my_isolines_dataset
```
### <a name="histogram"></a>11. Get the distribution of a variable (histogram)

We can get the histogram of a variable using [width_bucket](https://www.postgresql.org/docs/10/static/functions-math.html) function as following
```sql
WITH
a AS(
  SELECT
          min(my_field) as mn,
          max(my_field) as mx
  FROM my_dataset
)
SELECT
    width_bucket(my_field, a.mn, a.mx, number_of_buckets) as buckets,
    count(1)
FROM my_dataset, a
GROUP BY buckets
ORDER BY buckets
```
You will get back `number_of_buckets + 1` buckets, because it count  as `[bucket)`, so we get an extra bucket with the count of rows with `my_field = max(my_field) over()`

The buckets are equally distributed along your data, with a width of `(mx-mn)/number_of_buckets`. So, if you also want the limits per bucket ( **>= lower & < higher** ):
```sql
WITH
a AS(
    SELECT
        min(my_field) as mn,
        max(my_field) as mx
    FROM my_dataset
),
b as(
    SELECT
        width_bucket(my_field, a.mn, a.mx, number_of_buckets) as buckets,
        count(1)
    FROM my_dataset, a
    GROUP BY buckets
    ORDER BY buckets
)
SELECT
    b.*,
    a.mn + (b.buckets-1) * (a.mx-a.mn)/number_of_buckets as lower,
    CASE
        WHEN buckets <=number_of_buckets THEN
            a.mn + b.buckets * (a.mx-a.mn)/number_of_buckets
        ELSE null END AS higher
FROM b, a
```

### 12. The magic of round()

This function takes two parameter:
* the field or value to round
* the **negative** power of 10 your value its going to be rounded to.

You will find that
```sql
round(0.123456, 5) = 0.12345 -- rounded to 10^⁻5 => 5 decimals
round(0.123456, 3) = 0.123 -- rounded to 10^⁻3 => 3 decimals
round(0.123456, 0) = round(0.123456) = 0 -- rounded to 10^0 = 1 => closest unit
```
But... what if you use negative values for the second parameter?
```sql
round(67.81,0) = 68 -- rounded to 10^0 = 1 => closest unit
round(67.81,-1) = 70 -- rounded to 10^1 = 10 => closest ten
round(67.81,-2) = 100 -- rounded to 10^2 = 10 => closest hundred
round(67.81,-3) = 0 -- rounded to 10^3 = 10 => closest thousand
```
Cool, isn't it?

### 13. Some little known shorthand operators

* **||** concat strings

* **@**  abs

  * Small example: validating lat & long values

    ```sql
    SELECT * FROM my_dataset WHERE @latitude<90 AND @longitude<180
    ```

* **|/** square root

  * Example: equivalent radius of an arbitrary polygon

    ```sql
    SELECT |/( ST_Area(the_geom::geography) / PI() ) AS radius FROM my_dataset
    ```

* **||/** cube root

* **%** module

* **~**  regex comparision

  * Example: validate if a text field is numeric

    ```sql
    SELECT my_field ~ '^[0-9\.]+$' as isnumeric FROM my_dataset
    ```

    ​

### 14. NULL values management

You may find two different and uncomfortable null-related cases that can screw later aggregations (like averaging!!):

1. You receive a null where you expect a specific value
   * VG: buying a ticket with 100% discount should be a 0$ transaction, not a null one
2. You have an string like `NULL` , ` ` (empty string) or `-9999` in your source file for empty or invalid measures, but you want them to be proper nulls
   * VG:  heart rate monitor that fails to read the values from time to time and gives back a -9999 value when that happens

Let's fix them.

1. Just use `COALESCE`, the function that turns the null values in whatever you want
```sql
UPDATE transactions SET ticket = COALESCE(ticket, 0)
```
2. We have a pretty function called `NULLIF`, which is the opposite of COALESCE
```sql
UPDATE health_monitor SET heartrate = NULLIF(heartrate, -9999)
```

Other shitty situation: null values are placed before or after the rest of the values and screwing up your sorting. You can use the PostgreSQL instruction `NULLS FIRST` or `NULLS LAST` at the end of the sorting clause to let PostgreSQL know where you want the NULLs to be!

```sql
SELECT
    transaction_id,
    amount
ORDER BY amount DESC NULLS LAST
```

### 15. Double condition AOI to manage overlaps

Let's say that you need to solve this real live problem. You want to assign to each store the customers within 30min driving isochrone, but some isochrones overlap... so you need to define a new criteria to assign those customers in overlapped areas to an unique store. In this example we'll assign these customers to the closest store in terms of linear distance.

This is **NOT** a KNN problem. It's a two stages one:

1. Get the potential stores for each customer using isochrones
2. If there are more than one potential store, assign the customer to the closest store among the previous ones

OK, to speed the later intersection query, we are going to create an isochrones table for the stores

```sql
CREATE TABLE isochrones_30min AS
SELECT
    key_id,
    (my_isochrone_function(the_geom, 'car', ARRAY[1800]::integer[])).the_geom as the_geom
FROM my_stores_dataset
```

We're doing this to take advantage of the spatial indexes as we mention before, but if your tables are small enough you might want to just get the isochrones on the fly and inject the above SELECT query as the subquery `_s` in the query below.

First step: intersection between customers and isochrones, but we need to keep a row per customer and we may have overlaps. So lets aggregate the potential results in the subquery `_a`, using the technique described in [#2](#lateral) (but no need to use `LATERAL` keyword in this case, remember?):

```sql
SELECT
    c.*,
    array_agg(_s.key_id) as id_30mins
FROM
    customers c
LEFT JOIN
    isochrones_30min _s
ON ST_within(c.the_geom, _s.the_geom)
GROUP BY c.key_id
```

Check that the potentials stores IDs are aggregated in an array per customer, and that array might be an empty one for those customers further than 30min driving from any store.

Then, we should check the best store in the , using our beloved `LATERAL` keyword again for our subquery `_b`:

```sql
SELECT
    key_id
FROM
    my_stores_dataset
where
    array_length(_a.id_30mins, 1) > 0 AND
    array_position(_a.id_30mins, key_id) is not null
ORDER BY _a.the_geom <-> the_geom
LIMIT 1
```

So, the final assignation query would be like:

```sql
SELECT
    _a.*
    _b.key_id as store_id
FROM
(
    SELECT
        c.*,
        array_agg(_s.key_id) as id_30mins
    FROM
        customers c
    LEFT JOIN
        isochrones_30min _s
    ON ST_within(c.the_geom, _s.the_geom)
    GROUP BY c.key_id
) _a
LEFT JOIN
LATERAL(
    SELECT
        key_id
    FROM
        my_stores_dataset
    where
        array_length(_a.id_30mins, 1) >0 AND
        array_position(_a.id_30mins, key_id) is not null
    ORDER BY _a.the_geom <-> the_geom
    LIMIT 1
) _b
ON 1=1
```

Why the `ON 1=1` stuff at the end? Because we're using a `LEFT JOIN`  between `_a` and `_b`  to include all the customers in the original dataset, and the `ON` clause is compulsory for all the joins but the `CROSS` one. But we don't need to add any relation there because we're doing the filtering within the `LATERAL` subquery, hence the `1=1`.



### 16. Index over expressions

What if you want to render a query with calculated fields, or query a dataset scanning on a calculated field, and the performance is not as expected? You can add an [index on the calculation](https://www.postgresql.org/docs/9.5/static/indexes-expressional.html)!!

VG: indexing a distance in different units...
```sql
CREATE INDEX idx_distance_km ON routes(round(tripdistancemeters::numeric/1000.00, 2))
```

Or any other expression, using the example in the docs

```sql
CREATE INDEX people_names ON people ((first_name || ' ' || last_name))
```

Same with spatial indexes!! VG: indexing the centroids of a polygons dataset

```sql
CREATE INDEX idx_centroid on countries USING GIST(st_centroid(the_geom))
```

### 17. Spatial index clustering

I know that the average PostGIS user is aware of the basic requirement of indexing the geometry fields in order to get the expected performance, but I'm going one step beyond here.

We have the [1st Law of Geography](https://en.wikipedia.org/wiki/Tobler%27s_first_law_of_geography):
```
Everything is related to everything else, 
but near things are more related than distant things
```

So, it's easy to infer that most of the spatial analysis that the user might want to perform on a spatial dataset will be focused on neighbor features. So, it would be great if we can use this `neighborhood` property to speed up our spatial queries. 

PostgreSQL has a nice functionality called [CLUSTER](https://www.postgresql.org/docs/current/static/sql-cluster.html) that reshuffles the data in a dataset based on a given index, so the rows with close index are physically placed close to each other. 

So, what if we cluster our dataset based on our spatial index? Features that are close in the real world will be physically placed close to each others in the dataset! 

```sql
CLUSTER mytable USING my_spatial_index;
```

This action takes less than 2s / 100K rows, and depending on the size of the dataset and how spatially spread is it, can lead to a **performance improvement of up to 90%** in later spatial queries.

As a real live example, working on [QuadGrid](https://abelvm.github.io/sql/quadgrid/), clustering lowered the processing time for a 400K rows dataset, worldwide, from 4:15 min to 39s. That's ~ 85% improvement!