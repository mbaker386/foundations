# foundation.py tutorial

This tutorial is a practical guide to the `foundation.py` module.
It is written for a student who wants to use the package in a notebook,
especially in Google Colab.

The module covers four closely related tasks:

1. constructing finite **matroids**,
2. computing their **foundations** and **inner Tutte groups**,
3. defining and manipulating **pastures**,
4. computing **morphisms** and testing **isomorphisms** between pastures.

For new work, `foundation.py` is the only module you need.

A word on correctness: the algorithms follow Baker–Jin–Lorscheid (for the
foundation) and Chen–Zhang, arXiv:2307.14275 (for morphisms), and the module
is regression-tested against the Macaulay2 `foundations.m2` package: unit
groups, hexagon data, morphism counts, and pasture isomorphism all agree on a
15-matroid corpus. You can re-run those tests yourself at any time with
`python3 test_foundation.py`.

---

## 1. Files you should keep together

For ordinary use, keep these files in the same folder:

- `foundation.py`
- `FOUNDATION_TUTORIAL.md`
- `README.md`
- `test_foundation.py` (optional, the regression suite)

---

## 2. Loading `foundation.py` in Google Colab

There are two easy ways to use the module in Colab.

### Option A. Upload the file from your computer

Run:

```python
from google.colab import files
files.upload()
```

Select `foundation.py` from your computer.

Then import it with:

```python
import foundation as fd
```

If you edit or re-upload the file in the same Colab session, reload it with:

```python
import importlib
import foundation as fd
importlib.reload(fd)
```

### Option B. Create the file directly inside Colab

If you want to paste the whole module text directly into Colab, run:

```python
%%writefile foundation.py
# paste the full contents of foundation.py here
```

Then import it with:

```python
import foundation as fd
```

### Recommended: install python-flint

The module runs on pure sympy, but installing **python-flint** speeds up the
integer linear algebra dramatically (often 10–100× on larger examples). In
Colab, run this once per session, *before* importing the module:

```python
!pip install python-flint -q
import foundation as fd
```

### Quick sanity check

```python
import foundation as fd
print(fd.uniform_matroid(2, 4))
print(fd.foundation(fd.uniform_matroid(2, 4)))
```

If that prints a matroid and the presentation
`F_1^±(x, y) / < x + y - 1 >`, the module loaded correctly.

---

## 3. Loading the module locally

If you are using a local Jupyter notebook or a Python script, put
`foundation.py` in the same folder as the notebook or script and run:

```python
import foundation as fd
```

Optionally `pip install python-flint` in the same environment.

---

## 4. Constructing matroids

The main class is `fd.Matroid`.
Internally it stores a matroid by its bases (as bitmasks, for speed), but you
should usually build it with one of the class methods below.

### From bases

```python
M = fd.Matroid.from_bases(
    ground=[1, 2, 3, 4],
    bases=[
        {1, 2}, {1, 3}, {1, 4},
        {2, 3}, {2, 4}, {3, 4},
    ],
    name="U24",
)
```

### From nonbases

```python
M = fd.Matroid.from_nonbases(
    ground=[1, 2, 3, 4, 5, 6, 7],
    nonbases=[
        {1, 4, 5}, {1, 6, 7}, {2, 4, 6},
        {2, 5, 7}, {3, 4, 7}, {3, 5, 6},
    ],
    rank=3,
    name="F7-",
)
```

### From circuits

```python
M = fd.Matroid.from_circuits(
    ground=[1, 2, 3, 4],
    circuits=[
        {1, 2, 3},
        {1, 2, 4},
        {1, 3, 4},
        {2, 3, 4},
    ],
    rank=2,
    name="U24",
)
```

### From hyperplanes

```python
M = fd.Matroid.from_hyperplanes(
    ground=[1, 2, 3, 4],
    hyperplanes=[{1}, {2}, {3}, {4}],
    name="U24",
)
```

Useful methods on a matroid: `M.rank(S)`, `M.closure(S)`, `M.hyperplanes()`,
`M.corank2_flats()`, and the properties `M.rank_M`, `M.ground`, `M.bases`.

---

## 5. Built-in matroid families and examples

### Families

```python
fd.uniform_matroid(r, n)     # U_{r,n} on {1, ..., n}
fd.elliptic_matroid(n)       # T_n on Z/n: nonbases = triples summing to 0
```

### Named examples

The named catalogue uses ground set `{0, ..., n-1}` with **the same labelings
as Macaulay2's `specificMatroid`**, so results can be compared line by line
with M2:

```python
fd.fano_matroid()            # F_7;      foundation F_2
fd.nonfano_matroid()         # F_7^-;    foundation D (dyadic)
fd.pappus_matroid()          # Pappus
fd.nonpappus_matroid()       # non-Pappus (not representable over any field)
fd.vamos_matroid()           # Vamos (rank 4, not representable, orientable)
fd.t8_matroid()              # T_8;      foundation F_3
fd.betsy_ross_matroid()      # Betsy Ross; foundation G (golden ratio)
fd.ag23_matroid()            # AG(2,3);  foundation H (hexagonal)
fd.v8plus_matroid()          # V_8^+
fd.nondesargues_matroid()    # non-Desargues
fd.r9a_matroid(); fd.r9b_matroid()   # R_9^A, R_9^B (see Chen--Zhang, Sec. 6)
fd.r10_matroid()             # R_10 (regular; foundation F_1^±)
```

### Library helper

```python
L = fd.matroid_library()
print(sorted(L.keys()))

U37 = L["Uniform"](3, 7)
T17 = L["Elliptic"](17)
V   = L["Vamos"]()
```

---

## 6. Computing a foundation

The main user function is

```python
F = fd.foundation(M)
```

This returns a **raw foundation presentation**: the incidence-variable
generators, the (T1) multiplicative relations, and one fundamental pair per
modular quadruple of hyperplanes. It is fast even for fairly large examples.

You can also construct the matroid inline:

```python
F = fd.foundation(ground=[1, 2, 3, 4], bases=[{1,2},{1,3},{1,4},{2,3},{2,4},{3,4}])
```

### Strategies

```python
F = fd.foundation(M, strategy="auto")        # currently = "incidence"
F = fd.foundation(M, strategy="incidence")
```

The implemented backend is the hyperplane-incidence backend of
Baker–Jin–Lorscheid; the code is structured so that additional backends can
be registered later.

By default the backend emits a *reduced* set of (T1) relations that provably
generates the same relation lattice as the full set (verified symbolically
and by lattice-equality checks in the test suite); this is what makes the
larger examples fast. If you ever want the full Baker–Jin–Lorscheid
generating set, pass

```python
F = fd.foundation(M, t1_mode="full")
```

---

## 7. Small-example workflow

### Example: `U_{2,4}`

```python
M = fd.uniform_matroid(2, 4)
F = fd.foundation(M)

print(F)
print(F.details(limit_generators=10, limit_relations=10, limit_pairs=10))
```

Small examples print a quotient-style presentation such as

```text
F_1^±(x, y) / < x + y - 1 >
```

### Example: non-Fano

```python
M = fd.nonfano_matroid()
print(fd.foundation(M))
```

Output:

```text
F_1^±(x) / < x + x - 1 >
```

(the dyadic partial field D).

---

## 8. Large-example workflow

For larger examples, use the raw presentation and inspect summaries.

### Example: `T_13`

```python
M = fd.elliptic_matroid(13)
F = fd.foundation(M, strategy="incidence")
print(F.summary())
```

This gives a compact summary: number of generators, number of multiplicative
relations, number of raw fundamental pairs, and the unit group (computed via
Smith normal form; for T_13 it is `Z/2Z ⊕ Z^5`).

### Useful inspection methods

```python
print(F.show_generators(limit=20))
print(F.show_relations(limit=20))
print(F.show_pairs(limit=20))
print(F.num_hexagons())        # number of distinct hexagons of F(M)
```

### How far can you push it?

Expect roughly (single core): Pappus, Vámos, U(3,7) well under a second;
T_13 ≈ 0.2 s; T_17 ≈ 0.4 s; T_19 ≈ 1 s; T_23 ≈ 2 s; T_29 ≈ 25 s. Foundations
are nearly as fast without python-flint; flint mainly speeds up the morphism
search.

---

## 9. Canonical quotient-level foundations

For a `(unit group, hexagons)` model of the foundation, pass to the canonical
quotient-level form:

```python
CF = fd.canonical_foundation(M)
print(CF.pretty_presentation())
print(CF.unit_group.format_group())
print(CF.num_hexagons())
```

Or from a raw foundation object:

```python
CF = F.canonical()
```

`CF.representative_pairs` contains one fundamental pair per hexagon;
`CF.fundamental_pairs` is the full hexagon-closed list of ordered pairs.

### Typeset output in notebooks

In Jupyter or Colab, just evaluate the object as the last expression of a
cell:

```python
CF = fd.canonical_foundation(fd.uniform_matroid(2, 5))
CF          # renders the presentation in typeset math, plus a hexagon table
```

This works for matroids, raw and canonical foundations, pastures, and
morphisms. A few useful variants:

```python
CF.latex()                       # paper-ready LaTeX string (no dollar signs)
P = fd.hexagonal()
P                                # renders  F_1^±(t) / < t^3 = -1, t + t^{-1} - 1 >
ms = fd.morphisms(fd.u24(), fd.gf(5))
ms[0]                            # renders  x ↦ g, y ↦ g^2
```

`print(obj)` still gives the plain-text form, so scripts are unaffected. To
preview the rich rendering outside a notebook, write a standalone HTML file
(it loads MathJax from a CDN, so open it in any browser):

```python
fd.display_demo_html([CF, fd.hexagonal(), fd.gf(7)], "demo.html")
```

Long relation lists break automatically over several lines, and the hexagon
table includes a provenance column showing which modular quadruple of
hyperplanes each hexagon comes from. The cards are self-typesetting: they
work both in classic Jupyter and in Google Colab (whose sandboxed outputs
lack MathJax; the first card in a session loads it, so allow a second or two
for the very first render). Large foundations render a fast summary
card instead of the full presentation; call `.canonical()` first if you want
the typeset presentation for those.

### Where do the variables come from?

The generators of a computed foundation are abstract Smith-normal-form
coordinates. Three commands connect them back to the geometry of the
matroid:

```python
F = fd.foundation(fd.elliptic_matroid(11))

print(F.describe_hyperplanes())   # the labels H0, H1, ... as element sets
print(F.describe_variables())     # each generator as a product of cross-ratios
print(F.describe_hexagon(0))      # which modular quadruples give hexagon 0
```

For example, `describe_variables()` for `T_11` reports that each of the four
free generators *is* a single cross-ratio `cr(Ha,Hb;Hc,Hd)` of four
hyperplanes through a common corank-2 flat, and lists those hyperplanes as
sets. Here `cr(Ha,Hb;Hc,Hd)` and `cr(Ha,Hc;Hb,Hd)` are the two members of
the fundamental pair attached to the modular quadruple `(Ha,Hb,Hc,Hd)`, so
they sum to 1 in the foundation. Every printed expression is re-evaluated in
the unit group and machine-checked before being shown.

Programmatic access: `CF.variable_expressions()` (a dict of coefficient
lists), `CF.hexagon_provenance(i)` (a list of `{flat, hyperplanes, ...}`
dicts per hexagon), and `F.hyperplane_dict()`.

---

## 10. Creating pastures directly

The main class is `fd.Pasture`. For most users, the simplest way to build a
pasture is

```python
P = fd.presented_pasture(...)
```

or the synonym `fd.pasture(...)`.

### Example: near-regular partial field

```python
P = fd.presented_pasture(
    name="Near-regular",
    generators=["x", "y"],
    additive_relations=["x + y = 1"],
)
print(P)
```

### Example: hexagonal partial field

The hexagonal partial field needs **both** an additive and a multiplicative
relation (`x + x^{-1} = 1` alone, or `x^2 - x + 1 = 0` alone, presents a
larger pasture with unit group `Z/2 ⊕ Z` instead of `Z/6`):

```python
H = fd.presented_pasture(
    name="Hexagonal",
    generators=["z"],
    additive_relations=["z^2 - z + 1 = 0"],
    multiplicative_relations=["z^3 = -1"],
)
print(fd.are_isomorphic(H, fd.hexagonal()))    # True
```

### Example: several generators and relations

```python
P = fd.presented_pasture(
    name="Toy example",
    generators=["A", "B", "C", "D", "E"],
    additive_relations=[
        "A + C*D = 1",
        "B + D*E = 1",
    ],
    multiplicative_relations=[
        "A*B*C = D*E",
    ],
)
print(P.summary())
```

### Supported relation syntax

An additive relation must be a **null triple**: after moving everything to
the left-hand side, exactly three signed Laurent monomials summing to zero.
Examples:

```python
"x + y = 1"
"x + y - 1 = 0"
"z^2 - z + 1 = 0"
"x + x^(-1) = 1"
"x - 1 = 1"          # the pair (x, -1); presents the dyadic partial field
"-1 - 1 - 1"         # presents F_3
```

Multiplicative relations may be written in forms such as

```python
"x*y = z"
"x^3 = -1"
"A*B*C = D*E"
```

---

## 11. Built-in pasture library

### Basic pastures and fields

```python
fd.f1()          # F_1^±
fd.f2()          # the field F_2
fd.f3()          # the field F_3
fd.gf(q)         # the finite field GF(q), q any prime power
```

### Hyperfields

```python
fd.krasner()     # the Krasner hyperfield K
fd.sign()        # the sign hyperfield S  (morphisms F_M -> S = orientations)
```

### Named partial fields / standard foundations

```python
fd.dyadic()          # D  = foundation of the non-Fano
fd.near_regular()    # U  = foundation of U(2,4)     (alias: fd.u24())
fd.hexagonal()       # H  = foundation of AG(2,3)
fd.golden()          # G  = foundation of the Betsy Ross matroid
fd.i_pasture()       # I  (Chen--Zhang)
fd.u25()             # V  = foundation of U(2,5)
fd.t11_foundation()  # verified presentation of the foundation of T_11
```

Or by name, mirroring M2's `specificPasture`:

```python
fd.specific_pasture("S")        # sign
fd.specific_pasture("golden")   # G
fd.specific_pasture("V")        # foundation of U(2,5)
```

### Library helper

```python
L = fd.pasture_library()
print(sorted(L.keys()))

F4 = L["GF"](4)
S  = L["Sign"]
```

Note: an earlier version of this module contained a conjectural presentation
`t11_candidate()`. That presentation turned out **not** to be isomorphic to
the foundation of T_11 (it has the same unit group and hexagon count, but no
GF(7)-points, whereas F(T_11) has five). `t11_foundation()` is the corrected,
machine-verified presentation; `t11_candidate()` now returns the same
corrected pasture.

---

## 12. Building a pasture from a computed foundation

To use the morphism machinery on a foundation, convert it to a `Pasture`:

```python
M = fd.uniform_matroid(2, 4)
F = fd.foundation(M)
P = fd.Pasture.from_foundation(F, name="Foundation(U24)")
print(P.summary())
```

`from_foundation` accepts a `Matroid`, a raw foundation, or a canonical
foundation.

---

## 13. Morphisms and isomorphisms of pastures

The morphism layer implements the Chen–Zhang algorithm and is **exact**: it
returns the complete (finite) list of pasture morphisms, with no search
bounds to tune.

### Compute all morphisms

```python
P = fd.Pasture.from_foundation(fd.foundation(fd.uniform_matroid(2, 4)))
ms = fd.morphisms(P, fd.gf(5))
print(len(ms))         # 3
print(ms[0])           # e.g. PastureMorphism(z1 -> g, z2 -> g^2)
```

Options:

```python
fd.morphisms(P, Q, find_one=True)     # stop at the first morphism
fd.morphisms(P, Q, find_iso=True)     # only isomorphisms
fd.morphisms(P, Q, max_results=10)    # stop after 10
```

### Yes/no tests

```python
fd.has_morphism(P, Q)       # exact decision
fd.are_isomorphic(P, Q)     # exact decision
```

(Old notebooks that pass `search_bound=...` or `include_products=...` still
run; those arguments are accepted and ignored.)

### Example: near-regular and `U_{2,4}`

```python
P = fd.u24()
Q = fd.near_regular()
print(fd.are_isomorphic(P, Q))     # True
```

### What the counts mean

For a matroid M with foundation F_M and a pasture Q, the morphisms
F_M → Q are in bijection with **rescaling classes of Q-representations of
M**. Two favourite instances:

```python
FP = fd.Pasture.from_foundation(fd.foundation(fd.pappus_matroid()))
print(len(fd.morphisms(FP, fd.gf(8))))    # 18 representations of Pappus over F_8
print(len(fd.morphisms(FP, fd.sign())))   # 18 orientation classes
```

Matroid-level shortcuts:

```python
fd.is_representable(fd.fano_matroid(), fd.gf(2))   # True
fd.is_orientable(fd.vamos_matroid())               # True
fd.is_orientable(fd.fano_matroid())                # False
```

### Scope of the algorithm

The search is exact whenever the multiplicative group of the **source** is
generated by its fundamental elements together with −1. This holds
automatically for foundations of matroids, for finite fields, and for all the
built-in pastures. For other sources the module falls back to brute-force
enumeration when the target is finite, and raises an informative error when
the target is infinite (in which case the morphism set need not be finite).

### From morphisms to explicit representations and orientations

A morphism count tells you *how many* rescaling classes exist; these
functions hand you an explicit representative of each class:

```python
# one verified matrix over GF(q) per rescaling class of representations
reps = fd.representations(fd.fano_matroid(), 2)
print(reps[0])          # the classical 3 x 7 Fano matrix over GF(2)

reps = fd.representations(fd.ag23_matroid(), 4)
print(len(reps))        # 2 (entries involve the primitive element g of GF(4))

# one explicit chirotope per reorientation-plus-rescaling class
chs = fd.orientations(fd.uniform_matroid(2, 4))
print(len(chs))                     # 3
print(chs[0].chirotope_string())    # e.g. '++++--' over the 2-subsets
print(chs[0].chi(1, 3))             # evaluate chi on an ordered basis

ch = fd.orientations(fd.vamos_matroid(), max_results=1)[0]
print(ch.verified)      # 'exchange + 3-term GP'
```

Every returned matrix is verified (its column matroid is checked against M
on all r-subsets), and every chirotope is verified for basis-exchange
consistency and the 3-term Grassmann–Plücker sign conditions — so the Vámos
example above really is a certified non-realizable oriented matroid. In a
notebook these objects render as a matrix table / chirotope string
automatically.

### Matroid operations, cross-ratios, and minors

Standard matroid operations (all benchmarked against Macaulay2):

```python
M = fd.pappus_matroid()
M.dual()                      # or fd.dual_matroid(M)
M.delete({0, 1})              # deletion
M.contract({0})               # contraction
M.minor(delete={1}, contract={2})
fd.direct_sum(M, fd.uniform_matroid(2, 4))
fd.two_sum(fd.uniform_matroid(2, 3), fd.uniform_matroid(2, 3), 1, 1)
M.flats(2)                    # flats of rank 2; M.flats() for all
M.circuits(); M.loops(); M.coloops()
```

More operations and tests, all benchmarked against Macaulay2 (labeled data
or booleans):

```python
fd.nonbases(M); fd.cocircuits(M); fd.hyperplanes(M)
fd.fundamental_circuit(M, {0, 1}, 6)     # the circuit in I + e
fd.fundamental_cocircuit(M, B, e)        # complement of cl(B - e)
fd.is_matroid_bases(ground, candidate)     # basis-exchange axiom check
fd.matroids_are_isomorphic(M, N)             # exact; quick_are_isomorphic for a fast screen
fd.has_minor(fd.pappus_matroid(), fd.uniform_matroid(2, 4))   # True
fd.is_connected(M); fd.is_3_connected(M); fd.components(M)
fd.is_binary(M); fd.is_ternary(M); fd.is_regular(M)           # via foundation morphisms
fd.is_positively_orientable(M)           # positroid test (any ground-set order)
fd.wheel(4); fd.whirl(4); fd.graphic_matroid(4, [(0,1), (1,2), ...])
fd.projective_geometry(2, 2)             # ~ Fano
fd.lattice_of_flats(M)                   # Hasse diagram rendered as SVG in notebooks
```

The inverse of `describe_variables()`: look up a universal cross-ratio as an
element of the foundation, in either form:

```python
F = fd.foundation(fd.uniform_matroid(3, 6))
u = F.cross_ratio({1,2}, {1,3}, {1,4}, {1,5})    # hyperplane form (sets, indices, or "H7")
v = F.cross_ratio(2, 3, 4, 5, J=[1])             # element form [2,3;4,5]_{1}
print(u == v)                                    # True
print(F.cross_ratio_string(2, 3, 4, 5, J=[1]))   # e.g. -x_2^-1 * x_4
```

And minor functoriality — the induced morphism F(N) → F(M) for an embedded
minor N = M/contract\delete:

```python
phi = fd.minor_hom(fd.uniform_matroid(3, 6), contract={1})   # F(U(2,5)) -> F(U(3,6))
print(phi.is_pasture_morphism())    # True (verified on every hexagon)
print(phi)                          # generator images as products of cross-ratios
```

Each generator of F(N) is expressed in cross-ratios of N, and each
[a,b;c,d]_J is lifted to [a,b;c,d]_{J + C'} in M (C' a maximal independent
subset of the contracted set). Composing with a morphism F(M) → GF(q)
restricts a representation of M to the minor N.

### Pasture operations: products, tensor products, universal rings, lifts

```python
U = fd.near_regular()
fd.num_morphisms(U, fd.gf(5))          # 3
fd.automorphisms(U)                    # the 6 automorphisms of U
sub, incl = fd.fundamental_sub_pasture(U)   # sub-pasture on the fundamental elements

# coproducts: by Baker-Jin-Lorscheid, foundations of direct sums and 2-sums
# are tensor products of foundations
T = fd.tensor_product(fd.dyadic(), fd.sign())
phi = fd.morphisms(fd.f1(), fd.dyadic())[0]
psi = fd.morphisms(fd.f1(), fd.sign())[0]
fd.are_isomorphic(fd.fiber_coproduct(phi, psi), T)      # True

# products (= fiber products over the Krasner hyperfield)
PR = fd.pasture_product(fd.dyadic(), fd.sign())

# the universal ring Z[P^x]/(eps + 1, a + b - 1): renders in typeset math
R = fd.universal_ring(fd.hexagonal())
print(R)
print(R.count_points(7))               # 2 = |Mor(H, GF(7))|

# the GRS lift: only the hexagon-internal relations survive
L, counit = fd.grs_lift(fd.gf(4))
fd.are_isomorphic(L, fd.hexagonal())   # True - the classical PvZ example

fd.is_rigid(fd.gf(7))                  # True: Hom(GF(7), T) is a point
fd.is_orientable_pasture(U)            # True: Hom(U, S) is nonempty
```

### The BLZ appendix pastures

```python
fd.blz("FHydra3")        # the Hydra-3 partial field, by its FP2 appendix tag
fd.blz("FPappus")        # the Pappus foundation (== F(pappus), verified)
sorted(fd.blz_pastures()) # all available tags
fd.BLZ_F21a()            # tags are also individual functions
fd.weak_sign()           # W = F3 (x) S
```

Every entry is verified against the appendix's stated (hexagons, rank) data
and flags, and — where the appendix lists the minimal matroid's short
circuits — against the computed foundation of that matroid.

### The Oxley appendix

```python
fd.oxley_R8()            # the real affine cube [I4 | J - 2I] over GF(3)
fd.oxley_S8(); fd.oxley_P8(); fd.oxley_L8(); fd.oxley_T12()
fd.oxley_S_5_6_12()      # the extended ternary Golay code matroid
fd.binary_spike(4)       # Oxley's Z_4  (Z_4 \ t = AG(3,2))
fd.free_swirl(4)         # Psi_4; Psi_3 = U(3,6)
fd.theta_matroid(4)      # for segment-cosegment exchange
fd.matroid_from_gfq_matrix([[1,0,1],[0,1,1]], 2)   # column matroids over GF(p)
```

Every constructor follows the recipe in Oxley's appendix, and the test suite
checks the appendix's own cross-facts (relaxation chains, minors,
self-duality, representability patterns).

### The lattice-of-flats presentation and hexagon pictures

```python
# an independent computation of the foundation via BLZ FP2, Theorem 5.9
P = fd.flats_foundation(fd.ag23_matroid())
fd.are_isomorphic(P, fd.Pasture.from_foundation(fd.foundation(fd.ag23_matroid())))   # True

# draw the hexagons of a pasture as actual hexagons (BL's picture):
# solid red edges join elements summing to 1, dashed blue edges join inverses
F = fd.foundation(fd.uniform_matroid(2, 4))
F.show_hexagons()             # renders as the last expression of a cell
                              # (also a plain HTML string, and embedded
                              # automatically in small display cards)

# Oxley matroids (Q6 and P6 verified via their foundations against BLZ FP2)
fd.oxley_Q6(); fd.oxley_P6(); fd.oxley_W3(); fd.oxley_AG32()
```

### The tropical hyperfield: Hom(P, T) and the reduced Dressian

Morphisms to the tropical hyperfield 𝕋 form a polyhedral fan rather than a
finite set, and the package computes it exactly:

```python
# the tropical line: Hom of the near-regular partial field
Ft = fd.tropical_hom(fd.near_regular())
print(Ft.summary())        # dimension 1, three rays
print(Ft.rays())           # [(-1, -1), (0, 1), (1, 0)]

# the reduced Dressian of a matroid: Hom(F(M), T)
Ft = fd.reduced_dressian(fd.uniform_matroid(2, 5))
print(Ft.f_vector())       # {2: 15} - the 15 trivalent trees on 5 leaves
print(len(Ft.rays()))      # 10     - the 10 splits
print(Ft.verify())         # sampled soundness + completeness check

Ft = fd.reduced_dressian(fd.elliptic_matroid(11))
print(Ft.summary())        # a 1-dimensional fan with 5 rays
print(Ft.describe_rays())  # the rays as valuations of the generators
```

Here `reduced_dressian(M)` is the Dressian of M with the lineality space of
rescalings already removed: the coordinates are the free generators of the
foundation (i.e., valuations of cross-ratios — `describe_variables()` tells
you which), and each hexagon `a + b = 1` imposes the tropical condition that
the minimum of (v(a), v(b), 0) be attained at least twice. `contains(w)`
tests membership of any integer vector directly against these conditions,
independently of the computed cones, and `verify()` uses this as an oracle.
In a notebook the fan renders as a summary card with a table of rays.

For larger examples pass `max_cones=...` (intermediate cell counts can
exceed the final count) and `progress=True` to watch the intersection.

---

## 13c. Saving and loading with JSON

```python
# save
F = fd.foundation(fd.elliptic_matroid(11))
fd.to_json(F, path="t11_foundation.json")
fd.to_json(fd.hexagonal(), path="hexagonal.json")

# load
rec = fd.from_json("t11_foundation.json")   # a FoundationRecord
P = rec.pasture                              # ready for morphism computations
print(len(fd.morphisms(P, fd.gf(7))))        # 5
print(rec.describe_hyperplanes())            # hyperplane labels survive
H = fd.from_json("hexagonal.json")           # a Pasture
```

Matroids, pastures, and foundations all round-trip; foundations are stored
with their canonical pasture, hyperplane list, and hexagon provenance
(compact — the T_11 foundation is about 19 kB). Ground-set elements must be
integers or strings for JSON serialization.

---

## 14. Typical workflow for comparing a foundation to a known pasture

```python
import foundation as fd

M = fd.ag23_matroid()
P = fd.Pasture.from_foundation(fd.foundation(M))
Q = fd.hexagonal()

print(P.summary())
print(Q.summary())
print(fd.are_isomorphic(P, Q))    # True: F(AG(2,3)) is the hexagonal partial field
```

---

## 15. A few useful commands to remember

```python
import foundation as fd

# matroids
M = fd.uniform_matroid(2, 5)
M = fd.elliptic_matroid(13)
M = fd.vamos_matroid()

# foundations
F = fd.foundation(M)
print(F.summary())
CF = fd.canonical_foundation(M)
P = fd.Pasture.from_foundation(F)

# pastures
Q = fd.gf(4)
R = fd.presented_pasture(
    name="Example",
    generators=["x"],
    additive_relations=["x + x = 1"],
)

# comparisons (all exact)
fd.morphisms(P, Q)
fd.has_morphism(P, Q)
fd.are_isomorphic(R, fd.dyadic())
fd.is_orientable(M)

# geometry of the generators
F.describe_variables()
F.describe_hexagon(0)
F.describe_hyperplanes()

# explicit representatives
fd.representations(M, 4)
fd.orientations(M)

# save / load
fd.to_json(F, path="example.json")
rec = fd.from_json("example.json")

# tropical
fd.tropical_hom(P)
fd.reduced_dressian(M)

# operations / cross-ratios / minors
M.dual(); M.delete(S); M.contract(S); M.minor(delete=D, contract=C)
F.cross_ratio(a, b, c, d, J=J)
fd.minor_hom(M, delete=D, contract=C)
```

---

## 16. If something seems wrong

### Import fails in Colab

Check that `foundation.py` is really present:

```python
import os
print(os.listdir())
```

### The notebook is still using an old version of the module

Reload it:

```python
import importlib
import foundation as fd
importlib.reload(fd)
```

### A computation is too slow

- Install python-flint: `!pip install python-flint` (then restart the
  runtime / re-import).
- Stay with the raw foundation layer (`fd.foundation(M)`) and `F.summary()`.
- Morphism counts into large finite fields grow quickly with the number of
  "free" (type 3) hexagons of the source; `find_one=True` is much faster
  when you only need existence.

### You do not trust a result

Run the regression suite:

```bash
python3 test_foundation.py
```

Every expected value in it was verified against the Macaulay2
`foundations.m2` package.

---

## 17. Suggested exercises for a first session

1. Compute the foundations of `U_{2,4}`, `U_{2,5}`, `F_7`, and `F_7^-`.
2. Check that `fd.u24()` and `fd.near_regular()` are isomorphic, and that
   neither is isomorphic to `fd.dyadic()`.
3. Build the hexagonal partial field with `presented_pasture(...)` (don't
   forget `z^3 = -1`!) and compare it to `fd.hexagonal()`.
4. Compute the foundation of `T_11` and verify it is isomorphic to
   `fd.t11_foundation()`.
5. Count the representations of the Pappus matroid over `GF(8)` (you should
   find 18), and verify that the non-Pappus matroid admits no morphism to any
   `GF(q)` for q ≤ 8 but has 36 orientation classes.
6. Verify that `F_7` is orientable over no field... more precisely: check
   `fd.is_orientable(fd.fano_matroid())` and explain the answer via
   `fd.morphisms(P, fd.sign())`.

---

That is the basic workflow of the current `foundation.py` module.
