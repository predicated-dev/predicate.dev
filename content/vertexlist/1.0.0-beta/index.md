---
title: "Vertex List Descriptor"
version: "1.0.0-beta"
latest: true
---
## Summary
A general descriptor for most common sequences of vertices in lists, arrays or linked lists. 

## Terms
1. **Descriptor**: A set of values that describe an associated dataset.
1. **Coordinate**: A numeric value member of a set of values that define a specific location.
1. **Vertex**: A structure that holds a set of **Coordinates** defining a single point. 
1. **Dimensionality**: Number of **Coordinates** required to define the location of a **Vertex**.  
1. **Pointer**: A platform native integer value representing a memory address
1. **Offset**: A numeric value added or subtracted from a **Pointer** to retrieve data at a relative memory location.
1. **Array**: A contiguous block of memory with elements directly after each other
1. **Linked List**: A chain of **Nodes**
1. **Node**: A structure with a **Pointer** to another **Node**
1. **Stride**: The distance in bytes from element to element in an **Array**, in case of a **Linked List** the distance from a **Node** start to the **Pointer** it holds for the next **Node** 


## The Vertex List Descriptor
For brevity we will call a structure that holds coordinates a `Vertex`, but it may be more complex than just the simple defintion of a spatial point position.

The following values are provided by the descriptor. We provide the byte offset in brackets `[]`, followed by the field and size.

-  [0] `version`: an 8-bit unsigned integer version of the descriptor, which may be different than the version number of this specification
-  [1] `data_type`: an 8-bit value from an enumeration of coordinate data types with the following ordinal values:
    - 0: Known. From API call naming, protocol, context, etc.
    - 1: 32-bit signed integer
    - 2: 64-bit signed integer
    - 3: 32-bit floating point value (IEEE 754 single-precision)
    - 4: 64-bit floating point value (IEEE 754 double-precision)
-  [2] `list_type`: an 8-bit value from an enumeration of list or array structure types with the following ordinal values:
    - 0: Array
    - 1: Linked List
-  [3] `indirection` : an 8-bit integer value with the level of indirection to the coordinate information (how many pointers to follow). This version of the specification defines implementation cases with at most one indirection.
-  [4] `count`: A 64-bit integer value holding the number of elements in the vertex array.
- [12] `data`: A platform specific pointer sized variable (32-bit or 64-bit). Pointer to the first element of an array or the first node of a linked list.
- [16] `_padding_`: 32-bit pad. Only on 32-bit platforms. 
- [20] `stride`: a 16-bit unsigned integer size 
    - For arrays: The distance from element to element in bytes (also the size of elements in the array). 
    - For linked lists: the distance in bytes from the start of the node to the pointer of the next node 
- [22] `structure_offset`: an 16-bit unsigned integer offset (in bytes) from the start of a `Vertex` to its first coordinate
- [24] `pointer_offset`: a 16-bit unsigned integer offset (in bytes) from the start of a structure to a member pointer to the **Vertex**. Applies to linked lists of pointers and arrays of structures with pointers to `Vertices`. 
- [26] `dimensionality`: an 8-bit integer value indiciating the number of coordinates. May pass 0 if dimensionality is known from API naming, context, protocol. etc.
- [27] `coordinate_system`: an 8 bit value from an enumeration of coordinate systems. 
     - 0: Known. Either from API call naming, protocol or context
     - 1: Cartesian
     - 2: Polar
     - 3: Cylindrical    


### Sample C++ implementation

{{< codebox filename="VertexArrayDescriptor.h" lang="cpp" >}}
#include <cstdint>

struct VertexArrayDescriptor {
    std::uint8_t version = 1;
    std::uint8_t data_type = 0;
    std::uint8_t list_type = 0;
    std::uint8_t indirection = 0;
    std::uint64_t count = 0;
    void*         data = nullptr;

#if UINTPTR_MAX == 0xFFFFFFFFULL
    std::uint32_t _padding_ = 0;
#endif
    std::uint16_t stride = 0;
    std::uint16_t structure_offset = 0;
    std::uint16_t pointer_offset = 0;
    std::uint8_t data_type = 0;
    std::uint8_t data_type = 0;        

};
{{< /codebox >}}

Implementations should add static asserts to ensure alignment and may add strong types via getters and setters.

## Constraints
- **Coordinates** are all of the same single datatype as specified by the **descriptor**. For instance, if we use polar **coordinates** we can't have 32-bit integer values for the radial **coordinate** and 64-bit floating values for the angular **coordinate**. 
- **Coordinate** values for a specific **vertex** are stored in continguous adjacent memory locations. Meaning if we have say a 2 dimensional cartesian vertex its X and Y coordinates are stored directly one after the other. (e.g., `double x, y` or `long coords[2]`)
- Only structures smaller than 2^16 (65,536) bytes supported by offset and stride variables.

## Common Vertex List types
A non-exhaustive list of common representations of coordinate data.

In illustrations below:
- `n`: Number of vertices  
- `p1`, `p2`, etc:  a pointer to a structure that holds coordinates (`Vertex`).
- `{}`: the bounds of a structure. 
- `[]`: the bounds of an array.
- `()`: the bounds of the contiguous set of coordinates that define our vertex
- `(x1, y1)`, `(x2, y2)`, etc: denotes a coordinate group defining a vertex (only 2-dimensions shown in examples, but any dimensionality up to 255 is supported). Coordinate groups may be any form of contiguous values: a series of fields, an array of values, a structure with fields, etc.
- `n1`, `n2`, etc. denotes pointers to nodes in a linked list
- `pre_1`, `post_1`, `inter_1`, etc. denotes optional extraneous data and padding
- `->` shows the value at the memory address held by the pointer (`pointer -> value`) 

Note that 1-based subscript is used in illustrations (simplifies the syntax for the last element in the series).

In all cases below will the vertex array descriptor have

- **version**: Default 1 from definition and should not be altered
- **count**: Number of elements in the array or list
- **data_type**: The type of coordinates 
- **dimensionality**: 2 for all examples below (we only illustrate with `X`, `Y` cartesian coordinates)

 ## Structure holding Coordinates ('Vertex`)
 Even though the structures that hold coordinates can be more complex than simple vertices we will still call them `Vertices` for simplicity. A `Vertex` is a structure of the form `{pre, (x, y), post}` where `pre` and `post` are optional data before and after the coordinates (`X` and `Y`).

### Arrays of Vertices

Contiguous memory of `Vertices` of the form `[v1, v2, ... vn]`. If we expand with our definition of `Vertex` the memory can be seen like this: `[{pre_1, (x1, y1), post_1}, {pre_2, (x2, y2), post_2}, ... {pre_n, (xn, yn), post_n}]`


To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first `Vertex` in array (`&v1`)
- **structure_offset**: The distance in bytes from the start of a `Vertex` to its first coordinate (`offsetof(Vertex, X)`)
- **stride**: The number of bytes from element to element in the array (`sizeof(Vertex)`)
- **listtype**: 0 (Array) 
- **indirection**: 0 (no pointers to follow)

### Arrays of Pointers to Vertices

Pointers adjacent in memory with a stride from element to element as size of pointer: `[p1, p2 ..., pn]` where `p1`, `p2`, etc. are pointers to `Vertices` e.g. `p1 -> v1`, `p2 -> v2`, etc.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first element in array (`&p1`)
- **structure_offset**: The distance in bytes from the start of `Vertex` to its first coordinate (`offsetof(Vertex, X)`)
- **stride**: The size of a pointer (`sizeof(void*)`)
- **listtype**: 0 (`VertexListType::Array`) 
- **indirection**: 1
- **pointer_offset**: 0 (Points directly to structures)

### Arrays of structures with pointers to Vertices

Arrays hold structures that in turn hold pointers to `Vertices`. These structures can hold other data as well. 
For instance we could have `{pre1, p1, post1}, {pre2, p2, post2},...`. 

Assume the elements are of type `Elem` with pointer `p` to Vertices, e.g. `Elem1.p -> v1`, `Elem2.p -> v2`, etc.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first element in array (`&Elem1`)
- **structure_offset**: The distance in bytes from the start of `Vertex` to its first coordinate (`offsetof(Vertex, X)`)
- **stride**: The size of a array elements (`sizeof(Elem)`)
- **listtype**: 0 (`VertexListType::Array`) 
- **indirection**: 1
- **pointer_offset**: The distance in bytes from the start of `Elem` to its pointer to a (`offsetof(Elem, p)`)


## Linked list of structures

Linked lists are chains of pointers to `Nodes`. `Nodes` can be located anywhere in memory, each node has a pointer to the next `Node` in the list and also holds the coordinate values of a specific vertex. Coordinates are held directly by the `Node` or in a nested `Vertex` structure. 

For example: `n1 -> {pre1, (x1, y1), inter1, n2, post1}`, `n2 -> {pre2, (x2, y2), inter2, n3, post2}`, the location of the pointer to the next node is specified by a **pointer offset**, and the **structure offset**, as before, is the distance from the start of the structure to the first coordinate.

Please note that the pointer to next could be before the vertex also, for instance:  `n1 -> [pre1, n2, inter1, (x1, y1), post1]`. 

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first node in the linked list (`&n1`)
- **structure_offset**: The distance in bytes from the start of structure with vertices to its first coordinate (`offsetof(Node, X)`). If coordinates are in a nested `Vertex` structure `v` then it is calculated as the offset of `X` in `v` plus offset of `v` in `Node` (`offsetof(Node, v) + offsetof(v, X)`)
- **stride**: The offset in each `Node` to the pointer to the next `Node` (`offsetof(Node, n)`) 
- **listtype**: 0 (`VertexListType::Array`) 
- **indirection**: 1
- **pointer_offset**: 0 {`Nodes` hold coodinates directly or via composed `Vertex`}


## Linked list of pointers to structures

We can have a linked lists each node does not directly hold our vertex, but rather point to it.
`n1 -> [pre1, p1, inter1, n2, post1]`, where `p1 -> v1`.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first node in the linked list (`&n1`)
- **structure_offset**: The distance in bytes from the start of structure with vertices to its first coordinate (`offsetof(Vertex, X)`).
- **stride**: Offset from `Node` to its pointer to next `Node` (`offsetof(Node, n)`)
- **listtype**: 1 (`VertexListType::LinkedList`) 
- **indirection**: 1
- **pointer_offset**: The distance in bytes from the start of `Node` to its pointer to the `Vertex` (`offsetof(Node, p)`)



