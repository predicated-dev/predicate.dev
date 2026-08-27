---
version: "1.0.0-beta.2"
latest: true
---
## Summary
A cross-platform way to describe vertex memory layouts so that APIs can interoperate without requiring knowledge of the underlying data structures.

## Terms
1. **Descriptor**: A set of values that describe an associated dataset.
1. **Coordinate**: A numeric value member of a set of values that define a specific location.
1. **Vertex**: A structure that holds the **Coordinate** values defining a point. It may contain other fields.
1. **Dimensionality**: Number of **Coordinates** required to define the location of a **Vertex**.  
1. **Pointer**: A platform native integer value representing a memory address
1. **Offset**: A numeric value added or subtracted from a **Pointer** to retrieve data at a relative memory location.
1. **Array**: A contiguous block of memory with elements directly after each other
1. **Linked List**: A chain of **Nodes**
1. **Node**: A structure with a **Pointer** to another **Node**
1. **Stride**: The distance in bytes from element to element in an **Array**, in case of a **Linked List** the distance from a **Node** start to the **Pointer** it holds for the next **Node** 
1. **Axis**: The set of **Coordinate** values at one ordinal position across every **Vertex** in a list. A 2-dimensional list has two **Axes**, all `X` values and all `Y` values.
1. **Axis Array**: An **Array** holding the **Coordinates** of a single **Axis**.
1. **Structure of Arrays**: A layout holding one **Axis Array** per **Axis** rather than one **Array** of **Vertices**.
1. **Axis Stride**: The value describing how to step from one **Axis** to the next, whether that step is from **Coordinate** to **Coordinate** within a **Vertex**, from **Axis Array** to **Axis Array**, or from **Pointer** to **Pointer**.
1. **Axis Step**: The byte distance from one **Axis** to the next, derived from **Axis Stride** and the layout it describes.


## The Vertex List Descriptor

### Memory Layout

```
0   uint8   version
1   uint8   data_type
2   uint8   list_type
3   uint8   indirection
4   uint8   dimensionality
5   uint8   coordinate_system
6   uint16  stride
8   uint64  count
16  void*   data
20  uint32  _padding_    (32-bit only)
24  uint16  structure_offset
26  uint16  pointer_offset
28  uint16  axis_stride
30  uint16  _reserved_
```

Consider that the size of `void*` at offset 16 may be 32-bit on some platforms, but we need to ensure that `structure_offset` is at byte offset 24. One way to ensure this is to add padding for 32-bit (see `_padding_` below).

Fields are ordered by descending alignment requirement, so each one falls on its natural boundary and the descriptor is exactly 32 bytes on every platform with no implicit padding. This ordering is not optional.

For brevity we will call a structure that holds coordinates a `Vertex`, but it may be more complex than just the simple definition of a spatial point position.

The descriptor carries two independent steps. `stride` is the step taken to advance the vertex index, and `axis_stride` describes the step taken to advance the axis index. What distinguishes an array of `Vertices` from a Structure of Arrays is only which of those two steps spans the data.

The following values are provided by the descriptor. The byte offsets are in brackets `[]`:

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
    - 2: Structure of Arrays
-  [3] `indirection` : an 8-bit integer value with the level of indirection to the coordinate information (how many pointers to follow).
    - 0: Array or linked list elements hold `Vertices` directly. For a Structure of Arrays the `Axis Arrays` are held directly in the block at `data`
    - 1: Array or linked list elements hold pointers to `Vertices`. For a Structure of Arrays the storage at `data` holds one pointer per `Axis Array`
-  [4] `dimensionality`: an 8-bit integer value indicating the number of coordinates. May pass 0 if dimensionality is known from API naming, context, protocol. etc. For a Structure of Arrays it is also the number of `Axis Arrays`.
-  [5] `coordinate_system`: an 8 bit value from an enumeration of coordinate systems. 
     - 0: Known. Either from API call naming, protocol or context
     - 1: Cartesian
     - 2: Polar
     - 3: Cylindrical    
-  [6] `stride`: a 16-bit unsigned integer size, the step taken to advance from one `Vertex` to the next
    - For arrays: The distance in bytes from element to element in bytes (also the size of elements in the array). 
    - For linked lists: The bytes offset from the start of the node to the pointer of the next node. It is **not** the size of the node.
    - For a Structure of Arrays: The distance in bytes from element to element within a single `Axis Array`. For a tightly packed array of coordinates it is the size of one coordinate.
-  [8] `count`: A 64-bit integer value holding the number of elements in the vertex array. For a Structure of Arrays it is the number of `Vertices`, which is also the number of elements in each `Axis Array`.
- [16] `data`: A platform specific pointer sized variable (32-bit or 64-bit). Pointer to the first element of an array or the first node of a linked list. For a Structure of Arrays it points to the storage holding the `Axis Arrays`, or the pointers to them.
- [20] `_padding_`: 32-bit pad. Only on 32-bit platforms. This forces alignment of the next field `structure_offset` to be at offset 24
- [24] `structure_offset`: a 16-bit unsigned integer offset (in bytes) from the start of a `Vertex` to its first coordinate. For a Structure of Arrays it is the offset from the start of an `Axis Array` element to its coordinate, which is 0 where the `Axis Array` holds coordinates directly
- [26] `pointer_offset`: a 16-bit unsigned integer offset (in bytes). If `indirection` is 1 then it is the offset in bytes from the start of the Array element or Linked List `Node` to the `Vertex` pointer. For a Structure of Arrays it is the offset from `data` to the first `Axis Array`, or to the first pointer to an `Axis Array` where `indirection` is 1. In all other cases set it to 0
- [28] `axis_stride`: a 16-bit unsigned integer describing the step from one `Axis` to the next. A value of 0 always means the `Axes` are as tightly packed as the layout allows and the step is derived. Its meaning is determined by `list_type` and `indirection`:
     - `list_type` 0 or 1 (Array or Linked List): the distance in bytes from one `Coordinate` of a `Vertex` to the next. A value of 0 means the coordinates are adjacent and the distance is the size of one coordinate
     - `list_type` 2 with `indirection` 1: the distance in bytes from one pointer to an `Axis Array` to the next. A value of 0 means the pointers are adjacent and the distance is the size of a pointer (`sizeof(void*)`)
     - `list_type` 2 with `indirection` 0: the distance in bytes from the end of one `Axis Array` to the start of the next, so that the distance from the start of one to the start of the next is `count * stride + axis_stride`. A value of 0 means the `Axis Arrays` are adjacent
- [30] `_reserved_`: 16-bit pad reserved for a future field. Writers set it to 0 and readers ignore it

### Resolving a coordinate

The address arithmetic below is normative. It is the definition of what the offset and stride fields mean, not a suggested implementation, and where the prose above and the arithmetic here disagree the arithmetic governs.

For axis `a` in `[0, dimensionality)` and vertex `i` in `[0, count)`, where `T` is the type given by `data_type`, `base` is the address held in `data`, and all arithmetic is on byte addresses.

The step from one axis to the next is derived once, from the fields that select its form:

```
list_type 0 or 1              axis_step = (axis_stride != 0) ? axis_stride : sizeof(T)
list_type 2, indirection 1    axis_step = (axis_stride != 0) ? axis_stride : sizeof(void*)
list_type 2, indirection 0    axis_step = count * stride + axis_stride
```

#### Array (`list_type` 0)

```
elem   = base + i * stride
vertex = (indirection == 0) ? elem : *(void**)(elem + pointer_offset)
coord  = *(T*)(vertex + structure_offset + a * axis_step)
```

#### Linked List (`list_type` 1)

```
node(0)   = base
node(i+1) = *(void**)(node(i) + stride)
vertex    = (indirection == 0) ? node(i) : *(void**)(node(i) + pointer_offset)
coord     = *(T*)(vertex + structure_offset + a * axis_step)
```

#### Structure of Arrays (`list_type` 2)

```
axis(a) = (indirection == 0) ? base + pointer_offset + a * axis_step
                             : *(void**)(base + pointer_offset + a * axis_step)
coord   = *(T*)(axis(a) + i * stride + structure_offset)
```

The two families differ only in which index selects the block that indirection is applied to. In an array or list, `i * stride` locates a `Vertex` and `a * axis_step` reaches within it. In a Structure of Arrays, `a * axis_step` locates an `Axis Array` and `i * stride` reaches within it. A reader that walks whole vertices will find the first form cheaper and a reader that walks whole axes will find the second cheaper, which is the only reason both exist.

Behaviour is undefined where `i` is not less than `count` or `a` is not less than `dimensionality`.

### Sample C++ implementation

{{< codebox filename="VertexArrayDescriptor.h" lang="cpp" >}}
#include <cstdint>
#include <cstddef>

struct VertexArrayDescriptor {
    std::uint8_t  version = 1;
    std::uint8_t  data_type = 0;
    std::uint8_t  list_type = 0;
    std::uint8_t  indirection = 0;
    std::uint8_t  dimensionality = 0;
    std::uint8_t  coordinate_system = 0;
    std::uint16_t stride = 0;
    std::uint64_t count = 0;
    void*         data = nullptr;
#if UINTPTR_MAX == 0xFFFFFFFFULL
    std::uint32_t _padding_ = 0;
#endif
    std::uint16_t structure_offset = 0;
    std::uint16_t pointer_offset = 0;
    std::uint16_t axis_stride = 0;
    std::uint16_t _reserved_ = 0;
};

static_assert(offsetof(VertexArrayDescriptor, version)           ==  0);
static_assert(offsetof(VertexArrayDescriptor, data_type)         ==  1);
static_assert(offsetof(VertexArrayDescriptor, list_type)         ==  2);
static_assert(offsetof(VertexArrayDescriptor, indirection)       ==  3);
static_assert(offsetof(VertexArrayDescriptor, dimensionality)    ==  4);
static_assert(offsetof(VertexArrayDescriptor, coordinate_system) ==  5);
static_assert(offsetof(VertexArrayDescriptor, stride)            ==  6);
static_assert(offsetof(VertexArrayDescriptor, count)             ==  8);
static_assert(offsetof(VertexArrayDescriptor, data)              == 16);
static_assert(offsetof(VertexArrayDescriptor, structure_offset)  == 24);
static_assert(offsetof(VertexArrayDescriptor, pointer_offset)    == 26);
static_assert(offsetof(VertexArrayDescriptor, axis_stride)       == 28);
static_assert(offsetof(VertexArrayDescriptor, _reserved_)        == 30);
static_assert(sizeof(VertexArrayDescriptor)                      == 32);

{{< /codebox >}}

Implementations should add static asserts to ensure alignment and may add strong types via getters and setters. The asserts above are the whole set rather than a representative sample, since it is the field order that produces the layout and a reordering will otherwise pass unnoticed.

## Constraints
- **Coordinates** are all of the same single datatype as specified by the **descriptor**. For instance, if we use polar **coordinates** we can't have 32-bit integer values for the radial **coordinate** and 64-bit floating values for the angular **coordinate**. 
- **Coordinate** values for a specific **Vertex** are stored at a uniform spacing given by **Axis Stride**, in ascending memory order and in axis order. Adjacent is the default and is what a **Axis Stride** of 0 selects (e.g., `double x, y` or `long coords[2]`). This does not apply to a **Structure of Arrays**, where the **Coordinates** of one **Vertex** are by definition distributed across the **Axis Arrays**, one to each.
- Only Structures that can be described with offset and stride values less than 2^16 (65,536) are supported.
- The **Axis Arrays** of a **Structure of Arrays** must be uniformly spaced and in ascending memory order, since a single **Axis Stride** describes the distance between all of them. Where they are located by **Pointers**, those **Pointers** must be contiguous in the **Array** or structure that holds them, or each be a member of an element of an **Array** of equally sized structures. **Pointers** held at irregular spacing are not expressible.
- Every **Axis Array** of a **Structure of Arrays** holds at least `count` elements and shares the same **Stride** and `structure_offset` as the others. Where they are held in a single block and an **Axis Array** holds more than `count` elements, the surplus is absorbed by **Axis Stride** along with any deliberate padding, and where that total exceeds 65,535 bytes the **Axis Arrays** must be located by **Pointers** with `indirection` set to 1.
- The value of the next-node **Pointer** held by the last **Node** of a **Linked List** is unspecified. `count` bounds the walk.
- All steps are forward. A list traversed in descending memory order, or **Axes** laid out in descending order, is not expressible.

## Common Vertex List types
A non-exhaustive list of common representations of coordinate data.

In illustrations below:
- `n`: Number of vertices  
- `p1`, `p2`, etc:  a pointer to a structure that holds coordinates (`Vertex`).
- `{}`: the bounds of a structure. 
- `[]`: the bounds of an array.
- `()`: the bounds of the contiguous set of coordinates that define our vertex. Where the coordinates of a vertex are spaced rather than adjacent they are shown individually instead
- `(x1, y1)`, `(x2, y2)`, etc: denotes a coordinate group defining a vertex (only two dimensions shown in examples, but any dimensionality up to 255 is supported). Coordinate groups may be any form of contiguous values: a series of fields, an array of values, a structure with fields, etc.
- `n1`, `n2`, etc. denotes pointers to nodes in a linked list
- `pre_1`, `post_1`, `inter_1`, etc. denotes optional extraneous data and padding
- `->` shows the value at the memory address held by the pointer (`pointer -> value`) 
- `pX`, `pY`, etc. denotes a pointer to the `Axis Array` of the named axis

Note that 1-based subscript is used in illustrations (simplifies the syntax for the last element in the series).

In all cases below, will the vertex array descriptor have:

- **version**: Default 1 from definition and should not be altered
- **count**: Number of elements in the array or list
- **data_type**: The type of coordinates 
- **dimensionality**: 2 for all examples below (we only illustrate with `X`, `Y` Cartesian coordinates)

 ## Structure holding Coordinates (`Vertex`)
 Even though the structures that hold coordinates can be more complex than simple vertices we will still call them `Vertices` for simplicity. A `Vertex` is a structure of the form `{pre, (x, y), post}` where `pre` and `post` are optional data before and after the coordinates (`X` and `Y`).

### Arrays of Vertices

Contiguous memory of `Vertices` of the form `[v1, v2, ... vn]`. If we expand with our definition of `Vertex` the memory can be seen like this: `[{pre_1, (x1, y1), post_1}, {pre_2, (x2, y2), post_2}, ... {pre_n, (xn, yn), post_n}]`


To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first `Vertex` in array (`&v1`)
- **structure_offset**: The distance in bytes from the start of a `Vertex` to its first coordinate (`offsetof(Vertex, X)`)
- **stride**: The number of bytes from element to element in the array (`sizeof(Vertex)`)
- **list_type**: 0 (`VertexListType::Array`) 
- **indirection**: 0 (no pointers to follow)
- **axis_stride**: 0 (the coordinates are adjacent)

### Arrays of Vertices with spaced Coordinates

The coordinates of a `Vertex` need not be adjacent, only uniformly spaced. A structure carrying a field between its coordinates, such as `{pre_1, x1, inter_1, y1, post_1}`, is described by giving that spacing as **axis stride**. Assume a type `Vertex` where `X` and `Y` are separated by other members.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first `Vertex` in array (`&v1`)
- **structure_offset**: The distance in bytes from the start of a `Vertex` to its first coordinate (`offsetof(Vertex, X)`)
- **stride**: The number of bytes from element to element in the array (`sizeof(Vertex)`)
- **list_type**: 0 (`VertexListType::Array`) 
- **indirection**: 0
- **axis_stride**: The distance in bytes from one coordinate to the next (`offsetof(Vertex, Y) - offsetof(Vertex, X)`)

The same applies to any of the array or linked list forms below, and the spacing between coordinates is independent of the spacing between elements.

### Arrays of Pointers to Vertices

Pointers adjacent in memory with a stride from element to element as size of pointer: `[p1, p2 ..., pn]` where `p1`, `p2`, etc. are pointers to `Vertices` e.g. `p1 -> v1`, `p2 -> v2`, etc.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first element in array (`&p1`)
- **structure_offset**: The distance in bytes from the start of `Vertex` to its first coordinate (`offsetof(Vertex, X)`)
- **stride**: The size of a pointer (`sizeof(void*)`)
- **list_type**: 0 (`VertexListType::Array`) 
- **indirection**: 1
- **pointer_offset**: 0 (points directly to structures)
- **axis_stride**: 0 (the coordinates are adjacent)

### Arrays of structures with pointers to Vertices

Arrays hold structures that in turn hold pointers to `Vertices`. These structures can hold other data as well. 
For instance we could have `{pre1, p1, post1}, {pre2, p2, post2},...`. 

Assume the elements are of type `Elem` with pointer `p` to Vertices, e.g. `Elem1.p -> v1`, `Elem2.p -> v2`, etc.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first element in array (`&Elem1`)
- **structure_offset**: The distance in bytes from the start of `Vertex` to its first coordinate (`offsetof(Vertex, X)`)
- **stride**: The size of an array element (`sizeof(Elem)`)
- **list_type**: 0 (`VertexListType::Array`) 
- **indirection**: 1
- **pointer_offset**: The distance in bytes from the start of `Elem` to its pointer to a `Vertex` (`offsetof(Elem, p)`)
- **axis_stride**: 0 (the coordinates are adjacent)


## Linked list of structures

Linked lists are chains of pointers to `Nodes`. `Nodes` can be located anywhere in memory, each node has a pointer to the next `Node` in the list and also holds the coordinate values of a specific vertex. Coordinates are held directly by the `Node` or in a nested `Vertex` structure. 

For example: `n1 -> {pre_1, (x1, y1), inter_1, n2, post_1}`, `n2 -> {pre_2, (x2, y2), inter_2, n3, post_2}`, the location of the pointer to the next node is specified by the **stride**, and the **structure offset**, as before, is the distance from the start of the structure to the first coordinate.

Please note that the pointer to next `Node` could also appear before the vertex, for instance:  `n1 -> [pre_1, n2, inter_1, (x1, y1), post_1]`. 

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first node in the linked list (`&n1`)
- **structure_offset**: The distance in bytes from the start of structure with vertices to its first coordinate (`offsetof(Node, X)`). If coordinates are in a nested `Vertex` structure `v` then it is calculated as the offset of `X` in `v` plus offset of `v` in `Node` (`offsetof(Node, v) + offsetof(v, X)`)
- **stride**: The offset in each `Node` to the pointer to the next `Node` (`offsetof(Node, n)`) 
- **list_type**: 1 (`VertexListType::LinkedList`) 
- **indirection**: 0 (each `Node` holds coordinates directly)
- **pointer_offset**: 0 (`Nodes` hold coordinates directly or via composed `Vertex`)
- **axis_stride**: 0 (the coordinates are adjacent)


## Linked list of pointers to structures

We can have a linked list where each node does not directly hold our vertex, but rather points to it.
`n1 -> {pre1, p1, inter_1, n2, post_1}`, where `p1 -> v1`.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first node in the linked list (`&n1`)
- **structure_offset**: The distance in bytes from the start of structure with vertices to its first coordinate (`offsetof(Vertex, X)`).
- **stride**: Offset from `Node` to its pointer to next `Node` (`offsetof(Node, n)`)
- **list_type**: 1 (`VertexListType::LinkedList`) 
- **indirection**: 1
- **pointer_offset**: The distance in bytes from the start of `Node` to its pointer to the `Vertex` (`offsetof(Node, p)`)
- **axis_stride**: 0 (the coordinates are adjacent)


## Structure of Arrays

Rather than one array of `Vertices`, coordinates are held in one `Axis Array` per axis, and the coordinates of a single vertex are no longer adjacent. Vertex `i` is assembled from element `i` of each `Axis Array`.

For two Cartesian dimensions the coordinate data is `X = [x1, x2, ... xn]` and `Y = [y1, y2, ... yn]`. What differs between the forms below is only how those two arrays are located, which is what **axis stride** describes.

### Arrays of Pointers to Axis Arrays

Pointers adjacent in memory, one per axis: `[pX, pY]` where `pX -> [x1, x2, ... xn]` and `pY -> [y1, y2, ... yn]`. This is the form of a `double*[2]`.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first element in the array of axis pointers (`&pX`)
- **structure_offset**: 0 (`Axis Arrays` hold coordinates directly)
- **stride**: The size of a coordinate (`sizeof(double)`)
- **list_type**: 2 (`VertexListType::StructureOfArrays`) 
- **indirection**: 1 (one pointer to follow per axis)
- **pointer_offset**: 0 (the first axis pointer is at `data`)
- **axis_stride**: 0 (the pointers are adjacent, so the distance is `sizeof(void*)`)

### Structures with pointers to Axis Arrays

A structure holds the pointers to the `Axis Arrays` and may hold other data before, between or after them: `{pre, pX, pY, post}`. Assume the structure is of type `Soa` with pointers `x` and `y`, e.g. `Soa.x -> [x1, x2, ... xn]`.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to the structure (`&Soa`)
- **structure_offset**: 0 (`Axis Arrays` hold coordinates directly)
- **stride**: The size of a coordinate (`sizeof(double)`)
- **list_type**: 2 (`VertexListType::StructureOfArrays`) 
- **indirection**: 1
- **pointer_offset**: The distance in bytes from the start of `Soa` to its first axis pointer (`offsetof(Soa, x)`)
- **axis_stride**: The distance in bytes from one axis pointer to the next (`offsetof(Soa, y) - offsetof(Soa, x)`), or 0 where they are adjacent

### Adjacent Axis Arrays in a single block

All axes are held in one block of memory, one `Axis Array` after the other, and no pointers are involved: `[(x1, x2, ... xn)(y1, y2, ... yn)]`. The block may hold a header before the first coordinate.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to the block (`&x1`, or the start of the header)
- **structure_offset**: 0 (`Axis Arrays` hold coordinates directly)
- **stride**: The size of a coordinate (`sizeof(double)`)
- **list_type**: 2 (`VertexListType::StructureOfArrays`) 
- **indirection**: 0 (no pointers to follow)
- **pointer_offset**: The distance in bytes from the start of the block to the first coordinate, or 0 where there is no header
- **axis_stride**: 0 (the `Axis Arrays` are adjacent, so the distance from one to the next is `count * stride`)

### Spaced Axis Arrays in a single block

As above, but the `Axis Arrays` are deliberately spaced further apart than their contents require, for instance so that each begins on a cache line: `[(x1, ... xn) post_x (y1, ... yn) post_y]`.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to the block
- **structure_offset**: 0 (`Axis Arrays` hold coordinates directly)
- **stride**: The size of a coordinate (`sizeof(double)`)
- **list_type**: 2 (`VertexListType::StructureOfArrays`) 
- **indirection**: 0
- **pointer_offset**: The distance in bytes from the start of the block to the first coordinate
- **axis_stride**: The distance in bytes from the end of one `Axis Array` to the start of the next, which is the size of `post_*`. The `Axis Arrays` themselves may be of any size, since only the padding between them has to fit in 16 bits (this reprenntation limits padding to 65,535 bytes per axis). 

### Axis Arrays of structures

An `Axis Array` may hold structures rather than coordinates directly, in which case `structure_offset` locates the coordinate within an element exactly as it does for an array of `Vertices`: `pX -> [{pre_1, x1, post_1}, {pre_2, x2, post_2}, ...]`. Assume the elements are of type `AxElem` with coordinate `v`.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first element in the array of axis pointers (`&pX`)
- **structure_offset**: The distance in bytes from the start of an element to its coordinate (`offsetof(AxElem, v)`)
- **stride**: The size of an element (`sizeof(AxElem)`)
- **list_type**: 2 (`VertexListType::StructureOfArrays`) 
- **indirection**: 1
- **pointer_offset**: 0
- **axis_stride**: 0 (the axis pointers are adjacent)

### Axis Arrays over an array of Vertices

An `Axis Array` need not be a dedicated allocation. Given an existing array of `Vertices` `[{pre_1, (x1, y1), post_1}, {pre_2, (x2, y2), post_2}, ...]`, pointing each axis pointer at the corresponding coordinate of the first element describes the same data as a Structure of Arrays, with a **stride** that steps over the rest of each `Vertex`. This lets a producer present an array of `Vertices` to a reader of a Structure of Arrays without copying it.

Note that this form requires an array of axis pointers to exist. Where the consumer has no preference, the same data is described as an Array with no additional storage at all, and where the coordinates within the `Vertex` are spaced rather than adjacent the **axis stride** of that Array form carries the spacing. Use this form when the consumer specifically wants Structure of Arrays.

To Define the Vertex Array Descriptor we provide
- **data**: Pointer to first element in the array of axis pointers, where `pX` is `&v1.X` and `pY` is `&v1.Y`
- **structure_offset**: 0 (the axis pointers already address coordinates)
- **stride**: The size of a `Vertex` (`sizeof(Vertex)`)
- **list_type**: 2 (`VertexListType::StructureOfArrays`) 
- **indirection**: 1
- **pointer_offset**: 0
- **axis_stride**: 0 (the axis pointers are adjacent)


## Notes for implementers

This section is not normative. Nothing here changes the meaning of a descriptor, and a reader that follows the arithmetic above is correct whether or not it follows any of this.

**Derive once.** `axis_step` and the resolved coordinate size depend only on the descriptor, not on `a` or `i`. Resolve them when the descriptor is accepted rather than per coordinate.

**Zero is the fast path.** A writer that can express a layout either with 0 or with the equivalent explicit value should write 0, and a reader must treat the two as identical. Where `axis_stride` and `structure_offset` are both 0 and `stride` equals `dimensionality` times the coordinate size, an Array of `Vertices` is a packed array of coordinates and may be consumed as one. Where `axis_stride`, `structure_offset` and the difference between `stride` and the coordinate size are all 0, each `Axis Array` of a Structure of Arrays is likewise a packed array.

**Validate before dereferencing.** A reader is receiving addresses from another module. Check the `version` against what it implements, that `data` is not null, that `dimensionality` matches what the call requires where it is not 0, and that `data_type` and `list_type` are values it handles. A descriptor it cannot resolve should be rejected rather than partially interpreted.

**Do not assume the axes are disjoint.** Two axes may resolve into the same memory, and `stride` may be smaller than the space a coordinate group appears to occupy. The descriptor grants read access to the addresses the arithmetic produces and says nothing else about the surrounding allocation.

**Bound the walk by `count`.** This matters most for linked lists, where there is no terminator defined and the pointer held by the last node may be anything.

## Revision history

While this specification is in beta the descriptor `version` field stays at 1 and does not distinguish one beta revision from another. A descriptor is therefore not self-describing across beta revisions, and implementations tracking the beta must agree on which revision they are speaking out of band. Changes listed as breaking will cause a reader of an earlier beta revision to resolve a valid descriptor to a wrong address, or to reject a valid descriptor, with nothing in the descriptor to indicate that it has happened.

### 1.0.0-beta.2

Breaking:

- `axis_stride` under `list_type` 2 with `indirection` 0 is now the distance from the end of one `Axis Array` to the start of the next, where it was previously the distance from the start of one to the start of the next. The distance between starts is now `count * stride + axis_stride`. A value of 0 continues to mean the `Axis Arrays` are adjacent, so descriptors for the adjacent case are unaffected and only deliberately spaced blocks change.
- `axis_stride` is now meaningful under `list_type` 0 and 1, where it was previously required to be 0. It gives the distance from one `Coordinate` of a `Vertex` to the next, with 0 meaning adjacent. Writers of the previous revision set 0 and remain correct; readers of the previous revision ignore the field and will resolve every coordinate after the first to a wrong address if given a descriptor that uses it.
- `axis_stride` of 0 under `list_type` 2 with `indirection` 1 now means the axis pointers are adjacent, where the previous revision required the size of a pointer to be stated explicitly. Both forms are accepted from this revision on, and a reader must treat them as identical.
- The constraint that the `Coordinates` of a `Vertex` occupy contiguous adjacent memory is relaxed to uniform spacing, as a consequence of the above.
- The constraint capping the distance between `Axis Arrays` in a single block at 65,535 bytes is removed. Only padding between them is now subject to that limit, and the requirement to locate widely spaced `Axis Arrays` by `Pointers` now applies only where an `Axis Array` holds enough surplus elements beyond `count` to overflow it.

Clarifying, with no change to the meaning of any previously valid descriptor:

- `Resolving a coordinate` is promoted to a normative section covering all list types, rather than appearing only under Structure of Arrays, and governs where the prose describing a field disagrees with it.
- `Notes for implementers` is added and is not normative.
- The next-node `Pointer` held by the last `Node` of a `Linked List` is stated to be unspecified, and `count` to bound the walk.
- Two `Axes` are stated to be permitted to resolve into the same memory.
- All steps are stated to be forward, and descending traversal and descending axis order to be inexpressible, which is now also listed as a known limitation.
- `count` no longer restates that it is not the number of `Axes`.
- `Axis Step` is added to the terms and `Axis Stride` is redefined to cover all three of its forms.
- An example of an Array of `Vertices` whose coordinates are spaced rather than adjacent is added.

### 1.0.0-beta

Initial publication.

## Future versions

`Version` field needs to remain fixed as the first value and incremented when the memory layout or structure changes. The `Version` value should be incremented by 1 when the structure is changed.

The beta is the only period in which the meaning of a field may change without the descriptor `version` being incremented, and Revision history above records where that has happened. Once a non-beta version is published, a change to the meaning of an existing field is a breaking change in the same way that a change of layout is, because a reader of the older version will resolve a valid descriptor to the wrong address without any indication that it has done so.

Older readers may not assume the structure of a descriptor with a newer unknown version, but a best effort should be made to avoid breaking changes:
- New fields should be appended, not inserted
- Existing offsets should remain stable
- Older readers should not read beyond known fields

If a breaking change is made it should be clearly noted in an updated specification. Doing this will allow API developers to check what versions their API can support. Once a breaking change is approved, the entire structure with exception of the `Version` field may be restructured and the size of fields modified. While this is not foreseen, it cannot be precluded.

Note: Version numbers refer to the descriptor format, not the specification version

`_reserved_` at offset 30 is the only space remaining in the descriptor. A field that does not fit there requires the descriptor to grow to 40 bytes, that being the next size its alignment allows. Being the last free space, it is better spent on a flags word than on a typed field, since flag bits can be allocated one at a time across several versions while a typed field is spent all at once. A `descending` bit, for instance, would buy back reverse traversal without widening any stride to a signed type.

Known limitations that a future version may address:
- Doubly-linked lists are expressible by taking the forward pointer offset as **stride** and ignoring the backward pointer, but there is no way to state that a backward pointer exists.
- True 3-level indirection, where an element points to a pointer that points to a `Vertex`, is not expressible with the current fields.
- A Structure of Arrays whose `Axis Arrays` are located at irregular spacing needs a second offset rather than a single **axis stride**.
- All steps are unsigned and therefore forward. Descending traversal and descending axis order are not expressible.

----
Copyright 2025 Jasper Schellingerhout. All rights reserved.
