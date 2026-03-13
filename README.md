# OME-Zarr specification reformat

This project is not normative; it is a personal experiment on whether the specification could be made more approachable for implementors.
It is a restatement of the [OME-Zarr v0.5 specification](https://ngff.openmicroscopy.org/specifications/0.5/index.html).

The specification document is at [SPECIFICATION.md](./SPECIFICATION.md).

## Goals

- Minimise blocks of prose
- Minimise context switches, define one object at a time
- Avoid restating Zarr implementation details; refer only to the data model
- Remain 100% forward and backward compatible with the OME-Zarr specification

## To do

- [ ] bioformats2raw.layout metadata
- [ ] omero metadata
- [ ] more descriptions

## Prior art

Layout inspired by the [OpenRTB specification v2.6+](https://github.com/InteractiveAdvertisingBureau/openrtb2.x/blob/main/2.6.md).
