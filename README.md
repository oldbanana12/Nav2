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
    - [Subsection 1](#subsection-1)
    - [Subsection 2](#subsection-2)
  
# Concepts

.nav2 files contain 3D geometry in the form of faces and vertices along with graph data structures that the AI pathfinding system uses to traverse the mesh. There are a number of ways this is divided up and to fully understand the format, we have to define various terms:

* We'll use the word __tile__ to define one section of the game map when larger maps are involved (e.g. afgh). You see these tiles in the filenames of various files (i.e. `afgh_139_141_nav.nav2`)
* A __group__ is a logical division of the data structures in a single .nav2 file. If a navmesh is self contained and is not part of a tiled map, it will likely only have one group. If the .nav2 file forms part of a tiled map, it will have one main group and one group for each bordering tile.
* A __segment__ is a subdivision of the navmesh within the nav2 file. Within each group, the navmeshes are generally divided into these segments. These form a grid, but some of the segments are further subdivided if there is a lot of geometry in a particular area.

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

| Field                       | Type                | Description                                                                                                                                                                                |
| --------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Common Header               | Common Entry Header | 16 bytes as described in the "Common Entry Header" described above                                                                                                                         |
| Subsection 1 Offset         | uint                | The offset from the _end_ of the common entry header to the start of "Subsection 1" which describes the 3D positions of the points within the graph data structure.                        |
| Subsection 2 Offset         | uint                | The offset from the _end_ of the common entry header to the start of "Subsection 2" which provides some array lengths for this node and some indexes into other subsections.               |
| Subsection 4 Offset         | uint                | The offset from the _end_ of the common entry header to the start of "Subsection 4" which is an array of edges between the nodes in the __NavWorld__ graph.                                |
| Subsection 3 Offset         | uint                | The offset from the _end_ of the common entry header to the start of "Subsection 3" which details which other nodes a node is connected to.                                                |
| u1                          | uint                | Unknown. Usually `0`.                                                                                                                                                                      |
| u2                          | uint                | Unknown. Usually `0`.                                                                                                                                                                      |
| Subsection 5 Offset         | uint                | The offset from the _end_ of the common entry header to the start of "Subsection 5" which seems to be a way of providing flags for different __Segments__.                                 |
| u3                          | uint                | Unknown. Usually `0`.                                                                                                                                                                      |
| Subsection 6 Offset         | uint                | The offset from the _end_ of the common entry header to the start of "Subsection 6" which provides pointers to the mesh to indicate which part of the mesh a node in the graph belongs to. |
| Number of Points            | ushort              | The number of nodes within this graph structure.                                                                                                                                           |
| Number of Edges             | ushort              | The number of edges within this graph structure.                                                                                                                                           |
| Padding?                    | ushort              | Probably padding, usually `0`                                                                                                                                                              |
| Padding?                    | ushort              | Probably padding, usually `0`                                                                                                                                                              |
| Padding?                    | ushort              | Probably padding, usually `0`                                                                                                                                                              |
| Number of Section 5 Entries | ushort              | The number of entries within "Subsection 5". This should be equal to the number of __Segments__ within the __Group__                                                                       |

### Subsection 1
| Field                      | Type               | Description                                                                                                                                                                                                     |
| -------------------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| points[`Number of points`] | Vertex (3x ushort) | An array of 3D points (encoded as ushorts), where each entry is a node in the graph structure. To get the proper floating point 3D co-ordinates, divide each component by the divisors in the main file header. |

### Subsection 2
