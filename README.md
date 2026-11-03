# mis3060-bi-ai-HW05 #
## Jacob Srivastava ##
## HW 05 ##

## Structure ##

- [ ] `README.md` (structure, how to run, Claude Code findings)
- [ ] `star_design.md` (Parts A, B, C, E and F)
- [ ] `powerbi/hw05.pbix` · `powerbi/model_view.png` ·

---

## How to Run ##
1. Open terminal in the HW05 folder after clone command from githib. 
2. Set up the virtual environment: `source .venv/bin/activate`
3. Run the following scripts: 
    - explore.py: This shows each tables rows, columns, and blanks
    - star_design.py: this checks the keys and the grain
    - additivity.py: this labels each number column as additive, semi-additive, or non-additive
4. Open powerbi/hwo5.pbix to see the model. Or view the model in the model_view.png. 
** I used Claude Code to create the python code files. The output from the files was used to answer questions in the star_design.md file. 
---

## Claude Code Findings ##


- All the keys are clean: no duplicates and no missing matches.
- The fact tables join to the date table on `date`, not `date_id`.
- Some transactions have no `security_id` because they're cash moves (deposits, withdrawals, fees). That's normal.
- Fees and snapshots have one row per client per month. Transactions have one row per transaction.
- Don't add up AUM across months, because it's a balance.
- Don't add up returns, percentages or prices. Average them instead.
- I didn't link `dim_client` to `dim_advisor`, because the fact tables already connect to advisors directly.

**(These findings were from Claude Code and then I reviewed them)
