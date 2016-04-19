% That SQL looks pretty complicated. Where are the tests?
% James E. Marca
% 2016-04-20

# Overview

* Database tables are super useful
* But they can get a little messy
* and the SQL used to create them can get complex


# Motivation: Clean up and test SQL

--------

![Urban area highways have loop detectors](./figures/detectors_on_map.png)


--------

![Focusing on just the highways](./figures/detectors_on_map_4.png)


# To segmentize the points


![Segmentize points](./figures/segmentation.png)


# I wrote this SQL

```sql
INSERT INTO osm_upgrade.vds_segment_geometry
   (vds_id,adj_pm,refnum,direction,seggeom)
SELECT vds_id,adj_pm,refnum,direction,seggeom
FROM (
  SELECT pdist,ptsec,ndist,vds_id,adj_pm,refnum,
         relation_direction as direction,
         -- vdsid, order, freeway id, freeway dir

         -- AND the computed segment geometry associated with the vds
         CASE WHEN ptsec <= ntsec

-- The previous vds comes before the next.
-- Create a line segment by taking the piece of the
-- route line between ptsec and ntsec

              THEN ST_line_substring( q.rls, ptsec,ntsec )

              ELSE

-- The "previous" vds comes /after/ the
-- "next". Probably this means that they are pointing
-- at different line segments.  Hardcode to zero or one

                    case
                      WHEN mydist > pdist then ST_line_substring( q.rls, ptsec, 1 )
                      WHEN mydist < pdist then ST_line_substring( q.rls, 0, ntsec )
                    end
            END AS seggeom
       FROM
       (

-- here we select the points halfway between a target vds and the
-- vds's on either side of it (p=previous,n=next) along the route.  The
-- points are specified as the distance along the route linestring.

         SELECT vrr.vds_sequence_id AS sequence_id,
                vrr.vds_id,vrr.adj_pm,vrr.refnum,vrr.relation_direction,
                -- vdsid,pm, and osm relation
                vrr.line AS rls,        -- the linestring the vds snaps to
                vrr.dist as mydist,
                vrr.numline AS myline,  -- to which multilines are we snapped?
                p.dist AS pdist,      -- the distance along the
                                      -- line of the previous vds
                CASE WHEN p.dist IS NOT NULL   -- this computes the bisection
                     THEN (vrr.dist+p.dist)/2  -- distance between the previous
                     ELSE 0                    -- and target vds
                     END AS ptsec,
                n.dist AS ndist,      -- the distance along the
                                      -- line of the next vds
                CASE WHEN n.dist IS NOT NULL   -- this computes the bisection
                     THEN (vrr.dist+n.dist)/2  -- distance between the target
                     ELSE 1                    -- and next vds
                     END AS ntsec

            FROM osm_upgrade.vds_route_relation vrr
                 -- joining the previous vdsid
                 LEFT JOIN osm_upgrade.vds_route_relation p
                      ON ( vrr.vds_sequence_id=p.vds_sequence_id+1 AND
                           vrr.refnum=p.refnum AND
                           vrr.relation_direction=p.relation_direction AND
                           vrr.numline=p.numline)
                 -- and joining the next vdsid
                 LEFT JOIN osm_upgrade.vds_route_relation n
                      ON ( n.vds_sequence_id=vrr.vds_sequence_id+1 AND
                           vrr.refnum=n.refnum AND
                           vrr.relation_direction=n.relation_direction AND
                           vrr.numline=n.numline)

                 -- We need to the route line too as all our
                 -- computations are based upon that
                 LEFT JOIN tempseg.numbered_route_lines rl
                      ON (rl.refnum=vrr.refnum AND
                          vrr.relation_direction=rl.direction)
                 -- WHERE
                 -- rl.refnum=99 and rl.direction='north'
                 -- rl.refnum=580
            ORDER BY vrr.vds_sequence_id
       ) q
       ORDER BY sequence_id
) qq WHERE geometrytype(seggeom) !~* 'point'
;
```
... and several other files of similar functions.

--------

!["Brutti ma buoni!"](./figures/brutti_ma_buoni__mg_9082-15.jpg)


# Can't test that!


# Talk objectives

* Give real-world examples of using pgTAP and Sqitch
* Hoping others can learn from my mistakes.


# The examples

1. Cleaning up after a SQL disaster
1. Fixing a bad design choice (another SQL disaster)
2. Modularizing, testing, and porting that monstrous SQL mapping hack
3. Switching from OSM to official Caltrans map data



# About me

* James Marca
* Transportation Researcher at UC\ Irvine
* Activimetrics LLC
* twitter @jmarca
* github https://github.com/jmarca
* occasional posts at https://contourline.wordpress.com

# My PostgreSQL experience

* Ever since 2001 (ish)
* Needed a geo-enabled DB
* MySQL didn't at the time
* PostgreSQL had PostGIS
* Stored and processed streaming GPS data
    * (live, from cars, over a CDPD modem!)


# Also

* I set up Slashcode using PostgreSQL
* (That's also how I learned Perl)


# Acknowlegments

Shout out to the agencies funding my research over the years

# Caltrans
## DRI/SI and District 12

Mission: Provide a safe, sustainable, integrated and efficient
transportation system to enhance California's economy and livability.

# California Air Resources Board

Mission: To promote and protect public health, welfare and ecological
resources through the effective and efficient reduction of air
pollutants while recognizing and considering the effects on the
economy of the state.


# Context

# CalVAD: California Vehicle Activity Database

Estimate, from data, vehicle movements on state roads and highways

# Input data sources

* 10,000+ inductive loop detectors (VDS)
* 100+ Weigh In Motion (WIM) stations
* Estimated annual vehicle counts on every street

----------

![Loop detector](figures/ramp_loops.jpg)


----------

![WIM bending plate](figures/bendingplate.jpg)

----------

![pneumatic tubes for two-day counts](figures/tube.scaled.jpeg)



# Part 1: SQL disaster




# The original project (2002)

* Q: "Can you display our safety model results on a map?"
* A: "Sure"

# VDS Loop Data

```csv
ID,       Fwy,  Dir, District, County, City,  State_PM, Abs_PM, Latitude,  Longitude,   Length, Type, Lanes,    Name,    User_ID_1, User_ID_2, User_ID_3, User_ID_4
1201044,  133,  S,     12,      59,    36770,  9,        8.991, 33.66184,  -117.7553,     ,      OR,    1,   BARRANCA2,    565,,,
1201052,  133,  S,     12,      59,    36770,  9,        8.991, 33.66184,  -117.7553,     ,      FR,    1,   BARRANCA 2,   565,,,
1201054,  133,  S,     12,      59,    36770,  9,        8.991, 33.66184,  -117.7553,   2.685,   ML,    3,   BARRANCA2,    565,,,
1201058,  133,  N,     12,      59,    36770,  8.866,    8.857, 33.659542, -117.756294,   ,      OR,    1,   BARRANCA1,    551,,,
...
```

# Solution

* Put the VDS data into PostgreSQL
* Use PostGIS to make Point geometries
* Show the points as a layer on a web map
* Wire up "click" to call up modeling results

# Solution

* Put the VDS data into PostgreSQL
* Use PostGIS to make Point geometries
* ~~Show the points as a layer on a web map~~
* ~~Wire up "click" to call up modeling results~~


# VDS loop detector tables

1. Loop detector metadata
2. Loop detector geometry in SRID 4269 (original)
3. Loop detector geometry in SRID 4326 (for Google Maps)
4. Join tables


--------

!["Normalized" Tables](./figures/vds_geom_diagram.png)

# Load up using perl

* Parse the CSV metadata file
* For each detector
    * find or create entry in `vds_id_all`
    * find or create entry in `geom_points_4269`
    * use PostGIS to transform geometry for `geom_points_4326`
    * make the join table entries

# Loading code

``` {.perl}
my $vds = $self->get_vds_or_die($pk,$data);
my $lon = $vds->longitude;
my $lat = $vds->latitude;
my $new_geoid = $self->get_new_geoid;
my $gid        = $new_geoid->gid;
my $creategeom = $self->create_geompoint_4269($lon,$lat,$gid);
$self->join_geom($vds,$gid,4269);
$new_geoid = $self->get_new_geoid();
my $projectedgid        = $new_geoid->gid;
my $projectedpoint = $self->project_geom_4326($gid,$projectedgid);
$self->join_geom($vds,$projectedgid,4326);
```

----

![](./figures/vds_geom_diagram_1.png)

``` {.perl}
my $vds = $self->get_vds_or_die($pk,$data);
```

----

![](./figures/vds_geom_diagram_2.png)

``` {.perl}
my $new_geoid = $self->get_new_geoid;
my $gid        = $new_geoid->gid;
my $creategeom = $self->create_geompoint_4269($lon,$lat,$gid);
```

----

![](./figures/vds_geom_diagram_3.png)

``` {.perl}
$self->join_geom($vds,$gid,4269);
```

----

![](./figures/vds_geom_diagram_4a.png)

``` {.perl}
$new_geoid = $self->get_new_geoid();
my $projectedgid        = $new_geoid->gid;
my $projectedpoint = $self->project_geom_4326($gid,$projectedgid);
```

------

```{.perl .tall}
method project_geom_4326(Int $gid4269,Int $gid4326){
    my $transform;
    my $test_eval = eval {
        my $arr = [q{ST_AsText(ST_Transform(me.geom,4326))}];
        ($transform) = $self->resultset('Public::GeomPoints4269')->search(
            { 'gid' => $gid4269 },
            {
                select => [ 'gid', \$arr ],
                as => [qw/gid  wkt_transform /],
            },
            );
    };
    if ($EVAL_ERROR) {    # find or create failed
        carp "can't fetch transform $EVAL_ERROR";
        croak;
    }
    my $projectedpoint;
    $test_eval = eval {
        my $arr = [
            q{ST_GeometryFromText(?,4326)},
            ['dummy'=>$transform->get_column('wkt_transform')]
            ];
        $projectedpoint = $self->resultset('Public::GeomPoints4326')->create(
            {
                gid  => $gid4326,
                geom => \$arr,
            }
            );
    };
    if ($EVAL_ERROR) {    # find or create failed
        carp "can't create new tranformed geometry $EVAL_ERROR";
        croak;
    }
    return $projectedpoint;
}
```

----

![](./figures/vds_geom_diagram_4.png)

``` {.perl}
$new_geoid = $self->get_new_geoid();
my $projectedgid        = $new_geoid->gid;
my $projectedpoint = $self->project_geom_4326($gid,$projectedgid);
```

----

![](./figures/vds_geom_diagram_5.png)

``` {.perl}
$self->join_geom($vds,$projectedgid,4326);
```


# 2010 CalVAD project

# Add Weigh-In-Motion stations

* About 150 WIM stations
* Similar approach as with VDS sites
* Metadata table, join table, geometry table

# But I had an idea!

* Why not just reuse the geom\_point\_xxxx tables?

--------

![WIM points and VDS points pull from the same points tables](./figures/add_wim_stations_colored.png)



# Similar create/populate perl

* Parse the CSV metadata file
* For each detector
    * find or create entry in `wim_stations`
    * find or create entry in `geom_points_4269`
    * use PostGIS to transform geometry for `geom_points_4326`
    * make the join table entries




# Years go by

-----

![](./figures/2010_10_15.jpg)


-----

![](./figures/2015_11_11.jpg)


# Transfer CalVAD to ARB

* Some database tables are way too big to copy
* Many are manageable, but
* Learning by doing
* Teach my ARB counterpart what to do:
    * Load the required software
    * Download the data
    * Run the code

# No metadata overlap...

* My DB had processed through 2012
* ARB downloaded 2013, 2014, 2015 data
* I cleaned up [my perl code](https://github.com/jmarca/CalVAD-PEMS-StationsParse) (now using Dist::Zilla)
* I added tests to perl code
* On ARB server, we
    * Installed perl, etc
    * Installed PostgreSQL, PostGIS
    * Parsed and loaded 2013, 2014, 2015 VDS metadata


--------

![Recall the DB tables...](./figures/add_wim_stations.png)


# Weeks pass

* Regular phone calls working through CalVAD code
* Setting up software
* Populating DB tables
* Processing data

# Time to use the WIM detector metadata

* For WIM sites, about the same data now as in 2010
* Slower pace of change
* I decided *not* to upgrade/modernize my WIM perl code
* I decided *instead* to just copy the WIM metadata to ARB


# Oops!


--------

![The WIM join tables got overwritten](./figures/add_wim_stations_c2.png)



# I broke the join table

* We created new VDS points in the new DB
* The actual GID values are completely different
* But I copied the wim\_points\_xxx tables from my DB
* *with the wrong GID values*


# Fix this

* pgTAP
     * to prove there is a problem
     * to prove the fix worked
* Sqitch
     * to organize the fix

# pgTAP

* TAP = test anything protocol
* Lets you run tests in the database
* github: [https://github.com/theory/pgtap](https://github.com/theory/pgtap)
* docs: [http://pgtap.org](http://pgtap.org)

# pgTAP "Hello World"

``` sqlpostgresql
SET client_min_messages TO warning;
CREATE EXTENSION IF NOT EXISTS pgtap;
RESET client_min_messages;

BEGIN;
SELECT no_plan();
-- SELECT plan(1);

SELECT pass('Hello Brooklyn!');

SELECT finish();
ROLLBACK;
```

# Testing

> The basic purpose of pgTAP ... is to print out either\ "ok #" or
> "not ok #", depending on whether a given test succeeded or failed.

---from the README

# pgTAP basic usage

* As a simple test script
* In xUnit-style test functions that you install into your database
  and run all at once


# pgTAP advanced features

* Can integrate with any TAP testing system
* TAP consumers exist in most languages
* xUnit-style usage
* compatible with continuous integration servers
    * Hudson, Travis etc


# I don't really know what that previous slide means

-----

![This is a can of Shinola, a shoe polish brand that went out of business](./figures/shinola_container.png)


# My testing needs

1. Make sure the database tables exist
2. Verify primary keys
3. Make sure the tables reference each other
4. Make sure each WIM station has a geometry
4. Verify metadata (lat,lon) matches the linked geometry

# Testing approach

* Simple test script
* Run the script using pg_prove

# pg_prove

* http://search.cpan.org/dist/TAP-Parser-SourceHandler-pgTAP

```
cpanm --sudo TAP::Parser::SourceHandler::pgTAP
```

# cpanm?

```
cpan install App::cpanminus
```
... then later ...

```
cpanm -S --self-upgrade
```

# Testing tables, schemas

* has_schema()
* has_table()
* schemas_are()
* tables_are()

# The 'has_' test variants

* Checks that something exists
* The test fails if that thing is not in the database
* For example

``` sqlpostgresql
SELECT has_table('public','wim_stations','there is a wim_stations table');
```

# The ‘_are' test variants

* Checks that the DB contains exactly the list you specify
* Will fail if something is missing
* Will fail if something extra exists
* For example

``` sqlpostgresql
SELECT schemas_are(ARRAY[ 'public', 'wim','osm','vds','sqitch','pgRouting' ],
                   'expected schemas exist in database');
```

# Basic tests

``` sqlpostgresql
-- is there a wim_stations table?
SELECT has_table('public', 'wim_stations', 'there is a wim_stations table');

-- are  there join tables?
SELECT has_table('public', 'wim_points_4269', 'there is a wim_points_4269 table');
SELECT has_table('public', 'wim_points_4326', 'there is a wim_points_4326 table');

-- are there geometry tables?
SELECT has_table('public', 'geom_points_4269','there is a geom_points_4269 table');
SELECT has_table('public', 'geom_points_4326','there is a geom_points_4326 table');
```

# quick demo

# Testing table characteristics

* col_is_pk()
* has_pk()
* col_is_fk()
* has_fk()


# Primary keys?


``` sqlpostgresql
SELECT col_is_pk('public', 'wim_stations', 'site_no', 'wim stations site_no is pk' );
SELECT col_is_pk('public', 'geom_points_4269','gid', 'geom points 4269 gid is pk' );
SELECT col_is_pk('public', 'geom_points_4326','gid', 'geom points 4326 gid is pk' );

SELECT col_is_pk('public', 'wim_points_4326','wim_id', 'geom points 4326 gid is pk' );
SELECT col_is_pk('public', 'wim_points_4269','wim_id', 'geom points 4269 gid is pk' );
```

# quick demo


# Testing foreign keys

``` sqlpostgresql
SELECT fk_ok( :fk_schema, :fk_table,   :fk_columns,
              :pk_schema,  :pk_table, :pk_columns,
              :description );


```

--------

```{.sqlpostgresql .tall}
-- 4269
SELECT fk_ok(
    'public','wim_stations','site_no',
    'public','wim_points_4269','wim_id',
    'wim_points_4269 connects to wim_stations'
);
SELECT fk_ok(
    'public','geom_points_4269','gid',
    'public','wim_points_4269','gid',
    'wim_points_4269 connects to geom_points_4269'
);


--4326
SELECT fk_ok(
    'public','wim_stations','site_no',
    'public','wim_points_4326','wim_id',
    'wim_points_4326 connects to wim_stations'
);
SELECT fk_ok(
    'public','geom_points_4326','gid',
    'public','wim_points_4326','gid',
    'wim_points_4326 connects to geom_points_4326'
);
```

# quick demo


# Not limited to simple tests

```sqlpostgresql
SELECT results_eq(   A,      B,    :description );

SELECT results_eq( :sql,    :sql,    :description );
SELECT results_eq( :sql,    :array,  :description );
SELECT results_eq( :cursor, :cursor, :description );
SELECT results_eq( :sql,    :cursor, :description );
SELECT results_eq( :cursor, :sql,    :description );
SELECT results_eq( :cursor, :array,  :description );
```

results_eq() is a workhorse for my kind of problems.

# Caveat

* Need to quote `:sql` arguments
    * so use `prepare` statements

* results_eq() does item by item comparison of A vs B
    * Be careful to order results explicitly


# Have enough entries?

``` sqlpostgresql
-- make sure join relations are all there
select results_eq(
    'select site_no from wim_stations order by site_no',
    'select wim_site from wim_points_4269 order by wim_site'
);

select results_eq(
    'select site_no from wim_stations order by site_no',
    'select wim_site from wim_points_4326 order by wim_site'
);

```

# quick demo


# Geometries vs metadata

```{.sqlpostgresql .tall}
-- do the geometries actually match the metadata in all cases?
PREPARE build_geoms_4269 AS
  SELECT ST_GeomFromEWKT('SRID=4269;POINT('||a.longitude||' '||a.latitude||')')
  FROM wim_stations a
  ORDER BY site_no;

PREPARE existing_geoms_4269 AS
  SELECT b.geom
  FROM wim_points_4269 a
  JOIN geom_points_4269 b USING (gid)
  ORDER BY wim_id;

-- expect that the metadata matches the stored 4269 geoms
SELECT results_eq(
    'build_geoms_4269',
    'existing_geoms_4269'
);
```

# Geometries vs metadata

```{.sqlpostgresql .tall}
-- now test projection
PREPARE projected_geoms_4269_to_4326 AS
  SELECT ST_TRANSFORM(b.geom,4326)
  FROM wim_points_4269 a
  JOIN geom_points_4269 b USING (gid)
  ORDER BY wim_id;

PREPARE existing_geoms_4326 AS
  SELECT b.geom
  FROM wim_points_4326 a
  JOIN geom_points_4326 b USING (gid)
  ORDER BY wim_id;


--expect that the transformed 4269 geoms matches the stored 4326 geoms
SELECT results_eq(
    'projected_geoms_4269_to_4326',
    'existing_geoms_4326'
);
```

# quick demo


# So, it is broken. How do I fix it?

# use Sqitch

From the same team of one that brought use pgTAP comes Sqitch

(No 'u')

# What is Sqitch?

* Docs: <http://sqitch.org/>
* Code: <https://github.com/theory/sqitch>
* A database change management application.
* Supports PostgreSQL 8.4+,
    * (also SQLite 3.7.11+, MySQL 5.0+, Oracle 10g+,
    Firebird 2.0+, and Vertica 6.0+.)


* Feels like:
    * git when you create and store changes to your plan
    * a package manager when you deploy, require dependencies, etc

# Sqitch documentation is great

Really.  Its superb.


# Sqitch "Hello World"

```bash
sqitch init calvad_wim_geometries --uri https://github.com/jmarca/calvad_wim_geometries --engine pg
sqitch target add brokendb db:pg:slash@127.0.0.1/brokendb
sqitch engine add pg brokendb
sqitch add brooklyn  -m 'Hello Brooklyn!'
sqitch deploy brooklyn
```

# TODO here:  test.  test case

broken database, deploy the fix.  Use the test files I just copied




# Part 2: A bad design decision


# VDS data changes

* Regular releases of new metadata files
* Need to download and parse the metadata files for
    * new loop detectors coming on line
    * missing loop detectors that were retired

# Metadata

+------------------------------+------------------------------+
| Things that don't change     | Things that change           |
+==============================+==============================+
| ID                           | State PM                     |
+------------------------------+------------------------------+
| Fwy                          | Absolute PM                  |
+------------------------------+------------------------------+
| Direction                    | Length                       |
+------------------------------+------------------------------+
| District                     | User ID fields               |
+------------------------------+------------------------------+
| County                       |                              |
+------------------------------+------------------------------+
| City                         |                              |
+------------------------------+------------------------------+
| Latitude                     |                              |
+------------------------------+------------------------------+
| Longitude                    |                              |
+------------------------------+------------------------------+
| Type                         |                              |
+------------------------------+------------------------------+
| Name                         |                              |
+------------------------------+------------------------------+





# Part 3: modularize and test detector geolocation/segmentation code

Show the big code and how I use Sqitch inter-package dependencies


# part 4: migrate to caltrans state highway network

Not sure what the point of this one is in terms of sqitch and pgtap
usage
