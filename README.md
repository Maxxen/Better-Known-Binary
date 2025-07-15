<div align="center">
  <picture>
    <img alt="BKB Logo" src="logo.png" height="300">
  </picture>
</div>
<br>


# Better Known Binary (BKB)

A proposal for a successor format to the "Well Known Binary" (WKB) encoding of geometries, designed for simpler and more efficient geospatial processing in the context of modern data systems.

## Overview

The "Well Known Binary" (WKB) format is widely used for serializing "Simple Features" style geometries into binary. It remains even more relevant today as its ubiquity has made it part of modern standards that deal with large-scale data systems, such as `Parquet`, `GeoParquet`, `Iceberg V3` and `Spark`. However, while WKB is well suited as a transport format, it has some deficiencies that make it less suitable as an execution format in high-performance analytical processing.

- No standardized way to handle `EMPTY` geometries, specifically for `POINT EMPTY`.
- Supporting big-endian byte order is not relevant in most modern systems and greatly complicates implementations.
- Unaligned coordinate data prevents zero-copy access. While unaligned memory access itself is less of a performance concern on modern hardware, it still constrains implementations in environments where unaligned access through pointers is considered undefined behavior (e.g., C/C++, Rust).

The Better Known Binary (BKB) format aims to address these issues by providing a more straightforward and efficient way to encode geometries, while ensuring that it can be gradually adopted on top of WKB.

## Format

### Geometry Header
BKB supports the 7 standard geometry types, that is:

- `POINT = 1`
- `LINESTRING = 2`
- `POLYGON = 3` 
- `MULTIPOINT = 4`
- `MULTILINESTRING = 5`
- `MULTIPOLYGON = 6`
- `GEOMETRYCOLLECTION = 7`

Each BKB geometry encoded geometry always starts with the following 8-byte "header" structure:

```
1 byte: 0x02                // magic byte indicating BKB format
1 byte: 0x01                // reserved, must be 0x01
1 byte: <flags>             // 0x00 for no flags, 0x01 for Z, 0x02 for M, 0x03 for ZM
1 byte: <geometry type>     // 0x01 for POINT, 0x02 for LINESTRING, etc.
4 byte: <count>             // number of parts/vertices (N)
...
```

#### 1st byte: BKB identifier magic byte

The "magic byte" 0x02 at the first byte indicates that this is a BKB format. This allows for easy identification of the BKB encoding and ensures that parsers can distinguish it from the WKB format, which always starts with 0x00 or 0x01 based on the byte order. An existing robust WKB parser should therefore not misinterpret BKB data, as it should reject the first byte as an invalid byte order indicator. Similarly, the other way around, A BKB parser can identify a WKB-encoded geometry and "fall back" to WKB parsing.

#### 2nd byte: reserved

After the magic byte, the second byte is reserved and must always be 0x01. A future iteration of the BKB format may use this byte to indicate additional features or flags, but for now, parsers should check that this is 0x01 to ensure compatibility. Consider this byte as a "version" indicator for the BKB format.

#### 3rd byte: flags

The third byte contains flags that indicate whether the geometry has Z (3D) or M (measure) coordinates. The flags are encoded as follows:

- `0x00`: No Z or M coordinates
- `0x01`: Z coordinates present
- `0x02`: M coordinates present
- `0x03`: Both Z and M coordinates present

More flags may be added in the future, but parsers should ignore any flags they do not recognize.

#### 4th byte: geometry type

The fourth byte indicates the geometry type, encoded as an integer from 1 to 7, corresponding to the standard geometry types listed above. 0 corresponds to an invalid geometry type and should be treated as an error. Further types may be added in the future, but parsers should reject any type they do not recognize.

#### 5th to 8th byte: count

After the header, the next 4 bytes contain the `count` field, which indicates the number of parts or vertices in the geometry body. The meaning of this field depends on the geometry type.

### The Geometry Body

There are two main cases for the geometry body: "single part" geometries and "multi part" geometries.
`POINT` and `LINESTRING` geometries are considered "single part" geometries, while `POLYGON`, `MULTIPOINT`, `MULTILINESTRING`, `MULTIPOLYGON`, and `GEOMETRYCOLLECTION` are considered "multi part" geometries.

#### Single Part Geometries

If the geometry is a "single part" geometry, i.e., `POINT` and `LINESTRING`, the `count` field specifies the number of vertices that follow. Each vertex is encoded as a pair (or triple, or quadruple, depending on if the Z and M flags are set) of IEEE 754 double-precision floating-point numbers (8 bytes each). The vertices are stored in a flat sequence, with interleaved coordinate components, so that the X value is followed by the Y value, and so on. For example, a `POINT` with vertices (1.0, 2.0) would be encoded as

```
1 byte: 0x02                // magic byte indicating BKB format
1 byte: 0x01                // reserved
1 byte: 0x00                // flags
1 byte: 0x01                // geometry type: POINT
4 byte: 0x00000001          // count: 1 part (vertex)
8 byte: 0x3FF0000000000000  // X coordinate: 1.0
8 byte: 0x4000000000000000  // Y coordinate: 2.0
```

If the geometry has Z or M coordinates, the additional coordinates are appended after the Y coordinate. For example, a `POINT` with Z coordinate 3.0 would be encoded as:

```
1 byte: 0x02                // magic byte indicating BKB format
1 byte: 0x01                // reserved
1 byte: 0x01                // flags: Z coordinate present
1 byte: 0x01                // geometry type: POINT
4 byte: 0x00000001          // count: 1 part (vertex)
8 byte: 0x3FF0000000000000  // X coordinate: 1
8 byte: 0x4000000000000000  // Y coordinate: 2
8 byte: 0x4000000000000000  // Z coordinate: 3
```

Because all geometry parts have a `count` field, the BKB format can also represent `EMPTY` geometries in a standardized way. For example, `POINT EMPTY` is represented as a `POINT` with zero parts:

```
1 byte: 0x02                // magic byte indicating BKB format
1 byte: 0x01                // reserved
1 byte: 0x00                // flags
1 byte: 0x01                // geometry type: POINT
4 byte: 0x00000000          // count: 0 parts (vertices)
```

#### Multi Part Geometries

Otherwise, if the geometry is a "multi part" geometry, i.e., `POLYGON`, or any of the collection types (`MULTIPOINT`, `MULTILINESTRING`, `MULTIPOLYGON`, `GEOMETRYCOLLECTION`), the format continues recursively with the same header structure for each part, followed by the data of each part (either more geometries or coordinates).

Example of a `MULTIPOINT` with two points:
```
1 byte: 0x02                // magic byte indicating BKB format
1 byte: 0x01                // reserved
1 byte: 0x00                // flags
1 byte: 0x04                // geometry type: MULTIPOINT
4 byte: 0x00000002          // count: 2 parts (sub-geometries, in this case POINTs)
    
    // First part (POINT)
    1 byte: 0x02                // magic byte indicating BKB format
    1 byte: 0x01                // reserved
    1 byte: 0x00                // flags
    1 byte: 0x01                // geometry type: POINT
    4 byte: 0x00000001          // count: 1 part
    8 byte: 0x3FF0000000000000  // X coordinate:
    
    //  Second part (POINT)
    1 byte: 0x02                // magic byte indicating BKB format
    1 byte: 0x01                // reserved
    1 byte: 0x00                // flags
    1 byte: 0x01                // geometry type: POINT
    4 byte: 0x00000001          // count: 1 part
    8 byte: 0x4000000000000000  // Y coordinate:
```

`POLYGON` geometries are encoded similarly, with each ring represented as a separate part of type `LINESTRING`. The first part is the outer ring, followed by any inner rings (holes).

Example of a `POLYGON` with one outer ring and one inner ring:
```
1 byte: 0x02                // magic byte indicating BKB format
1 byte: 0x01                // reserved
1 byte: 0x00                // flags
1 byte: 0x03                // geometry type: POLYGON
4 byte: 0x00000002          // count: 2 parts
    
    // Outer ring (LINESTRING)
    1 byte: 0x02                // magic byte indicating BKB format
    1 byte: 0x01                // reserved
    1 byte: 0x00                // flags
    1 byte: 0x02                // geometry type: LINESTRING
    4 byte: 0x00000004          // count: 4 parts
    8 byte: 0x3FF0000000000000  // X coordinate
    8 byte: 0x4000000000000000  // Y coordinate
    8 byte: 0x3FF0000000000001  // X coordinate
    8 byte: 0x4000000000000001  // Y coordinate
    8 byte: 0x3FF0000000000002  // X coordinate
    8 byte: 0x4000000000000002  // Y coordinate
    8 byte: 0x3FF0000000000003  // X coordinate
    8 byte: 0x4000000000000003  // Y coordinate
    
    // Inner ring (LINESTRING)
    1 byte: 0x02                // magic byte indicating BKB format
    1 byte: 0x01                // reserved
    1 byte: 0x00                // flags
    1 byte: 0x02                // geometry type: LINESTRING
    4 byte: 0x00000003          // count: 3 parts
    8 byte: 0x3FF0000000000004  // X coordinate
    8 byte: 0x4000000000000004  // Y coordinate
    8 byte: 0x3FF0000000000005  // X coordinate
    8 byte: 0x4000000000000005  // Y coordinate
    8 byte: 0x3FF0000000000006  // X coordinate
    8 byte: 0x4000000000000006  // Y coordinate
```

The benefit of having the sub-geometries encoded in the same way as the main geometry is that it allows for a consistent and straightforward parsing strategy. Each part can be processed independently, and the structure remains uniform across different geometry types. Additionally, it makes it possible to concatenate geometries into collections (e.g. `ST_Collect`, `ST_MakePolygon`), without actually needing to parse the individual geometries.

## Key Features

- **Simplicity and Efficiency**

    BKB can be written and read streaming in a single pass. Since every geometry part starts with an aligned 8-byte header, all vertices are guaranteed to be aligned on 8-byte boundaries as well, allowing for efficient memory access and zero-copy processing in many programming languages and environments. Additionally, the generalized structure of "single-part" and "multi-part" geometries enables many algorithms to be implemented without having to handle each type explicitly, reducing complexity.

- **Standardized Representation of `EMPTY` Geometries**

    BKB provides a standardized way to represent `EMPTY` geometries, specifically `POINT EMPTY`, which is now represented as a `POINT` with zero parts.

- **Small metadata**

    All the information required to identify a BKB geometry: the BKB identifier byte (0x02), the geometry type and the Z/M flags fit within the first 4 bytes, making it suitable for use in systems that utilize "german strings" or similar techniques to process large binary blobs.

## Limitations

- **Only supports little-endian byte order**

Not supporting big-endian byte order simplifies the implementation and is sufficient for most modern systems, but it may limit interoperability with systems that run on big-endian architectures. However, this information could be provided "out-of-band" if necessary, such as through metadata in a data catalog or schema definition, rather than being part of the binary format itself.
    
- **Slightly larger size compared to WKB for some geometries**

Notably `POINT` geometries are 3 bytes larger than WKB (about ~14% larger for a normal point) due to the additional "count" field, but `POINT EMPTY` geometries are actually smaller (compared to the non-standard WKB convention of representing empty points as points of `NaN`). `POLYGON` geometries also gain 4 bytes of overhead for each ring, but in practice, most `POLYGONS` tend to contain a lot of vertices, making the extra 4 bytes less significant as a whole. Additionally, most large-scale data systems apply some sort of compression at the storage level, which makes the size difference even less of a concern.

## Example code

TODO

## Implementations

None, but the format is heavily based on the internal format used in DuckDBs spatial extension and may be a candidate for inclusion in the future. The spatial extensions format is itself heavily inspired from the PostGIS "GSERIALIZED" internal format. The ideas and changes taken from spatial's format and formalized in BKB are thus based on practical implementation experience.
