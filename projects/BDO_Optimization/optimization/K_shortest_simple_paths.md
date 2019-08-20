---
layout: page
title: K shortest simple paths
permalink: /projects/BDO_Optimization/optimization/k-shortest
---


# K shortest simple paths

```python
# Importing packages
import numpy as np
import pandas as pd
import re
import MySQLdb

# Display settings
pd.options.display.max_rows = 6

# Server connection to MySQL:
conn = MySQLdb.connect(host= "localhost",
                  user="yourusername",
                  passwd="yourpassword",
                  db="bdodae_new")

x = conn.cursor()

# Importing SQL data
node = pd.read_sql("SELECT * FROM node;", con = conn)

# Create connection/adjacency matrix weighted with CP costs
cp_connection_matrix = pd.DataFrame(np.zeros((len(node["name"]), len(node["name"]))), index = node["name"], columns = node["name"])

for x, name in zip(node["connections"], node["name"]):
    connections = re.split(", ", x)
    for connected_nodes in connections:
        # Edges going into cities have weight 0 and thus disappear from our graph, but we don't want this to happen
        # So we will instead set their weights to a very small value
        if (node[node["name"] == connected_nodes]["cp"].values[0] == 0):
            cp_connection_matrix.at[name, connected_nodes] = 0.000000001
        else:
            cp_connection_matrix.at[name, connected_nodes] = node[node["name"] == connected_nodes]["cp"].values[0]

# Import networkx
import networkx as nx

# Create graph
G = nx.from_numpy_matrix(np.matrix(cp_connection_matrix), create_using = nx.DiGraph())
# Add node attributes
node_attributes = node[["name", "cp", "area", "type"]].to_dict('index')
nx.set_node_attributes(G, node_attributes)

### Helper functions
# Takes in list as input (use get_nx_node_id for a single node)
def get_node_id(name):
    return([node[node['name'] == x].index[0] for x in name])

def get_node_name(idx):
    return(node.iloc[idx]['name'])

def get_total_cp(idx):
    return(sum(node.iloc[idx]['cp'].values))

def get_nx_node_id(name):
    return([x for x,y in G.nodes(data=True) if y["name"] == name][0])
```


```python
# Get the total value of each subnode
subnode_values = pd.read_sql("""
SELECT item_yield.node_name, item_price.subnode_id, item_yield.subnode_name, item_yield.cp,
    SUM(item_price.user_average * item_yield.amount) AS subnode_value_user,
    SUM(item_price.recent_value * item_yield.amount) AS subnode_value_recent,
    SUM(item_price.vendor_sell * item_yield.amount) AS subnode_value_vendor
FROM
    (SELECT subnode_item.subnode_id, item.name, prices.*
    FROM subnode_item JOIN item ON subnode_item.item_id = item.item_id
      JOIN prices ON item.item_id = prices.item_id
    ORDER BY subnode_item.subnode_id, item.item_id) AS item_price
    JOIN
    (SELECT subnode.subnode_id, item.item_id, item.name, yield.amount, subnode.name AS subnode_name, subnode.cp, node.name AS node_name
    FROM subnode JOIN yield ON subnode.subnode_id = yield.subnode_id
      JOIN item ON yield.item_id = item.item_id
      JOIN node ON subnode.node_id = node.node_id
    ORDER BY subnode.subnode_id, item.item_id) AS item_yield
    ON item_price.subnode_id = item_yield.subnode_id
        AND item_price.item_id = item_yield.item_id
        AND item_price.name = item_yield.name
GROUP BY item_price.subnode_id
ORDER BY item_price.subnode_id
""", con = conn)

# Add per CP values
subnode_values["subnode_value_user_per_cp"] = subnode_values["subnode_value_user"] / subnode_values["cp"]
subnode_values["subnode_value_recent_per_cp"] = subnode_values["subnode_value_recent"] / subnode_values["cp"]

# Add subnode values to node graph G
for i in range(subnode_values.shape[0]):
    nx.set_node_attributes(G, {i+1000: {"subnode_value_user": subnode_values.loc[i]["subnode_value_user"],
                                        "subnode_value_recent": subnode_values.loc[i]["subnode_value_recent"],
                                        "subnode_value_vendor": subnode_values.loc[i]["subnode_value_vendor"]}})
    
# Adding subnode ranks based on their value per CP
subnode_values["value_rank"] = subnode_values["subnode_value_user_per_cp"].rank(ascending = False)
subnode_values["recent_rank"] = subnode_values["subnode_value_recent_per_cp"].rank(ascending = False)
```


```python
import itertools
# Find the k shortest paths using CP costs as edge weights
def k_shortest_paths(G, source, target, k, weight = 'weight'):
    return list(itertools.islice(nx.shortest_simple_paths(G, source, target, weight = weight), k))

# k = 20 is arbitrarily chosen, we will tune this later
paths = k_shortest_paths(G, get_node_id(["Calpheon"])[0], get_node_id(["Altinova"])[0], 20)
for path in paths:
    print(path)
```

    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 230, 109, 78, 318, 332, 176, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 78, 318, 332, 176, 38, 160, 23, 22]
    [69, 101, 97, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 255, 40, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 316, 328, 38, 160, 23, 22]
    [69, 101, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 255, 327, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 208, 332, 176, 38, 160, 23, 22]
    [69, 122, 219, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 298, 28, 29, 74, 316, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 230, 109, 78, 171, 12, 328, 38, 160, 23, 22]
    [69, 101, 97, 61, 244, 206, 155, 108, 171, 12, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 329, 189, 227, 255, 40, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 316, 227, 255, 40, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 329, 189, 227, 255, 327, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 316, 227, 255, 327, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 230, 241, 109, 78, 318, 332, 176, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 230, 109, 78, 318, 332, 208, 12, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 230, 109, 78, 318, 332, 176, 364, 25, 1, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 230, 109, 78, 318, 332, 176, 38, 255, 40, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 78, 318, 332, 208, 12, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 78, 318, 332, 176, 364, 25, 1, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 78, 318, 332, 176, 38, 255, 40, 23, 22]
    [69, 101, 97, 260, 65, 244, 206, 155, 230, 109, 78, 318, 332, 176, 38, 160, 23, 22]
    [69, 101, 97, 260, 65, 244, 206, 155, 108, 171, 78, 318, 332, 176, 38, 160, 23, 22]
    [69, 101, 122, 220, 260, 65, 244, 206, 155, 230, 109, 78, 318, 332, 176, 38, 160, 23, 22]
    [69, 101, 122, 220, 260, 65, 244, 206, 155, 108, 171, 78, 318, 332, 176, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 230, 109, 78, 318, 332, 176, 38, 255, 327, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 78, 318, 332, 176, 38, 255, 327, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 316, 227, 255, 40, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 160, 23, 0, 2, 7, 22]
    [69, 101, 97, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 255, 40, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 316, 328, 38, 255, 40, 23, 22]
    [69, 101, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 255, 40, 23, 22]
    [69, 122, 101, 97, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 160, 23, 22]
    [69, 122, 219, 238, 261, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 298, 28, 29, 171, 12, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 298, 28, 29, 74, 329, 189, 227, 255, 40, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 298, 28, 29, 74, 316, 227, 255, 40, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 329, 189, 227, 255, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 316, 227, 255, 38, 160, 23, 22]
    [69, 101, 97, 260, 65, 244, 206, 155, 108, 171, 29, 74, 316, 328, 38, 160, 23, 22]
    [69, 101, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 316, 328, 38, 160, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 316, 227, 255, 327, 23, 22]
    [69, 101, 97, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 255, 327, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 171, 29, 74, 316, 328, 38, 255, 327, 23, 22]
    [69, 101, 122, 220, 260, 65, 244, 206, 155, 108, 171, 12, 328, 38, 255, 327, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 298, 28, 29, 74, 329, 189, 227, 255, 327, 23, 22]
    [69, 122, 220, 260, 65, 244, 206, 155, 108, 298, 28, 29, 74, 316, 227, 255, 327, 23, 22]
    


```python
# Works best if you have a lot of CP, because this assumes you will take all subnodes
# Very greedy algorithm, seeks to maximize value per CP along a single path, ignoring potentially better neighboring nodes
def get_best_path(paths, output = False):
    """
    Gets the best value per CP path from paths generated by k_shortest_paths
    
    Args:
        paths: Output generated from k_shortest_paths
            Ex. paths = k_shortest_paths(G, get_node_id(["Calpheon"])[0], get_node_id(["Altinova"])[0], 20)
        output: Option to print detailed output. Set output = True to see this.
        
    Returns:
        The best value per CP path out of the k paths generated by k_shortest_paths
    """
    path_df_list = []

    # For each path generated by k_shortest_paths, get a dataframe containing all subnodes connected to nodes in the path
    # and then sum all values to get path value and additional CP cost
    for path in paths:
        path_subnode_values_df = pd.DataFrame()   
        for node_id in path:
            path_subnode_values_df = path_subnode_values_df.append(subnode_values.loc[subnode_values["node_name"] == G.nodes[node_id]["name"]])
        path_df_list.append(path_subnode_values_df)
        
    # Very messy, just summing up values for each path (naive approach)
    path_node_cp = []
    path_subnode_cp = []
    path_subnode_value_user = []
    path_subnode_value_recent = []
    path_subnode_value_vendor = []
    for path, path_df in zip(paths, path_df_list):
        path_node_cp.append(get_total_cp(path))
        path_subnode_cp.append(path_df["cp"].sum())
        path_subnode_value_user.append(path_df["subnode_value_user"].sum())
        path_subnode_value_recent.append(path_df["subnode_value_recent"].sum())
        path_subnode_value_vendor.append(path_df["subnode_value_vendor"].sum())
        
    # Element-wise addition of 2 lists
    from operator import add

    path_values = pd.DataFrame({
        "path": paths,
        "total_cp": list(map(add, path_node_cp, path_subnode_cp)),
        "value_user": path_subnode_value_user,
        "value_recent": path_subnode_value_recent,
        "value_vendor": path_subnode_value_vendor
    })
    
    # Add value per CP for each path
    path_values["value_user_per_cp"] = path_values["value_user"] / path_values["total_cp"]
    path_values["value_recent_per_cp"] = path_values["value_recent"] / path_values["total_cp"]
    
    # Get the best path and its subnodes
    best_path = path_values.loc[path_values["value_user_per_cp"].idxmax()]["path"]
    best_subnodes_df = path_df_list[path_values["value_user_per_cp"].idxmax()]
    
    # Print output if turned on
    if output == True:
        print("Start: " + get_node_name(best_path[0]) + " | End: " + get_node_name(best_path[len(best_path)-1]))
        print("\tNode CP cost: " + str(get_total_cp(best_path)))
        print("\tSubnode CP cost: " + str(best_subnodes_df["cp"].sum()))
        print("\tTotal CP cost: " + str(get_total_cp(best_path) + best_subnodes_df["cp"].sum()))
        print("\tPath value: " + str(best_subnodes_df["subnode_value_user"].sum()))
        print("\tPath value per CP: " + str(best_subnodes_df["subnode_value_user"].sum() / (get_total_cp(best_path) + best_subnodes_df["cp"].sum())))
    
    # Returns best path and corresponding subnode dataframe
    return(best_path, best_subnodes_df)
```


```python
# Demonstration of get_best_path
get_best_path(k_shortest_paths(G, get_node_id(["Calpheon"])[0], get_node_id(["Altinova"])[0], 20), output = True)
```

    Start: Calpheon | End: Altinova
    	Node CP cost: 21
    	Subnode CP cost: 24
    	Total CP cost: 45
    	Path value: 165422.21986860037
    	Path value per CP: 3676.0493304133415
    




    ([69,
      122,
      220,
      260,
      65,
      244,
      206,
      155,
      108,
      171,
      29,
      74,
      329,
      189,
      227,
      255,
      40,
      23,
      22],
                           node_name  subnode_id  \
     162                    Oze Pass         163   
     149  Northern Plain Of Serendia         150   
     150  Northern Plain Of Serendia         151   
     ..                          ...         ...   
     140                Mediah Shore         141   
     158              Omar Lava Cave         159   
     159              Omar Lava Cave         160   
     
                                      subnode_name  cp  subnode_value_user  \
     162                    Oze Pass - Lumbering.1   1        10570.759717   
     149  Northern Plain Of Serendia - Gathering.1   1         2826.930099   
     150  Northern Plain Of Serendia - Lumbering.2   1         6183.050148   
     ..                                        ...  ..                 ...   
     140                   Mediah Shore - Mining.1   3        10792.679879   
     158                 Omar Lava Cave - Mining.1   3        14428.530141   
     159                 Omar Lava Cave - Mining.2   3        15385.450207   
     
          subnode_value_recent  subnode_value_vendor  subnode_value_user_per_cp  \
     162          13332.749623           1626.549969               10570.759717   
     149           3481.250122            261.510009                2826.930099   
     150           6242.200157           1148.250031                6183.050148   
     ..                    ...                   ...                        ...   
     140          11103.299851           1932.789973                3597.559960   
     158          38878.100433           2127.240011                4809.510047   
     159          44988.250780           2707.420044                5128.483402   
     
          subnode_value_recent_per_cp  value_rank  recent_rank  
     162                 13332.749623        67.0         68.0  
     149                  3481.250122       178.0        175.0  
     150                  6242.200157        91.0        128.0  
     ..                           ...         ...          ...  
     140                  3701.099950       160.0        170.0  
     158                 12959.366811       126.0         73.0  
     159                 14996.083593       117.0         55.0  
     
     [12 rows x 11 columns])




```python
def optimize_paths(locations, output = False):
    """
    Extension of the get_best_path function. Can take multiple start and end location pairings and will combine results into
    one final path removing duplicates.
    
    Args:
        locations: Pairings of start and end locations. They should both be start nodes.
            Ex. [("Calpheon", "Altinova"), ("Calpheon", "Heidel"), ("Heidel", "Velia")]
        output: Option to print detailed output. Set output = True to see this.
        
    Returns:
        One master path that connects all location pairings given as input.   
    """
    master_path_df = pd.DataFrame()
    master_path = []
    
    for path_location in locations:
        path, path_df = get_best_path(k_shortest_paths(G, get_node_id([path_location[0]])[0], get_node_id([path_location[1]])[0], 20), output = output)
        master_path.append(path)
        master_path_df = pd.concat([master_path_df, path_df])
        if output == True:
            # Print a line for cleaner output
            print("_______________________________________________________________________________________________________________")
    
    # Output
    print("Combining paths...")
    master_path = list(dict.fromkeys(itertools.chain(*master_path)))
    print("Done")
    print("Cleaning subnode dataframes...")
    master_path_df = master_path_df.drop_duplicates()
    print("Done")
    print("_______________________________")
    print("Node CP: " + str(get_total_cp(master_path)))
    print("Subnode CP: " + str(master_path_df["cp"].sum()))
    print("Total CP: " + str(get_total_cp(master_path) + master_path_df["cp"].sum()))
    print("Total value: " + str(master_path_df["subnode_value_user"].sum()))
    print("Value per CP: " + str(master_path_df["subnode_value_user"].sum() / (get_total_cp(master_path) + master_path_df["cp"].sum())))
    
```


```python
# Demonstration of optimize_paths
optimize_paths([("Calpheon", "Altinova"), ("Calpheon", "Heidel"), ("Heidel", "Velia")], output = True)
```

    Start: Calpheon | End: Altinova
    	Node CP cost: 21
    	Subnode CP cost: 24
    	Total CP cost: 45
    	Path value: 165422.21986860037
    	Path value per CP: 3676.0493304133415
    _______________________________________________________________________________________________________________
    Start: Calpheon | End: Heidel
    	Node CP cost: 7
    	Subnode CP cost: 5
    	Total CP cost: 12
    	Path value: 42284.31983745098
    	Path value per CP: 3523.693319787582
    _______________________________________________________________________________________________________________
    Start: Heidel | End: Velia
    	Node CP cost: 13
    	Subnode CP cost: 9
    	Total CP cost: 22
    	Path value: 71020.9696764946
    	Path value per CP: 3228.225894386118
    _______________________________________________________________________________________________________________
    Combining paths...
    Done
    Cleaning subnode dataframes...
    Done
    _______________________________
    Node CP: 33
    Subnode CP: 31
    Total CP: 64
    Total value: 213739.60967189074
    Value per CP: 3339.6814011232927
    
