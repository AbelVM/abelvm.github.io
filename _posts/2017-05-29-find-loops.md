---
title: "Dymanically find loops in a tracks dataset"
tags: gpx track loops
category: SQL
---

To write te article [Spies in the sky](https://www.buzzfeed.com/peteraldhous/spies-in-the-skies), the authors needed a multi-step analysis, explained [here](https://buzzfeednews.github.io/2016-04-federal-surveillance-planes/analysis.html)

Now, most of the work cand be replicated with a simple function applied using [CARTO SQL Node analysis](https://github.com/CartoDB/camshaft/blob/master/docs/deprecated-sql-function.md).

{% raw %}
<iframe width="100%" height="520" frameborder="0" src="https://team.carto.com/u/abel/builder/3e25257e-4219-11e7-a133-0ecd1babdde5/embed" allowfullscreen webkitallowfullscreen mozallowfullscreen oallowfullscreen msallowfullscreen></iframe>
{% endraw %}

[Link to the map](https://team.carto.com/u/abel/builder/3e25257e-4219-11e7-a133-0ecd1babdde5/embed)

The function used is based on the approach published in [GIS STack Exchange](https://gis.stackexchange.com/questions/206815/seeking-algorithm-to-detect-circling-and-beginning-and-end-of-circle), and modified to accept multiple flights:

```sql
CREATE OR REPLACE FUNCTION DEP_EXT_findloops(
        operation text,
        table_name text,
        primary_source_query text,
        primary_source_columns text[],
        cat_column text,
        temp_column text
    )
    RETURNS VOID AS $$
        DECLARE
            tail text;
            categorized text;
            cat_string text;
            cat_string2 text;
            sub_q text;
            s record;
            gSegment            geometry = NULL;
            gLastPoint          geometry = NULL;
            gLastTrackID        text = NULL;
            gLoopPolygon        geometry = NULL;
            gRadius             numeric;
            iLoops              integer := 0;
            cdbi                bigint := 0 ;
        BEGIN

            IF operation = 'create' THEN

                EXECUTE 'DROP TABLE IF EXISTS ' || table_name;

                EXECUTE 'CREATE TABLE ' || table_name || '(cartodb_id bigint, track_id text, loop_id integer, the_geom geometry(Geometry,4326), radius numeric)';

            ELSEIF operation = 'populate' THEN

                IF  trim(temp_column) = '0' THEN
                   temp_column := 'cartodb_id';
                END IF;

                IF  trim(cat_column) = '0' THEN
                    categorized := ' order by ' || temp_column;
                    cat_string := '';
                ELSE
                    categorized := 'partition by ' || cat_column || ' order by ' || temp_column;
                    cat_string := cat_column;
                END IF;

                -- partition and sorting of the input
                sub_q := 'WITH '
                    ||  'prequery as('
                    ||      'SELECT '
                    ||      cat_string || ' as cat,'
                    ||      temp_column || ' as temp_column,'
                    ||      'the_geom as point'
                    ||      ' FROM ('
                    ||          primary_source_query
                    ||      ') _q'
                    ||   '),'
                    ||  'pts as('
                    ||      'SELECT '
                    ||      ' cat'
                    ||      ', temp_column'
                    ||      ', point'
                    ||      ', row_number() over(partition by cat order by temp_column) as index'
                    ||      ' FROM prequery'
                    ||      ' ORDER BY cat, temp_column'
                    ||  ')'
                    ||      'SELECT '
                    ||      ' b.cat::text as track_id'
                    ||      ', ST_MakeLine(ARRAY[a.point, b.point]) AS geom'
                    ||      ' FROM pts as a, pts as b'
                    ||      ' WHERE b.index > 1'
                    ||      ' AND a.index = b.index - 1'
                    ||      ' AND a.cat = b.cat '
                    ||      ' ORDER BY b.cat, b.temp_column';

                FOR s IN EXECUTE sub_q
                LOOP

                    -- restart when new track
                    if gLastTrackID <> s.track_id then
                        gSegment := null;
                        gLastPoint := null;
                        iLoops := 0;
                    end if;

                    -- build segments
                    if gSegment is null then
                        gSegment := s.geom;
                    elseif ST_equals(s.geom, gLastPoint) = false then
                        gSegment := ST_Makeline(gSegment, s.geom);
                    end if;

                    gLoopPolygon := ST_BuildArea(ST_Node(ST_Force2D(gSegment)));


                    if gLoopPolygon is not NULL and ST_Numpoints(gSegment) > 3 then

                        iLoops := iLoops + 1;
                        gRadius := (|/ ST_area(gLoopPolygon::geography)/PI());
                        gSegment := null;

                        EXECUTE
                        'INSERT INTO '
                        || quote_ident(table_name)
                        || ' VALUES('
                        || cdbi || ', '
                        || quote_literal(s.track_id) || ', '
                        || iLoops ||', '
                        || 'ST_GeomFromText(' || quote_literal(ST_astext(gLoopPolygon)) || ', 4326), '
                        || gRadius || ')';

                        cdbi := cdbi +1;

                    end if;

                    IF  trim(cat_column) <> '0' THEN
                        gLastTrackID = s.track_id;
                    END IF;

                    gLastPoint := s.geom;

                END LOOP;


            END IF;

        END;
$$ LANGUAGE plpgsql;
```
