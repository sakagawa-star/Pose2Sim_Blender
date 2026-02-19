# CLAUDE.md

| 項目 | 内容 |
|------|------|
| 文書番号 | DOC-001 |
| 更新日 | 2026-02-19 |

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pose2Sim_Blender is a Blender add-on (v0.7.0) that visualizes OpenSim biomechanics data and Pose2Sim markerless motion capture results in Blender. It bridges motion analysis research tools and 3D animation software.

**Tested with:** Blender 3.6, 4.0, 4.2, and 5.0

## Architecture

```
__init__.py              # Add-on entry point, UI panel, all operator classes
Pose2Sim_Blender/
├── common.py            # Shared utilities (message boxes, material creation)
├── cameras.py           # Camera calibration import/export, video overlays, ray projection
├── markers.py           # Marker data (.trc, .c3d) import and armature rigging
├── model.py             # OpenSim .osim model import (XML parsing, VTP→STL conversion)
├── motion.py            # Motion animation (.mot, .csv) with OpenSim forward kinematics
├── forces.py            # Ground reaction force visualization (.mot)
├── skeletons.py         # Skeleton hierarchy definitions (HALPE_26, COCO_133, etc.)
└── Geometry/            # Pre-built 3D geometry files (STL, VTP, PLY)
```

## Key Patterns

**Operator Pattern:** All user-facing features are Blender operators in `__init__.py`:
```python
class MyOperator(bpy.types.Operator):
    bl_idname = "object.my_operator"
    bl_label = "My Operator"

    def execute(self, context):
        # Implementation
        return {'FINISHED'}
```

**Module Organization:**
- Data parsing functions go in the appropriate module (cameras, markers, model, motion, forces)
- `__init__.py` imports these and creates operator wrappers
- Panel layout defined in `panel1` class in `__init__.py`

**Coordinate Systems:** Code supports 'zup' and 'yup' coordinate systems throughout.

## Dependencies

**Always Available (Blender built-in):** bpy, mathutils, bmesh, numpy

**Bundled:** anytree, toml

**Optional (for advanced features):**
- `opensim` - Required for .mot motion file processing (forward kinematics)
- `vtk` - Required for VTP→STL geometry conversion
- `c3d` - Required for .c3d marker file support

## Data Formats

| Format | Extension | Module | Purpose |
|--------|-----------|--------|---------|
| TOML | .toml | cameras | Camera calibration (Pose2Sim format) |
| TRC | .trc | markers | Marker positions (tab-separated) |
| C3D | .c3d | markers | Marker positions (binary) |
| OpenSim | .osim | model | Skeletal model (XML) |
| MOT | .mot | motion/forces | Joint angles or forces |
| CSV | .csv | motion | Body positions/orientations |

## Development Notes

- No build system - distributed as .zip for Blender add-on installation
- No automated tests - manual testing on multiple Blender versions required
- Uses numpy (ships preinstalled with Blender) - avoid heavy external dependencies
- Follow existing docstring format with INPUTS/OUTPUTS sections
