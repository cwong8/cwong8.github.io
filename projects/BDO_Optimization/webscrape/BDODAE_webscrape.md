---
layout: page
title: Black Desert Online Worker Optimization
permalink: /projects/BDO_Optimization/webscrape/
---

# Web Scraping
I web scraped the website [bdodae.com](https://www.bdodae.com/) for data to use for my project. Since I am optimizing profits I need item values and information about the nodes.

### Tools of choice
* Python because I want to learn another language for data analysis (having used R throughout college)
* Selenium because [bdodae.com](https://www.bdodae.com/) is written in JavaScript as opposed to HTML. This means that the website is dynamic and has to be loaded before web scraping.
* BeautifulSoup to parse the webpage once it is loaded in Selenium.
* MySQLdb because I want to save the scraped information to a MySQL database that I can pull information from later.

# Scraping item values
[Item page](https://www.bdodae.com/nodes/index.php?page=items)


```python
# Import packages
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import Select
from bs4 import BeautifulSoup
import re
import pandas as pd
import os
import time

# Server Connection to MySQL:
import MySQLdb
conn = MySQLdb.connect(host= "localhost",
                  user="yourusername",
                  passwd="yourpassword",
                  db="bdodae")

x = conn.cursor()

# Create table for prices
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS PRICES (
    ITEM_ID INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    ITEM CHAR(50) NOT NULL,
    USER_AVERAGE INT,
    RECENT_VALUE INT,
    VENDOR_SELL INT,
    VENDOR_BUY INT DEFAULT NULL)
    """)
    conn.commit()
except:
    conn.rollback()


# Scraping item values
url = "https://www.bdodae.com/nodes/index.php?page=items"

# Create a new Firefox session and go to the URL above
driver = webdriver.Firefox()
driver.implicitly_wait(30)
driver.get(url)

# Not mandatory
#driver.maximize_window()
```


```python
# Initial page scrape
soup_values = BeautifulSoup(driver.page_source, 'lxml')

# Getting item names
items_soup = soup_values.find_all('a', class_ = "item_popup")

# Initialize empty list we are going to append item names to
items = []
for item in items_soup:
    item_name = item.get_text()
    items.append(item_name)

# Define a function to clean the text/values
def clean_values(values):
    for i in range(len(values)):
        values[i] = re.search("(\d+)", values[i].get_text()).group(0)
    if (len(values) == 3):
        # Add NULL to vendor_buy if it does not have one
        values.insert(2, None)
    
# Find the value buttons to click on
value_buttons = driver.find_elements_by_class_name('value_button')
wait = WebDriverWait(driver, 10)
```


```python
# Big loop that:
# 1.) Clicks on the values boxes to open up the hidden table
# 2.) Scrapes the page, including the now revealed hidden table
# 3.) Finds the values in the hidden table and cleans it
# 4.) Writes the item name and corresponding values into a MySQL database
# 5.) Waits until the hidden table is actually visible before we click off of it to close it
# 6.) Repeats steps 1-5 for all items/rows on the page
for item, button, i in zip(items, value_buttons, range(len(items))):
    # Click on value box to open hidden table
    button.click()
    # If it clicks too fast, it'll throw an error since the table will cover other values
    time.sleep(1)
    # Now we scrape the hidden table
    soup_values = BeautifulSoup(driver.page_source, 'lxml')
    values = soup_values.find_all('div', class_ = "value_extra value_extra_on")[-1].find_all('div', class_ = "value_option")
    # Clean the values output
    clean_values(values)
    values_list = [item] + values
    # Safety check for when our code stops or does not find the values
    if (len(values_list) != 5): 
        print("ERROR missing values") 
        print(values_list)
        break
    # Input data into database
    try:
        x.execute(
        """
        INSERT INTO prices (item, user_average, recent_value, vendor_buy, vendor_sell)
        VALUES (%s, %s, %s, %s, %s)
        """, values_list)
        conn.commit()
    except:
        conn.rollback()
    # Wait until hidden table pops up
    # If one of the values in the table is a duplicate, then the CSS selector has a 3 instead of 2 at the end
    # Weird, but it's just how it is
    if (len(set(values_list)) == 3 or len(set(values_list)) == 4):
        element = wait.until(EC.visibility_of_element_located((By.CSS_SELECTOR, "tr.search:nth-child(" + str(i+2) + ") > td:nth-child(2) > div:nth-child(1) > div:nth-child(3)")))
    else:
        element = wait.until(EC.visibility_of_element_located((By.CSS_SELECTOR, "tr.search:nth-child(" + str(i+2) + ") > td:nth-child(2) > div:nth-child(1) > div:nth-child(2)")))
    # Click off the values box to close the hidden table
    off_button = driver.find_element_by_css_selector("tr.search:nth-child(" + str(i+2) + ") > td:nth-child(3)")
    off_button.click()
    # As of 10/24/2018 reset_actions() does NOT clear locally in Firefox
    # So there will be lots of extra random clicking if you run this in Firefox
    #action.move_to_element_with_offset(button, -5, 0).click().perform()
    #action.reset_actions()
    time.sleep(1)
```


```python
pd.options.display.max_rows = 6
# Table 'prices' that we obtained from web scraping bdodae.com
prices = pd.read_sql('SELECT * FROM prices', con = conn)
prices
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ITEM_ID</th>
      <th>ITEM</th>
      <th>USER_AVERAGE</th>
      <th>RECENT_VALUE</th>
      <th>VENDOR_SELL</th>
      <th>VENDOR_BUY</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>Acacia Sap</td>
      <td>387</td>
      <td>662</td>
      <td>70</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>Acacia Timber</td>
      <td>874</td>
      <td>907</td>
      <td>105</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>Aloe</td>
      <td>212</td>
      <td>174</td>
      <td>40</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>198</th>
      <td>199</td>
      <td>White Umbrella Mushroom</td>
      <td>273</td>
      <td>531</td>
      <td>46</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>199</th>
      <td>200</td>
      <td>Withered Leaf</td>
      <td>110</td>
      <td>110</td>
      <td>110</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>200</th>
      <td>201</td>
      <td>Zinc Ore</td>
      <td>1016</td>
      <td>1058</td>
      <td>90</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>201 rows × 6 columns</p>
</div>



# Scraping node data
[Node page](https://www.bdodae.com/nodes/)


```python
# Import packages
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.support.ui import Select
from bs4 import BeautifulSoup
from itertools import chain
import re
import pandas as pd
import os
import time

# Launch URL
url = "https://www.bdodae.com/"

# Create a new Firefox session
driver = webdriver.Firefox()
driver.implicitly_wait(30)
driver.get(url)

python_button = driver.find_element_by_id("nodes_nav")
python_button.click() # Click node link
```


```python
# Storing page source in a variable
soup_level1 = BeautifulSoup(driver.page_source, 'lxml')

# Grab all page links to nodes
links = soup_level1.find_all('a', {'href' : re.compile("^index\.php\?node"), 'class' : False})
```


```python
# Server Connection to MySQL:
import MySQLdb
conn = MySQLdb.connect(host= "localhost",
                  user="yourusername",
                  passwd="yourpassword",
                  db="bdodae")

x = conn.cursor()

# Create table for nodes
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS NODE (
    NODE_ID INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    NAME CHAR(50) NOT NULL,
    CP INT DEFAULT 0,
    AREA CHAR(50),
    TYPE CHAR(50),
    REGION_MOD_PERCENT FLOAT DEFAULT NULL,
    CONNECTIONS CHAR(255))
    """)
    conn.commit()
except:
    conn.rollback()

# Create table for subnodes
try:
    x.execute(
    """
    CREATE TABLE IF NOT EXISTS SUBNODE (
    SUBNODE_ID INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    NAME CHAR(50) NOT NULL,
    CP INT DEFAULT 0,
    BASE_WORKLOAD INT DEFAULT NULL,
    CURRENT_WORKLOAD INT DEFAULT NULL,
    ITEM1 CHAR(50) DEFAULT NULL,
    ITEM2 CHAR(50) DEFAULT NULL,
    ITEM3 CHAR(50) DEFAULT NULL,
    ITEM4 CHAR(50) DEFAULT NULL,
    YIELD1 FLOAT DEFAULT NULL,
    YIELD2 FLOAT DEFAULT NULL,
    YIELD3 FLOAT DEFAULT NULL,
    YIELD4 FLOAT DEFAULT NULL,
    DISTANCE1 INT DEFAULT 0,
    DISTANCE2 INT DEFAULT NULL,
    DISTANCE3 INT DEFAULT NULL,
    CITY1 CHAR(50),
    CITY2 CHAR(50) DEFAULT NULL,
    CITY3 CHAR(50) DEFAULT NULL)
    """)
    conn.commit()
except:
    conn.rollback()
```


```python
# Go through every node page
for link in links:
    # Get node link
    link = link.get_text()
    
    # Click on link
    python_button = driver.find_element_by_link_text(link)
    python_button.click()
    
    # soup_level 2 contains the page source for the node clicked on
    soup_level2 = BeautifulSoup(driver.page_source, 'lxml')
    
    # General node info
    node_title = soup_level2.find(class_ = 'n_title_link').get_text()
    node_name = re.split('\s(?=\dCP)', node_title)[0]
    node_cp = re.split('\s(?=\dCP)', node_title)[1][0]
    node_area = soup_level2.find(class_ = 'n_area').get_text()
    node_type = soup_level2.find(class_ = 'n_type').get_text()
    node_connections = re.split('Connected: ', soup_level2.find(class_ = 'n_connected').get_text())[1]
    # Check if region modifier exists
    if (soup_level2.find(class_ = 'n_region')):
        node_region_mod = re.search('\d', soup_level2.find(class_ = 'n_region').get_text()).group(0)
    else:
        node_region_mod = None

    # Combine info into one variable
    node_list = [node_name,
                 node_cp,
                 node_area,
                 node_type,
                 node_region_mod,
                 node_connections]

    # Insert into MySQL database 'node'
    try:
        x.execute(
        """
        INSERT INTO node (name, cp, area, type, region_mod_percent, connections)
        VALUES (%s, %s, %s, %s, %s, %s)
        """, node_list)
        conn.commit()
    except:
        conn.rollback()
        
    
    # Get subnode information if it exists
    # Otherwise this part does nothing
    node_subnode_subtype = soup_level2.find_all(class_ = 's_subtype')
    node_subnode_workload = soup_level2.find_all(class_ = 's_workload')
    node_subnode_tables = soup_level2.find_all('table', class_ = re.compile("^(?!worker_select_)"))
    # Check if there are subnode tables
    if node_subnode_tables:
        node_subnode_tables = pd.read_html(str(node_subnode_tables[1:]), header = 0)

    node_subnode_name = []
    node_subnode_cp = []
    base_workload = []
    current_workload = []
    for text, i in zip(node_subnode_subtype, range(len(node_subnode_subtype))):
        node_subnode_name = [node_name + " - " + re.split('\s(?=\dCP)', text.get_text())[0] + "." + str(i+1)]
        node_subnode_cp = [re.split('\s(?=\dCP)', text.get_text())[1][0]]
        # Workload information should come in pairs: base and current workload
        if (len(node_subnode_workload) % 2) == 0:
            base_workload = [re.search('\d+', node_subnode_workload[(2*i)-1].get_text()).group(0)]
            current_workload = [re.search('\d+', node_subnode_workload[2*i].get_text()).group(0)]
        # Subnode tables should come in pairs: one containing items and yields, other containing distances and cities
        # Also add None until list is the correct length
        if (len(node_subnode_tables) % 2) == 0:
            node_subnode_item = node_subnode_tables[2*i]["Item"]
            node_subnode_item_list = []
            for item in node_subnode_item:
                node_subnode_item_list.append(item)
            while len(node_subnode_item_list) < 4:
                node_subnode_item_list.append(None)
            node_subnode_yield = node_subnode_tables[2*i]["Avg"]
            node_subnode_yield_list = []
            for yields in node_subnode_yield:
                node_subnode_yield_list.append(yields)
            while len(node_subnode_yield_list) < 4:
                node_subnode_yield_list.append(None)
            node_distance = node_subnode_tables[(2*i)+1]["Range"]
            node_distance_list = []
            for distance in node_distance:
                node_distance_list.append(distance)
            while len(node_distance_list) < 3:
                node_distance_list.append(None)
            node_city = node_subnode_tables[(2*i)+1]["City"]
            node_city_list = []
            for city in node_city:
                node_city_list.append(city)
            while len(node_city_list) < 3:
                node_city_list.append(None)

            # Combine info into one variable
            subnode_list = [node_subnode_name,
                            node_subnode_cp,
                            base_workload,
                            current_workload,
                            node_subnode_item_list,
                            node_subnode_yield_list,
                            node_distance_list,
                            node_city_list]
            subnode_list = list(chain.from_iterable(subnode_list))

            # Insert into MySQL database 'subnode'
            try:
                x.execute(
                """
                INSERT INTO subnode (name, cp, base_workload, current_workload,
                                  item1, item2, item3, item4, yield1, yield2, yield3, yield4,
                                  distance1, distance2, distance3, city1, city2, city3)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                """, subnode_list)
                conn.commit()
            except:
                conn.rollback()

                
        
    # Back it up
    driver.execute_script("window.history.go(-1)")
```


```python
# Post-processing cleaning
x.execute(
"""
UPDATE node
    SET connections = 'North Kaia Ferry, Catfishman Camp, Calpheon Castle'
    WHERE node_id = 69;
"""
)
```


```python
pd.options.display.max_rows = 6
# Table 'node' that we obtained from web scraping bdodae.com
node = pd.read_sql('SELECT * FROM node', con = conn)
node
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>NODE_ID</th>
      <th>NAME</th>
      <th>CP</th>
      <th>AREA</th>
      <th>TYPE</th>
      <th>REGION_MOD_PERCENT</th>
      <th>CONNECTIONS</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>Abandoned Iron Mine</td>
      <td>2</td>
      <td>Mediah</td>
      <td>Worker Node</td>
      <td>0.0</td>
      <td>Abandoned Iron Mine Saunil District, Abandoned...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>Abandoned Iron Mine Entrance</td>
      <td>1</td>
      <td>Mediah</td>
      <td>Connection Node</td>
      <td>NaN</td>
      <td>Highland Junction, Alumn Rock Valley</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>Abandoned Iron Mine Rhutum District</td>
      <td>1</td>
      <td>Mediah</td>
      <td>Connection Node</td>
      <td>NaN</td>
      <td>Abandoned Iron Mine, Abun</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>356</th>
      <td>357</td>
      <td>Yalt Canyon</td>
      <td>1</td>
      <td>Valencia</td>
      <td>Connection Node</td>
      <td>NaN</td>
      <td>Shakatu, Gahaz Bandit's Lair</td>
    </tr>
    <tr>
      <th>357</th>
      <td>358</td>
      <td>Yianaros's Field</td>
      <td>1</td>
      <td>Kamasylvia</td>
      <td>Connection Node</td>
      <td>NaN</td>
      <td>Western Valtarra Mountains</td>
    </tr>
    <tr>
      <th>358</th>
      <td>359</td>
      <td>Zagam Island</td>
      <td>1</td>
      <td>Margoria</td>
      <td>Connection Node</td>
      <td>NaN</td>
      <td>Nada Island</td>
    </tr>
  </tbody>
</table>
<p>359 rows × 7 columns</p>
</div>




```python
# Table 'subnode' that we obtained from web scraping bdodae.com
subnode = pd.read_sql('SELECT * FROM subnode', con = conn)
subnode
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>SUBNODE_ID</th>
      <th>NAME</th>
      <th>CP</th>
      <th>BASE_WORKLOAD</th>
      <th>CURRENT_WORKLOAD</th>
      <th>ITEM1</th>
      <th>ITEM2</th>
      <th>ITEM3</th>
      <th>ITEM4</th>
      <th>YIELD1</th>
      <th>YIELD2</th>
      <th>YIELD3</th>
      <th>YIELD4</th>
      <th>DISTANCE1</th>
      <th>DISTANCE2</th>
      <th>DISTANCE3</th>
      <th>CITY1</th>
      <th>CITY2</th>
      <th>CITY3</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>Abandoned Iron Mine - Mining.1</td>
      <td>3</td>
      <td>200</td>
      <td>250</td>
      <td>Zinc Ore</td>
      <td>Powder Of Time</td>
      <td>Platinum Ore</td>
      <td>None</td>
      <td>7.86</td>
      <td>1.86</td>
      <td>0.95</td>
      <td>NaN</td>
      <td>1113</td>
      <td>2063</td>
      <td>NaN</td>
      <td>Altinova</td>
      <td>Tarif</td>
      <td>None</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>Abandoned Iron Mine - Mining.2</td>
      <td>3</td>
      <td>500</td>
      <td>100</td>
      <td>Iron Ore</td>
      <td>Powder Of Darkness</td>
      <td>Rough Black Crystal</td>
      <td>None</td>
      <td>10.53</td>
      <td>1.80</td>
      <td>1.08</td>
      <td>NaN</td>
      <td>1113</td>
      <td>2063</td>
      <td>NaN</td>
      <td>Altinova</td>
      <td>Tarif</td>
      <td>None</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>Ahto Farm - Farming.1</td>
      <td>2</td>
      <td>200</td>
      <td>100</td>
      <td>Cotton</td>
      <td>Cotton Yarn</td>
      <td>None</td>
      <td>None</td>
      <td>12.53</td>
      <td>0.85</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>702</td>
      <td>2000</td>
      <td>2325.0</td>
      <td>Tarif</td>
      <td>Heidel</td>
      <td>Altinova</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>169</th>
      <td>170</td>
      <td>Weenie Cabin - Gathering.1</td>
      <td>2</td>
      <td>140</td>
      <td>140</td>
      <td>Volcanic Umbrella Mushroom</td>
      <td>Green Pendulous Mushroom</td>
      <td>None</td>
      <td>None</td>
      <td>7.70</td>
      <td>6.90</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>1608</td>
      <td>3025</td>
      <td>6005.0</td>
      <td>Grana</td>
      <td>Old Wisdom Tree</td>
      <td>Trent</td>
    </tr>
    <tr>
      <th>170</th>
      <td>171</td>
      <td>Weita Island - Fish Drying Yard.1</td>
      <td>1</td>
      <td>2400</td>
      <td>1200</td>
      <td>Dried Surfperch</td>
      <td>Dried Bluefish</td>
      <td>Dried Maomao</td>
      <td>Dried Nibbler</td>
      <td>3.60</td>
      <td>1.80</td>
      <td>0.90</td>
      <td>0.08</td>
      <td>1620</td>
      <td>3291</td>
      <td>3340.0</td>
      <td>Iliya Island</td>
      <td>Velia</td>
      <td>Olvia</td>
    </tr>
    <tr>
      <th>171</th>
      <td>172</td>
      <td>Wolf Hills - Lumbering.1</td>
      <td>1</td>
      <td>150</td>
      <td>100</td>
      <td>Ash Timber</td>
      <td>None</td>
      <td>None</td>
      <td>None</td>
      <td>4.43</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>340</td>
      <td>1902</td>
      <td>NaN</td>
      <td>Olvia</td>
      <td>Velia</td>
      <td>None</td>
    </tr>
  </tbody>
</table>
<p>172 rows × 19 columns</p>
</div>


