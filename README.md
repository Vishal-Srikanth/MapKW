# MapKW
### Every street is a decision.

**What if finding the fastest way across a city was actually a graph theory problem?**

It is. And that's what I wanted to explore with **MapKW**, an urban routing project built around the real road network of **Kitchener–Waterloo, Ontario**.

I used Python to turn OpenStreetMap roads into a weighted graph, put classic shortest-path algorithms to the test, generate alternative routes, and bring the results to life in an interactive Tableau Public dashboard.

<!-- Centered Dashboard Preview -->
<p align="center">
  <img src="MapKW SS1.png" alt="MapKW" width="600px">
</p>

<!-- Centered Dashboard Preview -->
<p align="center">
  <img src="MapKW SS2.png" alt="About MapKW" width="600px">
</p>


[Explore my interactive MapKW Tableau dashboard](https://public.tableau.com/app/profile/vishal.srikanth/viz/MapKW/01InteractiveMap)


*Choose an origin and destination, then explore the optimal route and its alternatives. Route times are estimates under static road conditions, not live traffic predictions.*

---

##  The idea

A road map looks like streets and intersections. To a routing algorithm, it's a **graph**:

- **Nodes** represent intersections or points in the road network.
- **Directed edges** represent permitted road connections, including one-way streets.
- **Edge weights** represent estimated travel time.
- **Paths** represent possible journeys between an origin and a destination.

Once the city becomes a graph, a simple question gets surprisingly interesting: **Which route is best, and what does “best” actually mean?**

The shortest route by distance isn't necessarily the fastest. The path with the fewest nodes isn't necessarily the quickest either. And finding one optimal route is only part of the story; sometimes you want to know what your other options look like.

##  What MapKW does

1. **Builds a real road network** using OpenStreetMap data and OSMnx.
2. **Prepares a routing graph** by projecting coordinates, consolidating nearby intersections, and estimating edge travel times.
3. **Compares routing algorithms** — Dijkstra, A*, bidirectional Dijkstra, and breadth-first search (BFS).
4. **Finds alternative routes** using Yen's K-shortest simple paths approach.
5. **Measures route characteristics** such as estimated driving time, distance, and number of path nodes.
6. **Precomputes regional journeys** and exports a Tableau-ready dataset for interactive exploration.

##  MapKW by the numbers

| Metric | Result |
|:--|--:|
| Landmarks | **47** |
| Possible ordered origin–destination pairs | **2,162** |
| Directed pairs displayed in the dashboard | **2,154** |
| Route options per displayed pair | **4** |
| Precomputed routes displayed in the dashboard | **8,616** |
| Rows in the Tableau export | **292,930** |

*The 2,162 figure is the theoretical number of ordered pairs among 47 distinct landmarks (47 × 46). The dashboard reports 2,154 directed pairs with precomputed routes.*

##  The algorithms

| Algorithm | What I used it to investigate |
|:--|:--|
| **Dijkstra's algorithm** | Find the minimum estimated travel-time path on a weighted road network. |
| **A\* search** | Use a distance-based heuristic to guide the search toward the destination. |
| **Bidirectional Dijkstra** | Explore whether searching from both ends can reduce computation time. |
| **Breadth-first search (BFS)** | Show why minimizing the number of edges is not the same as minimizing driving time. |
| **Yen's K-shortest paths** | Generate multiple loopless candidate routes instead of returning only one. |

### One route, four algorithms

For a test journey from **Wilfrid Laurier University to Conestoga Mall**, the notebook produced these estimated travel times:

| Algorithm | Estimated driving time |
|:--|--:|
| Dijkstra | 274.92 s |
| A* | 274.92 s |
| Bidirectional Dijkstra | 274.92 s |
| BFS | 301.88 s |

**The interesting part:** BFS found a path with fewer nodes, but its estimated driving time was approximately **27 seconds longer**. Fewer steps through a graph do not automatically mean a faster drive.

In a separate queue-removal experiment, A* recorded **398** removals versus **1,255** for Dijkstra, approximately **68.3% fewer** for that particular route. These counts describe that experiment's search behavior, not a universal performance guarantee. Similarly, the notebook's millisecond timings are illustrative single-run measurements rather than a comprehensive benchmark.

##  From notebook to interactive map

The Python notebook handles the computational work; **Tableau Public handles exploration**.

Rather than asking Tableau to run a shortest-path algorithm every time someone changes a filter, I precomputed candidate routes between regional landmarks and exported the results as a structured CSV.

The export includes origins, destinations, route identifiers, route order, geographic coordinates, estimated driving times, and route distances. A `Path_Order` field keeps the coordinates in the sequence needed to draw each path.

That makes it possible to switch between locations and compare candidate routes in the dashboard without executing graph algorithms in Tableau.

**[Open MapKW on Tableau Public](https://public.tableau.com/app/profile/vishal.srikanth/viz/MapKW/01InteractiveMap)**

##  Built with

**Python · OSMnx · NetworkX · GeoPandas · Pandas · Folium · ipywidgets · pyproj · Tableau Public**

- **OSMnx / OpenStreetMap:** Road-network acquisition and preparation.
- **NetworkX:** Graph representation, pathfinding, and algorithm experiments.
- **Pandas:** Route metrics and tabular exports.
- **Folium + ipywidgets:** Interactive exploration inside the notebook.
- **pyproj:** Coordinate transformation for geographic visualization.
- **Tableau Public:** Interactive map and route-selection dashboard.

##  Explore the project

The **Jupyter notebook** is the technical walkthrough: network construction, graph theory explanations, algorithm comparisons, route generation, visualizations, and findings.

The **Tableau dashboard** is the interactive end product: choose landmarks and explore the precomputed routes on a map.

The **routing CSV** is the bridge between them: a path-point dataset that Tableau can use to draw routes and display their associated metrics.

##  What the model doesn't know

MapKW is a graph-theory and routing exploration project, **not a live navigation service**.

- Travel times are estimated from road-network speeds and lengths; they do **not** incorporate live traffic, incidents, or weather.
- Landmark locations are snapped to nearby road-network nodes, so plotted access points may differ slightly from exact landmark coordinates.
- The notebook's algorithm timings come from limited test cases, so they should not be interpreted as definitive performance rankings.
- The alternative-path implementation simplifies the original multigraph for compatibility, which can affect how parallel road edges are represented.
- Route-level metrics repeat across coordinate rows in the export; aggregating those columns directly in Tableau requires care to avoid double-counting.

### What's next?

I'd like to expand the experiments to more origin–destination pairs, use repeated benchmark runs, strengthen route-data validation, and explore how routing changes when distance and travel time are treated as competing objectives.

##  Data, research & attribution

Road-network data comes from **[OpenStreetMap](https://www.openstreetmap.org/)**, processed with **[OSMnx](https://osmnx.readthedocs.io/)**. Map tiles and underlying geographic data retain their respective provider attributions.

**OSMnx reference:**

Boeing, G. (2025). [Modeling and Analyzing Urban Networks and Amenities with OSMnx](https://doi.org/10.1111/gean.70009). *Geographical Analysis*, **57**(4), 567–577. https://doi.org/10.1111/gean.70009

---


[Explore the live dashboard](https://public.tableau.com/app/profile/vishal.srikanth/viz/MapKW/01InteractiveMap)
