# Taichi DEM
A discrete element method (DEM) simulation demo written in Taichi.

![](./large_radius.gif)

## Installation
Make sure your `pip` is up-to-date:

```bash
$ pip3 install pip --upgrade
```

Assume you have a Python 3 environment, to install Taichi:

```bash
$ pip3 install -U taichi
```

## Assumptions
The `dem.py` implements a minimal DEM solver with the following assumptions:

- All paricles are round circles with variable radius.
- Only the normal force between particles is considered - the tangential force is not included.
- The deformation of the particles is not included.
- Ignore the angular momentum of the particle and only consider the translation of the particle.

## Demo

- `python dem.py`: this is the same demo as the one in the [taichi_dem](https://github.com/taichi-dev/taichi_dem) repo. I only modify the code so that we can play around with different grid sizes. I also set the number of simulation frames to be 200.
- `python dem_large_radius.py`: In this demo, four large particles are added to the top (see the GIF above). The simulation method is the same as in `dem.py`. Due to the limitation of the method, the smallest grid size we can use is 0.2 (`grid_n=5`). 
- `python dem_large_radius_grouped.py`: The system in this demo is the same as in `dem_large_radius.py`. I made a simple change t the method so that we can use smaller grid size. To make the exmplanation simple, I assume there are only two groups of particles - small particles and large particles. (The idea can be extended to multiple groups of particles of different range of sizes.) The largest radius of small particles satisfy `small_grain_r_max * 2 < grid_size`. There are three types of collisions in the system - small-small, small-large and large-large. If a small particle want to resolve all possible small-small collision it is involved, it only need to search small particles in the neighboring 3x3 grids. For small-large and large-large collisions, as they involve large particles, they can be resolved in the threads that handles large particle collisions, where a large neighboring region need to be searched. Thus, a smaller grid size can be used and we overcome the limitation of the original method. The core code snippet is shown below. 

```python
    # Fast collision detection
    for i in range(n):
        if i < ns: # small particles
            grid_idx = ti.floor(gf[i].p * grid_n, int)
            # small particles only search neighboring 3x3 grid
            # to resolve small-small collision
            x_begin = max(grid_idx[0] - 1, 0)
            x_end = min(grid_idx[0] + 2, grid_n)

            y_begin = max(grid_idx[1] - 1, 0)
            y_end = min(grid_idx[1] + 2, grid_n)

            for neigh_i in range(x_begin, x_end):
                for neigh_j in range(y_begin, y_end):
                    neigh_linear_idx = neigh_i * grid_n + neigh_j
                    for p_idx in range(list_head[neigh_linear_idx],
                                    list_tail[neigh_linear_idx]):
                        j = particle_id[p_idx]
                        if j < ns and i < j:
                            resolve(i, j) # small-small collision
        else: # large particles
            grid_idx = ti.floor(gf[i].p * grid_n, int)
            # large particles resolve all possible collisions 
            # so they need to search in a larger region
            x_begin = max(grid_idx[0] - SEARCH_NUM_GRID, 0)
            x_end = min(grid_idx[0] + 1 + SEARCH_NUM_GRID, grid_n)

            y_begin = max(grid_idx[1] - SEARCH_NUM_GRID, 0)
            y_end = min(grid_idx[1] + 1 + SEARCH_NUM_GRID, grid_n)

            for neigh_i in range(x_begin, x_end):
                for neigh_j in range(y_begin, y_end):
                    neigh_linear_idx = neigh_i * grid_n + neigh_j
                    for p_idx in range(list_head[neigh_linear_idx],
                                    list_tail[neigh_linear_idx]):
                        j = particle_id[p_idx]
                        if j >= ns and i < j: # large-large collision
                            resolve(i, j)
                        elif j < ns: # large-small collision
                            resolve(i, j)
```

Note that the performance of this parallel for loop is dominated by the threads that handles large particle collisions. 

## Performance

All the wall clock time reported here are with simulations of 200 frames. 

Here's a rough comparison of the ungrouped method and grouped method with different grid sizes. We can see the the grouped method slightly improves the performance if the right grid size is selected. 

| large radius method | ungrouped | grouped | grouped | grouped | grouped |
| --- | --- | --- | --- | --- | --- |
| grid_n | 5 | 5 | 32 | 64 | 128 |
| wall clock time (on gpu) | 37.41s | 40.48s | 35.21s | **33.66s** | 37.06s |

I wonder why `grid_n=64` performs better than `grid_n=128` so I did a test on grid size with the original system in `dem.py`. Here's a rough comparison.

| grid_n | 5 | 32 | 64 | 128 |
| --- | --- | --- | --- | --- |
| wall clock time (on gpu) | 34.25s | 23.61s | **21.28s** | 21.69s |

I guess even if the number of collision pairs need to be processed in `grid_n=64` is larger than those in `grid_n=128`, it takes less time to populate `particle_id`, `list_head` and `list_tail`. The take-away is that the grid_n is not the larger the better. 