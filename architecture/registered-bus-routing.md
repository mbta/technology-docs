# Bus routing in Registered

## Context
In order to properly make announcements, buses need to know the distance between stops and the compass direction at the end. Each of these distance/directions is called an Interval, between two Stops.

When a Stop moves during one of the quarterly ratings, the Intervals using that stop also need to be re-calculated. For short intervals, these can be calculated in the TransitMaster interface by clicking out the route on a map.

However, some intervals (mostly non-revenue) are too long and complicated to effectively click through in the TransitMaster interface. The current workaround is to use Google Maps to calculate the distance manually (converting from kilomenters to feet for the distance) and estimating the compass direction. This process can take days if there are many stops to update.

## Goals / non-goals
- query the database directly for missing intervals, but also be able to use a CSV for off-VPN use
- automatic calculation of the correct distance / direction for the given intervals
- generated routes should reflect traffic rules
	- one-way streets
	- no left turns / U-turns
- generated routes should reflect that the route is for a bus and not a passenger car. for example:
	- buses can drive through a busway
	- buses cannot drive on Storrow Drive (too high)
	- buses can not make sharp turns (large turning radius)
- generated routes should ensure that the start/end stops have the bus stop on their right side
	- this ensures that the bus is travelling in the correct direction to pick up/drop off passengers at the stop
- performance does not need to match Google Maps / OpenTripPlanner, but should not be "unreasonable"

## Design
As [Registered](https://github.com/mbta/registered) (the set of scripts used to automate parts of the TransitMaster rating process) is already in Python, the interval calculation is also in Python.

OSMnx is a Python library which can query OpenStreetMap data and turn it into a graph. We base our initial graph on this data. However, it's missing some features we need and so we'll build on top of it.

Edges (called Ways in OpenStreetMap) have tags to describe data like maximum speed and what types of vehicles can use it. We use a combination of OSMnx built-ins and custom code to calculate a time for each edge, which we'll minimize when finding the fastest route between two stops.

Turn restrictions are modeled in OpenStreetMap as a type of relation, between Ways and a Node. We use OSMnx to query for the relations in the relevant area around our stops, and build it into a data structure for later querying.

It's possible to incorporate turn restrictions into account with a normal shortest-path algorithm, but only by approximately doubling the size of the graph. Instead, we [write our own implementation](https://github.com/networkx/networkx/pull/4679) of the shortest-path algorithm which directly takes turn restrictions into account.

In order to use the shortest-path algorithm, we need to find the start and end nodes in the graph. We build an R* tree of all edges when building the graph. With that, we can find the closest edges to the given start/end point and split it at that location. If multiple edges are the same distance away (bi-directional edge), we use the edge which keeps the point on the right side of the edge.

These give us the ability to calculate the fastest path between arbitrary points covered by OpenStreetMap. To use this data, we build an HTML page containing all of the data for the intervals: 
- start/end points
- the type of interval
- the route/pattern the interval is on
- the shortest/fastest distances and compass directions
- a Leaflet map with the fastest/shortest routes

This HTML page can be opened in any browser, and the maps can be scrolled and zoomed to ensure that they're correct.

There are a number of unit tests of the routing algorithm. If any generated routes are found to be incorrect, we can add additional tests to ensure that the issue does not occur in the future. We can also make updated to the OpenStreetMap data itself, benefiting us as well as the globa OSM community.

## Alternatives considered
- OpenTripPlanner API
	- While using our existing OTP instance would be helpful, we'd need to run a separate instance configured to support routing the vehicle as a bus
- using existing Nodes instead of creating new ones
	- While re-using existing nodes is faster, it ran into a number of issues.
	- Roads which cross over each other (such as a highway crossing a local road) can assign the node to the wrong starting edge
	- Existing nodes may not be on the correct side of the street for a stop

## Cross-cutting concerns (security, privacy, scalabilty, observability)
These scripts run on a single developer's machine. This limits the performance/scalability to that of their machine: calculating 100 intervals can take up to an hour. 
