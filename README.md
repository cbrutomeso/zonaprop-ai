# zonaprop-ai

Course repository for **AI & Data Science for Economists**.

You will use the Cursor agent to download a public ZonaProp search into a CSV and explore the published USD price before fitting a model.

## Before you start

Install these tools:

- [Node.js](https://nodejs.org/) 20 or newer (the current LTS)
- [Python](https://www.python.org/downloads/) 3.11 or newer
- The latest version of [Git](https://git-scm.com/downloads)
- The latest version of [Cursor](https://cursor.com/)

Create a [GitHub account](https://github.com/signup).

Fork [cbrutomeso/zonaprop-ai](https://github.com/cbrutomeso/zonaprop-ai), then clone your fork and open it in Cursor:

```bash
git clone https://github.com/YOUR_USERNAME/zonaprop-ai.git
cd zonaprop-ai
```

The agent instructions for this assignment are in `.cursor/` and come with the clone.

Install the Playwright MCP before the first prompt. In Cursor, open Customize → Playwright → Add to Cursor. The page should list one MCP, `playwright`, with no error. The agent uses it to open the search in a real browser.

Two failures may show up on that page.

**Workspace collection is not available.** The toast says `Failed to install Playwright, please try again or reload the window.` The project installer stops before it writes the plugin. Create `.cursor/settings.json` in the repo:

```json
{
  "plugins": {
    "playwright": {
      "enabled": true
    }
  }
}
```

Reload the window (`Ctrl+Shift+P` → Developer: Reload Window). Playwright then shows as installed for this project. That file is gitignored with the rest of the local Cursor state.

**Request timed out.** The MCP row stays red and the log says `MCP error -32001: Request timed out`. On Windows the plugin starts the server with `npx`, and `npx.cmd` consumes the handshake on stdin, so the server never answers.

The package is already in the npm cache after that attempt. Find `cli.js` under the hash folder that contains `@playwright\mcp`:

```text
%LOCALAPPDATA%\npm-cache\_npx\<hash>\node_modules\@playwright\mcp\cli.js
```

Edit the plugin file `%USERPROFILE%\.cursor\plugins\cache\cursor-public\playwright\<commit>\mcp.json` so Node runs that file:

```json
{
  "mcpServers": {
    "playwright": {
      "command": "C:\\Program Files\\nodejs\\node.exe",
      "args": [
        "C:\\Users\\YOU\\AppData\\Local\\npm-cache\\_npx\\<hash>\\node_modules\\@playwright\\mcp\\cli.js"
      ]
    }
  }
}
```

Reload the window. A later plugin update can put `npx` back. If the red dot returns, point the command at `node.exe` and `cli.js` again.

## First prompt

Paste this as the first message to the Cursor agent:

```text
Bajá a un CSV todos los resultados de esta búsqueda:
https://www.zonaprop.com.ar/casas-departamentos-ph-venta-capital-federal.html

Campos que necesito, con nombres fijos: precio en USD, barrio, ambientes, dormitorios, baños, descripción, tipo (casa, departamento, ph o terreno) y superficie total, cubierta y descubierta.

Sumá las columnas que sirvan para un análisis o un modelo. Dejá un solo comando documentado. El CSV queda en data/.
```

The CSV belongs in `data/`. That directory is gitignored, so the download stays on your machine and is not part of what you push.

## Download

```bash
python src/scrape.py https://www.zonaprop.com.ar/casas-departamentos-ph-venta-capital-federal.html --workers 6
```

`url` is the search page. `--workers` is how many slices download at once (default 6). Each worker uses its own HTTP session and requests that slice's pages one after another: concurrent requests for different pages of the same search repeat listings. The command writes `data/<slug>.csv` and prints that path and how many listings were saved.

The first page is HTML. The records are in `window.__PRELOADED_STATE__`. Changing page loads `...-pagina-N.html` and also POSTs `/rplis-api/postings` with the filters already applied (`pagina`, operation, property types, location). Deep HTML answers with a Cloudflare check while that POST still returns the page you asked for, until a cap: past it, `currentPage` is no longer the page requested or the first listing id repeats page 1. That cap is far below the advertised page count, and pages past about 140 in a single slice start repeating listings already saved. The default order (relevance) repeats listings even earlier, so the download asks for the same filters ordered by most recent, which announces the same total. Slices above that page window are split with the property-type filter from the page, then by USD price, and deduplicated by listing id. The room filter is not a partition: listings with no room count fall out of every bucket. `precio_usd` and the square-meter columns stay empty when the source value is in another unit. `superficie_descubierta` is the published figure, or total minus covered when both are in m².

## Price exploration

`notebooks/eda_propiedades_caba.ipynb` explores the Capital Federal sales CSV before fitting a price model. It covers distributions, outliers, missing values, percentiles, the association of the published USD price with the other columns, a correlation matrix, a neighborhood map, and a subway heat map. Phrases in the description become amenity flags, and those columns enter the same charts.

The downloader uses only the Python standard library. The notebook needs `requirements.txt`:

```bash
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
```

Open the notebook and select that environment as the kernel. It reads `data/casas-departamentos-ph-venta-capital-federal.csv`, which stays out of git. On the first run it downloads official neighborhood and subway GeoJSON into `data/geo/`.

## Price models

`notebooks/modelo_precio_caba.ipynb` uses the same CSV to compare three estimators of the
published USD price: linear regression with train-only robust limits and median imputation,
LightGBM with median imputation, and LightGBM with its native handling of missing values.

Run every cell after installing `requirements.txt`. The notebook creates a fixed 70/15/15
train, validation, and test split; tunes five LightGBM hyperparameters on validation; and
shows the final test MAE, RMSE, MAPE, and R² in USD. Results and fitted models remain in the
notebook session; it does not write a model or predictions file.
