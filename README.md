# MGS .nav2 Information <!-- omit in toc -->

# Tools <!-- omit in toc -->

- [oldbanana12/Obj2Nav2](https://github.com/oldbanana12/Obj2Nav2) (Converts arbitrary wavefront .obj files to MGSV compatible .nav2 files)
- [oldbanana12/Nav2Parser](https://github.com/oldbanana12/Nav2Parser) (Parses .nav2 files and extracts the navmesh amongst other info)

This repository includes a .bt file that can be used with 010 Editor to inspect the data structures of .nav2 files.

# Contents <!-- omit in toc -->

- [Concepts](#concepts)
  - [Co-ordinates as integers](#co-ordinates-as-integers)
- [Structure](#structure)
- [Detailed Description](#detailed-description)
  - [Header](#header)
    - [Manifest](#manifest)
    - [ManifestEntry](#manifestentry)
  - [Common Entry Header](#common-entry-header)
  - [NavWorld](#navworld)
    - [Header](#header-1)
    - [Subsection 1 (Node positions)](#subsection-1-node-positions)
    - [Subsection 2 (Extra Node Info)](#subsection-2-extra-node-info)
      - [Subsection2Entry](#subsection2entry)
    - [Subsection 3 (Node adjacencies)](#subsection-3-node-adjacencies)
    - [Subsection 4 (Edges)](#subsection-4-edges)
      - [Subsection4Entry](#subsection4entry)
    - [Subsection 5 (Flags?)](#subsection-5-flags)
    - [Subsection 6 (Navmesh References)](#subsection-6-navmesh-references)
  - [NavmeshChunk](#navmeshchunk)
    - [Header](#header-2)
    - [Subsection 1 (Vertices)](#subsection-1-vertices)
    - [Subsection 2 (Face Offsets)](#subsection-2-face-offsets)
    - [Subsection 3 (Faces)](#subsection-3-faces)
      - [Subsection3Entry](#subsection3entry)
  
# Concepts

.nav2 files contain 3D geometry in the form of faces and vertices along with graph data structures that the AI pathfinding system uses to traverse the mesh. There are a number of ways this is divided up and to fully understand the format, we have to define various terms:

* We'll use the word __tile__ to define one section of the game map when larger maps are involved (e.g. afgh). You see these tiles in the filenames of various files (i.e. `afgh_139_141_nav.nav2`)
* A __group__ is a logical division of the data structures in a single .nav2 file. If a navmesh is self contained and is not part of a tiled map, it will likely only have one group. If the .nav2 file forms part of a tiled map, it will have one main group and one group for each bordering tile.
* A __segment__ is a subdivision of the navmesh within the nav2 file. Within each group, the navmeshes are generally divided into these segments. These form a grid, but some of the segments are further subdivided if there is a lot of geometry in a particular area, or if particular areas need to be specially divided (maybe to differentiate water etc).

## Co-ordinates as integers

A lot of the Vector3 positions in .nav2 files are encoded as unsigned integers. Turning these back into floating point numbers in the same co-ordinate space used elsewhere in the level involves dividing the values by some divisors found in the header of the file and offseting the result based on the origin of the file (also found in the header).

# Structure

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

Finally, .nav2 files also always seem to have a __NavSystem__ at the end of the file. Not a lot is known about the fields in this section, but it seems to be another way of dividing the .nav2 up into a grid. The header contains the co-ordinates of the origin of the navmesh and some integers that determine how many pieces to divide it into. Then for each piece, it describes a list of segments that fall within that piece.

# Detailed Description

## Header

| Field                    | Type               | Description                                                                                                                                                                                                                                                                                            |
| ------------------------ | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Magic                    | uint               | `201403242` for MGSV files. Other FOX engine games may use a different value as this just seems to be a date code of when the format was updated.                                                                                                                                                      |
| File Size                | uint               | Total count of all bytes in the file                                                                                                                                                                                                                                                                   |
| End of header offset     | uint               | The size of the __Header__ and __Manifest__ entries in the file. This can vary depending on the number of entries used.                                                                                                                                                                                |
| Entry count              | uint               | The number of entries used. This is usually 4 multiplied by the number of groups in the file. For standalone nav2 files, this will usually be 4. For nav2 files that are part of a tiled map, it will be `4 + (4 * number of bordering tiles)`.                                                        |
| __NavSystem__ offset     | uint               | Offset from the start of the file to the __NavSystem__ entry                                                                                                                                                                                                                                           |
| File Index               | byte               | Due to size limitations of some of the data types used, the .nav2 format supports splitting the data across multiple files if some of these limits are reached. This is the ID of a file in a "split set". It should start from 0. There is usually only 1 .nav2 for all but the largest of navmeshes. |
| u0a (name TBD)           | byte               | Not fully understood, but appears to relate to the tile reference within a tiled map. Can be set to `0` for standalone maps. Appears to always be set to `128` if the horizontal tile reference is odd.                                                                                                |
| u0b (name TBD)           | byte               | Similar to above. Using this and `u0a`, it seems to be possible to recover the horizontal tile reference of a tiled .nav2 file. See .bt comments for details.                                                                                                                                          |
| u0c (name TBD)           | byte               | Similar to above. Using this and `u0b`, it seems to be possible to recover the vertical tile reference of a tiled .nav2 file. See .bt comments for details.                                                                                                                                            |
| __Section2__ Offset      | uint               | Offset from the start of the file to the __Section2__ section. This section is present in a lot of files but does seem to be optional by setting this value to `0`.                                                                                                                                    |
| o6 (probably padding)    | uint               | There is a lot of 16-byte alignment in .nav2 files. So having this as padding starts the next field at the next 16-byte boundary.                                                                                                                                                                      |
| Origin                   | Vector3 (3x float) | The 3D origin of this navmesh                                                                                                                                                                                                                                                                          |
| __Section3__ Offset      | uint               | The offset fromt he start of the file to the __Section3__ section. It is very rare to see this included and is often set to `0`. An example of this section can be found in `mb_brdg_clst.nav2`.                                                                                                       |
| u1b                      | uint               | Unknown. Possibly another offset to another rarely used section, or maybe just reserved in the format for future use. Usually set to `0`.                                                                                                                                                              |
| __Manifest__ Offset      | uint               | Offset from the start of the file to the __Manifest__ section.                                                                                                                                                                                                                                         |
| __Manifest__ Size        | uint               | Total size of the manifest section in bytes                                                                                                                                                                                                                                                            |
| u1c                      | uint?              | Unknown, usually `0`                                                                                                                                                                                                                                                                                   |
| u1d                      | uint?              | Unknown, usually `0`                                                                                                                                                                                                                                                                                   |
| X co-ordinate divisor    | ushort             | The divisor to use when converting integer co-ordinates back to floating point                                                                                                                                                                                                                         |
| Y co-ordinate divisor    | ushort             | The divisor to use when converting integer co-ordinates back to floating point                                                                                                                                                                                                                         |
| Z co-ordinate divisor    | ushort             | The divisor to use when converting integer co-ordinates back to floating point                                                                                                                                                                                                                         |
| u1h                      | ushort             | Probably a divisor for the `w` component of a Vector4 ? Unsure whether this is relevant or important                                                                                                                                                                                                   |
| n7                       | byte               | Unknown. Possibly an entry count. Seems to be always set to `1`                                                                                                                                                                                                                                        |
| __Section2__ Entry Count | byte               | The number of entries in __Section2__                                                                                                                                                                                                                                                                  |
| n8 (probably padding)    | ushort             | Normally 0, so probably padding to the next 16-byte boundary                                                                                                                                                                                                                                           |
| vals                     | 8x hfloats?        | Not fully convinced that these are hfloats and also unsure what their purpose is. Possibly a record of what generation parameters were used to build the file.                                                                                                                                         |


### Manifest

The __Manifest__ provides an index of sorts for the entries within the .nav2 file. It seems to only index the __NavWorld__, __NavmeshChunk__ and __SegmentGraph__ entries. The __SegmentChunk__ entry is not included.

| Field                               | Type                       | Description                                                                                                             |
| ----------------------------------- | -------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Group Count                         | uint                       | The number of groups in the manifest (i.e. `1` for standalone maps, or `1 + number of bordering tiles` for tiled maps). |
| Manifest Entries[`Group Count * 3`] | Array of __ManifestEntry__ | See the following section for a description of the contents.                                                            |

### ManifestEntry

| Field                 | Type   | Description                                                                                                                                                                |
| --------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Group ID              | byte   | The __group__ that this entry belongs to. For a standalone map, this will be `4`. Groups `0-3` correspond to the small slithers of navmesh that border other tiles.        |
| u1b                   | byte   | Not fully understood, this possibly relates to `u0a` in the header somehow as it often seems to match.                                                                     |
| u2                    | ushort | Not fully understood. As above, this probably relates to the tiling system as it's usually 0 for standalone maps and has the same value across all entries for tiled maps. |
| Payload Offset        | uint   | Offset from the start of the file where the content of this entry is found. Each entry usually has a small header before it, and this offset skips over that.              |
| Entry Type            | byte   | `0` = __NavWorld__ , `1` = __NavmeshChunk__, `3` = __SegmentGraph__                                                                                                        |
| Entry Size            | ushort | Total size of the entry in bytes (including the header that the offset skips)                                                                                              |
| n4 (Probably padding) | byte   | Almost certainly padding                                                                                                                                                   |

## Common Entry Header

Each __NavWorld__, __NavmeshChunk__, __SegmentGraph__ and __SegmentChunk__ entry starts with a 16-byte header. The format and fields within this header are the same for all of the entry types as described below:

| Field             | Type   | Description                                                                                                                                                         |
| ----------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Entry Type        | ushort | `0` = __NavWorld__ , `1` = __NavmeshChunk__, `3` = __SegmentGraph__, `4` = __SegmentChunk__                                                                         |
| u1                | ushort | Unknown, usually `0`                                                                                                                                                |
| Next Entry Offset | uint   | The offset from the start of this header to start of header of the next entry in the nav2 file                                                                      |
| Payload Offset    | uint   | The offset from the start of this header to the payload of the entry. Always `16`?                                                                                  |
| Group ID          | byte   | The __group__ that this entry belongs to. For a standalone map, this will be `4`. Groups `0-3` correspond to the small slithers of navmesh that border other tiles. |
| u2                | byte   | Not fully understood. Seems to match `u1b` in __ManifestEntry__, see there for more details.                                                                        |
| n3                | ushort | Not fully understood. Seems to match `u2` in __ManifestEntry__, see there for more details.                                                                         |

## NavWorld

### Header

| Field                       | Type                                        | Description                                                                                                                                                                                |
| --------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Common Header               | [Common Entry Header](#common-entry-header) | 16 bytes as described in the [Common Entry Header](#common-entry-header).                                                                                                                  |
| Subsection 1 Offset         | uint                                        | The offset from the _end_ of the common entry header to the start of "Subsection 1" which describes the 3D positions of the points within the graph data structure.                        |
| Subsection 2 Offset         | uint                                        | The offset from the _end_ of the common entry header to the start of "Subsection 2" which provides some array lengths for this node and some indexes into other subsections.               |
| Subsection 4 Offset         | uint                                        | The offset from the _end_ of the common entry header to the start of "Subsection 4" which is an array of edges between the nodes in the __NavWorld__ graph.                                |
| Subsection 3 Offset         | uint                                        | The offset from the _end_ of the common entry header to the start of "Subsection 3" which details which other nodes a node is connected to.                                                |
| u1                          | uint                                        | Unknown. Usually `0`.                                                                                                                                                                      |
| u2                          | uint                                        | Unknown. Usually `0`.                                                                                                                                                                      |
| Subsection 5 Offset         | uint                                        | The offset from the _end_ of the common entry header to the start of "Subsection 5" which seems to be a way of providing flags for different __Segments__.                                 |
| u3                          | uint                                        | Unknown. Usually `0`.                                                                                                                                                                      |
| Subsection 6 Offset         | uint                                        | The offset from the _end_ of the common entry header to the start of "Subsection 6" which provides pointers to the mesh to indicate which part of the mesh a node in the graph belongs to. |
| Number of Points            | ushort                                      | The number of nodes within this graph structure.                                                                                                                                           |
| Number of Edges             | ushort                                      | The number of edges within this graph structure.                                                                                                                                           |
| Padding?                    | ushort                                      | Probably padding, usually `0`.                                                                                                                                                             |
| Padding?                    | ushort                                      | Probably padding, usually `0`.                                                                                                                                                             |
| Padding?                    | ushort                                      | Probably padding, usually `0`.                                                                                                                                                             |
| Number of Section 5 Entries | ushort                                      | The number of entries within "Subsection 5". This should be equal to the number of __Segments__ within the __Group__                                                                       |

### Subsection 1 (Node positions)
| Field                      | Type               | Description                                                                                                                                                                                                     |
| -------------------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| points[`Number of points`] | Vertex (3x ushort) | An array of 3D points (encoded as ushorts), where each entry is a node in the graph structure. To get the proper floating point 3D co-ordinates, divide each component by the divisors in the main file header. |

### Subsection 2 (Extra Node Info)
| Field                       | Type             | Description                                                                                                                                                    |
| --------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| entries[`Number of points`] | Subsection2Entry | An array of __Subsection2Entry__ structures (described below). This subsection is largely used to determine the size and related indexes of other subsections. |

#### Subsection2Entry
| Field              | Type   | Description                                                                                                                                                                                         |
| ------------------ | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Section 3 Offset   | ushort | `(offset / 2)` into "Subsection 3" of where to find the __Subsection3Entry__ that relates to this node. Multiply by `2` to get the actual byte offset from the start of "Subsection 3".             |
| Subsection 5 Index | ushort | The index into the "Subsection 5" array that relates to this node.                                                                                                                                  |
| countA             | byte   | The number of "type A" edges that connect to this node. These are the standard edges where two different nodes connect to each other.                                                               |
| countB             | byte   | The number of "type B" edges that connect to this node. These are for the special case where there are two nodes with the same physical position, because they exist in two different __Segments__. |

### Subsection 3 (Node adjacencies)
| Field                | Type      | Description                                                                                                                                                  |
| -------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| typeAEdges[`countA`] | 2x ushort | The first ushort is the index of the node connected by this edge. The second is the index into the edges array (subsection 4).                               |
| typeBEdges[`countB`] | ushort    | The index of the node connected by this edge. Remember, this node has the same physical position as the current node, but exists in a different __Segment__. |

### Subsection 4 (Edges)
| Field                    | Type             | Description                                                                                                                                |
| ------------------------ | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| edges[`Number of edges`] | Subsection4Entry | An array of `Subsection4Entry` structures (described below). These structures describe an individual edge within the graph data structure. |

#### Subsection4Entry
| Field              | Type    | Description                                                                                                                                                                                                                                                                                                 |
| ------------------ | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| weight             | ushort  | The weighting of this graph edge (i.e. distance between the two connected nodes). The formula used to determine this weighting by the MGSV .nav2 generator has not been determined. However, pathfinding algorithms usually just work on the relative weights between edges, so it's likely not important.  |
| Subsection 5 Index | ushort  | The index into the "Subsection 5" array that relates to this edge.                                                                                                                                                                                                                                          |
| nodes              | 2x byte | The "from" and "to" node indexes for this edge. Note that these indexes start again from `0` every time the "Subsection 5 Index" is incremented. So a value of `3` when "Subsection 5 Index" is `2`, is `3 + (total nodes associated with indexes 0 and 1 of subsection 5)` in terms of overall node index. |

### Subsection 5 (Flags?)
There is one Subsection 5 entry for each __Segment__ in the .nav2 file, so it provides a means of associating different nodes and edges with different __Segments__. For the most part, these __Segments__ just form a grid, but some maps have this grid system further divided into more natural formations (like rivers etc).
| Field                                   | Type   | Description                                                                                                                                                                                                                                                                                                                                                                      |
| --------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| flags[`Number of subsection 5 entries`] | ushort | Not fully understood. From the various values seen in the wild. These appears to be some kind of bitfield that probably forms some kind of flag system. It likely determines something like what kinds of entity are allowed to use this segment of the navigation graph, or possibly what animations/sounds to play. Common values are `1249` (`0xE104`) and `1257` (`0xE904`). |

### Subsection 6 (Navmesh References)
| Field                                       | Type   | Description                                                                                                                                                  |
| ------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| navmeshSubsection2Index[`Number of points`] | ushort | An array of indexes into the Subsection2 array of the __NavmeshChunk__ in the same group. Used to associate the graph node with a physical piece of navmesh. |

## NavmeshChunk
### Header
| Field               | Type                                        | Description                                                                                                                                          |
| ------------------- | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Common Header       | [Common Entry Header](#common-entry-header) | 16 bytes as described in [Common Entry Header](#common-entry-header).                                                                                |
| Subsection 1 Offset | uint                                        | The offset from the _end_ of the common entry header to the start of "Subsection 1" which describes the 3D positions of vertices that form the mesh. |
| Subsection 2 Offset | uint                                        | The offset from the _end_ of the common entry header to the start of "Subsection 2" which provides some offsets into the Subsection 3 Array.         |
| Subsection 3 Offset | uint                                        | The offset from the _end_ of the common entry header to the start of "Subsection 3" which describes the faces of the navmesh.                        |
| uu1                 | ushort                                      | Unknown. Usually `0`.                                                                                                                                |
| uu2                 | ushort                                      | Unknown. Usually `0`.                                                                                                                                |
| uu3                 | ushort                                      | Unknown. Usually `0`.                                                                                                                                |
| uu4                 | ushort                                      | Unknown. Usually `0`.                                                                                                                                |
| uu5                 | ushort                                      | Unknown. Usually `0`.                                                                                                                                |
| uu6                 | ushort                                      | Unknown. Usually `0`.                                                                                                                                |
| Number of faces     | ushort                                      | The number of navmesh faces contained in this entry                                                                                                  |
| Number of vertices  | ushort                                      | The number of navmesh vertices contained in this entry                                                                                               |
| u4                  | ushort                                      | Probably padding, usually `0`.                                                                                                                       |
| u5                  | ushort                                      | Probably padding, usually `0`.                                                                                                                       |

### Subsection 1 (Vertices)
| Field                          | Type               | Description                                                                                                                                                                                               |
| ------------------------------ | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| vertices[`Number of vertices`] | Vertex (3x ushort) | An array of 3D points (encoded as ushorts), where each entry is a vertex in the navmesh. To get the proper floating point 3D co-ordinates, divide each component by the divisors in the main file header. |

### Subsection 2 (Face Offsets)
| Field                     | Type | Description                                                                                                                                                                                                                                                                          |
| ------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| offset[`Number of faces`] | uint | Perform a bitwise AND with `0x3ffff` and multiply by `2` to get the offset into the Subsection 3 array for this face. If a bitwise right shift of this value by `0x12` results in an odd number, then this face has 4 vertices. Though 4-vertex faces haven't been seen in the wild. |

### Subsection 3 (Faces)
| Field                    | Type             | Description                                                                                                                                                                                                                                                                                                                                                                 |
| ------------------------ | ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| faces[`Number of faces`] | Subsection3Entry | An array of Subsection3Entry structures (described below). These structures describe an individual face within the navmesh. Note that it is not possible to re-assemble the whole Navmesh with just this __NavmeshChunk__ section. There is some additional information in the __SegmentChunk__ section that describes the relationship between the faces and the vertices. |

#### Subsection3Entry
| Field                   | Type    | Description                                                                                                                                                                                                                                                                                         |
| ----------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| adjacentFaces[`3 or 4`] | short[] | The indexes of the other faces that share an edge with this face. Or, `-1` if there isn't a face on that side. There are usually 3 entries here, but note that it may be possible to have 4-edge faces as described in the [Subsection 2 (Face Offsets)](#subsection-2-face-offsets) section above. |
| v1                      | byte    | The index of the first vertex of this face, note that this will be offset by an amount described in the __SegmentChunk__ section.                                                                                                                                                                   |
| v2                      | byte    | The index of the second vertex of this face, note that this will be offset by an amount described in the __SegmentChunk__ section.                                                                                                                                                                  |
| v3                      | byte    | The index of the third vertex of this face, note that this will be offset by an amount described in the __SegmentChunk__ section.                                                                                                                                                                   |
| v4 (optional)           | byte    | The index of the fourth vertex of this face, only present if this is a 4-vertex face as described in the [Subsection 2 (Face Offsets)](#subsection-2-face-offsets) section above.                                                                                                                   |
| edgeIndices[`3 or 4`]   | byte[]  | Every edge in the navmesh has an index that is unique per-__segment__.                                                                                                                                                                                                                              |
