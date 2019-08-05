---
layout: page
title: Guest House
permalink: /projects/sqlzoo/guesthouse
---

# [Guest House](https://sqlzoo.net/wiki/Guest_House)

### Guest House Easy Questions

1. Guest 1183. Give the booking_date and the number of nights for guest 1183.

```sql
SELECT booking_date, nights
FROM booking
WHERE guest_id = 1183;
```

2\. When do they get here? List the arrival time and the first and last names for all guests due to arrive on 2016-11-05, order the output by time of arrival.

```sql
SELECT booking.arrival_time, guest.first_name, guest.last_name
FROM booking JOIN guest ON booking.guest_id = guest.id
WHERE booking.booking_date = '2016-11-05'
ORDER BY booking.arrival_time ASC;
```

3\. Look up daily rates. Give the daily rate that should be paid for bookings with ids 5152, 5165, 5154 and 5295. Include booking id, room type, number of occupants and the amount.

```sql
SELECT booking.booking_id, booking.room_type_requested, booking.occupants, rate.amount
FROM booking JOIN rate ON booking.room_type_requested = rate.room_type AND booking.occupants = rate.occupancy
WHERE booking.booking_id IN (5152, 5165, 5154, 5295);
```

4\. Who’s in 101? Find who is staying in room 101 on 2016-12-03, include first name, last name and address. 

```sql
SELECT guest.first_name, guest.last_name, guest.address
FROM guest JOIN booking ON guest.id = booking.guest_id
WHERE booking.room_no = 101 AND booking.booking_date = '2016-12-03';
```

5\. How many bookings, how many nights? For guests 1185 and 1270 show the number of bookings made and the total number of nights. Your output should include the guest id and the total number of bookings and the total number of nights. 

```sql
SELECT booking.guest_id, COUNT(nights), SUM(nights)
FROM booking
WHERE booking.guest_id IN (1185, 1270)
GROUP BY guest_id;
```

### Guest House Medium Questions

6\. Ruth Cadbury. Show the total amount payable by guest Ruth Cadbury for her room bookings. You should JOIN to the rate table using room_type_requested and occupants. 

```sql
SELECT SUM(nights*amount)
FROM booking JOIN rate ON booking.room_type_requested = rate.room_type AND booking.occupants = rate.occupancy
  JOIN guest ON booking.guest_id = guest.id
WHERE guest.first_name = 'Ruth' AND guest.last_name = 'Cadbury';
```

7\. Including Extras. Calculate the total bill for booking 5346 including extras.

'Extra' table is empty for me when I run it in SQLZOO.

```sql
SELECT (rate.amount + SUM(extra.amount)) AS amount
FROM booking JOIN rate ON booking.room_type_requested = rate.room_type AND booking.occupants = rate.occupancy
  JOIN extra ON booking.booking_id = extra.booking_id
WHERE booking.booking_id = 5346;
```

8\. Edinburgh Residents. For every guest who has the word “Edinburgh” in their address show the total number of nights booked. Be sure to include 0 for those guests who have never had a booking. Show last name, first name, address and number of nights. Order by last name then first name. 

```sql
SELECT guest.last_name, guest.first_name, guest.address, 
  @nights := CASE WHEN SUM(booking.nights) IS NOT NULL
                  THEN @nights := SUM(booking.nights)
                  ELSE @nights := 0
             END AS nights
FROM guest LEFT JOIN booking ON guest.id = booking.guest_id
WHERE guest.address LIKE '%Edinburgh%'
GROUP BY guest.last_name, guest.first_name, guest.address;
```

9\. How busy are we? For each day of the week beginning 2016-11-25 show the number of bookings starting that day. Be sure to show all the days of the week in the correct order. 

DATE_ADD adds 6 days because BETWEEN is inclusive, so we already have the first day and we need the end of the week which is first day + 6 days.

```sql
SELECT booking.booking_date, COUNT(*) AS arrivals
FROM booking
WHERE booking.booking_date BETWEEN '2016-11-25' AND DATE_ADD('2016-11-25', INTERVAL 6 DAY)
GROUP BY booking.booking_date;
```

10\. How many guests? Show the number of guests in the hotel on the night of 2016-11-21. Include all occupants who checked in that day but not those who checked out. 

We make a time interval when the guest is staying at the hotel using their arrival date and nights booked. Then we check which intervals contain the date '2016-11-21'.

```sql
SELECT SUM(occupants)
FROM booking
WHERE booking.booking_date <= '2016-11-21' AND DATE_ADD(booking.booking_date, INTERVAL nights DAY) > '2016-11-21'
```

### Guest House Hard Questions

11\. Coincidence. Have two guests with the same surname ever stayed in the hotel on the evening? Show the last name and both first names. Do not include duplicates. 

Minus one in DATE_ADD because booking_date counts as one night.

```sql
SELECT b.last_name, b.first_name, a.first_name
FROM
(SELECT *
FROM guest JOIN booking ON guest.id = booking.guest_id) AS a
JOIN
(SELECT *
FROM guest JOIN booking ON guest.id = booking.guest_id) AS b
ON a.last_name = b.last_name AND a.first_name <> b.first_name
WHERE (a.booking_date BETWEEN b.booking_date AND DATE_ADD(b.booking_date, INTERVAL b.nights-1 DAY)) OR (b.booking_date BETWEEN a.booking_date AND DATE_ADD(a.booking_date, INTERVAL a.nights-1 DAY))
GROUP BY a.last_name
```

12\. Check out per floor. The first digit of the room number indicates the floor – e.g. room 201 is on the 2nd floor. For each day of the week beginning 2016-11-14 show how many rooms are being vacated that day by floor number. Show all days in the correct order.

I made it really weird and had to switch rows and columns to get the desired table: https://stackoverflow.com/questions/1241178/mysql-rows-to-columns.

```sql
SELECT a.i, SUM(1st) AS 1st, SUM(2nd) AS 2nd, SUM(3rd) AS 3rd
FROM
(SELECT DATE_ADD(booking.booking_date, INTERVAL nights DAY) AS i,
  @floor := CASE WHEN booking.room_no LIKE '1%' THEN @floor := '1st'
                 WHEN booking.room_no LIKE '2%' THEN @floor := '2nd'
                 WHEN booking.room_no LIKE '3%' THEN @floor := '3rd'
            ELSE @floor := '?????'
            END AS floor,
  COUNT(*),
  CASE WHEN @floor = '1st' THEN COUNT(*) END AS 1st, 
  CASE WHEN @floor = '2nd' THEN COUNT(*) END AS 2nd,
  CASE WHEN @floor = '3rd' THEN COUNT(*) END AS 3rd
FROM booking
WHERE DATE_ADD(booking.booking_date, INTERVAL nights DAY) BETWEEN '2016-11-14' AND DATE_ADD('2016-11-14', INTERVAL 6 DAY)
GROUP BY DATE_ADD(booking.booking_date, INTERVAL nights DAY), floor) AS a
GROUP BY a.i
```

13\. Free rooms? List the rooms that are free on the day 25th Nov 2016. 

```sql
SELECT DISTINCT(room_no)
FROM booking
WHERE room_no NOT IN
(SELECT room_no
FROM booking
WHERE '2016-11-25' BETWEEN booking_date AND DATE_ADD(booking_date, INTERVAL nights-1 DAY))
```

14\. Single room for three nights required. A customer wants a single room for three consecutive nights. Find the first available date in December 2016.

Super messy because I spent a few hours on this and got tired at the end. My result is not exactly the expected output because it calculates for each single room, but you can add LIMIT 1 at the end.

```sql
SELECT e.room_no, e.earliest_opening
FROM
(SELECT d.room_no, MAX(DATE_ADD(d.next_booking, INTERVAL d.nights DAY)) AS earliest_opening
FROM
(SELECT c.*, 
  TIMESTAMPDIFF(DAY, c.booking_date, c.next_booking) - c.nights
FROM
(SELECT a.room_no, a.nights, a.booking_date, MIN(b.booking_date) AS next_booking
FROM booking AS a JOIN booking AS b ON a.booking_date < b.booking_date AND a.room_no = b.room_no
WHERE a.room_no IN (101, 201, 301) AND a.booking_date LIKE '2016-12%' AND b.booking_date LIKE '2016-12%'
GROUP BY a.room_no, a.nights, a.booking_date
ORDER BY a.room_no, a.booking_date) AS c) AS d
GROUP BY d.room_no) AS e
ORDER BY e.earliest_opening;
```

15\. Gross income by week. Money is collected from guests when they leave. For each Thursday in November and December 2016, show the total amount of money collected from the previous Friday to that day, inclusive. 

I did this in MySQL so it will not work in sqlzoo. If anybody has an easy way to get a range of dates in November and December 2016 please let me know.

```sql
SELECT weeks.Thursday, SUM(checkout_bill.bill)
FROM
(SELECT DATE_ADD(booking_date, INTERVAL -6 DAY) AS previous_Friday, booking_date AS Thursday 
FROM booking
WHERE DAYOFWEEK(booking_date) = 5
GROUP BY booking_date) AS weeks
JOIN
(SELECT booking.*, DATE_ADD(booking_date, INTERVAL nights DAY) AS checkout_date, ((rate.amount * booking.nights) + COALESCE(SUM(extra.amount), 0)) AS bill
FROM booking JOIN rate ON booking.room_type_requested = rate.room_type AND booking.occupants = rate.occupancy
  LEFT JOIN extra ON booking.booking_id = extra.booking_id
GROUP BY booking.booking_id) AS checkout_bill
ON checkout_bill.checkout_date BETWEEN weeks.previous_Friday AND weeks.Thursday
GROUP BY weeks.Thursday;
```
