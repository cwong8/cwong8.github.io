---
layout: page
title: Warframe Alerts visualization
permalink: /projects/warframe_alerts/webscrape/
---

# Warframe alerts web scrape


```python
# Server connection to MySQL:
import MySQLdb
conn = MySQLdb.connect(host= "localhost",
                  user="yourusername",
                  passwd="yourpassword",
                  db="warframe")

x = conn.cursor()

# Create table for alerts
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS ALERTS (
    TIME DATETIME,
    DESTINATION CHAR(50),
    DESTINATION_PLANET CHAR(50),
    CREDITS INT DEFAULT NULL,
    LOOT_TYPE CHAR(50) DEFAULT NULL,
    LOOT CHAR(50))
    """)
    conn.commit()
except:
    conn.rollback()
```


```python
# Web scraping
from bs4 import BeautifulSoup
import requests
import re

response = requests.get("https://raidtime.net/en/game/tool/warframe/alarm")
url = response.url
soup = BeautifulSoup(response.content, "lxml")

# Get max pages
max_pages = int(re.search("\d+", soup.findAll(class_ = "btn btn-default")[-1]["title"]).group(0))
```


```python
def scrape_page(soup):
    for i in range(len(soup.tbody.findAll('tr'))):
        # Alert time
        alert_time = soup.tbody.findAll('tr')[i].find(class_ = "warframe-countdown")['data-source']
        alert_time = str(datetime.strptime(alert_time, "%Y-%m-%dT%H:%M:%S+%f"))

        # Alert destination
        destination = "".join(soup.tbody.findAll('tr')[i].find(class_ = "alternative").stripped_strings)
        # Split by space followed by ( or ending with )
        alert_destination = re.split(' \(|\)', destination)[0]
        alert_destination_planet = re.split(' \(|\)', destination)[1]

        # Check if there is a credit reward
        if ("".join(soup.tbody.findAll('tr')[i].findAll('td')[1].stripped_strings) == "?"):
            alert_credits = None
        else:
            alert_credits = int("".join(soup.tbody.findAll('tr')[i].findAll('td')[1].stripped_strings).replace(",", ""))

        # Check if there is loot
        if (soup.tbody.findAll('tr')[i].findAll('td')[2].find(class_ = "tag")):
            alert_loot_type = soup.tbody.findAll('tr')[i].findAll('td')[2].find(class_ = "tag").get_text()
        else:
            alert_loot_type = None
        if (soup.tbody.findAll('tr')[i].findAll('td')[2].find(class_ = "alternative")):
            alert_loot = "".join(soup.tbody.findAll('tr')[i].findAll('td')[2].find(class_ = "alternative").stripped_strings)
        else:
            alert_loot = None

        # Insert into MySQL database 'warframe', table 'alerts'
        try:
            x.execute(
            """
            INSERT INTO alerts (time, destination, destination_planet, credits, loot_type, loot)
            VALUES (%s, %s, %s, %s, %s, %s)
            """, [alert_time, alert_destination, alert_destination_planet, alert_credits, alert_loot_type, alert_loot])
            conn.commit()
        except:
            conn.rollback()
    
```


```python
# Deletes all rows, FOR TESTING ONLY
x.execute("DELETE FROM alerts")
conn.commit()
```


```python
# Loop
for j in range(max_pages):
    response = requests.get(url + "/" + str(j+1))
    soup = BeautifulSoup(response.content, "lxml")
    scrape_page(soup)
```
