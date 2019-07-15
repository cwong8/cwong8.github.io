---
layout: page
title: Warframe.market API scrape
permalink: /projects/warframe_market/JSON_SQL/
---

# Schema design
We need to design a schema for our database.

We are using the [Warframe.market API](https://docs.google.com/document/d/1121cjBNN4BeZdMBGil6Qbuqse-sWpEXPpitQH5fb_Fo/edit).


```python
# MySQL create database - make sure your MySQL bin (...\MySQL\MySQL Server 8.0\bin) is in your %PATH% global environment
# Then run this in your command prompt: 
#    mysql -u user -p
#    CREATE DATABASE warframe_market;

# Server connection to MySQL:
import MySQLdb
conn = MySQLdb.connect(host= "localhost",
                  user="yourusername",
                  passwd="yourpassword",
                  db="warframe_market")

x = conn.cursor()
```


```python
import requests
import numpy as np
import pandas as pd

# Start session
s = requests.Session()
s.headers.update({"Platform":"pc", "Language":"en"})

# Get all items JSON data from API
items = s.get("https://api.warframe.market/v1/items").json()
### Partial view
items["payload"]["items"]["en"][0:6]
```




    [{'thumb': 'sub_icons/handle_128x128.png',
      'item_name': 'Scindo Prime Handle',
      'id': '54a73e65e779893a797fff5d',
      'url_name': 'scindo_prime_handle'},
     {'thumb': 'sub_icons/neuroptics_128x128.png',
      'item_name': 'Ember Prime Neuroptics',
      'id': '54a73e65e779893a797fff6d',
      'url_name': 'ember_prime_neuroptics'},
     {'thumb': 'sub_icons/blueprint_128x128.png',
      'item_name': 'Loki Prime Blueprint',
      'id': '54a73e65e779893a797fff7c',
      'url_name': 'loki_prime_blueprint'},
     {'thumb': 'sub_icons/chassis_128x128.png',
      'item_name': 'Ember Prime Chassis',
      'id': '54a73e65e779893a797fff7e',
      'url_name': 'ember_prime_chassis'},
     {'thumb': 'icons/en/thumbs/Hammer_Shot.fb11fe7457d821871f4c92b9acea585e.128x128.png',
      'item_name': 'Hammer Shot',
      'id': '54a74454e779892d5e51558d',
      'url_name': 'hammer_shot'},
     {'thumb': 'icons/en/thumbs/Seeking_Force.9efcc6f0d43c53ca9a57a65d753de6f3.128x128.png',
      'item_name': 'Seeking Force',
      'id': '54a74454e779892d5e51558f',
      'url_name': 'seeking_force'}]



#### Create items table


```python
# Create 'items' table
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS items (
    item_id VARCHAR(255) NOT NULL PRIMARY KEY,
    item_name VARCHAR(255),
    url_name VARCHAR(255))
    """)
    conn.commit()
except:
    conn.rollback()

# Populate table 'items' with item JSON data
for item in items["payload"]["items"]["en"]:
    try:
        x.execute(
        """
        INSERT INTO items (item_id, item_name, url_name)
        VALUES (%s, %s, %s)
        """, [item["id"], item["item_name"], item["url_name"]])
        conn.commit()
    except:
        conn.rollback()
```

    C:\Users\Christopher Wong\Anaconda3\lib\site-packages\ipykernel_launcher.py:9: Warning: (1050, "Table 'items' already exists")
      if __name__ == '__main__':
    

#### Now we want to get the item information for each item.
After a quick manual inspection, we find that different items have different dictionary headers. Thus, we need to find all possible dictionary headers for every item.


```python
# Using time.sleep to not flood API requests. Recommend 3 requests per second
import time

# Using set because we want only unique elements
all_keys = set([])

############################################################################
# HEY YO, THIS TAKES LIKE 25 MINUTES TO RUN SO DON'T USE IT MORE THAN ONCE #
############################################################################

# Get all dictionary keys from JSON for all items
for item in items["payload"]["items"]["en"]:
    # Url switch-a-roo that goes through all item urls
    url_name = item["url_name"]
    req = s.get("https://api.warframe.market/v1/items/" + url_name).json()
    # Grabs all the dictionary keys from each item page
    for item_in_set in req["payload"]["item"]["items_in_set"]:
        for key in list(item_in_set.keys()):
            all_keys.add(key)
    
    # I use 2 requests per second max
    time.sleep(0.5)
    
all_keys
```




    {'de',
     'ducats',
     'en',
     'fr',
     'icon',
     'icon_format',
     'id',
     'ko',
     'mastery_level',
     'mod_max_rank',
     'rarity',
     'ru',
     'set_root',
     'sub_icon',
     'sv',
     'tags',
     'thumb',
     'trading_tax',
     'url_name',
     'zh'}



We see that "tags" are a many-to-many relationship with items, as well as "drop" in the "en" header. Thus, we also need to find all tags and drops in our data. We do this so we can use a 3-table design with a bridge table to nicely store our many-to-many relationships.


```python
# Using set because we want only unique elements
all_tags = set([])

############################################################################
# HEY YO, THIS TAKES LIKE 25 MINUTES TO RUN SO DON'T USE IT MORE THAN ONCE #
############################################################################

# Get all tags
for item in items["payload"]["items"]["en"]:
    # Url switch-a-roo that goes through all item urls
    url_name = item["url_name"]
    req = s.get("https://api.warframe.market/v1/items/" + url_name).json()
    # Grabs all tags from each item page
    for item_in_set in req["payload"]["item"]["items_in_set"]:
        for tag in item_in_set["tags"]:
            all_tags.add(tag)
    
    # I use 2 requests per second max
    time.sleep(0.5)
    
### Partial list
list(all_tags)[0:10]
```




    ['shot gun',
     'archwing gun',
     'k-drive',
     'titania',
     'gunblade',
     'dagger',
     'link',
     'exilus',
     'chassis',
     'unique']




```python
# Using set because we want only unique elements
all_drops = set([])

############################################################################
# HEY YO, THIS TAKES LIKE 25 MINUTES TO RUN SO DON'T USE IT MORE THAN ONCE #
############################################################################

# Get all drops
for item in items["payload"]["items"]["en"]:
    # Url switch-a-roo that goes through all item urls
    url_name = item["url_name"]
    req = s.get("https://api.warframe.market/v1/items/" + url_name).json()
    # Grabs all drops from each item page
    for item_in_set in req["payload"]["item"]["items_in_set"]:
        # Check if drop header exists
        if "drop" in item_in_set["en"]:
            for drop in item_in_set["en"]["drop"]:
                all_drops.add(drop["name"])
                
    # I use 2 requests per second max
    time.sleep(0.5)
    
### Partial list
list(all_drops)[0:10]
```




    ['Captain Vor',
     'Neo S2 Common',
     'The Grustrag Three',
     'Neo N3 Uncommon',
     'CorpusSniper Crewman',
     'Feyarch Specter',
     'Corrupted Butcher',
     'Meso V1 Rare',
     'Axi L3 Uncommon',
     'Dark Sector Defense']



#### Create tags and drops table (and the bridge tables)


```python
# Create 'tags' table
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS tags (
    tag_id INT AUTO_INCREMENT PRIMARY KEY,
    tag_name VARCHAR(255))
    """)
    conn.commit()
except:
    conn.rollback()
    
# Populate table 'tags' with all tags
try:
    x.executemany(
    """
    INSERT INTO tags (tag_name)
    VALUES (%s)
    """, list(all_tags))
    conn.commit()
except:
    conn.rollback()
    
# Create 'items_tags' bridge table
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS items_tags (
    item_id VARCHAR(255),
    FOREIGN KEY (item_id) REFERENCES items(item_id),
    tag_id INT,
    FOREIGN KEY (tag_id) REFERENCES tags(tag_id))
    """)
    conn.commit()
except:
    conn.rollback()
```


```python
# Create 'drops' table
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS drops (
    drop_id INT AUTO_INCREMENT PRIMARY KEY,
    drop_name VARCHAR(255))
    """)
    conn.commit()
except:
    conn.rollback()

# Populate table 'drops' with all drops
# Unicode "\u2019" is annoying
try:
    for drop in list(all_drops):
        x.execute(
        """
        INSERT INTO drops (drop_name)
        VALUES (%s)
        """, [drop.replace("\u2019", "'")])
        conn.commit()
except:
    conn.rollback()
    
# Create 'items_drops' bridge table
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS items_drops (
    item_id VARCHAR(255),
    FOREIGN KEY (item_id) REFERENCES items(item_id),
    drop_id INT,
    FOREIGN KEY (drop_id) REFERENCES drops(drop_id))
    """)
    conn.commit()
except:
    conn.rollback()
```

#### Item sets
We see that items are grouped by their item sets. Thus, we want to have a separate table containing all item sets in the game.


```python
# Using set because we want only unique elements
all_sets = set([])

############################################################################
# HEY YO, THIS TAKES LIKE 25 MINUTES TO RUN SO DON'T USE IT MORE THAN ONCE #
############################################################################

# Get all dictionary keys from JSON for all items
for item in items["payload"]["items"]["en"]:
    # Url switch-a-roo that goes through all item urls
    url_name = item["url_name"]
    req = s.get("https://api.warframe.market/v1/items/" + url_name).json()
    # Grabs sets from each item page if it is part of a set (i.e. more than one item is returned)
    if (len(req["payload"]["item"]["items_in_set"]) > 1):
        for item_in_set in req["payload"]["item"]["items_in_set"]:
            if item_in_set["set_root"]:
                all_sets.add(item_in_set["en"]["item_name"])
    
    # I use 2 requests per second max
    time.sleep(0.5)

### Partial list
list(all_sets)[0:10]
```




    ['Lex Prime Set',
     'Scindo Prime Set',
     'Ballistica Prime Set',
     'Tiberon Prime Set',
     'Wolf Sledge Set',
     'Akbolto Prime Set',
     'Rathbone Set',
     'Akstiletto Prime Set',
     'Ash Prime Set',
     'Venka Prime Set']




```python
# Create 'sets' table
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS sets (
    set_id INT AUTO_INCREMENT PRIMARY KEY,
    set_name VARCHAR(255))
    """)
    conn.commit()
except:
    conn.rollback()
    
# Populate table 'sets' with all sets
try:
    x.executemany(
    """
    INSERT INTO sets (set_name)
    VALUES (%s)
    """, list(all_sets))
    conn.commit()
except:
    conn.rollback()
```

#### Item info
Just a big scraper.


```python
# Using time.sleep to not flood API requests. Recommend 3 requests per second
import time


# Smart scraper: if an item is a sibling to the requested item then it is already scraped and we skip it
scraped_items = set([])

# HTML tag cleaner
import re
def cleanhtml(raw_html):
  cleanr = re.compile('<.*?>')
  cleantext = re.sub(cleanr, '', raw_html)
  return cleantext

# Create 'item_info' table
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS item_info (
    item_id VARCHAR(255) PRIMARY KEY,
    FOREIGN KEY (item_id) REFERENCES items(item_id),
    item_name VARCHAR(255),
    item_description VARCHAR(1000) DEFAULT NULL,
    item_wiki VARCHAR(255) DEFAULT NULL,
    rarity VARCHAR(255) DEFAULT NULL,
    mastery_level INT DEFAULT NULL,
    trading_tax INT DEFAULT NULL,
    mod_max_rank INT DEFAULT NULL,
    ducats INT DEFAULT NULL,
    set_id INT DEFAULT NULL,
    FOREIGN KEY (set_id) REFERENCES sets(set_id))
    """)
    conn.commit()
except:
    conn.rollback()

for item in items["payload"]["items"]["en"]:
    # Navigate to item information page
    url_name = item["url_name"]
    req = s.get("https://api.warframe.market/v1/items/" + url_name).json()
    
    # If there is more than one item returned, then it is part of a set and we want to find which one
    set_id = None
    if len(req["payload"]["item"]["items_in_set"]) > 1:
        for item_in_set in req["payload"]["item"]["items_in_set"]:
            if (item_in_set["set_root"]):
                item_set = item_in_set["en"]["item_name"]
                set_id = int(pd.read_sql_query("""
                SELECT set_id
                FROM sets
                WHERE set_name LIKE %s
                """, conn, params = [item_set]).values)
    
    # Get all item information
    for item_in_set in req["payload"]["item"]["items_in_set"]:
        # Item ID
        item_id = item_in_set["id"]
        
        if item_id not in scraped_items:
            # Item name
            item_name = item_in_set["en"]["item_name"]

            # Description
            item_description = cleanhtml(item_in_set["en"]["description"])

            # Wiki link
            if "wiki_link" in item_in_set["en"]:
                item_wiki = item_in_set["en"]["wiki_link"]
            else:
                item_wiki = None

            # Drops (if applicable)
            # This populates our bridge table 'items_drops'
            if "drop" in item_in_set["en"]:
                for drop in item_in_set["en"]["drop"]:
                    # Get drop_id from drop_name
                    drop_id = pd.read_sql_query("""
                    SELECT drop_id
                    FROM drops
                    WHERE drop_name LIKE %s
                    """, conn, params = [drop["name"].replace("\u2019", "'")])

                    # Add to 'items_drops'
                    try:
                        x.execute(
                        """
                        INSERT INTO items_drops (item_id, drop_id)
                        VALUES (%s, %s)
                        """, [item_id, int(drop_id.values)])
                        conn.commit()
                    except:
                        conn.rollback()

            # Tags
            # This populates our bridge table 'items_tags'
            for tag in item_in_set["tags"]:
                # Get tag_id from tag_name
                tag_id = pd.read_sql_query("""
                SELECT tag_id
                FROM tags
                WHERE tag_name LIKE %s
                """, conn, params = [tag])

                # Add to 'items_tags'
                try:
                    x.execute(
                    """
                    INSERT INTO items_tags (item_id, tag_id)
                    VALUES (%s, %s)
                    """, [item_id, int(tag_id.values)])
                    conn.commit()
                except:
                    conn.rollback()

            # Ducats (if applicable)
            if "ducats" in item_in_set:
                ducats = int(item_in_set["ducats"])
            else:
                ducats = None

            # Mastery level (if applicable)
            if "mastery_level" in item_in_set:
                mastery_level = int(item_in_set["mastery_level"])
            else:
                mastery_level = None

            # Trading tax (if applicable)
            if "trading_tax" in item_in_set:
                trading_tax = item_in_set["trading_tax"]
            else:
                trading_tax = None

            # Max mod rank (if applicable)
            if "mod_max_rank" in item_in_set:
                mod_max_rank = item_in_set["mod_max_rank"]
            else:
                mod_max_rank = None

            # Rarity (if applicable)
            if "rarity" in item_in_set:
                rarity = item_in_set["rarity"]
            else:
                rarity = None

            # Insert into item_info table
            try:
                x.execute(
                """
                INSERT INTO item_info (item_id, item_name, item_description, item_wiki, rarity, mastery_level, trading_tax, mod_max_rank, ducats, set_id)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                """, [item_id, item_name, item_description, item_wiki, rarity, mastery_level, trading_tax, mod_max_rank, ducats, set_id])
                conn.commit()
            except:
                conn.rollback()

            # Smart scraper function
            scraped_items.add(item_id)
  
    # 2 requests per second
    time.sleep(0.5)
```


```python
# Save workspace
del conn
del x

import dill
#dill.dump_session('market.db') # Save
#dill.load_session('market.db') # Load

### https://github.com/WFCD - Warframe community developers
```
