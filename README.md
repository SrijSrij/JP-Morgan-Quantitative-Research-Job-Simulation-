# JPMorgan Chase Quantitative Research Job Simulation

This repository contains my work for the JPMorgan Chase & Co. Quantitative Research Job Simulation on Forage. I completed four tasks covering natural-gas price modelling, commodity storage-contract valuation, credit risk and FICO score bucketing.

I used the notebooks to document my analysis and kept reusable versions of the main calculations in `src/`, with automated checks in `tests/`.

> This is an independent educational project based on a Forage job simulation. It is not employment, investment advice, official JPMorgan Chase work product, or an endorsement by JPMorgan Chase or Forage.

## Tasks

### Tasks 1 and 2: Natural-gas pricing and storage-contract valuation

I kept Tasks 1 and 2 together in [`task1and2.ipynb`](task1and2.ipynb) because the storage-contract valuation directly uses the price curve produced in Task 1.

#### Task 1: Investigate and analyse price data

My objective was to analyse the supplied monthly natural-gas prices and create a function that could estimate a price for a requested date, including dates within a future forecast period.

My approach was to:

1. load and inspect the monthly price observations;
2. convert the dates into a consistent time-series format;
3. interpolate the monthly observations to obtain daily estimates;
4. split the data into training and holdout periods;
5. fit a Prophet model to capture the underlying trend and seasonality;
6. evaluate the forecast using RMSE; and
7. expose the result through `findprice(date)`.

The final local holdout run produced an RMSE of approximately `0.54`. As an example, the function estimated a price of `14.67` for 31 December 2025.

#### Task 2: Price a commodity storage contract

I then used the Task 1 price function to value a simplified natural-gas storage contract. The contract can contain multiple injection and withdrawal dates, so I treated it as a sequence of dated cash-flow and inventory events.

For each event, I accounted for:

- the price paid when gas is injected;
- the revenue received when gas is withdrawn;
- injection and withdrawal costs;
- monthly storage fees;
- the amount of gas currently held; and
- the facility's maximum capacity.

I sorted all events chronologically, started with empty inventory, increased inventory on injection and reduced it on withdrawal. The function rejects events that would exceed storage capacity or withdraw more gas than is available.

The valuation follows:

```text
Contract value = sales revenue
               - gas purchase cost
               - injection and withdrawal costs
               - storage fees
```

For the sample schedule in the notebook, my corrected calculation returned a contract value of `$16,000`.

### Task 3: Credit risk analysis

In [`Task3.ipynb`](Task3.ipynb), I built a model to estimate a borrower's probability of default using:

- credit lines outstanding;
- loan amount outstanding;
- total debt outstanding;
- income;
- years employed; and
- FICO score.

I used a stratified train-test split and a random forest with a fixed random state so that the results were reproducible. The final local holdout run produced approximately `99.8%` accuracy and a ROC-AUC of `0.99995`.

The important output for this task was the probability of default, rather than only a default/non-default classification. I therefore used `predict_proba()` and calculated expected loss as:

```text
Expected Loss = Probability of Default × Exposure at Default × Loss Given Default
```

With the task's 10% recovery assumption:

```text
Loss Given Default = 1 - 0.10 = 0.90
```

The `recovery_loss(loan)` function applies this calculation to an individual borrower.

### Task 4: Bucket FICO scores

In [`Task4.ipynb`](Task4.ipynb), I converted continuous FICO scores into a configurable number of credit-rating buckets. A lower rating number represents better credit quality, so rating `1` is assigned to the highest FICO range.

My first iteration used KMeans clustering to group similar FICO scores. Although it produced visually reasonable groups, KMeans only minimises the distance between score values. It does not use the observed defaults when deciding where the boundaries should be, so it was not directly optimising the credit-risk objective.

I also experimented with adjusting rounded cluster centres using a continuous optimiser. This was unreliable because rounding made the objective piecewise constant and non-differentiable: small changes to a centre often produced no change in the buckets or likelihood.

I replaced that approach with dynamic programming. For every possible FICO interval, I calculate the bucket's binomial log-likelihood from its number of borrowers and defaults. Dynamic programming then finds the combination of discrete boundaries that maximises the total log-likelihood across all requested buckets.

This approach was preferable because it:

- uses default behaviour when selecting boundaries;
- searches over valid integer FICO cut-offs;
- returns the globally optimal solution for the stated likelihood objective; and
- avoids applying a gradient-based optimiser to a rounded, discontinuous function.

The notebook is currently configured to demonstrate ten buckets, but the number of buckets can be changed through `num_clusters`.

## Installation

Clone the repository and create a virtual environment:

```bash
git clone <your-repository-url>
cd jpmorgan-quantitative-research

python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

On Windows, activate the environment with:

```powershell
.venv\Scripts\activate
```

## Data access

The two datasets were supplied through the Forage simulation and are not redistributed in this repository. After obtaining the files through authorised access, place them in the repository root with these names:

```text
Nat_Gas.csv
Task3and4_Loan_Data.csv
```

Further information is available in [`DATA_ACCESS.md`](DATA_ACCESS.md).

## Running the notebooks

Start Jupyter Lab:

```bash
jupyter lab
```

Run the notebooks in this order:

1. [`task1and2.ipynb`](task1and2.ipynb)
2. [`Task3.ipynb`](Task3.ipynb)
3. [`Task4.ipynb`](Task4.ipynb)

Tasks 1 and 2 are intentionally combined and should be run as a single notebook.

## Testing

I added tests for the reusable implementations in `src/`. They cover price interpolation and forecast limits, storage inventory constraints and cash flows, probability-based expected loss, and FICO boundary behaviour.

The credit-risk and storage-contract tests use small synthetic examples defined directly in the test files. Two checks use the full local simulation data: the price-analysis tests load `Nat_Gas.csv`, and one FICO-bucketing test loads `Task3and4_Loan_Data.csv` to confirm that five non-empty buckets are produced for the complete dataset.

Run the test suite with:

```bash
python -m unittest discover -s tests -v
```

The complete test suite therefore requires both Forage datasets to be present in the repository root. Without those files, only the self-contained credit-risk, storage-contract and synthetic FICO tests can run successfully.

## Main functions

| Function | Description |
| --- | --- |
| `findprice(date)` | Returns an interpolated or forecast natural-gas price for a requested date |
| `fin_value()` | Values the injection and withdrawal schedule defined in the combined notebook |
| `recovery_loss(loan)` | Calculates a borrower's expected loss using the predicted probability of default |
| `NaturalGasPriceModel.estimate(date)` | Provides a reusable and tested price-estimation interface |
| `price_storage_contract(...)` | Returns a validated storage-contract cash-flow breakdown |
| `expected_loss(...)` | Applies the `PD × EAD × LGD` calculation |
| `fit_fico_buckets(...)` | Finds likelihood-optimal FICO boundaries using dynamic programming |

## Limitations

- The project uses simulation data rather than live market or production banking data.
- The natural-gas model does not incorporate market forward curves, weather forecasts or price uncertainty.
- The credit model would require calibration, stability, explainability and fairness testing before any real lending use.
- Maximising likelihood alone can create narrow FICO buckets when many ratings are requested. A production scorecard may also impose minimum bucket sizes and monotonic default rates.

## Publication notice

I have included only my own code, explanations and tests. The supplied datasets, exact task instructions, example answers, generated data-dependent figures and unredacted certificate are excluded from the public repository.

“JPMorgan Chase” is used only to identify the sponsor of the simulation. JPMorgan Chase and its related names and marks belong to their respective owner. I have not used any JPMorgan Chase or Forage logos in this project.

## References

- [JPMorgan Chase Quantitative Research Job Simulation on Forage](https://www.theforage.com/virtual-internships/prototype/bWqaecPDbYAwSDqJy/Quantitative-Research)
- [Forage Terms of Use](https://www.theforage.com/terms)
