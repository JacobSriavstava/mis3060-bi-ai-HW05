# Star Design #

## Part A ##
| Table | Type | Primary key | Foreign keys (column → table[column]) |
|---|---|---|---|
| dim_advisor | Dimension | `advisor_id` | — |
| dim_benchmark | Dimension | `benchmark_id` | — |
| dim_client | Dimension | `client_id` | `advisor_id` → dim_advisor[advisor_id] |
| dim_date | Dimension | `date_id` | — |
| dim_security | Dimension | `security_id` | `benchmark_id` → dim_benchmark[benchmark_id] |
| fact_fees | Fact | `fee_id` | `client_id` → dim_client[client_id]<br>`advisor_id` → dim_advisor[advisor_id]<br>`fee_date` → dim_date[date] |
| fact_portfolio_snapshot | Fact | `snapshot_id` | `client_id` → dim_client[client_id]<br>`advisor_id` → dim_advisor[advisor_id]<br>`benchmark_id` → dim_benchmark[benchmark_id]<br>`snapshot_date` → dim_date[date] |
| fact_transactions | Fact | `txn_id` | `client_id` → dim_client[client_id]<br>`advisor_id` → dim_advisor[advisor_id]<br>`security_id` → dim_security[security_id]<br>`txn_date` → dim_date[date] |

## Notes

- Every primary key was checked as unique and never blank: all 8 pass.
- Fact date columns join on `dim_date[date]` (the calendar date), not the `date_id` surrogate key; the script checked both.
- `dim_client.client_since` is **not** listed as a foreign key: only 39.1% of its distinct values exist in `dim_date[date]` (1,006 don't). Its values run from 2015-01-01 to 2022-12-30, while dim_date covers 2020-01-01 to 2024-12-31. It is a descriptive attribute, not a join key.
- `fact_transactions.security_id` is blank on 101,528 of 298,572 rows. Every blank row is a cash-only transaction (`txn_type` = Deposit, Withdrawal or Advisory Fee). No security is bought, sold or paid out, so `security_id`, `shares` and `price` are all blank. Buy, Sell and Dividend rows always have a `security_id`. The key is optional, and a blank is valid, not an orphan.

## Grain

| Fact table | Grain (one row per…) | Unique key tested |
|---|---|---|
| fact_fees | client per month (fee_date = 1st of month) | (client_id, fee_date): 0 duplicates |
| fact_portfolio_snapshot | client per month (snapshot_date = 1st of month) | (client_id, snapshot_date): 0 duplicates |
| fact_transactions | individual transaction | txn_id: 0 duplicates |

- Adding `fee_type` or `account_type` changes nothing; each client has only one row per month.
- Monthly dates fall on the 1st, not month-end (0% have `is_month_end = 1`), and run from 2020-01-01 to 2024-12-01.
---

## Part B ##

- **fact_fees:** One row represents the single fee (Management, Advisory, Custodial or Performance) charged to one client for one month, dated the 1st of that month.
- **fact_portfolio_snapshot:** One row represents one client's portfolio as of one month (dated the 1st), with its AUM, year-to-date return against its benchmark, and cash percentage at that point.
- **fact_transactions:** One row represents one individual transaction (a buy, sell, dividend, deposit, withdrawal or advisory fee) made in one client's account on one day.


---

## Part C ##

## Measures and additivity

For each numeric column: *if I add this value across clients, across months, and across both, what does the total represent?*

- **Additive:** the sum makes sense across every dimension.
- **Semi-additive:** the sum makes sense across some dimensions (e.g. clients) but not others (usually time).
- **Non-additive:** the sum never makes sense (rates, percentages, prices); use a weighted average or a ratio of sums instead.

### fact_fees

| Column | Type | Sum across clients | Sum across months | Sum across both | What it means for summing / averaging |
|---|---|---|---|---|---|
| `aum_basis` | **Semi-additive** | total AUM billed on that month | the same dollars counted once per month (meaningless) | meaningless | Sum across clients within one month, but never across months; over time use the average or the latest month. |
| `bps_charged` | **Non-additive** | meaningless (a sum of rates) | meaningless | meaningless | A rate is never summed; report the AUM-weighted rate SUM(amount) * 12 * 10,000 / SUM(aum_basis), not a plain average. |
| `amount` | **Additive** | total fees billed to those clients | total fees billed over those months | total fee revenue | Can be summed across clients, advisors, fee types and months; SUM is the right aggregate. |

### fact_portfolio_snapshot

| Column | Type | Sum across clients | Sum across months | Sum across both | What it means for summing / averaging |
|---|---|---|---|---|---|
| `aum` | **Semi-additive** | firm (or advisor) AUM on that date | the same dollars counted once per month (meaningless) | meaningless | Balance-type measure: sum across clients for a single snapshot_date, but over time take the latest snapshot or an average, never a sum. |
| `return_ytd` | **Non-additive** | meaningless (a sum of percentages) | meaningless (each YTD value already includes earlier months) | meaningless | Never sum; across clients use an AUM-weighted average, and for a year take the last month's YTD value instead of adding months. |
| `benchmark_return_ytd` | **Non-additive** | meaningless | meaningless (cumulative YTD) | meaningless | Never sum; it is the same for every client on a benchmark, so take one value per benchmark and date (or a weighted average). |
| `active_alpha_ytd` | **Non-additive** | meaningless | meaningless (cumulative YTD) | meaningless | Never sum; compute it as the weighted return minus the weighted benchmark return for the group, or AUM-weight the row values. |
| `cash_pct` | **Non-additive** | meaningless (a sum of percentages) | meaningless | meaningless | Never sum; use the AUM-weighted average SUM(cash_pct * aum) / SUM(aum). |

### fact_transactions

| Column | Type | Sum across clients | Sum across months | Sum across both | What it means for summing / averaging |
|---|---|---|---|---|---|
| `shares` | **Semi-additive** | total shares traded in that security | total shares traded in that security over time | meaningful only for one security | Can be summed across clients and dates only within a single security (and txn_type); adding shares of different securities is meaningless. Blank on cash rows. |
| `price` | **Non-additive** | meaningless (a sum of prices) | meaningless | meaningless | A unit price is never summed; use SUM(amount) / SUM(shares) for a weighted average price, or MIN/MAX/latest. Blank on cash rows. |
| `amount` | **Additive** | total dollars moved for those clients | total dollars moved over those days | total dollar volume | Can be summed across clients, advisors, securities and dates, but always group by or filter on txn_type, because every amount is positive and the direction (buy vs. sell, deposit vs. withdrawal) lives in txn_type. |

**Data checks behind these labels**

- `fact_fees.amount` = `aum_basis × bps_charged / 10,000 / 12` on 100.0% of rows, so it is the monthly fee: a flow, which makes it additive.
- `active_alpha_ytd` = `return_ytd − benchmark_return_ytd` on 100.0% of rows, so it is a difference of two rates and is also non-additive.
- The smallest `fact_transactions.amount` is 13.33: every amount is positive, so direction must come from `txn_type`.
- 977 snapshot rows have a `return_ytd` above ±100%; 100.0% of them are flagged `is_return_suspect = 1`. Exclude flagged rows before averaging returns.

### Numeric columns in fact tables that are not measures

| Table | Column | Kind | Why it is not summed |
|---|---|---|---|
| fact_fees | `fee_id` | Key | Primary key; a row identifier, not a quantity. |
| fact_fees | `client_id` | Foreign key | Points to dim_client; use it to group or count, never to sum. |
| fact_fees | `advisor_id` | Foreign key | Points to dim_advisor; use it to group or count, never to sum. |
| fact_portfolio_snapshot | `snapshot_id` | Key | Primary key; a row identifier, not a quantity. |
| fact_portfolio_snapshot | `client_id` | Foreign key | Points to dim_client; use it to group or count, never to sum. |
| fact_portfolio_snapshot | `advisor_id` | Foreign key | Points to dim_advisor; use it to group or count, never to sum. |
| fact_portfolio_snapshot | `benchmark_id` | Foreign key | Points to dim_benchmark; use it to group or count, never to sum. |
| fact_portfolio_snapshot | `year` | Date part | Copy of the snapshot_date year; use it to filter or group, never to sum. |
| fact_portfolio_snapshot | `month` | Date part | Copy of the snapshot_date month; use it to filter or group, never to sum. |
| fact_portfolio_snapshot | `is_return_suspect` | Flag (0/1) | Marks returns to exclude or review; SUM gives a count of flagged rows, which is the same as COUNT with a filter. |
| fact_transactions | `txn_id` | Key | Primary key; a row identifier, not a quantity. |
| fact_transactions | `client_id` | Foreign key | Points to dim_client; use it to group or count, never to sum. |
| fact_transactions | `advisor_id` | Foreign key | Points to dim_advisor; use it to group or count, never to sum. |
| fact_transactions | `security_id` | Foreign key | Points to dim_security; stored as a float only because cash rows are blank. |
| fact_transactions | `is_post_departure_txn` | Flag (0/1) | Marks transactions after the advisor left; SUM gives a count of flagged rows, not a dollar amount. |

### Numeric attributes in dimension tables that should never be summed

| Table | Column | Why it should never be summed |
|---|---|---|
| dim_advisor | `advisor_id` | Key; a label, not a quantity. |
| dim_advisor | `years_experience` | Summed across advisors it means nothing, and once joined to fact rows it is repeated for every fee or trade; use AVG, MIN or MAX. |
| dim_advisor | `aum_target_mm` | A per-advisor target; joined to the facts it repeats on every row and inflates the total. Compare it with the advisor's actual AUM instead. |
| dim_benchmark | `benchmark_id` | Key; a label, not a quantity. |
| dim_client | `client_id` | Key; a label, not a quantity. |
| dim_client | `age` | Summing ages has no meaning; use AVG or age bands. |
| dim_client | `zip_code` | A numeric code, not a number; it should really be stored as text (leading zeros get lost). |
| dim_client | `initial_aum` | A one-time starting value per client; joined to monthly facts it repeats every month, so the sum is inflated. Use the fact table's aum for totals. |
| dim_client | `advisor_id` | Foreign key to dim_advisor; a label, not a quantity. |
| dim_client | `is_active` | True/false flag; count clients with it, never sum it as an amount. |
| dim_date | `date_id` | Surrogate key; a label, not a quantity. |
| dim_date | `year` | Calendar label; summing years has no meaning. |
| dim_date | `quarter` | Calendar label; summing quarters has no meaning. |
| dim_date | `month` | Calendar label; summing months has no meaning. |
| dim_date | `week_of_year` | Calendar label; summing weeks has no meaning. |
| dim_date | `is_month_end` | 0/1 flag for filtering; its sum is only a day count. |
| dim_date | `is_quarter_end` | 0/1 flag for filtering; its sum is only a day count. |
| dim_date | `is_year_end` | 0/1 flag for filtering; its sum is only a day count. |
| dim_date | `is_trading_day` | 0/1 flag for filtering; its sum is only a day count. |
| dim_security | `security_id` | Key; a label, not a quantity. |
| dim_security | `benchmark_id` | Foreign key to dim_benchmark; a label, not a quantity. |
| dim_security | `is_etf` | True/false flag; count ETFs with it, never sum it as an amount. |
| dim_security | `expense_ratio_bps` | A rate; adding rates is meaningless, so use an AUM-weighted average. Blank for non-fund securities. |

---

## Part E ##
(a) Some transaction rows have no security_id. Why? and is that a defect?
- Some transaction rows have no security_id because they are cash-only transactions. This includes a deposit, withdrawl, or advisory fee. No security_id is bought, sold, or paid out in those transactions so there is noting for security_id to point towards. This is not a defect. It is an optional key on purpose. 

(b) advisor_id appears in dim_client and on all three fact tables. Which relationship(s) did you keep active, and why?
- I decided to keep three active relationships from advisor_id on fact_fees, fact_portfolio_snapshot, and fact_transactions to dim_advisor (advisor_id). I did not create dim_client (advisor_id) relationship with dim_advisor because then there would be two filter paths from dim_advisor to each fact table and Power BI would reject it. I checked the data and the tables matched so leaving the link out does not lose information.

(c) The date dimension has its own date_id, but the facts store calendar dates. Which column did you join on, and why?
- I joined each fact's date column to dim_date (date), not the date_id. The facts store calendar date while date_id is just a sequence of numbers. Date matches with the date column in the fact sheets. I also marked dim_date as the date table using the date column. 

(d) Star or snowflake: does anything in the eight tables connect one dimension to another? Did you keep it?
- Two links in the data connect one dimension to another. The link of dim_client (advisor_id) to dim_advisor and dim_security (benchmarch_id) to dim_benchmark. The model is a star schema with one snowflaked branch. 

---

## Part F ## 

1. fact_fees
Question: How much did we earn in fees this year?
Does not require DAX

2. fact_transactions
Question: How many transactions did we make this year? How many of each type? 
Does not require DAX

3. fact_portfolio_snapshot
Question: How much money does Wildcat Capital manage each year?
Can not be answered without a DAX. To fix this, I would add a flag to fact_portfolio_snapshot like is year end. I would set it to 1 on each clients last snapshot of the year and 0 otherwise. Then I could filter the visual and sum aum by year. 
