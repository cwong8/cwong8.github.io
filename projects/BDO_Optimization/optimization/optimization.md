---
layout: page
title: Black Desert Online Worker Optimization
permalink: /projects/BDO_Optimization/optimization/
---

This part is active again! I am going to test and compare a bunch of algorithms to find a solution.

## [K shortest simple paths]({{ site.baseurl }}/projects/BDO_Optimization/optimization/k-shortest)

This algorithm uses networkx's [shortest_simple_paths](https://networkx.github.io/documentation/networkx-1.10/reference/generated/networkx.algorithms.simple_paths.shortest_simple_paths.html) to compute k simple paths.
From these k simple paths, this algorithm calculates the value of the path with the assumption that the player will take all subnodes available in the path.
It will then find the total CP cost of taking all nodes and subnodes in the path and take the path value and total CP cost to get a value per CP ratio to rank these k simple paths.
The path with highest value per CP ratio will be chosen as the best path from k simple paths.

Note that this algorithm is VERY GREEDY! It assumes that players will take all nodes and subnodes in the best chosen path, which is unrealistic as there is a CP softcap in the game.
I have another algorithm coming that is not as greedy as this one.

## Iterative path builder (in progress)

This algorithm finds the shortest path between two given nodes using networkx's [bidirectional Dijkstra](https://networkx.github.io/documentation/networkx-1.8.1/reference/generated/networkx.algorithms.shortest_paths.weighted.bidirectional_dijkstra.html) to compute the shortest path.
From this shortest path, it iterates over subnodes in the path AS WELL AS subnodes of neighboring nodes up to some depth value (i.e. depth 1 would look at neighbors, depth 2 neighbors of neighbors, etc.).

This becomes complicated quickly because for neighboring nodes, we have to include the base node CP cost into the calculation because it is not in our shortest path.
Take a neighboring node with two subnodes A and B:
  - The value per CP of any subnode A or B will have to be calculated using A or B's subnode CP cost as well as the node CP cost because we have to add the node to our path before accessing its subnodes.
  - If we already added A or B to our path, then we do not have to use the base node's CP cost again because that would be double-counting.
  - Consider neighbors of neighbors and this becomes very complicated.
  
Once the hard part described above is done, then we can add subnodes to our path that increase its current value per CP (i.e. consider subnodes whose value per CP > path's current value per CP).
This should yield better results than K shortest simple paths because this algorithm is more selective of subnodes it adds.

## Recursive ant colony optimization

This was my first attempt at optimizing, using a modified version of the ant colony optimization algorithm from [Akavall's Github](https://github.com/Akavall/AntColonyOptimization).
My Python script can be downloaded here: [ant_colony_modified.py]({{ site.baseurl }}/projects/BDO_Optimization/optimization/ant_colony_modified.py).
Update: Not too sure where this algorithm fits in now since I want to give users control of which nodes they want to optimize between. This algorithm simply seeks out high value subnodes around a given starting node.

## igraph (Depreciated)

Second attempt used igraph before I ran into a problem of time complexity. [igraph_optimization.md]({{ site.baseurl }}/projects/BDO_Optimization/optimization/igraph_optimization/).
I switched from igraph to networkx in favor of networkx's bidirections Dijkstra algorithm. This eliminates the problem of time complexity I faced using igraph's functions.