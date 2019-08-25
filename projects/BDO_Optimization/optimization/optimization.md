---
layout: page
title: Black Desert Online Worker Optimization
permalink: /projects/BDO_Optimization/optimization/
---

As far as algorithms are concerned, I think K-depth is the best I can do. Further improvements would have to be made using machine learning, but I need A LOT of data (user-generated node networks) that is not readily available at the moment.

Things I still have to do:
  - Add in workers and worker variables
  - Maybe export these algorithms to standalone Python scripts

## [K shortest simple paths]({{ site.baseurl }}/projects/BDO_Optimization/optimization/k-shortest)

This algorithm uses networkx's [shortest_simple_paths](https://networkx.github.io/documentation/networkx-1.10/reference/generated/networkx.algorithms.simple_paths.shortest_simple_paths.html) to compute k simple paths.
From these k simple paths, this algorithm calculates the value of the path with the assumption that the player will take all subnodes available in the path.
It will then find the total CP cost of taking all nodes and subnodes in the path and take the path value and total CP cost to get a value per CP ratio to rank these k simple paths.
The path with highest value per CP ratio will be chosen as the best path from k simple paths.

Note that this algorithm is VERY GREEDY! It assumes that players will take all nodes and subnodes in the best chosen path, which is unrealistic as there is a CP softcap in the game.
K-depth iterative path generator is not as greedy as this one.

## [K-depth Iterative Path Generator]({{ site.baseurl }}/projects/BDO_Optimization/optimization/k-depth)

This algorithm finds the shortest path between two given nodes using networkx's [bidirectional Dijkstra](https://networkx.github.io/documentation/networkx-1.8.1/reference/generated/networkx.algorithms.shortest_paths.weighted.bidirectional_dijkstra.html) to compute the shortest path.
From this shortest path, it iterates over subnodes in the path AS WELL AS subnodes of neighboring nodes up to some depth value (i.e. depth 1 would look at neighbors, depth 2 neighbors of neighbors, etc.).

Starting with the shortest path between two nodes, the algorithm considers subnodes in the path and subnodes of neighoring nodes. It chooses subnodes that will increase its average value per CP (and thus iteratively improving itself).

## K shortest simple paths vs. K-depth iterative path generator (in progress)

Will have a short comparison between these two algorithms.

## Recursive ant colony optimization

This was my first attempt at optimizing, using a modified version of the ant colony optimization algorithm from [Akavall's Github](https://github.com/Akavall/AntColonyOptimization).
My Python script can be downloaded here: [ant_colony_modified.py]({{ site.baseurl }}/projects/BDO_Optimization/optimization/ant_colony_modified.py).
Update: Not too sure where this algorithm fits in now since I want to give users control of which nodes they want to optimize between. This algorithm simply seeks out high value subnodes around a given starting node.

## igraph (Depreciated)

Second attempt used igraph before I ran into a problem of time complexity. [igraph_optimization.md]({{ site.baseurl }}/projects/BDO_Optimization/optimization/igraph_optimization/).
I switched from igraph to networkx in favor of networkx's bidirections Dijkstra algorithm. This eliminates the problem of time complexity I faced using igraph's functions.