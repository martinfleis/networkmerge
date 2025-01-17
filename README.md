
# Prerequisites

To run the code in this repo, you need to have a working Python
installation with the following packages installed (with pip in this
case):

``` bash
pip install matplotlib pandas shapely geopandas osmnx networkx scipy folium mapclassify
```

# networkmerge

A minimal example dataset was created with the ATIP tool. The example
dataset can be found in the `data` folder.

To read-in the data into Python we used the following:

``` python
import matplotlib.pyplot as plt
import pandas as pd
import math
from typing import List, Tuple
from shapely.geometry import LineString
import geopandas as gpd
import osmnx as ox
import numpy as np
from scipy.spatial.distance import pdist, squareform
from shapely.geometry import Point
import networkx as nx
from pyproj import CRS
import folium
import os

def calculate_total_length(gdf, crs="EPSG:32630"):
    # Copy the GeoDataFrame
    gdf_projected = gdf.copy()

    # Change the CRS to a UTM zone for more accurate length calculation
    gdf_projected = gdf_projected.to_crs(crs)

    # Calculate the length of each line
    gdf_projected["length"] = gdf_projected.length

    # Calculate the total length
    total_length = gdf_projected["length"].sum()

    return total_length

# Download Leeds Road Network data from OSM
# Define the point and distance
point = (55.952227 , -3.1959271)
distance = 1300  # in meters

#########################################################################
#############function to plot GeoDataFrame with index label##############
#########################################################################
def plot_geodataframe_with_labels(gdf, gdf_name):

    # Create a new figure
    fig, ax = plt.subplots(figsize=(10, 10))

    # Plot the GeoDataFrame
    gdf.plot(ax=ax)

    # Add labels for each line with its index
    for x, y, label in zip(gdf.geometry.centroid.x, gdf.geometry.centroid.y, gdf.index):
        ax.text(x, y, str(label), fontsize=12)
    plt.savefig(f"pics/{gdf_name}.jpg")
    # Display the plot
    plt.show()

#########################################################################
##### Download the road network data for the area around the point ######
#########################################################################
# Only download if the data/edges.shp file does not exist:
if not os.path.exists("data/edges.shp"):
    # Download the road network data
    graph = ox.graph_from_point(point, dist=distance, network_type='all')

    # Save the road network as a shapefile
    ox.save_graph_shapefile(graph, filepath=r'data/')

# Read in data from CycleStreets + overline
gdf = gpd.read_file("data/rnet_princes_street.geojson")
gdf = gdf.rename(columns={'commute_fastest_bicycle_go_dutch': 'value'})

# Use the function to calculate the total length
total_length = calculate_total_length(gdf)
total_length
```

    49371.10154507467

``` python
# TODO: check total length after network simplification
total_distance_traveled = round(sum(gdf['value'] * gdf['length']))

gdf_road = gpd.read_file("data/edges.shp")
gdf.head()
```

       value  ...                                           geometry
    0    0.0  ...  LINESTRING (-3.20572 55.94693, -3.20568 55.94694)
    1    0.0  ...  LINESTRING (-3.19433 55.95394, -3.19430 55.95388)
    2    0.0  ...  LINESTRING (-3.19619 55.95291, -3.19608 55.952...
    3    0.0  ...  LINESTRING (-3.20238 55.95174, -3.20230 55.95162)
    4    0.0  ...  LINESTRING (-3.19457 55.95517, -3.19469 55.955...

    [5 rows x 4 columns]

``` python
# Create the plot
fig, ax = plt.subplots(figsize=(10, 10))

# Plot the Shapefile data
gdf_road.plot(ax=ax, color='blue')

# Plot the GeoJSON data
gdf.plot(ax=ax, color='red')

plt.savefig(f"pics/gdf_road.jpg")
plt.show()
```

![](README_files/figure-commonmark/read-in%20data-1.png)

``` python
gdf.explore()
```

    <folium.folium.Map object at 0x7f4a7199f2b0>

``` python

# # Try to Create interactive map with 2 layers: gdf and gdf_road:
# Create a map with OSMnx and add graph
m = ox.plot_graph_folium(graph, popup_attribute='name', edge_width=2)

# Create GeoJson objects with specified colors
gdf_layer = folium.GeoJson(gdf, style_function=lambda feature: {'color': 'red'})
gdf_road_layer = folium.GeoJson(gdf_road, style_function=lambda feature: {'color': 'blue'})

# Add GeoJson objects as layers to the map
gdf_layer.add_to(m)
gdf_road_layer.add_to(m)

# Add a layer control panel to the map
folium.LayerControl().add_to(m)

# Save the map as an HTML file
m.save('data/leeds.html')

# View the map
m
```

``` r
gdf = sf::read_sf("data/rnet_princes_street.geojson")
gdf_road = sf::read_sf("data/edges.shp")
library(tmap)
tmap_mode("view")
m = tm_shape(gdf_road) +
  tm_lines("blue", lwd = 9) +
  tm_shape(gdf) +
  tm_lines("red", lwd = 3) 
dir.create("maps")
tmap_save(m, "maps/edinburgh.html")
browseURL("maps/edinburgh.html")
```

``` python
#########################################################################
############### Find matching lines from Leeds road data ################
#########################################################################

# Define the buffer size
buffer_size = 0.00002

# Create a buffer around the geometries in gdf
gdf_buffered = gdf.copy()
gdf_buffered.geometry = gdf.geometry.buffer(buffer_size)

# Initialize an empty DataFrame to store the matching lines
matching_lines_large_intersection = gpd.GeoDataFrame(columns=gdf_road.columns)

# Define the intersection length threshold
intersection_length_threshold = 0.0001

# Iterate over the buffered geometries in the GeoJSON GeoDataFrame
for geojson_line in gdf_buffered.geometry:
    # Iterate over the geometries in the shapefile GeoDataFrame
    for _, edge_row in gdf_road.iterrows():
        shapefile_line = edge_row.geometry
        
        # Calculate the intersection of the GeoJSON line and the shapefile line
        intersection = geojson_line.intersection(shapefile_line)
        
        # If the length of the intersection exceeds the threshold, add the shapefile line to the matching lines DataFrame
        if intersection.length > intersection_length_threshold:
            matching_lines_large_intersection = pd.concat([matching_lines_large_intersection, pd.DataFrame(edge_row).T])

# Plot gdf_buffered (in blue) and matching_lines_buffered (in green) on the same plot
fig, ax = plt.subplots(figsize=(10, 10))

gdf_buffered.boundary.plot(ax=ax, color='blue', label='minimal-input.geojson (Buffered)')
matching_lines_large_intersection.plot(ax=ax, color='green', label='matching_lines_buffered')
gdf.plot(ax=ax, color='black')
ax.set_title('Comparison of minimal-input.geojson (Buffered) and matching_lines_buffered')
ax.legend()
plt.savefig(f"pics/matching_lines.jpg")
plt.show()
matching_lines_large_intersection.to_file("data/gdf_matching_lines.geojson", driver='GeoJSON')
gdf_matching_lines = gpd.read_file("data/gdf_matching_lines.geojson")
plot_geodataframe_with_labels(gdf_matching_lines, gdf_name ='gdf_matching_lines')

gdf_matching_lines.explore() 
gdf.explore() 
gdf_road.explore() 
#########################################################################
################### Function to split line by angle #####################
#########################################################################
def split_line_at_angles_modified(line, value, threshold=30):
    if isinstance(line, LineString):
        coords = np.array(line.coords)
    elif isinstance(line, MultiLineString):
        # Handle each LineString in the MultiLineString separately
        return [seg for geom in line.geoms for seg in split_line_at_angles_modified(geom, value, threshold)]
    else:
        raise ValueError(f"Unexpected geometry type: {type(line)}")

    # Compute the direction of each vector
    vectors = np.diff(coords, axis=0)
    directions = np.arctan2(vectors[:,1], vectors[:,0])

    # Compute the angle between each pair of vectors
    angles = np.diff(directions)
    
    # Convert the angles to degrees and take absolute values
    angles = np.abs(np.degrees(angles))

    # Identify the indices where the angle exceeds the threshold
    split_indices = np.where(angles > threshold)[0] + 1

    # Split the line at the points corresponding to the split indices
    segments = []
    last_index = 0
    for index in split_indices:
        segment = LineString(coords[last_index:index+1])
        segments.append((segment, value))
        last_index = index

    # Include all remaining parts of the line after the last split point
    segment = LineString(coords[last_index:])
    segments.append((segment, value))

    return segments

# Apply the function to each line in the gdf with threshold=30
gdf_split_list = gdf.apply(lambda row: split_line_at_angles(row['geometry'], row['value'], threshold=30), axis=1)

# Convert the list of tuples into a DataFrame
gdf_split = pd.DataFrame([t for sublist in gdf_split_list for t in sublist], columns=['geometry', 'value'])

# Convert the DataFrame to a GeoDataFrame
gdf_split = gpd.GeoDataFrame(gdf_split, geometry='geometry')

# Set the CRS of gdf_split to match that of gdf
gdf_split.crs = gdf.crs

plot_geodataframe_with_labels(gdf_split, gdf_name ='gdf_split')
gdf_split.explore()

gdf_split.to_file("data/gdf_split.geojson", driver='GeoJSON')  
# Use the function to calculate the total length
calculate_total_length(gdf_split)

#########################################################################
### Find the nearest line in the .shp for a given line in the GeoJSON ###
#########################################################################
# Convert buffer size to degrees
buffer_size = 0.00001

# Create buffer around each road
gdf_matching_lines['buffer'] = gdf_matching_lines['geometry'].buffer(buffer_size)

# Compute centroids of lines in gdf_split
gdf_split['centroid'] = gdf_split['geometry'].centroid

# Create a new GeoDataFrame for buffers
gdf_buffer = gpd.GeoDataFrame(gdf_matching_lines, geometry='buffer')
gdf_buffer.explore()
# Set up the plot
fig, ax = plt.subplots(figsize=(10, 10))

# Plot buffers
gdf_buffer.plot(ax=ax, color='blue', alpha=0.5, edgecolor='k')

# Plot centroids
gdf_split['centroid'].plot(ax=ax, markersize=5, color='red', alpha=0.5, marker='o')

# Set plot title
ax.set_title('Buffered Roads and Centroids of Line Segments')
plt.savefig(f"pics/Buffered Roads and Centroids of Line Segments.jpg")
plt.show()

# Copy the columns from gdf_matching_lines to gdf_split
for col in gdf_matching_lines.columns:
    if col not in gdf_split.columns and col != 'value':
        gdf_split[col] = None

# Iterate over each row in gdf_split
for i, row in gdf_split.iterrows():
    # Check if the centroid of the line falls within any buffer in gdf_matching_lines
    for j, road in gdf_matching_lines.iterrows():
        if row['centroid'].within(road['buffer']):
            # If it does, copy the attributes from gdf_matching_lines to gdf_split
            for col in gdf_matching_lines.columns:
                if col != 'value':
                    gdf_split.at[i, col] = gdf_matching_lines.at[j, col]
            break

gdf_split = gdf_split[['value', 'name', 'highway','geometry']]
gdf_split.head()
gdf_split['name'] = gdf_split['name'].apply(lambda x: ', '.join(map(str, x)) if isinstance(x, list) else x)
gdf_split['highway'] = gdf_split['highway'].apply(lambda x: ', '.join(map(str, x)) if isinstance(x, list) else x)

gdf_split.to_file("data/gdf_att.geojson", driver='GeoJSON')    
gdf_att = gpd.read_file("data/gdf_att.geojson")
gdf_att.explore()


from shapely.geometry import LineString
import numpy as np
from numpy.linalg import norm

def filter_parallel_lines_concat(gdf, name, angle_tolerance=25):
    # Filter the GeoDataFrame by the 'name' column
    filtered_gdf = gdf[gdf['name'] == name]

    # Create a list to store the parallel lines
    parallel_lines = []

    # Iterate through each pair of lines
    for i in range(len(filtered_gdf)):
        for j in range(i+1, len(filtered_gdf)):
            # Get the lines
            line1 = list(filtered_gdf.iloc[i].geometry.coords)
            line2 = list(filtered_gdf.iloc[j].geometry.coords)

            # Calculate the angle between the lines
            angle = calculate_angle(line1, line2)

            # If the angle is close to 0 or 180 degrees, add the lines to the list
            if abs(angle) <= angle_tolerance or abs(angle - 180) <= angle_tolerance:
                parallel_lines.append(filtered_gdf.iloc[i:i+1])
                parallel_lines.append(filtered_gdf.iloc[j:j+1])

    # Combine the lines into a new GeoDataFrame using pd.concat
    parallel_gdf = pd.concat(parallel_lines).drop_duplicates()

    return parallel_gdf

# Use the function to filter out parallel lines with the name 'Princes Street'
parallel_gdf_concat = filter_parallel_lines_concat(gdf_att, 'Princes Street')
parallel_gdf_concat.explore()
gdf.explore()

#########################################################################
######### Find start and end point to define the flow direction #########
#########################################################################
gdf_att = gdf_att.set_crs("EPSG:4326")
gdf_split["start_point"] = gdf_split.geometry.apply(lambda line: line.coords[0])
gdf_split["end_point"] = gdf_split.geometry.apply(lambda line: line.coords[-1])

# Create a list of all start and end points
points = gdf_split["start_point"].tolist() + gdf_split["end_point"].tolist()

# Create a function to calculate the distance between two points
def calculate_distance(point1, point2):
    return Point(point1).distance(Point(point2))

# Calculate the pairwise distances between all points
distances = pdist(points, calculate_distance)

# Convert the distances to a square matrix
dist_matrix = squareform(distances)

# Find the indices of the two points that are farthest apart
farthest_points = np.unravel_index(dist_matrix.argmax(), dist_matrix.shape)

# Get the coordinates of the two points that are farthest apart
start_point = points[farthest_points[0]]
end_point = points[farthest_points[1]]
start_point, end_point

#########################################################################
############ Find all paths using start_point and end_point #############
#########################################################################

def generate_all_paths(G, start_point, end_point, max_paths=100):
    # Initialize a counter for the number of paths
    path_count = 0
    
    # Use DFS to generate all paths
    for path in nx.all_simple_paths(G, start_point, end_point):
        yield path
        
        # Increment the path counter
        path_count += 1
        
        # If we've generated the maximum number of paths, stop
        if path_count >= max_paths:
            return

# Create a new NetworkX graph
G = nx.Graph()

# Add each line in the GeoDataFrame as an edge in the graph
for i, row in gdf_split.iterrows():
    # We'll use the length of the line as the weight
    weight = row.geometry.length
    G.add_edge(row.start_point, row.end_point, weight=weight)

# Find the shortest path from the start point to the end point
try:
    shortest_path = nx.shortest_path(G, start_point, end_point, weight='weight')
except nx.NetworkXNoPath:
    shortest_path = None

shortest_path

# Create a list to store all paths
all_paths = []

# Generate all paths
for path in generate_all_paths(G, start_point, end_point):
    all_paths.append(path)

len(all_paths), all_paths

def find_line_by_points(gdf, start_point, end_point, tolerance=1e-6):
    # Convert the start and end points to Point objects
    start_point = Point(start_point)
    end_point = Point(end_point)

    # Iterate over the lines in the GeoDataFrame
    for _, line in gdf.iterrows():
        # If the start and end points of the line are within a small distance of the given start and end points, return the line
        if line.geometry.distance(start_point) < tolerance and line.geometry.distance(end_point) < tolerance:
            return line

    # If no line was found, return None
    return None

def plot_paths(gdf, paths, shortest_path,gdf_name):
    # Create a new figure
    fig, ax = plt.subplots(figsize=(10, 10))

    # Plot each path
    for i, path in enumerate(paths):
        for start_point, end_point in zip(path[:-1], path[1:]):
            # Find the line in the GeoDataFrame that corresponds to this edge
            line = find_line_by_points(gdf, start_point, end_point)
            
            # If this line is in the shortest path, plot it in red, otherwise plot it in blue
            color = 'red' if path == shortest_path else 'blue'
            
            # Plot the line
            gpd.GeoSeries(line.geometry).plot(ax=ax, color=color)
            
            # Add a label with the line's index
            x, y = line.geometry.centroid.x, line.geometry.centroid.y
            ax.text(x, y, str(line.name), fontsize=12)
    plt.savefig(f"pics/gdf_{gdf_name}.jpg")        
    # Display the plot
    plt.show()

# Use the function to plot the paths
plot_paths(gdf_split, all_paths, shortest_path,gdf_name ='all_paths')

#########################################################################
######################## Find division subpaths #########################
#########################################################################

def find_division_subpaths(gdf, paths):
    division_subpaths = []
    
    # Initialize a set with the waypoints in the first path
    common_waypoints = set(paths[0])
    
    # Iterate over the rest of the paths
    for path in paths[1:]:
        # Update the set of common waypoints to be the intersection of the current set of common waypoints and the waypoints in this path
        common_waypoints &= set(path)
    
    # Find the last common waypoint among the paths (the division waypoint)
    division_waypoint = None
    for waypoint in paths[0]:
        if waypoint in common_waypoints:
            division_waypoint = waypoint
        else:
            break
    
    for path in paths:
        # Initialize the division subpath for this path
        division_subpath = []
        
        # Find the index of the division waypoint in this path
        division_waypoint_index = path.index(division_waypoint)

        # Iterate over the waypoints in the path from the division waypoint to the end waypoint
        for start_point, end_point in zip(path[division_waypoint_index:], path[division_waypoint_index+1:]):
            # Find the line in the GeoDataFrame that corresponds to this edge
            line = find_line_by_points(gdf, start_point, end_point)
            
            # Add the index of the line to the division subpath
            division_subpath.append(line.name)
        
        # Add the division subpath to the list of division subpaths
        division_subpaths.append(division_subpath)
    
    # Initialize a set with the lines in the first division subpath
    common_lines = set(division_subpaths[0])
    
    # Iterate over the rest of the division subpaths
    for division_subpath in division_subpaths[1:]:
        # Update the set of common lines to be the intersection of the current set of common lines and the lines in this division subpath
        common_lines &= set(division_subpath)

    # Remove the common lines from each division subpath
    for division_subpath in division_subpaths:
        for line in common_lines:
            if line in division_subpath:
                division_subpath.remove(line)

    return division_subpaths

# Use the function to find the division subpaths
division_subpaths = find_division_subpaths(gdf_split, all_paths)
division_subpaths

def plot_lines_by_indices(gdf, line_indices_lists, colors, gdf_name):
    # Create a new figure
    fig, ax = plt.subplots(figsize=(10, 10))
    
    # Plot each line
    for line_indices, color in zip(line_indices_lists, colors):
        for line_index in line_indices:
            # Get the line from the GeoDataFrame
            line = gdf.loc[line_index]
            
            # Plot the line
            gpd.GeoSeries(line.geometry).plot(ax=ax, color=color)
            
            # Add a label with the line's index
            x, y = line.geometry.centroid.x, line.geometry.centroid.y
            ax.text(x, y, str(line.name), fontsize=12)
    plt.savefig(f"pics/gdf_{gdf_name}.jpg")  
    # Display the plot
    plt.show()

# Use the function to plot the division subpaths
plot_lines_by_indices(gdf_split, division_subpaths, ['blue', 'red'], gdf_name ='division_subpaths')

#########################################################################
################ Simplify the road network by road type #################
#########################################################################
def simplify_subpaths(gdf, subpaths, road_type ='footway'):
    # Create a copy of the GeoDataFrame to avoid modifying the original data
    gdf = gdf.copy()

    # Find the subpaths that should be removed
    removed_subpaths = [subpath for subpath in subpaths if all(gdf.loc[line_index, 'highway'] == 'footway' for line_index in subpath)]

    # Calculate the mean 'value' of the lines in the removed subpaths, if any
    mean_value = 0
    if removed_subpaths:
        # Exclude NaNs from mean calculation
        mean_value = np.nanmean([gdf.loc[line_index, 'value'] for subpath in removed_subpaths for line_index in subpath])

    # Remove the removed subpaths from the division subpaths
    for removed_subpath in removed_subpaths:
        subpaths.remove(removed_subpath)

    # Add the mean value to the 'value' of the lines in the remaining subpaths
    for subpath in subpaths:
        for line_index in subpath:
            if line_index in gdf.index:
                if np.isnan(gdf.loc[line_index, 'value']):
                    gdf.loc[line_index, 'value'] = mean_value
                else:
                    gdf.loc[line_index, 'value'] += mean_value

    # Remove the footway lines from the DataFrame
    gdf = gdf[gdf['highway'] != road_type]

    return subpaths, gdf

# Use the function to simplify the division subpaths
division_subpaths, gdf_split_modified = simplify_subpaths(gdf_split, division_subpaths, road_type ='footway')

# Show the modified GeoDataFrame
plot_geodataframe_with_labels(gdf_split_modified, gdf_name ='gdf_split_modified')


#Following function may be useful in future but not used now
#########################################################################
#dissolve/merge lines that share the same 'highway' value, but only if they also share the same 'value'#
#########################################################################

# Dissolving/merging lines based on 'highway' and 'value'
gdf_att_dissolved = gdf_att.dissolve(by=['highway', 'value'])

# Resetting the index to have 'highway' and 'value' as normal columns
gdf_att_dissolved.reset_index(inplace=True)


plot_geodataframe_with_labels(gdf_att_dissolved)


#--------------------------------------------------------------#
#-----Function to find the lines have same start_end point-----#
#--------------------------------------------------------------#
def check_same_start_end(gdf, line1, line2):
    """
    Check if two lines have the same start and end points.
    
    """
    line1_start_end = (gdf.iloc[line1].geometry.coords[0], gdf.iloc[line1].geometry.coords[-1])
    line2_start_end = (gdf.iloc[line2].geometry.coords[0], gdf.iloc[line2].geometry.coords[-1])

    same_start = line1_start_end[0] == line2_start_end[0]
    same_end = line1_start_end[1] == line2_start_end[1]

    return same_start, same_end

check_same_start_end(geo_data, 0, 1)


# Merge lines 0, 1, 2, 3, 4
merged_line = LineString([pt for line in gdf_att.iloc[0:5].geometry for pt in line.coords])

# Calculate the total value of the merged line (which is 1 as per the user)
total_value = 1

# Add the total value to lines 5 and 6, divided equally
gdf_att.loc[[5, 6], 'value'] += total_value / 2

# Retain only lines 7, 5, 6, 8 in the final output
gdf_final = gdf_att.loc[[7, 5, 6, 8]]

# Plot the final GeoDataFrame
plot_geodataframe_with_labels(gdf_final)

#--------------------------------------------------------------#
#-----find_connected_lines-----#
#--------------------------------------------------------------#
def find_connected_lines(gdf):
    # Create a dictionary where the keys will be points (start or end points of the lines),
    # and the values will be lists of the indices of the lines with those points
    points_dict = defaultdict(list)

    # Iterate over the rows of the GeoDataFrame
    for index, row in gdf.iterrows():
        # Get the line (geometry)
        line = row.geometry

        # Get the start and end points of the line
        start_point = line.coords[0]
        end_point = line.coords[-1]

        # Add the index of the line to the lists of lines with these start and end points
        points_dict[start_point].append(index)
        points_dict[end_point].append(index)

    # Find the sets of connected lines
    connected_lines = []
    for indices in points_dict.values():
        if len(indices) > 1:
            # If there are multiple lines with this point (start or end point),
            # they are connected lines
            connected_lines.append(set(indices))

    return connected_lines

# Find the sets of connected lines in gdf_att
connected_lines = find_connected_lines(gdf_split)
connected_lines
```
