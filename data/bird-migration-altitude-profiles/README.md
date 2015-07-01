# Bird migration altitude profiles

This dataset contains bird migration data for [this case study](https://github.com/enram/case-study) at different altitudes, measured by five weather radars. It is standardized as a [data package](http://dataprotocols.org/data-packages/).

## Description

The dataset is a time series. Each record represents a **timeframe** (`start_time` and `end_time`, generally around 5 minutes) at which the radar (`radar_name` and `radar_id`, which can be joined with [radars](https://github.com/enram/case-study/tree/master/data/radars) to get a geospatial representation) scanned at multiple altitudes (`altitude`, generally 30, every 200 meters). For each altitude, the calculated density of the birds is given (`bird_density`) in birds/km<sup>3</sup>, as well as the vertical speed (`w_speed`), and horizontal speed (`u_speed` towards East, `v_speed` towards North) of the general movement in m/s. The horizontal speed is also expressed in `ground_speed` and `direction`.

Please note that the bird density is only valid and calculated if `radial_velocity_std` is above 2.0. `bird_density` is set to 0 when the `radial_velocity_std` is below 2.0. If using `ground_speed` or `direction`, use this same threshold to discriminate between birds and weather.

All the fields are described in detail in the [metadata](datapackage.json).

## Data

* Bulk download: [bird-migration-altitude-profiles.csv](bird-migration-altitude-profiles.csv)
* SQL API: [http://lifewatch.cartodb.com/api/v2/sql?q=SELECT * FROM bird_migration_altitude_profiles LIMIT 10](http://lifewatch.cartodb.com/api/v2/sql?q=SELECT%20*%20FROM%20bird_migration_altitude_profiles%20LIMIT%2010)

*To use the API, see the [CartoDB API documentation](http://docs.cartodb.com/cartodb-platform/sql-api.html).*

## Procedure for aggregating the data

For visualizations such as the [bird migration flow visualization](http://enram.github.io/bird-migration-flow-visualization/viz/) and [TIMAMP](http://timamp.github.io/), the raw bird migration altitude profile data are aggregated per time and altitude, and have to meet certain conditions. The procedure below describes the recommended method to do so and is the one implemented by the two visualizations.

### Conditions

* Only use measurements between 0.2 and 4.0km (both inclusive). This results in 19 altitudes, from 0.3 to 3.9km.
* For u speed/v speed:
    * Only consider speed if the radial velocity standard deviation is above or equal to 2 (to make sure we are only considering *birds*) and the bird density is above or equal to 1 birds/km<sup>3</sup> (for more precise *bird* speed).
    * Set speed to `null` if those conditions are not met. Original `null` values will remain `null`.
    * Since we check for the same conditions, either both u speed and v speed will meet the requirements or none.
* For bird density:
    * Only consider bird density if the radial velocity standard deviation is above or equal to 2 (to make sure we are only considering *birds*).
    * Set bird density to 0 if those conditions are not met.
    * Keep original `null` values as `null`.

### Aggregation

* Aggregate in two altitude bands†:
    * 0.2 to 1.6km. This aggregates 7 altitudes, from 0.3 to 1.5.
    * 1.6 to 4.0km. This aggregates 12 altitudes, from 1.7 to 3.9.
* Aggregate per 20 minutes†. This typically combines 4 different date/times.
* For u speed/v speed:
    * Average the speeds, if at least one of the altitudes within the altitude band has a speed which met the conditions.
    * Set the speed to 0 if none of the altitudes within the altitude band has a speed which met the conditions, but there were originaly speeds detected.
    * Keep `null` if no speeds were detected.
* For bird density (birds/km<sup>3</sup>):
    * Average the bird density (0 values will affect this, which is ok).
* For vertical integrated density (birds/km<sup>2</sup>):
    * For the lower altitude band, multiply the average bird density by the number of altitudes† (7) and divide by 5 (to get to 1km instead of 200m).
    * For the higher altitude band, multiply the average bird density by the number of altitudes† (12) and divide by 5 (to get to 1km instead of 200m).
    * Note: `null` values will affect this "sum", which is not good, but we haven't found such values in the data yet.

† The number of altitude bands and time window depend on the visualization.

### SQL

The above procedure as PostgreSQL on CartoDB:

```SQL
WITH conditional_data AS (
    SELECT
        radar_id,
        date_trunc('hour', start_time) + date_part('minute', start_time)::int / 20 * interval '20 min' AS interval_start_time,
        CASE
            WHEN altitude >= 0.2 AND altitude < 1.6 THEN 1
            WHEN altitude >= 1.6 THEN 2
        END AS altitude_band,
        u_speed,
        CASE
            WHEN radial_velocity_std >= 2 AND bird_density >= 1 THEN u_speed
            ELSE null
        END AS conditional_u_speed,
        v_speed,
        CASE
            WHEN radial_velocity_std >= 2 AND bird_density >= 1 THEN v_speed
            ELSE null
        END AS conditional_v_speed,
        CASE
            WHEN bird_density IS NULL THEN NULL
            WHEN radial_velocity_std >= 2 THEN bird_density
            ELSE 0
        END AS bird_density
    FROM
        lifewatch.bird_migration_altitude_profiles
    WHERE
        altitude >= 0.2
        AND altitude <= 4.0
)

SELECT
    radar_id,
    interval_start_time,
    altitude_band,
    CASE
        WHEN avg(conditional_u_speed) IS NOT NULL THEN round(avg(conditional_u_speed)::numeric,5)
        WHEN avg(u_speed) IS NOT NULL THEN 0
        ELSE null
    END AS avg_u_speed,
    CASE
        WHEN avg(conditional_v_speed) IS NOT NULL THEN round(avg(conditional_v_speed)::numeric,5)
        WHEN avg(v_speed) IS NOT NULL THEN 0
        ELSE null
    END AS avg_v_speed,
    round(avg(bird_density)::numeric,5) AS avg_bird_density,
    CASE
        WHEN altitude_band = 1 THEN round((avg(bird_density) * 7 /5)::numeric,5)
        WHEN altitude_band = 2 THEN round((avg(bird_density) * 12 /5)::numeric,5)
    END AS vertical_integrated_density,
    count(*) AS number_of_measurements
FROM conditional_data
GROUP BY
    radar_id,
    interval_start_time,
    altitude_band
ORDER BY
    interval_start_time,
    radar_id,
    altitude_band
```

## License

[Creative Commons Zero](http://creativecommons.org/publicdomain/zero/1.0/)

## Contact

[Hans van Gasteren](https://twitter.com/hvangasteren)
