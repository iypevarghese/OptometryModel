# File: financial_model.py
# Financial Modeling Application for Specialized Optometry Services
# Computes projections, cash flows, and key ratios based on user inputs

from dataclasses import dataclass
from typing import List
import pandas as pd

# Constants
MONTHS = [
    "Jan 2025", "Feb 2025", "Mar 2025", "Apr 2025", "May 2025", "Jun 2025",
    "Jul 2025", "Aug 2025", "Sep 2025", "Oct 2025", "Nov 2025", "Dec 2025"
]

@dataclass
class Inputs:
    avg_patient_visits: int
    avg_revenue_per_visit: float
    monthly_product_sales: float
    fixed_overheads: float
    variable_cost_per_visit: float
    equipment_purchase: float
    receivable_days: int
    payable_days: int
    debt_interest_rate: float  # annual rate (e.g., 0.085)
    equity_injection: float

    def to_dict(self):
        return self.__dict__

# 1. Build Projection
def build_projection(inputs: Inputs) -> pd.DataFrame:
    data = {
        'Month': MONTHS,
        'Patient Revenue': [inputs.avg_patient_visits * inputs.avg_revenue_per_visit] * 12,
        'Product Revenue': [inputs.monthly_product_sales] * 12
    }
    df = pd.DataFrame(data)
    df['Total Revenue'] = df['Patient Revenue'] + df['Product Revenue']
    df['Variable Costs'] = [inputs.avg_patient_visits * inputs.variable_cost_per_visit] * 12
    df['Gross Profit'] = df['Total Revenue'] - df['Variable Costs']
    df['Fixed Overheads'] = inputs.fixed_overheads
    df['EBITDA'] = df['Gross Profit'] - df['Fixed Overheads']
    # Depreciation: straight-line over 5 years
    dep = inputs.equipment_purchase / 60  # monthly
    df['Depreciation'] = dep
    df['EBIT'] = df['EBITDA'] - df['Depreciation']
    # Interest expense monthly
    monthly_debt = inputs.equity_injection  # assuming debt = equity for simplicity
    int_exp = monthly_debt * inputs.debt_interest_rate / 12
    df['Interest Expense'] = int_exp
    # Taxes: assume 25% of EBIT if positive
    df['Taxes'] = df['EBIT'].apply(lambda x: max(x * 0.25, 0))
    df['Net Profit'] = df['EBIT'] - df['Interest Expense'] - df['Taxes']
    return df

# 2. Build Cash Flow
def build_cashflow(inputs: Inputs, proj: pd.DataFrame) -> pd.DataFrame:
    cf = pd.DataFrame({'Month': MONTHS})
    cf['Net Profit'] = proj['Net Profit']
    cf['+ Depreciation'] = proj['Depreciation']
    # Change in receivables and payables
    rev = proj['Total Revenue']
    var = proj['Variable Costs']
    cf['Δ Receivables'] = -rev * inputs.receivable_days / 365
    cf['Δ Payables'] = var * inputs.payable_days / 365
    cf['Cash from Operations'] = cf[['Net Profit', '+ Depreciation', 'Δ Receivables', 'Δ Payables']].sum(axis=1)
    cf['CapEx'] = -inputs.equipment_purchase  # one time in Jan, else 0
    cf.loc[1:, 'CapEx'] = 0
    cf['Equity Injection'] = inputs.equity_injection
    cf['Net Borrowing'] = 0  # placeholder
    cf['Net Change in Cash'] = cf['Cash from Operations'] + cf['CapEx'] + cf['Equity Injection'] + cf['Net Borrowing']
    cf['Opening Cash'] = 0
    for i in range(len(cf)):
        if i == 0:
            cf.at[i, 'Opening Cash'] = inputs.equity_injection
        else:
            cf.at[i, 'Opening Cash'] = cf.at[i-1, 'Closing Cash']
        cf.at[i, 'Closing Cash'] = cf.at[i, 'Opening Cash'] + cf.at[i, 'Net Change in Cash']
    return cf

# 3. Calculate Ratios
def calculate_ratios(inputs: Inputs, proj: pd.DataFrame, cf: pd.DataFrame) -> pd.DataFrame:
    ratios = {
        'Current Ratio': [1.5],  # placeholder
        'Operating Margin': [proj['EBITDA'].sum() / proj['Total Revenue'].sum()],
        'ROE': [proj['Net Profit'].sum() / inputs.equity_injection],
        'DSCR': [cf['Cash from Operations'].sum() / (inputs.debt_interest_rate * inputs.equity_injection / 12 * 12)]
    }
    return pd.DataFrame(ratios)

# CLI entry-point
if __name__ == "__main__":
    import argparse, json
    parser = argparse.ArgumentParser(description="Run Optometry Financial Model")
    parser.add_argument("--inputs", type=str, help="Path to JSON file with inputs")
    parser.add_argument("--out_prefix", type=str, default="output", help="Prefix for output files")
    args = parser.parse_args()
    # Load inputs
    with open(args.inputs) as f:
        data = json.load(f)
    inp = Inputs(**data)
    proj = build_projection(inp)
    cf = build_cashflow(inp, proj)
    rat = calculate_ratios(inp, proj, cf)
    # Save outputs
    proj.to_csv(f"{args.out_prefix}_projection.csv", index=False)
    cf.to_csv(f"{args.out_prefix}_cashflow.csv", index=False)
    rat.to_csv(f"{args.out_prefix}_ratios.csv", index=False)
