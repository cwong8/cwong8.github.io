---
layout: page
title: K-depth Iterative Path Generator
permalink: /projects/BDO_Optimization/optimization/k-depth
---

# Setup

Loading packages, initializing data and variables.


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

# Subnode information for subnode attributes and edges
node_subnode = pd.read_sql(
    """
    SELECT node.name, subnode.name AS subnode_name, subnode.cp AS subnode_cp, base_workload, current_workload
    FROM node JOIN subnode ON node.node_id = subnode.node_id;
    """, con = conn)

# Get node ID given a single node's name
def get_nx_node_id(name):
    return([x for x,y in G.nodes(data=True) if y["name"] == name][0])

# Add subnodes to node graph G
for i in range(node_subnode.shape[0]):
    G.add_node(i+1000, name = node_subnode.loc[i]["subnode_name"], cp = node_subnode.loc[i]["subnode_cp"],
               base_workload = node_subnode.loc[i]["base_workload"], current_workload = node_subnode.loc[i]["current_workload"])
    G.add_edge(get_nx_node_id(node_subnode.loc[i]["name"]), i+1000, weight = node_subnode.loc[i]["subnode_cp"])

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
    
    
# Add subnode values to node graph G
for i in range(subnode_values.shape[0]):
    nx.set_node_attributes(G, {i+1000: {"subnode_value_user": subnode_values.loc[i]["subnode_value_user"],
                                        "subnode_value_recent": subnode_values.loc[i]["subnode_value_recent"],
                                        "subnode_value_vendor": subnode_values.loc[i]["subnode_value_vendor"]}})

### Helper functions
# Takes in list as input (use get_nx_node_id for a single node). Also only works on main nodes, no subnodes.
def get_node_id(name):
    return([node[node['name'] == x].index[0] for x in name])

def get_node_name(idx):
    return(node.iloc[idx]['name'])

# Gets total CP for a list of NODES only
def get_total_cp(idx):
    return(sum(node.iloc[idx]['cp'].values))

# This would be a simple call to a subnode_id's neighbors if I added them in the graph...
# But since I did not, we have this thing
def get_parent_node(subnode_id):
    return(get_nx_node_id(G.nodes[subnode_id]["name"].split(" - ")[0]))

def get_subnodes(node_id):
    return([x for x in list(G.neighbors(node_id)) if x >= 1000])
```


```python
import itertools
def get_neighbors(path, depth = 1):   
    """
    Given a path as input, return neighbors with depth/degrees of separation to nodes in the path.
    
    Args:
        path: Any list of nodes that you call a path. In my case, it is the output of networkx's bidirectional Dijkstra.
            Ex. _, path = nx.bidirectional_dijkstra(G, get_node_id(["Calpheon"])[0], get_node_id(["Altinova"])[0])
        depth: Depth/degrees of separation for neighbors of nodes in path. Default depth = 1 because I found that the search
               area for higher depths deviates from our goal of going from start node to end node.
            Depth = 1 returns neighbors of nodes in path
            Depth = 2 returns neighbors of neighbors of nodes in path
            .
            .
            .
            Depth = n returns neighbors of nodes returned by depth n-1
        
    Returns:
        Neighbors with depth/degrees of separation to nodes in the path.
    """
    # Main path will only contain main nodes in our path
    main_path = [x for x in path if x < 1000]
    # Keep track of what neighboring nodes we have seen. Important for higher depth values where we may not choose
    # the first neighbor of a node, but its neighbor's neighbor.
    nodes_considered = []
    if depth == 0:
        return (path)
    # Note that for each depth iteration, we cut off the start and end nodes of our path
    # This is so we do not stray too far from getting to our location
    # Visually creates an elliptical search area between start and end nodes rather than a large circle
    # This does not necessarily mean that our algorithm will not go "backwards"
    for node_id in main_path[1:-1]:
        neighbors = [x for x in set(G.neighbors(node_id)).difference(path) if x < 1000]
        for neighbor in neighbors:
            nodes_considered.append(neighbor)
            
    # Remove duplicates    
    nodes_considered = list(dict.fromkeys(itertools.chain(nodes_considered))) 
    return(get_neighbors(nodes_considered, depth = depth-1))
```

# K-depth iterative path generator

Let me tell ya, this algorithm hurt my head when building it. The amount of code here is only a fraction of what I had to write to test, build, and experiment with for this algorithm.

##### Pseudocode:
1. Find shortest path given a start and end node using bidirectional Dijkstra's.
2. Get all subnodes in the path.
3. Find neighbors of nodes in the path given some depth/degree of separation.
4. Get those neighbors' subnodes (if they exist).
5. Combine subnodes from (2) and (4).
6. Calculate value per CP for each subnode:
  - Value per CP is subnode value divided by subnode CP IF the main node is already in the path
  - Otherwise the value per CP is subnode value divided by subnode CP plus node CP

Basically
$$ \text{check_value} = \text{Value per CP(subnode)} =  \left\{
\begin{array}{ll}
      \frac{\text{Subnode value}}{\text{Subnode CP cost}} & \text{if main node in path} \\
      \frac{\text{Subnode value}}{\text{Subnode CP cost + Node CP cost}} & \text{if main node NOT in path} \\
\end{array} 
\right. $$

However, we only need to do this for the best value per CP subnode if there are multiple. See below.

##### How this all works
* If the node is not in the path (i.e. is a neighboring node), then include the main node CP in calculations.
* Then "add" the neighbor node to path so its other subnodes keep their original subnode_value_user_per_cp. Don't really add 
  the neighbor node to the path yet. We keep track of this in node_not_in_path. We only truly add the main node to the 
  path when the check_value of the "best" subnode of that neighbor increases our path value.
* This allows us to have the best check if a neighboring subnode is worth including in our path.

Ex:

  Neighboring node costs 3 CP with subnodes A and B with:
    * Subnode A has 70k value for 2CP (so 35k value per CP)
    * Subnode B has 50k value for 3CP (so 16.6k value per CP)
  We pick the best subnode (in this case subnode A), and add the node CP to it.
  So, when we do calculations, subnode A has a check_value of 70k value for 5CP (or 14k value per CP).
  Note this is worse than subnode B, now, but consider the other case if we added node CP to subnode B.
  Then subnode B would have a check_value of 50k value for 6 CP (or 8.33k value per CP).
  It has a much lower check_value than subnode A so our algorithm would most likely skip it, even though subnode A
  has an amazing 35k value per CP.
  By using the best subnode for each neighbor, we ensure that we maximize the check_value for neighbors' subnodes
  and thus their likelihood of being added to the path.
  So, let's go back and add subnode A with check_value 14k value per CP.
  Two cases exist:
    1. The other subnode is better value per CP than the best subnode's check_value, so in this case subnode B
        with value per CP of 16.6k is better than subnode A's check_value of 14k value per CP.
        For this case, we also add subnode B to our path because the path's value per CP would increase.
    2. The other subnode is worse value per CP. Let's call this subnode C with 5k value per CP.
        Then we just skip subnode C because it adds no value to our path.


```python
if x is not None: print("adf")
```

    adf
    


```python
def optimize_path(initpath = None, start = None, end = None, max_depth = 1, output = False, diagnostics = False):
    """
    Optimizes a path by maximizing value per CP.
    Turn on diagnostics (diagnostics = True) to see the algorithm in action.
    Turn on output (output = True) to see information of the final optimized path.
    
    Args:
        initpath: The path you want optimized. Recommend to use the path generated by networkx's bidirectional Dijkstra.
                  Use this if you have multiple paths you want to optimize on, otherwise specify a start and end node.
        max_depth: Maximum depth our algorithm searches through. See get_neighbors for more information on depth.
                   Recommended max_depth = 1 for the same reasons as in get_neighbors.
        output: Prints information of the final optimized path.
        Diagnostics: Prints the thought process of the algorithm.
        
    Returns:
        1. A path with the best value per CP from start to end, including searching for neighboring nodes up to max_depth.
        2. A dataframe containing information on chosen subnodes that added value to our path.
    """
    
    # If you choose to use start and end
    if (start and end) is not None:
        # Initial path is stored in "initpath", and the iterative path we build is stored in "path"
        # This is because of how I set up the get_neighbors function
        _, initpath = nx.bidirectional_dijkstra(G, get_node_id([start])[0], get_node_id([end])[0])
    elif (start is None and end is not None) or (start is not None and end is None):
        print("Error: You must specify both start and end if you are using them!")
        return ()
    
    # If you choose to use initpath
    # This is useful if you want to have multiple start and end locations. Then create initpath using combine_paths.
    if initpath is not None:
        path = initpath
    
    # Our initial path only contains nodes, so there is zero value
    path_value = 0
    # Get the total node CP cost for nodes in our initial path
    path_node_CP = get_total_cp(path)
    # We start with no subnodes in our path, so we do not have any subnode CP cost
    path_subnode_CP = 0
    # Simple division to get the path value per CP 
    path_value_per_cp = path_value / (path_node_CP + path_subnode_CP)
    # Will contain subnodes the algorithm chooses (i.e. subnodes that increase the path's value per CP)
    subnodes_chosen = []
    # Will be the dataframe for data about subnodes we have chosen
    subnodes_chosen_df = pd.DataFrame(columns = subnode_values.columns)
    # Master dataframe containing data on subnodes in our path and its neighbors
    all_subnodes_df = []

    # Diagnostics
    if diagnostics == True:
        print("Start path value: " + str(path_value))
        print("Node CP: " + str(path_node_CP) + "\tSubnode CP: " + str(path_subnode_CP) + "\tTotal CP: " + str(path_node_CP + path_subnode_CP))
        print("***********************************************************************************************")

    # Loop that gets neighbors up to a certain depth/degree of separation
    for d in range(max_depth+1):
        # Initial path thingy
        for node_id in path:
            path_subnodes_df = pd.DataFrame(columns = subnode_values.columns)
            # Get subnodes in shortest path
            for node_id in path:
                # path_subnodes_df contains all subnodes along our path
                path_subnodes_df = path_subnodes_df.append(subnode_values.loc[subnode_values["node_name"] == G.nodes[node_id]["name"]])    


        # Neighbors and their subnodes
        neighbors = get_neighbors(initpath, depth = d)
        neighbor_subnodes_df = pd.DataFrame(columns = subnode_values.columns)
        for neighbor in neighbors:
            # For higher depths, neighbors of neighbors may contain nodes in our path so we check this
            if neighbor not in path:
                # Get subnode IDs of a neighboring node
                subnode_ids = get_subnodes(neighbor)
                # Get dataframe containing all neighbors' subnode data
                for subnode_id in subnode_ids:
                    subnode_name = G.nodes[subnode_id]["name"]
                    neighbor_subnodes_df = neighbor_subnodes_df.append(subnode_values.loc[subnode_values["subnode_name"] == subnode_name])


        # Put path subnodes and neighbors' subnodes together for comparison later, drop duplicates, order by value per CP
        all_subnodes_df = pd.concat([path_subnodes_df, neighbor_subnodes_df])
        all_subnodes_df = all_subnodes_df.drop_duplicates()
        all_subnodes_df = all_subnodes_df.sort_values(by = "subnode_value_user_per_cp", ascending = False)

        
        for index, row in all_subnodes_df.iterrows():
            # I use a binary variable for not in path as opposed to is in path because I want my binary variable to have
            # value 1 (or True) when the node is not in our path and 0 (or False) when the node is in our path
            # It makes sense when you see how I use it to calculate check_value
            node_not_in_path = get_nx_node_id(row["node_name"]) not in path
            # Get the subnode ID to check if we already added it to the path
            subnode_id = get_nx_node_id(row["subnode_name"])
            # Main node CP for the corresponding subnode
            node_cp = G.nodes[get_nx_node_id(row["node_name"])]["cp"]
            # check_value is value we check, basically a modified "subnode_value_user_per_cp" column for neighbors' subnodes
            check_value = row["subnode_value_user_per_cp"] / (row["cp"] + node_cp * node_not_in_path)
            # Actually I can just use an if/else statement for the above so I might be dumb.

            # If the adjusted value per CP will improve our path value, then add both node and subnode to path
            if (check_value > path_value_per_cp and subnode_id not in subnodes_chosen):
                # If the node is not in the path already, then add it along with its CP cost
                if node_not_in_path:
                    path.append(get_nx_node_id(row["node_name"]))
                    path_node_CP += G.nodes[get_nx_node_id(row["node_name"])]["cp"]
                    # Diagnostics
                    if diagnostics == True:
                        print("Node added: " + row["node_name"])
                # Add subnode to path
                path.append(get_nx_node_id(row["subnode_name"]))
                # Update path value
                path_value += row["subnode_value_user"]
                # Add subnode CP
                path_subnode_CP += row["cp"]
                # Update path value per CP
                path_value_per_cp = path_value / (path_node_CP + path_subnode_CP)
                # Keeping track of chosen subnodes
                subnodes_chosen.append(subnode_id)
                subnodes_chosen_df = subnodes_chosen_df.append(row)
                
                # Diagnostics
                if diagnostics == True:
                    print("Subnode added: " + row["subnode_name"])
                    print("New path value: " + str(path_value))
                    print("Node CP: " + str(path_node_CP) + "\tSubnode CP: " + str(path_subnode_CP) + "\tTotal CP: " + str(path_node_CP + path_subnode_CP))
                    print("Updated path value per CP: " + str(path_value_per_cp))
                    print("_______________________________________________________________________________________________")
        
    # Now let's offload lower value subnodes
    subnodes_chosen_df = subnodes_chosen_df.sort_values(by = "subnode_value_user_per_cp", ascending = True)
    # Basically reverse iteration, going through subnodes that we have chosen to see which ones can be dropped to increase
    # our average path value per CP
    for index, row in subnodes_chosen_df.iterrows():
        if (row["subnode_value_user_per_cp"] <= path_value_per_cp):
            # Remove subnode from path
            path.remove(get_nx_node_id(row["subnode_name"]))
            # Decrease path value because subnode is no longer in path
            path_value -= row["subnode_value_user"]
            # Decrease subnode CP because subnode is no longer in path
            path_subnode_CP -= row["cp"]
            # Recalculate value per CP (it should increase if you need a sanity check)
            path_value_per_cp = path_value / (path_node_CP + path_subnode_CP)
            
            # Diagnostics
            if diagnostics == True:
                print("Subnode removed: " + row["subnode_name"])
                print("New path value: " + str(path_value))
                print("Node CP: " + str(path_node_CP) + "\tSubnode CP: " + str(path_subnode_CP) + "\tTotal CP: " + str(path_node_CP + path_subnode_CP))
                print("Updated path value per CP: " + str(path_value_per_cp))
                print("_______________________________________________________________________________________________")
                
    if output == True:
        print("%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%")
        if (start and end) is not None:
            print("Start: " + start + " | End: " + end)
        print("\tNode CP cost: " + str(path_node_CP))
        print("\tSubnode CP cost: " + str(path_subnode_CP))
        print("\tTotal CP cost: " + str(path_node_CP + path_subnode_CP))
        print("\tPath value: " + str(path_value))
        print("\tPath value per CP: " + str(path_value_per_cp))
        print("%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%")
    
    # Just reordering subnodes by value per CP, purely optional and up to your preference
    subnodes_chosen_df = subnodes_chosen_df.sort_values(by = "subnode_value_user_per_cp", ascending = False)
    return (path, subnodes_chosen_df)
```


```python
# Demonstration of optimize_path
_, _ = optimize_path(start = "Calpheon", end = "Altinova", max_depth = 1, output = True, diagnostics = True)
```

    Start path value: 0
    Node CP: 18	Subnode CP: 0	Total CP: 18
    ***********************************************************************************************
    Subnode added: Lynch Farm Ruins - Excavation.2
    New path value: 19963.259972453117
    Node CP: 18	Subnode CP: 1	Total CP: 19
    Updated path value per CP: 1050.697893287006
    _______________________________________________________________________________________________
    Subnode added: Oze Pass - Lumbering.1
    New path value: 30534.01968896389
    Node CP: 18	Subnode CP: 2	Total CP: 20
    Updated path value per CP: 1526.7009844481945
    _______________________________________________________________________________________________
    Subnode added: Northern Plain Of Serendia - Lumbering.2
    New path value: 36717.06983745098
    Node CP: 18	Subnode CP: 3	Total CP: 21
    Updated path value per CP: 1748.4318970214754
    _______________________________________________________________________________________________
    Subnode added: Ahto Farm - Farming.1
    New path value: 47777.66969001293
    Node CP: 18	Subnode CP: 5	Total CP: 23
    Updated path value per CP: 2077.289986522301
    _______________________________________________________________________________________________
    Subnode added: Kamasylve Temple - Farming.2
    New path value: 57252.50991791487
    Node CP: 18	Subnode CP: 7	Total CP: 25
    Updated path value per CP: 2290.100396716595
    _______________________________________________________________________________________________
    Subnode added: Northern Plain Of Serendia - Gathering.1
    New path value: 60079.44001716375
    Node CP: 18	Subnode CP: 8	Total CP: 26
    Updated path value per CP: 2310.747692967837
    _______________________________________________________________________________________________
    Subnode added: Lynch Farm Ruins - Gathering.1
    New path value: 62819.75991791487
    Node CP: 18	Subnode CP: 9	Total CP: 27
    Updated path value per CP: 2326.6577747375877
    _______________________________________________________________________________________________
    Node added: Ancient Ruins Excavation Site
    Subnode added: Ancient Ruins Excavation Site - Excavation.1
    New path value: 114314.3696629405
    Node CP: 19	Subnode CP: 12	Total CP: 31
    Updated path value per CP: 3687.560311707758
    _______________________________________________________________________________________________
    Node added: Alejandro Farm
    Subnode added: Alejandro Farm - Farming.2
    New path value: 130023.76940125227
    Node CP: 21	Subnode CP: 13	Total CP: 34
    Updated path value per CP: 3824.2285118015375
    _______________________________________________________________________________________________
    Node added: Lynch Ranch
    Subnode added: Lynch Ranch - Farming.1
    New path value: 142246.8889234662
    Node CP: 23	Subnode CP: 14	Total CP: 37
    Updated path value per CP: 3844.5105114450325
    _______________________________________________________________________________________________
    Subnode removed: Lynch Farm Ruins - Gathering.1
    New path value: 139506.5690227151
    Node CP: 23	Subnode CP: 13	Total CP: 36
    Updated path value per CP: 3875.182472853197
    _______________________________________________________________________________________________
    Subnode removed: Northern Plain Of Serendia - Gathering.1
    New path value: 136679.6389234662
    Node CP: 23	Subnode CP: 12	Total CP: 35
    Updated path value per CP: 3905.132540670463
    _______________________________________________________________________________________________
    %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
    Start: Calpheon | End: Altinova
    	Node CP cost: 23
    	Subnode CP cost: 12
    	Total CP cost: 35
    	Path value: 136679.6389234662
    	Path value per CP: 3905.132540670463
    %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
    

##### Simple extension of our algorithm to do multiple paths at once


```python
def combine_paths(locations):
    """
    Can take multiple start and end location pairings and will combine all shortest paths between location pairings
    generated by networkx's bidirectional Dijkstra.
    
    Args:
        locations: Pairings of start and end locations. They should both be main nodes.
            Ex. [("Calpheon", "Altinova"), ("Calpheon", "Heidel"), ("Heidel", "Velia")]
        
    Returns:
        One master path that connects all location pairings given as input.   
    """
    master_path = []
    
    for path_location in locations:
        _, path = nx.bidirectional_dijkstra(G, get_node_id([path_location[0]])[0], get_node_id([path_location[1]])[0])
        master_path.append(path)
        
    master_path = list(dict.fromkeys(itertools.chain(*master_path)))
    
    return (master_path)
```


```python
initpath = combine_paths([("Calpheon", "Altinova"), ("Calpheon", "Heidel"), ("Heidel", "Velia")])
```


```python
main, _ = optimize_path(initpath, output = True, diagnostics = True)
```

    Start path value: 0
    Node CP: 25	Subnode CP: 0	Total CP: 25
    ***********************************************************************************************
    Subnode added: Lynch Farm Ruins - Excavation.2
    New path value: 19963.259972453117
    Node CP: 25	Subnode CP: 1	Total CP: 26
    Updated path value per CP: 767.8176912481969
    _______________________________________________________________________________________________
    Subnode added: Oze Pass - Lumbering.1
    New path value: 30534.01968896389
    Node CP: 25	Subnode CP: 2	Total CP: 27
    Updated path value per CP: 1130.8896181097737
    _______________________________________________________________________________________________
    Subnode added: Northern Plain Of Serendia - Lumbering.2
    New path value: 36717.06983745098
    Node CP: 25	Subnode CP: 3	Total CP: 28
    Updated path value per CP: 1311.3239227661065
    _______________________________________________________________________________________________
    Subnode added: Ahto Farm - Farming.1
    New path value: 47777.66969001293
    Node CP: 25	Subnode CP: 5	Total CP: 30
    Updated path value per CP: 1592.5889896670976
    _______________________________________________________________________________________________
    Subnode added: Forest Of Plunder - Gathering.1
    New path value: 53048.879828095436
    Node CP: 25	Subnode CP: 6	Total CP: 31
    Updated path value per CP: 1711.2541880030785
    _______________________________________________________________________________________________
    Subnode added: Kamasylve Temple - Farming.2
    New path value: 62523.72005599737
    Node CP: 25	Subnode CP: 8	Total CP: 33
    Updated path value per CP: 1894.6581835150719
    _______________________________________________________________________________________________
    Subnode added: Northern Plain Of Serendia - Gathering.1
    New path value: 65350.65015524626
    Node CP: 25	Subnode CP: 9	Total CP: 34
    Updated path value per CP: 1922.077945742537
    _______________________________________________________________________________________________
    Subnode added: Lynch Farm Ruins - Gathering.1
    New path value: 68090.97005599737
    Node CP: 25	Subnode CP: 10	Total CP: 35
    Updated path value per CP: 1945.4562873142106
    _______________________________________________________________________________________________
    Node added: Ancient Ruins Excavation Site
    Subnode added: Ancient Ruins Excavation Site - Excavation.1
    New path value: 119585.579801023
    Node CP: 26	Subnode CP: 13	Total CP: 39
    Updated path value per CP: 3066.296917974949
    _______________________________________________________________________________________________
    Node added: Alejandro Farm
    Subnode added: Alejandro Farm - Farming.2
    New path value: 135294.97953933477
    Node CP: 28	Subnode CP: 14	Total CP: 42
    Updated path value per CP: 3221.309036650828
    _______________________________________________________________________________________________
    Node added: Lynch Ranch
    Subnode added: Lynch Ranch - Farming.1
    New path value: 147518.0990615487
    Node CP: 30	Subnode CP: 15	Total CP: 45
    Updated path value per CP: 3278.179979145527
    _______________________________________________________________________________________________
    Node added: Costa Farm
    Subnode added: Costa Farm - Farming.2
    New path value: 157459.48875111341
    Node CP: 32	Subnode CP: 16	Total CP: 48
    Updated path value per CP: 3280.406015648196
    _______________________________________________________________________________________________
    Subnode added: Costa Farm - Farming.1
    New path value: 160988.8688456416
    Node CP: 32	Subnode CP: 17	Total CP: 49
    Updated path value per CP: 3285.4871192988085
    _______________________________________________________________________________________________
    Subnode removed: Lynch Farm Ruins - Gathering.1
    New path value: 158248.5489448905
    Node CP: 32	Subnode CP: 16	Total CP: 48
    Updated path value per CP: 3296.8447696852186
    _______________________________________________________________________________________________
    Subnode removed: Northern Plain Of Serendia - Gathering.1
    New path value: 155421.6188456416
    Node CP: 32	Subnode CP: 15	Total CP: 47
    Updated path value per CP: 3306.8429541625874
    _______________________________________________________________________________________________
    %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
    	Node CP cost: 32
    	Subnode CP cost: 15
    	Total CP cost: 47
    	Path value: 155421.6188456416
    	Path value per CP: 3306.8429541625874
    %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
    

Final path our algorithm produced.


```python
for id_ in main:
    print(G.nodes[id_]["name"])
```

    Calpheon
    Falres Dirt Farm
    Marni Farm Ruins
    Oze Pass
    Bradie Fortress
    Northern Plain Of Serendia
    Lynch Farm Ruins
    Heidel
    Eastern Border
    Kamasylve Temple
    Ahto Farm
    Stonetail Horse Ranch
    Asula Highland
    Highland Junction
    Altinova Entrance
    Altinova
    Northern Guard Camp
    Heidel Pass
    Forest Of Plunder
    Velia
    Lynch Farm Ruins - Excavation.2
    Oze Pass - Lumbering.1
    Northern Plain Of Serendia - Lumbering.2
    Ahto Farm - Farming.1
    Forest Of Plunder - Gathering.1
    Kamasylve Temple - Farming.2
    Ancient Ruins Excavation Site
    Ancient Ruins Excavation Site - Excavation.1
    Alejandro Farm
    Alejandro Farm - Farming.2
    Lynch Ranch
    Lynch Ranch - Farming.1
    Costa Farm
    Costa Farm - Farming.2
    Costa Farm - Farming.1
    
