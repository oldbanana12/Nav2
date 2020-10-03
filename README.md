# MGS .nav2 Information

# Tools

- [oldbanana12/Obj2Nav2](https://github.com/oldbanana12/Obj2Nav2) (Converts arbitrary wavefront .obj files to MGSV compatible .nav2 files)
- [oldbanana12/Nav2Parser](https://github.com/oldbanana12/Nav2Parser) (Parses .nav2 files and extracts the navmesh amongst other info)

# File Format

## Concepts

.nav2 files contain 3D geometry in the form of faces and vertices along with graph data structures that the AI pathfinding system uses to traverse the mesh. There are a number of ways this is divided up and to fully understand the format, we have to define various terms:

* We'll use the word __tile__ to define one section of the game map when larger maps are involved (e.g. afgh). You see these tiles in the filenames of various files (i.e. `afgh_139_141_nav.nav2`)
* A __group__ is a logical division of the data structures in a single .nav2 file. If a navmesh is self contained and is not part of a tiled map, it will likely only have one group. If the .nav2 file forms part of a tiled map, it will have one main group and one group for each bordering tile.
* A __segment__ is a subdivision of the navmesh within the nav2 file. Within each group, the navmeshes are generally divided into these segments. These form a grid, but some of the segments are further subdivided if there is a lot of geometry in a particular area.

## Co-ordinates as integers

A lot of the Vector3 positions in .nav2 files are encoded as unsigned integers. Turning these back into floating point numbers in the same co-ordinate space used elsewhere in the level involves dividing the values by some divisors found in the header of the file and offseting the result based on the origin of the file (also found in the header).

## Structure

For each group in the file, there are generally 4 sections (NavWorld, NavmeshChunk, SegmentChunk, and SegmentGraph). In turn, each of these sections describes the following:

* The __NavWorld__ is a graph structure that defines all of the possible path nodes. For every edge within the navmesh, there is a note. This section encodes the data in the following way:

  * Vector3 positions of each node (encoded as integers)
  * Graph edges showing which nodes link to each other
  * Weighting data for each edge
  * Pointers to the navmesh that encode which part of the mesh a node/edge is on
  * Some data that appears to be flags (purpose TBD, possibly determines what types of AI are allowed to use the egde)

* The __NavmeshChunk__ section contains the navmesh geometry, consisting of:
  
  * Vector3 positions of Vertices (encoded as integers)
  * Triangle faces encoded as indices into vertex array. It looks as though quads are supported in the format, but haven't seen them used in the wild.
  * Details of which triangles are adjacent to each other

* The __SegmentChunk__ section details how the file is broken down into segments. Due to the way the navmesh is broken into these segments, it is necessary to use both the __NavmeshChunk__ and __SegmentChunk__ sections to re-assemble the navmesh geometry. It has the following information on each segment:

  * The bounds of the segment, encoded as 2 integer Vector3s
  * How many vertices, faces and edges are in the segment
  * References back to the __NavmeshChunk__ that allow assembling a chunk's geometry.

* The __SegmentGraph__ is very similar to the __NavWorld__ in that it is another graph data structure with nodes, edges and weights etc... However, instead of a detailed node for each edge in the mesh, there is only one node for each segment. It is unclear what the purpose of this is. It's likely an optimisation mechanism for the pathfinding algorithm.