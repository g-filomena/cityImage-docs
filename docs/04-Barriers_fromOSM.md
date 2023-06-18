## Barriers


```python
import cityImage as ci
import pandas as pd
import geopandas as gpd
from matplotlib.colors import LinearSegmentedColormap

%matplotlib inline

import warnings
warnings.simplefilter(action="ignore")
```


```python
```python
# Important: find EPSG of your case-study area
# Initialise path, names, etc.

city_name = 'Paris'
epsg = 27571
crs = 'EPSG:'+str(epsg)
place = 'Paris, Ile-de-France'
download_method = 'OSMplace'
```

## 1. Barriers Identification
Download from OSM

Choose between the following methods:

* `OSMplace`, providing an OSM place name (e.g. City).
* `polygon`, providing a Polygon (coordinates must be in units of latitude-longitude degrees).
* `distance_from_address`, providing a precise address and setting the `distance` parameter.
* `distance_from_point`, providing point coordinates (in units of latitude-longitude degrees) and setting the `distance` parameter to build the bounding box around the point.


```python
road_barriers = ci.road_barriers(place = place, download_method = download_method, epsg= epsg,
                                 include_primary = True, include_secondary = False)
water_barriers = ci.water_barriers(place = place, download_method = download_method, epsg= epsg)
railway_barriers = ci.railway_barriers(place = place, download_method = download_method, epsg= epsg)
park_barriers = ci.park_barriers(place = place, download_method = download_method, epsg= epsg, min_area = 100000)
```

    C:\Users\gabri\miniconda3\envs\testing\Lib\cityImage\barriers.py:118: FutureWarning: `unary_union` returned None due to all-None GeoSeries. In future, `unary_union` will return 'GEOMETRYCOLLECTION EMPTY' instead.
      sea = sea.unary_union
    


```python
barriers = pd.concat([road_barriers, water_barriers, railway_barriers, park_barriers], ignore_index=True)
barriers.reset_index(inplace = True, drop = True)
barriers['barrierID'] = barriers.index.astype(int)
```

### 1.1 Visualisation


```python
barriers.sort_values(by = 'barrier_type', ascending = False, inplace = True)  
colors = ['green', 'red', 'gray', 'blue']

cmap = LinearSegmentedColormap.from_list('cmap', colors, N=len(colors))
fig = ci.plot_gdf(gdf = barriers, column = 'barrier_type', black_background = False, title = city_name+': Barriers', 
                  legend = True, cmap = cmap)
```


    
![png](img/04/output_8_0.png)
    


## 2. Incorporating Barriers into the Street Network
This is an optional step that allows modelling the effect of barriers on pedestrian movement, for exmaple, in agent-based modelling, or in route-choice modelling.

### 2.1 Download the street network

Choose between the following methods:

* `OSMplace`, providing an OSM place name (e.g. City).
* `polygon`, providing a Polygon (coordinates must be in units of latitude-longitude degrees).
* `distance_from_address`, providing a precise address and setting the `distance` parameter.
* `distance_from_point`, providing point coordinates (in units of latitude-longitude degrees) and setting the `distance` parameter to build the bounding box around the point.

Downloading the graph and cleaning it (see the notebook *01-Nodes_Paths_fromOSM* for details on the cleaning process)


```python
nodes_graph, edges_graph = ci.get_network_fromOSM(place = place, download_method = download_method, network_type = 'walk',
                                                  epsg = epsg)
nodes_graph, edges_graph = ci.clean_network(nodes_graph, edges_graph, dead_ends = True, remove_islands = True,
                            self_loops = True, same_vertexes_edges = True)
```

### 2.2 Loading from local path


```python
loading_path = 'Outputs/'+city_name+'/largeNetwork/'
nodes_graph = gpd.read_file(loading_path+city_name+'_nodes.shp')
edges_graph = gpd.read_file(loading_path+city_name+'_edges.shp')

nodes_graph.index, edges_graph.index  = nodes_graph.nodeID, edges_graph.edgeID
nodes_graph.index.name, edges_graph.index.name  = None, None
```


```python
fig = ci.plot_gdf(edges_graph, black_background = False, geometry_size = 1.0, alpha = 1.0, 
                 color = 'black', title = city_name+': Street Network', figsize = (10,10))
```


    
![png](img/04/output_14_0.png)
    


### 2.3 Assigning barriers to street segments
Type of Barriers:

* *Positive barriers*, from a pedestrian perspective: Waterbodies, Parks.
* *Negative barriers*, from a pedestrian perspective: Major Roads, Railway Structures.
* *Structuring barriers* - Barriers which structure and shape the image of the city: Waterbodies, Major roads, Railways.


```python
# clipping barriers to case study area
envelope = edges_graph.unary_union.envelope
barriers_within = barriers[barriers.intersects(edges_graph.unary_union.envelope)]
```

#### Along and within Positive Barriers


```python
sindex = edges_graph.sindex
# rivers
edges_graph = ci.along_water(edges_graph, barriers_within)
# parks
edges_graph = ci.along_within_parks(edges_graph, barriers_within)
# altogheter
edges_graph['p_barr'] = edges_graph['a_rivers']+edges_graph['aw_parks']
edges_graph['p_barr'] = edges_graph.apply(lambda row: list(set(row['p_barr'])), axis = 1)
```

#### Along Negative Barriers


```python
tmp = barriers_within[barriers_within['type'].isin(['railway', 'road'])]
edges_graph['n_barr'] = edges_graph.apply(lambda row: ci.barriers_along(row['edgeID'], edges_graph, tmp, sindex,
                                            offset = 25), axis = 1)
```

#### Crossing any kind of barrier but parks - Structuring Barriers


```python
edges_graph = ci.assign_structuring_barriers(edges_graph, barriers_within)
```

### 2.4 Visualisation


```python
# positive barriers
edges_graph['p_bool'] = edges_graph.apply(lambda row: True if len(row['p_barr']) > 0 else False, axis = 1)
tmp = edges_graph[edges_graph.p_bool == True].copy()

# base map
base_map_dict = {'base_map_gdf': edges_graph, 'base_map_alpha' : 0.3, 'base_map_color' : 'black'}
ci.plot_gdf(tmp, black_background = False, figsize = (15, 15), color = 'red', title = city_name+': Streets along parks and rivers', 
              legend = False, base_map_dict**)
```


    
![png](img/04/output_24_0.png)
    



```python
# negative barriers
edges_graph['n_bool'] = edges_graph.apply(lambda row: True if len(row['n_barr']) > 0 else False, axis = 1)
tmp = edges_graph[edges_graph.n_bool == True].copy()


ci.plot_gdf(tmp, black_background = False, figsize = (15, 15), color = 'red', title = city_name+': Streets along negative barriers',
              legend = False, base_map_dict**)
```


    
![png](img/04/output_25_0.png)
    



```python
# separating barriers
tmp = edges_graph[edges_graph.sep_barr == True].copy()
ci.plot_gdf(tmp, black_background = False, figsize = (15, 15),  color = 'red', 
            title = city_name+': Streets crossing structuring barriers: water, highways, railways',
             base_map_dict**)
```


    
![png](img/04/output_26_0.png)
    


## Saving 


```python
# saving barriers_gdf
saving_path = 'Outputs/'+city_name+'/'
barriers.to_file(saving_path+city_name+"_barriers.shp", driver='ESRI Shapefile')

# converting list fields to string
to_convert = ['a_rivers', 'aw_parks','n_barr', 'p_barr']
edges_graph_string = edges_graph.copy()
for column in to_convert: 
    edges_graph_string[column] = edges_graph_string[column].astype(str)
    
edges_graph_string.to_file(saving_path+city_name+"_edges.shp", driver='ESRI Shapefile')
nodes_graph.to_file(saving_path+city_name+'_nodes.shp', driver='ESRI Shapefile')
```
