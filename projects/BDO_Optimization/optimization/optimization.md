---
layout: page
title: Black Desert Online Worker Optimization
permalink: /projects/BDO_Optimization/optimization/
---

This part of the project is currently on hold. As it turns out, optimizing paths on a graph with many variables is very difficult. I have tried two different methods so far: recursive ant colony optimization and using Python's igraph library to find and calculate "reasonable" paths. Calculating paths between 300+ nodes on a weighted graph has a high time complexity which seems to make the last method unfeasible.

I plan on learning machine learning to see if those methods will assist in finishing this project.

## Recursive ant colony optimization

This was my first attempt at optimizing, using a modified version of the ant colony optimization algorithm from [Akavall's Github](https://github.com/Akavall/AntColonyOptimization). My Python script can be downloaded here: [ant_colony_modified.py]({{ site.baseurl }}/projects/BDO_Optimization/optimization/ant_colony_modified.py)

## igraph

Second attempt used igraph before I ran into a problem of time complexity. [igraph_optimization.md]({{ site.baseurl }}/projects/BDO_Optimization/optimization/igraph_optimization/)