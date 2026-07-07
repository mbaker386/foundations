# foundations

Compute **foundations of matroids** and everything nearby: pastures and their
morphisms, hexagons and cross-ratios, representations and orientations,
tropical Hom fans and reduced Dressians -- following Baker-Lorscheid theory.
Pure Python, single file, only `sympy` required (`python-flint` optional but
recommended for large computations). Rich typeset output (MathJax) in
notebooks, including hexagons drawn as actual hexagons.

## Use it -- three ways, easiest first

### 1. In your browser (nothing to install)

**-> https://mbaker386.github.io/foundations/lab/index.html?path=intro.ipynb**

A full Python notebook running in your browser via WebAssembly. Open the
link, press Shift+Enter through `intro.ipynb`, and you are computing
foundations in under a minute. Best for exploration, teaching, and
small-to-medium computations. (Heavy runs -- e.g. `F(T_31)`, large
Dressians -- belong in Colab below, which has `python-flint`.)

### 2. In Google Colab (free cloud compute)

**-> [Open the Colab workbench](https://colab.research.google.com/github/mbaker386/foundations/blob/main/notebooks/foundations_colab.ipynb)**

The first cell fetches the current `foundation.py` from this repository, so
it is always up to date -- nothing to upload, nothing to go stale.

### 3. Anywhere Python runs

```bash
wget https://raw.githubusercontent.com/mbaker386/foundations/main/foundation.py
python3 -c "import foundation as fd; print(fd.__version__)"
```

See **[FOUNDATION_TUTORIAL.md](FOUNDATION_TUTORIAL.md)** for a guided tour
of the whole API.

## One-time setup of this repository (for the maintainer)

1. Upload the contents of the kit to the repo root, **preserving the folder
   structure** (`content/`, `notebooks/`, `.github/workflows/`). Easiest:
   unzip locally, then drag the unzipped files and folders into
   *Add file -> Upload files* on GitHub and commit to `main`.
2. In the repo: **Settings -> Pages -> Build and deployment -> Source ->
   "GitHub Actions"**.
3. Go to the **Actions** tab; if prompted, enable workflows. The
   "Build and deploy the JupyterLite site" workflow runs on every push to
   `main`; the first green run publishes the site at
   `https://mbaker386.github.io/foundations/`.

## Releasing a new version of foundation.py

Replace `foundation.py` at the repo root (drag-and-drop upload, commit).
That single commit updates all three tiers automatically: the website
rebuilds itself via the Action, the Colab notebook fetches the new file on
its next run, and the raw-URL download always points at the latest version.
Check `fd.__version__` to confirm what you are running.

## Layout

```
foundation.py                       the package (single source of truth)
FOUNDATION_TUTORIAL.md              guided tour of the API
content/intro.ipynb                 starter notebook for the browser site
notebooks/foundations_colab.ipynb   the Colab workbench
.github/workflows/deploy.yml        builds & publishes the site on every push
requirements.txt                    build requirements for the site
```

License: MIT (see LICENSE; feel free to change it -- it's your project).
