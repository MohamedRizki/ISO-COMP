# IsoComp :

This project is developed as part of an academic project at the **ENSG – National School of Geographic Sciences**, in collaboration with the company **Ciril Group**.

**IsoComp** is a geospatial web application for comparing **isochrone** from multiple mapping service APIs (IGN, Mapbox, Ciril Group, and Here). It allows users to visualize the areas reachable within a given time or distance from a location, as computed by each provider, side-by-side on an interactive map. The app evaluates each API on performance metrics – including response time, area coverage, precision, routing accuracy, Hausdorff distance, and reliability – to highlight differences. Users can overlay isochrone polygons for each service, inspect quantitative comparisons, and export the results as GeoJSON files or a detailed PDF report.

## Project Overview
IsoComp was created to **compare isochrone services** and help identify strengths and weaknesses of different mapping APIs. An *isochrone* is a contour that connects points reachable in the same travel time from a start location (an isoline of equal travel time) ([Isochrone - OpenStreetMap Wiki](https://wiki.openstreetmap.org/wiki/Isochrone#:~:text=An%20isochrone%20is%20an%20isoline,such%20as%20distance%2C%20or%20even)). Different providers often produce varying isochrone shapes and sizes for the same inputs due to differences in road data, routing algorithms, and optimization parameters. 

This application provides a unified interface to generate isochrones from four providers – **IGN** (France's National Geographic Institute), **Mapbox**, **Ciril Group** (CG), and **Here** – and compares them directly. The goal is to evaluate which service offers the best performance and accuracy under various conditions, and to illustrate how isochrone results can differ between platforms.

## Features :
- **Interactive Map Interface:** The front-end provides a responsive map (built with Leaflet) where users can select a start location (by clicking on the map or entering coordinates/address) and set parameters like travel time or distance. The map displays isochrone polygons from each selected API in distinct colors for easy visual comparison.
- **Multi-Provider Isochrone Comparison:** IsoComp fetches isochrone polygons from IGN, Mapbox, Ciril Group, and Here for the same location and travel parameter. Users can toggle which providers to include, enabling side-by-side comparison of up to four isochrones on one map.
- **Performance Metrics Display:** For each API, the app calculates key metrics such as the API's response time (latency), the area of the resulting isochrone polygon, and other quality indicators (precision of shape, routing accuracy, etc.). These metrics are displayed in a comparison table or panel, allowing users to quickly gauge differences.
- **GeoJSON Download:** Users can download the raw GeoJSON data for the generated isochrones. This feature is useful for offline analysis or integrating the results into GIS software. Each provider's isochrone can be downloaded individually, or all together in a combined file.
- **PDF Report Generation:** IsoComp can generate a comprehensive PDF report summarizing the comparison. The report includes the map snapshot or description of the selected area, all the metrics computed for each provider, and possibly charts or rankings. This allows users to save or share the evaluation results easily.
- **Customizable Parameters:** Users may adjust travel mode (e.g., driving, walking) and travel time or distance thresholds for the isochrones. The interface and analysis adapt to these inputs, letting users test different scenarios. (For example, compare 10-minute driving isochrones vs 10-minute walking isochrones across providers.)

## Architecture Overview
IsoComp follows a classic **client–server architecture** split between a web front-end and a Python back-end:
- **Front-End:** The user interface is a single-page web app built with HTML, CSS, and JavaScript, using the **Leaflet** library for interactive maps. Users interact with a Leaflet map and controls (forms, buttons) to set input parameters. When the user requests a comparison, the front-end sends an AJAX request to the Flask server and then displays the results (polygons and metrics) returned by the server. The front-end is responsible for rendering the isochrone polygons on the map with different styles per provider and showing the metrics in a PDF.
- **Back-End:** The server is implemented in Python using **Flask**. Upon receiving a request, the Flask server orchestrates the calls to external APIs and performs geospatial computations. The back-end handles all heavy processing: it requests isochrone data from IGN, Mapbox, CG, and Here (using their web APIs, typically via HTTP requests), processes the returned geospatial data using libraries like **GeoPandas/Shapely** for analysis, and then returns a consolidated result to the front-end. It also uses a PDF generation library to create the comparison report on demand.

The diagram below provides a high-level overview of the system architecture:

![Application Architecture](Diagrammes/diag1.png)

In this architecture, the front-end and back-end communicate via HTTP. The heavy lifting (API calls, data crunching) is done server-side to avoid exposing API keys and to leverage Python's geospatial libraries. The front-end remains lightweight, only handling user interaction and rendering.

## How It Works
From a user's perspective, using IsoComp involves a few simple steps. Internally, each step triggers a series of back-end operations to gather data and compute the comparison:
1. **User Input:** The user selects a start location on the map (e.g., by clicking a point or entering an address) and sets the desired travel time or distance for the isochrone. The user also chooses which APIs to compare (they can select any combination of the four services).
2. **API Request Trigger:** The user clicks a button (such as "Generate Isochrones" or "Compare") to submit the request. The front-end JavaScript collects the input parameters (location coordinates, travel time, selected providers) and sends an HTTP request to the Flask back-end with these details.
3. **External API Calls:** The Flask server receives the request and begins querying each selected provider's isochrone API. For each service, it formats an HTTP request according to that API's specifications (including API keys, the location, travel time/distance, mode if applicable, etc.) and sends the request. This is done for IGN, Mapbox, CG, and Here in parallel. As each API responds, the server records the response time and retrieves the resulting isochrone polygon data (typically in GeoJSON or a similar format).
4. **Geo-Analysis:** Once all responses are received, the server uses **GeoPandas/Shapely** to load the polygons and perform comparisons. It calculates the area of each polygon, the pairwise Hausdorff distances between polygons, overlaps or gaps, and checks routing accuracy by sampling points.
5. **Aggregate Results:** The server compiles all the metrics and the geometry data into a unified response. It might create a GeoJSON containing all polygons with attributes or separate GeoJSONs per provider, and a JSON or table of metrics. It may also compute a **score** for each provider by combining the metrics (e.g., a weighted score out of 100).
6. **Send Response:** The Flask server sends back the aggregated results to the front-end. The response includes the geometry of the isochrones (for display on the map) and the metrics/scoring information (for display in the UI).
7. **Visualization:** The front-end JavaScript receives the data. It uses Leaflet to draw each isochrone polygon on the map in a different color or style (with a legend or labels identifying the provider of each polygon). It also updates the metrics panel, for example filling in a table where each row is a metric and each column is a provider, or showing a small chart or ranking.
8. **Export :** If the user wants to save the results, they can click the "Download GeoJSON" button to get the data. The front-end simply triggers a download of the GeoJSON file(s) prepared by the server. Similarly, if the user clicks "Download PDF Report," the browser sends a request to the Flask server's `/report` endpoint.
9. **PDF Generation (Optional):** When the server receives a report request, it uses the previously computed results (it might recompute or use cached results from the last query) to generate a PDF. The **FPDF** library is used to compose a document containing text (and possibly images). This report typically includes the input parameters (location, travel time, date of comparison), a summary table of all metrics, and possibly a small map or diagram illustrating the overlap of isochrones. Once generated, the PDF is sent back to the browser for the user to download.
10. **Iteration:** The user can then change parameters or location and run another comparison, which repeats the process.

The sequence diagram below illustrates the interaction between the user, the front-end, and the back-end for a typical scenario of generating isochrones and then downloading a report:

![The sequence diagram](Diagrammes/diag2.png)

## Metrics Compared 
IsoComp evaluates each isochrone result using several **metrics** to provide a well-rounded comparison:
- **Response Time:** The time taken for each API to return an isochrone result. This is measured from when the request is sent to when the first byte of the response is received (or the full response, depending on implementation). A lower response time indicates a faster service, which is crucial for real-time applications.
- **Area Coverage:** The geographic area covered by the isochrone polygon. After receiving the polygon geometry, the application computes its area. This metric shows how expansive the reachable region is for the given travel limit.
- **Precision of Shape:** This metric reflects how detailed or fine-grained the isochrone geometry is. It can be indicated by the number of vertices in the polygon or the inclusion of intricate details (following the road network closely vs. a generalized shape). A highly generalized isochrone might omit peninsulas of reachable area or smooth out the boundary, whereas a high-precision isochrone will have a complex boundary that closely follows actual roads. Precision can affect the accuracy of area and overlap calculations.
- **Routing Accuracy:** To validate that an isochrone truly represents reachable areas, IsoComp can sample random points on each polygon's boundary (or specific key points like farthest extent in each direction) and compute the actual travel time via a routing engine from the center to those points. The difference between the polygon's nominal time (say 10 minutes) and the real travel time needed is an indicator of accuracy. If an isochrone polygon extends too far into areas that actually take longer than the threshold to reach, it has lower routing accuracy (potentially overestimating coverage). Conversely, if it misses areas that are reachable, that's another accuracy issue. In simpler terms, this metric checks if the polygon from each API is neither overstating nor understating the reachable region for the given conditions.
- **Hausdorff Distance:** Hausdorff distance is used to quantify how different two shapes are. It measures the greatest distance from any point on one polygon to the closest point on another polygon ([Measuring the similarity of two polygons in PostGIS - Geographic Information Systems Stack Exchange](https://gis.stackexchange.com/questions/362560/measuring-the-similarity-of-two-polygons-in-postgis#:~:text=As%20other%20users%20mentioned%2C%20the,similarity%20between%20the%20two%20shapes)). In the context of IsoComp, we compute the Hausdorff distance between the isochrone polygons of different providers. A small Hausdorff distance means the two isochrones are very similar in shape and overlap closely. A large Hausdorff distance highlights that there are portions of one isochrone far away from any part of the other, indicating a significant disparity in their coverage.
- **Composite Score:** To give an overall assessment, IsoComp combines the above metrics into a single **score** for each provider. This scoring system may weight each metric (for example, giving more importance to area coverage and accuracy than to response time, or vice versa depending on user priorities) and produce a numeric rating or grade. The scoring allows a quick identification of which API performed best overall for the given scenario. 

Each metric is calculated using standard geospatial methods. For instance, area is computed via GeoPandas' geometry area calculation (after projecting coordinates to an appropriate projection for accuracy), and Hausdorff distance can be obtained using Shapely's `hausdorff_distance` function between two polygons.

## Technologies Used
- **Flask (Python)** 
- **Leaflet (JavaScript)** 
- **HTML5 & CSS3**
- **JavaScript (ES6+)**
- **GeoPandas & Shapely (Python)**
- **Requests (Python)**
- **FPDF (Python)**
- **External APIs**
  - **IGN Isochrone API** 
  - **Mapbox Isochrone API**
  - **Ciril Group API**
  - **Here Isochrone API** 

## Application interface :
![Application Architecture](Diagrammes/Application.JPG)


## Project Team :
- Mohamed RIZKI  
- Rieulle BRUSQ 
- Clément TYCHYJ  
- Clélia CHAMPEYROUX  
