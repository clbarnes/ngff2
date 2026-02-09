# OME-Zarr specification

OME-Zarr uses the Zarr v3 data model.
A Zarr node is identified by a storage prefix and may either be a group or an array.
Groups may contain other groups and/ or arrays.
Every node has a metadata document storing information using the JSON data model.
Every metadata document has a location for attributes.

The below defines specifications for bioimaging data and associated metadata,
in terms of hierarchies of Zarr nodes and OME-Zarr metadata stored in Zarr attributes.

## Conventions

### Paths

Paths are encoded as strings.
They use `/` as a separator in the POSIX fashion.
The tree of prefixes can be traversed upwards using the `..` to mean "parent" in the POSIX fashion.

## Hierarchy specifications

### Hierarchy: Multiscale image

In OME-Zarr, a multiscale image is defined as a set of Zarr arrays within a hierarchy:

- exactly one Zarr array representing the base (highest) resolution of the imaging data
- zero or more Zarr arrays containing downscaled (lower-resolution) representations of all or part of the base data
- zero or more Zarr arrays containing labels or masks segmenting the imaging data, at any downscale level or crop

The Zarr group representing the multiscale image MUST contain an attribute with the key `"ome"`, whose value MUST be an [OME object](#object-ome).
This OME object MUST contain the `multiscales` key.

Each array representing a single scale level MUST be in a Zarr array inside this group,
and SHOULD be direct children of this group.
It is RECOMMENDED that their keys clearly describe the downscaling sequence, e.g.
`s0` for the base resolution, `s1` for the first downscale level, `s2` for the second, etc..
These arrays are referred to in the `multiscales` field of the OME object.

```text
my_image/  # attributes contain $.ome.multiscales
├── s0  # base Zarr array
├── s1  # downscaled Zarr array
└── labels/  # attributes contain $.ome.labels
    └── my_label/  # attributes contain $.ome.multiscales
        ├── lbl0  # integer base Zarr array
        └── lbl1  # integer downscaled Zarr array
```

N.B. OME-Zarr always stores images as multiscales even if only the base scale level exists.

#### Multiscales with labels

If the group contains any label multiscale images, the multiscale image group MUST contain a child group called `labels`.
The Zarr metadata of the `labels` group MUST contain an attribute with the key `"ome"`, whose value MUST be an [OME object](#object-ome).
This OME object MUST contain the `labels` key.

Label images are themselves multiscale.
These label multiscales MUST be Zarr groups inside this array;
there MAY be intermediate groups.
These label multiscales are referred to in the `labels` field of the OME object.
Zarr arrays in label multiscales MUST have an integer data type.

The OME object in the Zarr attributes of label multiscales SHOULD have the `image-label` key
(as well as the `multiscales` key).

### Hierarchy: High-Content Screening

In OME-Zarr, a high-content screen (HCS) is be represented by

- a Zarr group representing the plate
  - the Zarr metadata attributes MUST contain an [OME object](#object-ome) under the `"ome"` key
  - this OME object MUST contain the `plate` key
- Zarr groups which are direct children of the plate group, representing rows of the plate
- Zarr groups which are direct children of the row groups, representing wells on that row
  - the Zarr metadata attributes MUST contain an [OME object](#object-ome) under the `"ome"` key
  - this OME object MUST contain the `well` key
- Zarr groups which are direct children of the well groups, representing fields of view in that well
  - these fields of view are [multiscale images](#hierarchy-multiscale-image) as defined above
  - fields of view multiscale images MAY have labels

```text
my_hcs_assay/  # plate; attributes contain $.ome.plate
├── r0/
│   ├── w0/  # well; attributes contain $.ome.well
│   │   ├── f0/  # multiscale image; attributes contain $.ome.multiscales
│   │   │   ├── s0  # base Zarr array
│   │   │   ├── s1  # downscaled Zarr array
│   │   │   ├── ...
│   │   │   └── labels/  # labels container; attributes contain $.ome.labels
│   │   │       ├── my_label/  # label multiscale; attributes contain $.ome.multiscales
│   │   │       │   ├── lbl0  # integer base Zarr array
│   │   │       │   ├── lbl1  # integer downscaled Zarr array
│   │   │       │   └── ...
│   │   │       └── ...
│   │   ├── f1/
│   │   └── ...
│   ├── w1/
│   └── ...
├── r1/
└── ...
```

## Object Specifications

OME-Zarr objects are defined within the JSON data model; JSON nomenclature is generally used.
However, as JSON only defines a single `number` type, for clarity we further define

- integer: a JSON number with no decimal part
- unsigned integer: an integer with no leading `-`
- float: alias for JSON number, to distinguish numbers in which decimal parts are expected
  - Note that JSON can only represent decimal numbers; conversion to/from IEEE-754 floating point values may be lossy

### Object: OME

The OME object acts as a container for all OME-Zarr metadata and MUST be stored under the `"ome"` key in the attributes map of a Zarr node.
Different keys MAY be present depending on which type of [hierarchy](#hierarchy-specifications) is being represented.

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `version` | MUST | string | MUST have the value `"0.5"` |
| `multiscales` | MUST for [multiscale images](#hierarchy-multiscale-image) | array of [Multiscale](#object-multiscale) | |
| `labels` | MUST for [parent of label multiscales](#multiscales-with-labels) | array of string | |
| `image-label` | SHOULD for [label multiscales](#multiscales-with-labels) | array of [ImageLabel](#object-imagelabel) | |
| `plate` | MUST for [HCS plates](#hierarchy-high-content-screening) | [Plate](#object-plate) | |
| `well` | MUST for [HCS wells](#hierarchy-high-content-screening) | [Well](#object-well) | |
| `bioformats2raw.layout` | MAY | `3` | transitional |
| `series` | MAY | array of string | transitional |
| `omero` | MAY | object | transitional |

### Object: Multiscale

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `axes` | MUST | array of [Axis](#object-axis) | Axis information for this multiscale's coordinate system. |
| `datasets` | MUST | array of [Dataset](#object-dataset) | Single-scale datasets composing this multiscale dataset |
| `name` | SHOULD | string | Name of the multiscale dataset |
| `type` | SHOULD | string | Type of downscaling method |
| `metadata` | SHOULD | object | Additional information about the downscaling method and a new line |
| `coordinateTransformations` | MAY | array of [Coordinate Transformation](#tagged-union-coordinate-transformation) | Transformations applied in sequence from some common coordinate system into physical coordinates |

#### Multiscale field: `datasets`

- MUST be ordered highest-resolution (largest) to lowest-resolution (smallest)

#### Multiscale field: `axes`

- MUST have the same length as the dimensionality of all Zarr arrays in this multiscale
- Axis objects must have unique `name` fields in this metadata array
- All Zarr arrays MUST have the `dimension_names` field set, which MUST match the order and values of the `name` fields of this `axes` array
- MUST contain between 2 and 5 elements, with types in the order
  - optional axis with `type` `"time"`
  - optional axis with `type` `"channel"` (conventionally with `name` `"c"`), OR a custom string `type`, OR `null`, or omitted
  - optional axis with `type` `"space"`, conventionally the anisotropic slicing axis, conventionally with `name` `"z"`
  - required axis with `type` `"space"`, conventionally with `name` `"y"`
  - required axis with `type` `"space"`, conventionally with `name` `"x"`

#### Multiscale field: `coordinateTransformations`

Scale datasets' [`coordinateTransformations`](#dataset-field-coordinatetransformations) MAY transform the arrays' voxel coordinates into physical coordinates, in which case this field SHOULD be omitted.

If, instead, they transform voxel coordinates into some other common coordinate system, this field converts those system's coordinates into physical coordinates, in which case this field

- MUST contain 1 or 2 elements, with types in this order
  - required [Scale](#coordinate-transformation-variant-scale) transformation
  - optional [Translation](#coordinate-transformation-variant-translation) transformation

### Object: Axis

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `name` | MUST | string | Name of this axis |
| `type` | SHOULD | [AxisType](#enum-axistype) or string | Type of this axis |
| `unit` | SHOULD | [SpaceUnit](#enum-spaceunit) or [TimeUnit](#enum-timeunit) or string | Unit of measurement for this axis |

#### Axis field: `unit`

- If a listed unit is used, the `type` SHOULD match the unit type

### Object: Dataset

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `path` | MUST | string | Relative path to a Zarr array |
| `coordinateTransformations` | MUST | array of [Coordinate Transformation](#tagged-union-coordinate-transformation) | Transformations applied in sequence to convert from voxel coordinates into the multiscale's common coordinate system |

#### Dataset field: `coordinateTransformations`

- MUST contain 1 or 2 elements, with types in this order
  - required [Scale](#coordinate-transformation-variant-scale) transformation
  - optional [Translation](#coordinate-transformation-variant-translation) transformation

The multiscale's common coordinate system MAY be physical coordinates (as specified in the multiscale's [`axes`](#multiscale-field-axes)),
or into an intermediate coordinate system which is transformed to physical coordinates by the multiscale's [`coordinateTransformations`](#multiscale-field-coordinatetransformations).

### Object: ImageLabel

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `colors` | SHOULD | array of [ColorInformation](#object-colorinformation) | Display information associated with each label value |
| `version` | SHOULD | `"0.5"` | OME-Zarr specification version |
| `source` | MAY | [LabelSource](#object-labelsource) | Image data to which these labels are applied |
| `properties` | MAY | array of [LabelProperties](#object-labelproperties) | Arbitrary properties associated with each label value |

### Object: ColorInformation

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `label-value` | MUST | integer | The value in the Zarr array representing this label |
| `rgba` | MAY | 4-length array of unsigned integer in `[0,256)` | red, green, blue, and alpha (opacity) values |

Applications MAY add additional keys.

### Object: LabelProperties

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `label-value` | MUST | integer | The value in the Zarr array representing this label |

Applications MAY add additional keys.

### Object: LabelSource

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `image` | MAY | string | Relative path to a Zarr image group; default `../..` if omitted |

### Object: Plate

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `columns` | MUST | array of [Column](#object-column) | |
| `rows` | MUST | array of [Row](#object-row) | |
| `wells` | MUST | array of [Plate Well](#object-plate-well) | |
| `field_count` | SHOULD | unsigned integer | Maximum number of fields of view across all wells |
| `name` | SHOULD | string | |
| `acquisitions` | MAY | array of [Acquisition](#object-acquisition) | |

#### Plate field: `columns`

- MUST represent every column on the physical plate, even if a column has no wells
- MUST be in the same order as on the physical plate
- columns' `name` field MUST be unique within this array

#### Plate field: `rows`

- MUST represent every row on the physical plate, even if a row has no wells
- MUST be in the same order as on the physical plate
- rows' `name` field MUST be unique within this array

#### Plate field: `wells`

Plate wells refer to a location where one plate column and one row, both as defined in this plate metadata, intersect.
A plate well's fields reflect this, such that

- the plate well's `path` MUST comprise the row's `name`, a slash, then the column's `name`, e.g. `row1/column2`
- the plate well's `rowIndex` MUST be the index into the plate's `rows` array where the row is defined
- the plate well's `columnIndex` MUST be the index into the plate's `columns` array where the column is defined

### Object: Acquisition

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `id` | MUST | unsigned integer | MUST be unique within this plate |
| `name` | SHOULD | string | |
| `maximumfieldcount` | SHOULD | unsigned integer | maximum number of fields of view for this acquisition |
| `description` | MAY | string | |
| `starttime` | MAY | unsigned integer | Seconds since the UNIX Epoch at the start of acquisition |
| `endtime` | MAY | unsigned integer | Seconds since the UNIX Epoch at the start of acquisition |

### Object: Column

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `name` | MUST | string | |

### Object: Row

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `name` | MUST | string | |

### Object: Plate Well

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `path` | MUST | string | |
| `rowIndex` | MUST | unsigned integer | |
| `columnIndex` | MUST | unsigned integer | |

### Object: Well

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `images` | MUST | array of [FieldOfView](#object-fieldofview) | |

#### Field: `images`

If multiple fields of view were acquired for this well

- each element MUST contain the `acquisition` key
- the `acquisition` value MUST match the `id` of one of the [Acquisition](#object-acquisition)s defined by the [Plate](#object-plate) to which this well belongs

### Object: FieldOfView

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `path` | MUST | string | |
| `acquisition` | MAY | unsigned integer | |

## Tagged Unions

### Tagged Union: Coordinate Transformation

#### Coordinate Transformation Variant: Scale

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `type` | MUST | `"scale"` | Tag |
| `path` | MAY | string | Relative path to a 1D Zarr array of float scale coefficients |
| `scale` | MAY | array of float | Scale coefficients |

Exactly ONE of `path` or `scale` MUST be defined.

The scale coefficients' length MUST equal the dimensionality of the coordinates.

For input coordinate `i[N]` and coefficients `c[N]`, the output coordinate `o[n] = i[n] * c[n]`.

#### Coordinate Transformation Variant: Translation

| key | necessity | type | description |
| --- | --------- | ---- | ----------- |
| `type` | MUST | `"translation"` | Tag |
| `path` | MAY | string | Relative path to a 1D Zarr array of float translation coefficients |
| `scale` | MAY | array of float | Translation coefficients |

Exactly ONE of `path` or `scale` MUST be defined.

The translation coefficients' length MUST equal the dimensionality of the coordinates.

For input coordinate `i[N]` and coefficients `c[N]`, the output coordinate `o[n] = i[n] + c[n]`.

## Enums

### Enum: AxisType

| value | description |
| ----- | ----------- |
| space | Continuously-varying space axis; related units SHOULD be taken from [SpaceUnit](#enum-spaceunit) |
| time | Continuously-varying time axis; related units SHOULD be taken from [TimeUnit](#enum-timeunit) |
| channel | Discrete channel axis |

### Enum: SpaceUnit

As defined in UDUNITS-2

| value | description |
| ----- | ----------- |
| yoctometer | 1e-24 m |
| zeptometer | 1e-21 m |
| attometer | 1e-18 m |
| femtometer | 1e-15 m |
| picometer | 1e-12 m |
| nanometer | 1e-9 m |
| angstrom | 1e-10 m |
| micrometer | 1e-6 m |
| millimeter | 1e-3 m |
| centimeter | 1e-2 m |
| inch | ~2.54e-2 m |
| decimeter | 1e-1 m |
| meter | 1e0 m |
| foot | ~3.048e-1 m |
| yard | ~9.144e-1 m |
| dekameter | 1e1 m |
| hectometer | 1e2 m |
| kilometer | 1e3 m |
| mile | ~1.609344e3 m |
| megameter | 1e6 m |
| gigameter | 1e9 m |
| terameter | 1e12 m |
| parsec | ~3.085677581e16 m |
| petameter | 1e15 m |
| exameter | 1e18 m |
| zettameter | 1e21 m |
| yottameter | 1e24 m |

### Enum: TimeUnit

As defined in UDUNITS-2

| value | description |
| ----- | ----------- |
| yoctosecond | 1e-24 s |
| zeptosecond | 1e-21 s |
| attosecond | 1e-18 s |
| femtosecond | 1e-15 s |
| picosecond | 1e-12 s |
| nanosecond | 1e-9 s |
| microsecond | 1e-6 s |
| millisecond | 1e-3 s |
| centisecond | 1e-2 s |
| decisecond | 1e-1 s |
| second | 1e0 s |
| dekasecond | 1e1 s |
| minute | 6e1 s |
| hectosecond | 1e2 s |
| kilosecond | 1e3 s |
| hour | 3.6e3 s |
| day | 8.64e4 s |
| megasecond | 1e6 s |
| gigasecond | 1e9 s |
| terasecond | 1e12 s |
| petasecond | 1e15 s |
| exasecond | 1e18 s |
| zettasecond | 1e21 s |
| yottasecond | 1e24 s |

## References

- Requirement levels: <https://www.ietf.org/rfc/rfc2119.txt>
- JSON: <https://datatracker.ietf.org/doc/html/rfc8259>
- Zarr v3: <https://zarr-specs.readthedocs.io/en/latest/v3/core/index.html>
