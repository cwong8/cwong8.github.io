## FFXV Comrades Item Optimizer

Something I was curious about while playing the game. It works, but it's not "complete" because I want to add more features such as hosting on a dynamic webpage, calculating optimal solutions for all weapons, adding Cid buff variables, etc.

In general this was a very simple project; the only part that took some time and thought was formulating the problem as a linear programming (LP) problem which I just put through the package PuLP.


```python
# Solving a simple linear programming problem following this tutorial:
# https://pythonhosted.org/PuLP/CaseStudies/a_blending_problem.html

# Importing the usual packages
import numpy as np
import pandas as pd

# Import PuLP modeler functions
from pulp import *
```


```python
# Create the 'prob' variable to contain the problem data
prob = LpProblem("FF15 Comrades Weapon Upgrade Optimization", LpMinimize)
```


```python
# Importing spreadsheet containing all upgrade item data from "FFXV Comrades Cheat Sheet 2.0"
# https://docs.google.com/spreadsheets/d/1BEGmEHH5TlnlZhSRLx1ToZzijeLm7--jB_QafNSSBCk/edit
item_data = pd.read_excel("FFXV Comrades Cheat Sheet 2.0.xlsx", sheet_name = "Items", header = 0, usecols = "B:N")

# Fill missing values with 0
item_data = item_data.fillna(0)
item_data.head()
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
      <th>Item</th>
      <th>HP</th>
      <th>MP</th>
      <th>STR</th>
      <th>VIT</th>
      <th>MAG</th>
      <th>SPI</th>
      <th>FIR</th>
      <th>ICE</th>
      <th>LNG</th>
      <th>DRK</th>
      <th>SHT</th>
      <th>EXP</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Accursed Coin</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>2.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.02</td>
      <td>0.0</td>
      <td>490</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Ammonite Fossil</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>3.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>450</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Anak Antlers</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>5.0</td>
      <td>0.0</td>
      <td>3.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>890</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Anak Fetlock</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>4.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>550</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Ancient Cloth</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>3.0</td>
      <td>3.0</td>
      <td>5.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.00</td>
      <td>0.0</td>
      <td>860</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Creating list of items
Items = item_data["Item"].tolist()

# Creating dictionary "Items" to contained referenced variables. Also we can only have integer amounts of items used.
item_vars = LpVariable.dicts("Items", Items, 0, None, LpInteger)
```


```python
# Create dictionary of all stats
itemSTATS = {}
for stat in item_data.columns[1:]:
    itemSTATS[stat] = (item_data[["Item", stat]].set_index("Item").to_dict()[stat])
```


```python
# Add objective function to 'prob'
prob += lpSum([itemSTATS["EXP"][i] * item_vars[i] for i in Items]), "Total experience used"
```


```python
### Can put this in a loop in the final product that takes in a list [str, vit, ..., sht, exp]

# Add item-specific stat constraints
# In this case, we will use Corsesca, requiring 20 STR, 20 VIT, 20 MAG, 20 SPI to evolve at level 30 &
# 99 STR, 99 VIT to evolve at level 60
prob += lpSum([itemSTATS["STR"][i] * item_vars[i] for i in Items]) >= 20.0, "StrengthRequirement"
prob += lpSum([itemSTATS["VIT"][i] * item_vars[i] for i in Items]) >= 20.0, "VitalityRequirement"
prob += lpSum([itemSTATS["MAG"][i] * item_vars[i] for i in Items]) >= 20.0, "MagicRequirement"
prob += lpSum([itemSTATS["SPI"][i] * item_vars[i] for i in Items]) >= 20.0, "SpiritRequirement"
```


```python
# The problem data is written to an .lp file
prob.writeLP("ComradesTest.lp")
```


```python
# The problem is solved using PuLP's choice of Solver
prob.solve()
```




    1




```python
# The status of the solution is printed to the screen
print("Status:", LpStatus[prob.status])
```

    Status: Optimal
    


```python
# Each of the variables is printed with it's resolved optimum value
for v in prob.variables():
    if v.varValue > 0:
        print(v.name, "=", v.varValue)
```

    Items_Ancient_Cloth = 1.0
    Items_Dragon_Horn = 1.0
    Items_Octolegs = 1.0
    Items_Rough_Scale = 3.0
    


```python
# The optimised objective function value is printed to the screen
print("Total experience used = ", value(prob.objective))
```

    Total experience used =  5580.0
    
