---
layout: page
title: Congestion Charging
permalink: /projects/sqlzoo/congestion
---

# https://sqlzoo.net/wiki/Congestion_Charging

### Congestion Charging Easy Questions

1. Show the name and address of the keeper of vehicle SO 02 PSP.

```sql
SELECT keeper.name, keeper.address
FROM keeper JOIN vehicle ON keeper.id = vehicle.keeper
WHERE vehicle.id = 'SO 02 PSP';
```

2\. Show the number of cameras that take images for incoming vehicles. 

```sql
SELECT COUNT(*)
FROM camera
WHERE perim = 'IN';
```

3\. List the image details taken by Camera 10 before 26 Feb 2007. 

```sql
SELECT *
FROM image
WHERE camera = 10 AND DATE_FORMAT(whn, '%Y-%m-%d') < '2007-02-26';
```

4\. List the number of images taken by each camera. Your answer should show how many images have been taken by camera 1, camera 2 etc. The list must NOT include the images taken by camera 15, 16, 17, 18 and 19. 

```sql
SELECT camera, COUNT(*) AS images_taken
FROM image
WHERE camera < 15
GROUP BY camera;
```

5\. A number of vehicles have permits that start on 30th Jan 2007. List the name and address for each keeper in alphabetical order without duplication. 

```sql
SELECT DISTINCT(keeper.name), keeper.address
FROM keeper JOIN vehicle ON keeper.id = vehicle.keeper
  JOIN permit ON vehicle.id = permit.reg
WHERE DATE_FORMAT(permit.sDate, '%Y-%m-%d') = '2007-01-30'
ORDER BY keeper.name, keeper.address;
```

### Congestion Charging Medium Questions

6\. List the owners (name and address) of Vehicles caught by camera 1 or 18 without duplication. 

```sql
SELECT DISTINCT(keeper.name), keeper.address
FROM keeper JOIN vehicle ON keeper.id = vehicle.keeper
  JOIN image ON vehicle.id = image.reg
WHERE image.camera IN (1, 18);
```

7\. Show keepers (name and address) who have more than 5 vehicles. 

```sql
SELECT keeper.name, keeper.address, COUNT(*) AS num_vehicles
FROM keeper JOIN vehicle ON keeper.id = vehicle.keeper
GROUP BY keeper.name, keeper.address
HAVING COUNT(*) > 5;
```

8\. For each vehicle show the number of current permits (suppose today is the 1st of Feb 2007). The list should include the vehicle.s registration and the number of permits. Current permits can be determined based on charge types, e.g. for weekly permit you can use the date after 24 Jan 2007 and before 02 Feb 2007. 

```sql
SELECT a.reg, COUNT(*) AS current_permits
FROM
(SELECT *,
  CASE WHEN permit.chargeType = 'Daily' THEN DATE_ADD(permit.sDate, INTERVAL 1 DAY)
       WHEN permit.chargeType = 'Weekly' THEN DATE_ADD(permit.sDate, INTERVAL 1 WEEK)
       WHEN permit.chargeType = 'Monthly' THEN DATE_ADD(permit.sDate, INTERVAL 1 MONTH)
       WHEN permit.chargeType = 'Annual' THEN DATE_ADD(permit.sDate, INTERVAL 1 YEAR)
  END AS permit_expiration
FROM permit) AS a
WHERE '2007-02-01' BETWEEN a.sDate AND a.permit_expiration
-- Why group by? Nobody would buy overlapping permits for their cars...
GROUP BY a.reg;
```

9\. Obtain a list of every vehicle passing camera 10 on 25th Feb 2007. Show the time, the registration and the name of the keeper if available. 

```sql
SELECT image.whn, image.reg, keeper.name
FROM image JOIN vehicle ON image.reg = vehicle.id
  JOIN keeper ON vehicle.keeper = keeper.id
WHERE image.camera = 10 AND DATE_FORMAT(image.whn, '%Y-%m-%d') = '2007-02-25';
```

10\. List the keepers who have more than 4 vehicles and one of them must have more than 2 permits. The list should include the names and the number of vehicles. 

```sql
SELECT a.id, a.name, a.num_vehicles, b.id, b.num_permits
FROM
(SELECT keeper.id, keeper.name, COUNT(*) AS num_vehicles
FROM keeper JOIN vehicle ON keeper.id = vehicle.keeper
GROUP BY keeper.id, keeper.name
HAVING COUNT(*) > 4) AS a
JOIN
(SELECT vehicle.keeper, vehicle.id, COUNT(*) AS num_permits
FROM vehicle JOIN permit ON vehicle.id = permit.reg
  JOIN keeper ON vehicle.keeper = keeper.id
GROUP BY vehicle.id, vehicle.keeper
HAVING COUNT(*) > 2) AS b
ON a.id = b.keeper;
```

### Congestion Charging Hard Questions

11\. When creating a view in scott you must specify the schema name of the sources and the destination. 

Not really a question so we skip.

12\. There are four types of permit. The most popular type means that this type has been issued the highest number of times. Find out the most popular type, together with the total number of permits issued. 

```sql
SELECT chargeType, COUNT(*)
FROM permit
GROUP BY chargeType
ORDER BY COUNT(*) DESC;
```

13\. For each of the vehicles caught by camera 19 - show the registration, the earliest time at camera 19 and the time and camera at which it left the zone. 

```sql
SELECT b.*, image.camera AS left_camera
FROM image,
(SELECT image.reg, a.camera AS first_camera, a.first_capture, MIN(whn) AS left_zone
FROM image JOIN
(SELECT camera, reg, MIN(whn) AS first_capture
FROM image
WHERE camera = 19
GROUP BY reg, camera) AS a
ON image.reg = a.reg
WHERE whn > a.first_capture
GROUP BY image.reg, a.first_capture, a.camera) AS b
WHERE image.reg = b.reg AND image.whn = b.left_zone;
```

14\. For all 19 cameras - show the position as IN, OUT or INTERNAL and the busiest hour for that camera. 

```sql
SELECT a.id, a.perim_fixed, MAX(b.images_taken)
FROM
(SELECT id,
  CASE WHEN perim IS NULL THEN 'INTERNAL'
       ELSE perim
  END AS perim_fixed
FROM camera) AS a
LEFT JOIN
(SELECT camera.id,
  HOUR(image.whn),
  COUNT(*) AS images_taken
FROM camera JOIN image ON camera.id = image.camera
GROUP BY camera.id, HOUR(image.whn)) AS b
ON a.id = b.id
GROUP BY a.id;
```

15\. Anomalous daily permits. Daily permits should not be issued for non-charging days. Find a way to represent charging days. Identify the anomalous daily permits. 

Congestion charge information: https://tfl.gov.uk/modes/driving/congestion-charge/congestion-charge-zone

```sql
SELECT *
FROM permit
WHERE chargeType = 'Daily' AND DAYOFWEEK(sDate) IN (1, 7);
```

16\. Issuing fines: Vehicles using the zone during the charge period, on charging days must be issued with fine notices unless they have a permit covering that day. List the name and address of such culprits, give the camera and the date and time of the first offence. 

This returns nothing because all images except one were taken on Sunday, but the logic is there.

```sql
SELECT keeper.name, keeper.address, image.camera, image.whn
FROM image JOIN vehicle ON image.reg = vehicle.id
  JOIN keeper ON vehicle.keeper = keeper.id,
(SELECT permit.reg, permit.sDate,
  CASE WHEN permit.chargeType = 'Daily' THEN DATE_ADD(permit.sDate, INTERVAL 1 DAY)
       WHEN permit.chargeType = 'Weekly' THEN DATE_ADD(permit.sDate, INTERVAL 1 WEEK)
       WHEN permit.chargeType = 'Monthly' THEN DATE_ADD(permit.sDate, INTERVAL 1 MONTH)
       WHEN permit.chargeType = 'Annual' THEN DATE_ADD(permit.sDate, INTERVAL 1 YEAR)
  END AS permit_expiration
FROM permit) AS valid_permits
-- Not 100% confident in the last part. Gut feeling says I should group by something.
WHERE (DAYOFWEEK(image.whn) BETWEEN 2 AND 6) AND (HOUR(image.whn) BETWEEN 7 AND 18) AND (image.whn NOT BETWEEN valid_permits.sDate AND valid_permits.permit_expiration);
```
