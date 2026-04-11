# FasterCap

3D/2D capacitance extraction engine based on the Boundary Element Method (BEM).
Given a geometric description of conductors and dielectrics, FasterCap solves for the
full Maxwell capacitance matrix.

**Original author:** FastFieldSolvers S.R.L. (http://www.fastfieldsolvers.com)
**License:** LGPL 2.1+

---

## What it does

FasterCap computes the electrostatic capacitance between conductors by:

1. Discretizing conductor/dielectric surfaces into panels (3D) or segments (2D)
2. Adaptively refining the mesh until a user-specified error tolerance is met
3. Solving the resulting linear system with preconditioned GMRES
4. Outputting the Maxwell capacitance matrix

Key features:
- Hierarchical matrix compression (typically 80-90% compression ratio)
- Automatic adaptive mesh refinement with gradient meshing
- Complex permittivity support (lossy dielectrics)
- Out-of-core processing for large problems
- OpenMP parallelization

---

## Input

### File hierarchy

FasterCap reads a **list file** (`.lst` or `.txt`) as the top-level input.
This file references geometry files via `C` (conductor) and `D` (dielectric) statements.

### 3D format

**List file** — defines conductors and dielectrics with spatial offsets:
```
* Two unit cubes in free space (comments start with *)
C cube.txt 1.0  0.0 0.0 0.0
C cube.txt 1.0  2.0 0.0 0.0
```

Syntax:
```
C <geometry_file> <outperm> <xoff> <yoff> <zoff> [+]
D <geometry_file> <outperm> <inperm> <xoff> <yoff> <zoff> <xref> <yref> <zref> [-]
```

- `outperm` / `inperm`: relative permittivity (can be complex, e.g. `3.0-j0.02`)
- `+`: merge this conductor with the next one
- `-`: invert the reference side for dielectric interface

**Geometry file** — defines panels as quadrilaterals (`Q`) or triangles (`T`):
```
* 1m x 1m x 1m unit cube (6 faces)
Q mycube 1.0 1.0 0.0  1.0 0.0 0.0  1.0 0.0 1.0  1.0 1.0 1.0
Q mycube 0.0 1.0 0.0  1.0 1.0 0.0  1.0 1.0 1.0  0.0 1.0 1.0
Q mycube 1.0 0.0 0.0  0.0 0.0 0.0  0.0 0.0 1.0  1.0 0.0 1.0
Q mycube 0.0 0.0 0.0  0.0 1.0 0.0  0.0 1.0 1.0  0.0 0.0 1.0
Q mycube 0.0 0.0 0.0  1.0 0.0 0.0  1.0 1.0 0.0  0.0 1.0 0.0
Q mycube 0.0 0.0 1.0  1.0 0.0 1.0  1.0 1.0 1.0  0.0 1.0 1.0
```

Syntax:
```
Q <condname> <x1> <y1> <z1> <x2> <y2> <z2> <x3> <y3> <z3> <x4> <y4> <z4>
T <condname> <x1> <y1> <z1> <x2> <y2> <z2> <x3> <y3> <z3>
```

Panels sharing the same `condname` belong to the same conductor.

### 2D format

The first line of a 2D file **must** contain `2D` or `2d`.

**List file:**
```
* 2D - Coaxial cable cross-section
C inner_circle.txt 2.0  0.0 0.0
D interface.txt    2.0 1.0  0.0 0.0  0.0 0.0
C outer_circle.txt 1.0  0.0 0.0
```

**Geometry file** — defines segments (`S`):
```
* 2D circle approximation
S circle 0.100 0.000  0.099 0.016
S circle 0.099 0.016  0.095 0.031
S circle 0.095 0.031  0.088 0.045
...
```

---

## Output

### Maxwell capacitance matrix

The primary output is printed to stdout (and optionally to CSV):

```
Capacitance matrix is:
Dimension 2 x 2
g1_mycube   8.26757e-011  -2.73819e-011
g2_mycube  -2.73807e-011   8.26804e-011
```

- **3D:** units are Farads (F)
- **2D:** units are Farads/meter (F/m)
- Diagonal: self-capacitance of each conductor
- Off-diagonal: negated mutual capacitance between conductor pairs

### Optional outputs

| Flag | Output file | Description |
|------|-------------|-------------|
| `-e` | `<input>.csv` | Capacitance matrix in CSV format |
| `-o` | `<input>_ref.lst` | Refined mesh geometry (FastCap2 compatible) |
| `-c` | `<input>_<cond>.lst` | Charge density per conductor (FastModel format) |

### Full run log example

```
Running FasterCap version 6.0.7
Starting capacitance extraction with the following parameters:
Input file: cubes.lst
Auto calculation with max error: 0.01

Solution scheme: Collocation, GMRES tolerance: 0.005

Iteration number #0 ***************************
Refining the geometry..
Number of panels after refinement: 24
Number of links: 70 (uncompressed 576, compression ratio is 87.8%)
GMRES Iteration: 0 1

Capacitance matrix is:
Dimension 2 x 2
g1_mycube   6.49181e-011  -2.07808e-011
g2_mycube  -1.76613e-011   6.49235e-011

Total allocated memory: 12530 kilobytes
Total time: 0.311s
```

---

## Usage

```bash
# Basic: automatic mode with 1% error tolerance (headless)
./FasterCap -b cubes.lst -a0.01

# Manual mesh refinement and tighter GMRES tolerance
./FasterCap -b cubes.lst -m0.01 -t0.001

# Export capacitance to CSV and dump refined geometry
./FasterCap -b cubes.lst -a0.01 -e -o

# With Jacobi preconditioner
./FasterCap -b cubes.lst -a0.01 -pj
```

### Key flags

| Flag | Description | Default |
|------|-------------|---------|
| `-b` | Headless / console mode (required on Linux) | GUI |
| `-a<err>` | Automatic refinement with target relative error | — |
| `-m<val>` | Manual mesh refinement value | 0.1 |
| `-mc<val>` | Gradient meshing curvature coefficient | 3 |
| `-t<val>` | GMRES iteration tolerance | 0.01 |
| `-g` | Use Galerkin scheme instead of collocation | collocation |
| `-pj` | Jacobi preconditioner | none |
| `-ps<dim>` | Two-level preconditioner (block dimension) | 5 |
| `-d<val>` | Potential interaction / mesh refinement ratio | 1 |
| `-f<val>` | Out-of-core memory ratio (0 = disable) | 5 |
| `-e` | Export capacitance matrix to CSV | — |
| `-o` | Export refined geometry | — |
| `-c` | Export charge densities | — |
| `-v` | Verbose output | — |

---

## Building (Linux headless)

Prerequisites: CMake >= 2.8.12, GCC >= 4.8.1, wxWidgets >= 3.0.2

```bash
mkdir -p build && cd build
cmake -G "Unix Makefiles" \
      -DCMAKE_BUILD_TYPE=Release \
      -DFASTFIELDSOLVERS_HEADLESS=ON \
      ../
make -j$(nproc)

# Verify
./FasterCap -bv    # prints version
```

LinAlgebra and Geometry must be sibling directories — CMakeLists.txt includes them
via `add_subdirectory(../LinAlgebra ...)` and `add_subdirectory(../Geometry ...)`.

---

## Algorithm overview

1. **Surface discretization** — conductor/dielectric surfaces are defined as panels (quads/triangles in 3D, segments in 2D)
2. **Adaptive mesh refinement** — panels are recursively subdivided; gradient meshing places finer elements near edges and corners
3. **Hierarchical compression** — a binary-tree hierarchy groups distant panels, reducing the O(N^2) interaction matrix to ~10-30% of full size
4. **BEM linear system** — the compressed interaction matrix is solved via preconditioned GMRES (Jacobi, block, or hierarchical preconditioner)
5. **Convergence loop** (auto mode) — steps 2-4 repeat with progressively finer meshes until the capacitance matrix converges within the error tolerance
