# Natural State: Senior Data Scientist take-home

My submission for the take-home project ([brief](data/raw/NS%20-%20Senior%20Data%20Scientist%20-%20Take%20Home%20Project.pdf)). Each challenge has one report covering both questions (Q1: the analysis, Q2: the hand-off to the Tech team), and one python notebook for the analyses. With more time I would extract functions out to reusable scripts, but I thought the notebooks are sufficient to the process.

| Challenge | Question 1 | Question 2 | Notebook |
|---|---|---|---|
| Birds | [Observation labels for each prediction](birds_report.md#bird-challenge-1-observation-labels-for-each-prediction) | [Hand-off to the Tech team](birds_report.md#bird-challenge-2-hand-off-to-the-tech-team) | [`bird-data-investigation.ipynb`](bird-data-investigation.ipynb) |
| Vegetation | [Data quality report](vegetation_report.md#vegetation-challenge-1-data-quality-report) | [Hand-off to the Tech team](vegetation_report.md#vegetation-challenge-2-hand-off-to-the-tech-team) | [`vegetation-data-investigation.ipynb`](vegetation-data-investigation.ipynb) |

Full reports: [`birds_report.md`](birds_report.md) · [`vegetation_report.md`](vegetation_report.md)

## Running the notebooks
Requires [uv](https://docs.astral.sh/uv/) and Python 3.13.

```bash
uv sync
uv run --with jupyter jupyter lab   # or open the notebooks in VS Code with the .venv kernel
```

Run each notebook top to bottom. Paths are relative to the repository root. Raw data is in `data/raw/` as provided; the bird notebook writes its outputs to `data/processed/`.
