---
layout: page
title: Black Desert Online Worker Optimization
permalink: /projects/BDO_Optimization/optimization/igraph_optimization/
---

```python
# Loading environment from analysis notebook
import dill
dill.load_session('BDO_Optimization_env.db')
```


```python
import numpy as np
import pandas as pd
import re
pd.options.display.max_rows = 6
```


```python
# https://igraph.org/python/doc/igraph.Graph-class.html#shortest_paths_dijkstra
# https://igraph.org/python/doc/igraph.GraphBase-class.html#get_shortest_paths
```




    <igraph.Graph at 0x185963f5228>




```python
g.vs[0]
```




    igraph.Vertex(<igraph.Graph object at 0x00000185963F5228>, 0, {'name': 'Abandoned Iron Mine', 'area': 'Mediah', 'type': 'Worker Node', 'cp': 2})




```python
# Takes in list as input (i.e. get_node_id(["Abandoned Iron Mine"]) if you want a single node)
def get_node_id(name):
    return([node[node['NAME'] == x].index[0] for x in name])

def get_node_name(idx):
    return(node.iloc[idx]['NAME'])

def get_total_cp(idx):
    return(sum(node.iloc[idx]['CP'].values))
```

### Analyze with vs without weights for shortest paths
- Find the differences in profits between these two methods for all nodes
- Then try out different paths (i.e. paths under a certain length) and compare those


```python
# Dijkstra gives shortest path length
g.shortest_paths_dijkstra(get_node_id(["Abandoned Iron Mine"])[0], get_node_id(["Calpheon"])[0])
g.shortest_paths_dijkstra(get_node_id(["Abandoned Iron Mine"])[0], get_node_id(["Calpheon"])[0], weights = "weight")
```




    [[17.0]]




```python
a = g.get_shortest_paths(get_node_id(["Calpheon"])[0], get_node_id(["Abandoned Iron Mine"])[0])[0]
a
```




    [66, 98, 94, 59, 226, 192, 143, 102, 159, 11, 309, 36, 148, 21, 0]




```python
get_node_name(a).values
```




    array(['Calpheon', 'Dias Farm', 'Delphe Knights Castle', 'Biraghi Den',
           'Northern Plain Of Serendia', 'Lynch Farm Ruins', 'Heidel',
           'Eastern Border', 'Kamasylve Temple', 'Ahto Farm',
           'Stonetail Horse Ranch', 'Asula Highland', 'Highland Junction',
           'Altinova Entrance', 'Abandoned Iron Mine'], dtype=object)




```python
get_total_cp(a)

```




    23




```python
# Get all node IDs that edges point to
destinations = [x[1] for x in g.get_edgelist()]
cp_cost = [node.iloc[i, :]["CP"] for i in destinations]
g.es["weight"] = cp_cost
```


```python
g.es[0].attributes()
```




    {'weight': 1}




```python
b = g.get_shortest_paths(get_node_id(["Calpheon"])[0], get_node_id(["Abandoned Iron Mine"])[0], weights = "weight")[0]
b
```




    [66, 116, 204, 242, 62, 226, 192, 143, 102, 159, 11, 309, 36, 148, 21, 0]




```python
get_node_name(b)
```




    [66                Calpheon
     116       Falres Dirt Farm
     204       Marni Farm Ruins
                   ...         
     148      Highland Junction
     21       Altinova Entrance
     0      Abandoned Iron Mine
     Name: NAME, Length: 16, dtype: object]




```python
get_total_cp(b)
```




    20




```python
node_dist
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
      <th>NODE_NAME</th>
      <th>MIN_DIST</th>
      <th>MIN_CITY</th>
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
      <td>Abandoned Iron Mine</td>
      <td>1113.0</td>
      <td>Altinova</td>
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
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
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
      <td>NaN</td>
      <td>NaN</td>
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
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
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
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
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
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>359 rows × 10 columns</p>
</div>




```python
node.index
```




    RangeIndex(start=0, stop=359, step=1)


