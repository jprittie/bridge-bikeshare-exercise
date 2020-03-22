## Part 1: Missing Station Data

1) Write a query that fills out the `stations` table
Here, I am doing a union so that I have data for visits to all stations. Then I group by station_id and station_name and rank those groups; this way I can pick the best name for each station ID based on which name appears most frequently in the trip data.

```sql
INSERT INTO stations
(SELECT id, name FROM
  (
    SELECT station_name AS name, station_id AS id, COUNT(*) as visits, rank() OVER (PARTITION BY station_id ORDER BY COUNT(*) DESC) as rank
    FROM
      (
          SELECT from_station_name AS station_name, from_station_id AS station_id FROM trips
          UNION ALL
          SELECT to_station_name AS station_name, to_station_id AS station_id from trips
      ) AS all_stations
    WHERE station_id IS NOT NULL
    GROUP BY station_id, station_name
    ORDER BY station_id ASC, visits DESC
  ) AS grouped_stations
WHERE rank = 1
)
```

2) Should we add any indexes to the stations table, why or why not?

3) Fill in the missing data in the `trips` table based on the work you did above
There are three cases to cover:
- Where name is correct and id is NULL, fill in id
- Where name is incorrect and id is NULL, fill in id
- Where name is incorrect, make it correct

It will be easier to correct all the names first. So I will use the above query, and join it with a modification of the query where rank = 2. This will give me a table that matches correct names and incorrect names in the same row.

I will use this to first update the `from_station_name` column. Then I will modify it to update the `to_station_name` column. Here is the first of those queries (the second version will just update the second line and last line with `to_station_name`):

```sql
UPDATE trips
SET from_station_name = namecheck.right_name
FROM
    (SELECT subquery1.correct_station_name AS right_name, subquery2.wrong_station_name AS wrong_name FROM


            (SELECT id, name as correct_station_name FROM
                (
                    SELECT station_name AS name, station_id AS id, COUNT(*) as visits, rank() OVER (PARTITION BY station_id ORDER BY COUNT(*) DESC) as rank
                    FROM
                    (
                        SELECT from_station_name AS station_name, from_station_id AS station_id FROM trips
                        UNION ALL
                        SELECT to_station_name AS station_name, to_station_id AS station_id from trips
                    ) all_stations1
                    WHERE station_id IS NOT NULL
                    GROUP BY station_id, station_name
                    ORDER BY station_id ASC, visits DESC
                ) grouped_stations1
                WHERE rank = 1
            ) subquery1

            JOIN

            (SELECT id, name as wrong_station_name FROM
                (
                    SELECT station_name AS name, station_id AS id, COUNT(*) as visits, rank() OVER (PARTITION BY station_id ORDER BY COUNT(*) DESC) as rank
                    FROM
                    (
                        SELECT from_station_name AS station_name, from_station_id AS station_id FROM trips
                        UNION ALL
                        SELECT to_station_name AS station_name, to_station_id AS station_id from trips
                    ) all_stations2
                    WHERE station_id IS NOT NULL
                    GROUP BY station_id, station_name
                    ORDER BY station_id ASC, visits DESC
                ) grouped_stations2
                WHERE rank = 2
            ) subquery2

            ON subquery1.id = subquery2.id

    ) namecheck
WHERE from_station_name=namecheck.wrong_name
```

Then, we can use the `stations` table to fill in the missing station ids in the `trips` table.
```sql
    UPDATE trips_backup
    SET from_station_id = idcheck.id
    FROM
        (SELECT id, name
         FROM stations
        ) idcheck
    WHERE from_station_name = idcheck.name AND from_station_id IS NULL
```
and repeat that query to update the `to_station_id` column.

In theory, that should take care of cleaning up the station ids and station names in the trips table.

However, I can see that there are still lots of rows where `from_station_id` and/or `to_station_id` are still `NULL`. This may be because some stations had more than two variations on a name, or because I made some other error in my queries.