# MapKW
### *Route Optimization Project for the Region of Waterloo (KW Area)*

## Introduction

### Physical Geography to Mathematical Graphs
When navigating a city using commercial map applications, we experience a continusous lankscape of asphalt, trafficlights and street signs. However, computers cannot reason about physical space directly. To compute optimal routes, we must translate the physical world into a discrete mathematical structure known as a directed multigraph 'G' is comprised of Edges 'E' and Vertices 'V', where: 

$G=(V,E)$

- Vertices or Nodes (V): In our spatial network, every intersection or dead-end is represented as a Node. Each node possesses a precise spatial coordinate (latitude & longitude) allowing us to calculate exact geographic distances. 
- Edges (E): Every street segment conneccting two intersections is represented as an Edge. Since urban environments contain one-way streets, divided highways, and turning restrictions, these edges are directed. 

#### Why a MultiDiGraph? 
A standard graph allows only a single edge between any two nodes. Howeer, real-world street topology often features parallel edges, like two seperate physical lanes or bidirectional paths connecting the exact same two intersections, or muiltiple road segements linking adjacent blocks. A MultiDiGraph allows multiple directed edges between the same pari of nodes, ensuring we do not lose critical topological detail. 

### The Role of OpenStreetMap and OSMnx
Instead of manually plotting intersections and roads, we leverage OpenStreetMap (OSM), a crowdsourced global map database. OSMnx is a powerful Python library built on top of NetworkX that bridges raw OSM geographic data and graph theory algorithms. 
With OSMnx,we can:

- Query Spatial Boundaries: Download street networks automatically by specifying a place name (eg. Kitchener Go or Wilfrid Laurier University)
- Project Coordinate Systems: Convert lat/long angles (measured in degrees on a 3D sphere) into a projected Cartesian coordinate system (measured in meters on a 2D plane), which is mathematically required to compute accurate physical distances and travel times. 
- Handle Graph Extraction: Filter networks by infrastructure type (eg. drivable roads vs. walking trails) using customized network types (eg. network_type = 'drive)

### Why this matters to project MapKW
Understanding graph theory foundations ensures we do not treat routing as a black box. When the code computes the fastest path between Wilfrid Laurier University and Conestoga Mall, it isn't querying a remote server, it is evaluating mathematical weights across custom built topological grid stored entirely in the notebook's memory. 

### Graph Simplification, Speed Imputation & Travel-Time Weights
#### 1. The reality of raw OpenStreetMap Data
When OSMnx downloads a raw street network from OpenStreetMap, the resulting graph is topologically "messy".

- Redundant Geometry Nodes: Curved roads and long straightways are broken down into hundreds of tiny, unncessary intermediate shape nodes conneccted by micro-edges. These aren't intersections: they are just coordinates used to draw a curve on a map. 
- Cmplex Multi-Node Interactions: A single physical intersection (like a complex multi-lane multi-street junction) is often mapped in OSM as a cluster of multiple connected nodes rather than a single clan topological vertex. 

If we tried to run pathfinding algorithms on an unsimplified graph, our algorithms would waste massive amounts of compute time stepping hundreds of redundant nodes along a single straight road. 

#### 2. Graph simplification and Intersection consolidation
To fix this, two vital transformations are applied:
- "ox.simplify_graph(G)" : This function strips away all the intermediate shape nodes along straight or curved road segments, merging them into single, continuous directed edges between true intersections. 
- "ox.sonsolidate_intersections(G)" : This function idnetifies clusters of nodes that represent a single physical intersection (eg. large mulit-lane roundabout) and merges them into a single unified node. 

#### 3. Engineering Egde Weights: Distance vs. Travel-Time
In a basic graph, edges are often weighted purely by physical length ($d_e$, in meters). However, a 5-Km drive on a 90Km/h regionnal highway is much faster than a 3-Km drive through a residential 40Km/h zone with stop signs. 

To make the routing engine mroe realistic, we must compute travel time weights ($w_e$) for every edge, like so:

- Imputing missing speeds: Many secondary and residential streets in OSm lack explicit $maxspeed$ tags. We use $ox.add\_edge\_speeds()$ to automatically assign realistic default speed limits based on the road's highway classification (residential, secondary etc.)
- Calculating Travel Time: We use $ox.add\_edge\_travel\_times()$ to convert physical distances and posted speed limits into expected travel times in seconds, using: 
$w_e = {length_e}/{speed_e}$


## Search Algorithms 

#### I. Uninformed vs. Heuristic Search (Dijkstra vs. A*) 

After obrtaining a clean, projected, travel-time-weighted mulidigraph for KW, the next objective is to find the shortest path between an Origin Node ($u$) and a Destination Node ($v$). We begin by comparing an uninformed search to a heuristic guided search. 

1. Dijkstra's Algorithm: Uniform Cost Search
- Developed by Edsger W. Dijkstra in 1959
- Explores outward from the Origin Node ($u$) uniformly in all directions expanding the lowset-cost path tree step until the Destination Node ($v$) is reached. 
- At any Node ($n$), Dijkstra's Algorithm evalues a cost function:
$g(n)$ = *exact cummulative travel time from origin $(u)$.*
- Since Dijkstra has no concpt of where the target is located, it wastes immense computational effot exploring residential cul-de-sacs and directions away from the Destination ($v$). On a regional graph with 9000+ nodes, like here ($n)=9442$, Dijkstra's algorithm might explore thousands of irrelevant intersecctions before bumping into the target. 

2. A* Search: Heuristic-Guided Search
- A* Search direcctly build upon Dijkstra's algorithm by introducing a Heuristic function $h(n)$ to "steer" the search intelligently toward the destination. 
- Instead of exploring blindly in a circle, A* prioritizes nodes that are not only closer to the origin, but also estimated to be closer to the destination. 
- At any Node ($n$), A* Search evalues a cost function:
$f(n) = g(n) + h(n)$

    where, 
    - $g(n)$ = Exact travel cost from the Origin $(u)$ to the Present Node $(n)$.
    - $h(n)$ = Heuristic estimate of the cost from the Present Node $(n)$ to the Destination $(v)$.
- The Heuristic Estimate used here is the *Haversine Distance*, since we're working with spatial road networks. 

- The *Haversine Distance* or *Straight-Line Haversine Distance* is a measure of the straight-line travel time at maximum speed from Node $(n)$ to the Destination $(v)$, divided by top speed. Because a straight line is always the shortest possible physical distance between two points, this Heuristic is admissible, meaning it will never overestimate the true cost, and thus guarantees that A* will findthe mathematically optimal path while exploring a fraction of the nodes that Dijkstra touches. 


#### II. Advanced Traversal: Bidirectional Dijkstra Search vs Breadth-First Search (BFS)

1. Bidirectional Dijkstra - Meeting in the Middle

- Standard Dijkstra and A* search expansion works unidirectionally, starting outward from a single source until they hit the target. Although effecctive, searching outward in a full radial wave is inefficient. 
The Bidirectional Dijkstra Search solves this by running two simultaneous searches:

    1. A forward search originating from the Origin Node $(u)$ or the Source $(S)$
    2. A backward search originating from the Destination Node $(v)$ or the Target $(T)$
- If a unidirectional search explores a Radius $(R)$, its search space grows roughly proportional to $O(R^2)$. 
- By splitting the search into two frontiers of Radius' $(R/2)$, the combined search space becomes proportional to $2\cdot O((R/2)^2)$.
    - Thus, drastically reducing the number of nodes evaluated and cutting execution latency roughly in half for large regional graphs. 


2. Breadth-First Search (BFS) 

- The *Breadth-First Search* or *BFS* is a fundamental unweighted traversal algorithm. 
- It explores the grah level-by-level, treating every edge as having an equal weight of $1$ (effectively counting Hop Count or intersection turns rather than physical distance or travel time).
- BFS finds the path with the fewest number of street segments, completely ignoring whether those segments are high-speed expressways (100Km/h) or congested residential side streets with stop signs.

    - Thus, we use this a Control Group or Baseline. 
    - By comparing the travel-time of a BFS Hop-Count against the Weighted Dijkstra or A* Optimal Route, we can mathematically prove that *time travel penalty* of ignoring road weights. 
    - This demonstrates why Advanced Cost Engineering, weighting edges by real travel time $(w_e = d_e/v_e)$, is indispensable for real-world municipal navigation engines (like MapKW). 

### Combinatorial Path Deviation (Yen's Algorithm)

#### The Limitations of Single-Path Algorithms 
With Dijkstra, A* and Bidirectional search, the fundamental constraint is that they return only one path from Origin $(u)$ to Destination $(v)$. In real world navigation, one route is seldom enough, alternate routes are necessary in case of closures, accidents, preferences etc. 

#### Why Naive Approaches Fail (The Loop Problem)
One might wonder, *Why not just find the shortest path, delete and edge, and compute Dijkstra again?* 
Blindly deleting edges off the optimal path and re-running Dijkstra would result in a path that is bizzare and highly ineffficient, or even in disconnected graph segments. Furhtermore, the alternate route must be structurally distinct, meaning it shouldn't just be the optimal highway route with one single residnetial turn altered. It should represent meaningful alternative corridors across the city. 

#### How Yen's K-Shortest Path Algorithm Works
- Developed by Jin Y. Ken in 1971.
- Gold standard for finding the top $K$ shortest loopless paths in a graph. 
- For *MapKW* we target $K = 4$ (1 optimal route, and 3 distint alternatives)

Step-by-step logic of Yen's K-Shortest Path Algorithm: 

1. Find the 1st Shortest Path $(Path_1)$: 

    - Run a standard shortest-path algorithm (like Dijkstra or A*) from Source $(S)$ to Target $(T)$. This becomes the Optimal Route $(Route A)$
2. Iterative Deviation $(k = 2$ to $K)$:

    - The algorithm treats each node along th previous shortest path as a *spur node*.
    - It temporarily *spurs off* the established path, blocking specific edges or nodes to force the search engine to find a new route that diverges from that point onward.
    - This generates a candidate pool of valid alternative paths. 
3. Selection & Sorting:

    - The algorithm evaluates all candidate paths in the pool, calculates their total travel-time weights, and selects the absolute next-shortest loopless path to become $RouteB$, then $RouteC$ and so on. 
4. Guaranteed Looplessness: 

    - Every path generated is strictly loopless, no node is visited twice in a single route, and ordered strictly from fastest to slowest. 

#### Comparitive Metrics Aggregator

When Yen's algorithm ouputs $K = 4$ routes, we get four lists of node IDs and four raw travel-time values. However, raw node IDs and seconds are not enough for an interactive routing dashboard. To evaluate and present trade-offs effecctively to a user, we need a Comparitive Metrics Aggregator to evaluate three core operational dimensions for every route to produce holistic routing alternatives. 

1. Total Distance $(km)$: Summing $(\sum)$ the phsyical geometric lengths of all the edge segments in the path. While travel-time is the crown metric, total distance helps drivers understand the physical footprint of taking an alternate corridor. A real life use case could be when drivers want to add the least amount of milage to their drive. 
2. Estiamted Travel Time $(min/sec)$: The primary cost metric $(w_e = d_e/v_e)$ needs to be translated into minutes and secodns for human readability. 
3. Node Turn Complexity (Node Count): Countign the total number of intersection nodes traversed. In graph theory ad urban logistics, a higher node count relative to distance often serves as a proxy for turn complexity, traffic lights and stop signs, helping drivers spot routes that might looks fast on paper but involve stressful intersection maneuvers. An all way stop sign near a school zone at noon could be magnitudes slower than a 2-lane roundabout. 

#### Thus, a Comparitive Metrics Aggregator is created using Pandas. 
Instead of manually calcutaing these values for each route, writing a generator function that loops through the $K\_routes$ output that extracts graph attributes using $NetworkX$ and packs them into a clean, tabular $pd.DataFrame$ is much faster. This is how it works: 
1. Distance Extraction:
- We iterate through consecutive pairs of nodes in each path *$(zip(path[:-1], path[1:]))$*, look up the edge data in $G\_weighted$, and sum the $length$ attribute (converted from Km to m). 

2. Travel Time Formatting:
- Take the cumulative path weight $(sec)$ and divide by 60 to get it in $minutes$. 
3. Rank Ordering:
- The aggregator automatically labels the routes as $Route A$ (Optimal Route), $RouteB$ (Atl1), $RouteC$ (Alt2), and $RouteD$ (Alt3), establishing a clean comparison matrix ready for dashboard display. 

### Making MapKW Production worthy

The first 75% of the project was setting up the backgorund calculations and engine to compute routes. Now we need to make it relatable to the KW region. Upto this point the test coordinates have been hardcoded for Laurier and Conestoga Mall, however an interactive routing dashboard requires flexibility. People need to be able to type-in landmarks or click direcctly on an interactive map. 

- People provide inputs either as friendly text strings or click on places, not as lat/lon coordinates.
- The MapKW routing engine runs on $G\_weighted$, which is projected into local UTM meters (EPSG:4326), and consists of discrete integer intersecctions node IDs. So even if someone knew the exact lat/lon coordinates for a location, a raw lat/lon pais does not exist as an exact node in the graph. 

To bridge this gap, we use *$Spatial\ Coordinate\ Snapping$* powered by OSMnx's $ox.nearest\_node()$ function which utilizes spatial indexing R-Trees under the hood for fast lookups. 

1. Take the user's input
2. Project them dynamically into the graph's exact UTM coordinate system using GeoPandas so the distance metric matches meters. 
3. Query the spatial index to find the closest valid intersecction node in $V$

To ensure that whether a user clicks  a random street corner or selects a landmark from a dropdown, the systme instantly snaps their selection to the nearest drivable intersaction, guaranteeing that the pathfinding algorithms never throw a missing-node error. 

Next, to make the dashboard user fiendly in Jupyter notebook's limited scope, we use the $ipywidgets$ library. A Dropdown Selector with a pre-loaded dictionary of iconic KW locations allows users to dynamically swap origin and destination points to view on the Folium map. 

#### Multi-Route Visualization

Visual Hierarchy in Mapping. 

When presenting multiple routing options, maps design matters a lot, if all four Routes (A, B, C & D) are rendered in the same color it would look like visual spaghetti. So a visual hierarchy is created with the Optimal Route $(RouteA)$ rendered in a high-contrast, bold primary color, here Electric Blue with a thicker weight. The alternative routes (B, C & D) are rendered in a secondary uniform color, here Soft Orange, with slightly lighter weights. This groups the alternative corridors together while keeping them distinct from the optimal route. 

Rendering Polylines and Markers in Folium 

- Node-to-Coordinate Extraction: Because our routing algorithms output sequences of Node IDs, we must map those node IDs back into geographic coordinate pairs using graph attributes. 
- Polyline Generation: These coordinate lists are passed into $folium.Polyline$ objects and added to the map layer. 
- Custom Markers and Popups: Placing start and end pins equipped with HTML popups displaying trip summaries. 

HTML Export

One of the best features of Folium is its ability to compile the entire interactive maps, with vector tiles, polylines, tooltips and layer controls, into standalone HTML file. This allows users to visit my Github and open the interactive map directly in a web browser without needing a Python kernal or Jupyter server. 
