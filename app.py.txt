# File: app.py
# Streamlit UI for the Optometry Services Financial Model

import streamlit as st
from financial_model import Inputs, build_projection, build_cashflow, calculate_ratios

# -- Sidebar Inputs --
st.set_page_config(page_title="Optometry Financial Model", layout="wide")
st.sidebar.title("Your Assumptions")
visits  = st.sidebar.number_input("Monthly number of patients you’ll serve", min_value=0, value=1000)
rev     = st.sidebar.number_input("Average revenue per patient (₹)", min_value=0.0, value=1500.0)
prod    = st.sidebar.number_input("Monthly sales of glasses & lenses (₹)", min_value=0.0, value=200000.0)
fixed   = st.sidebar.number_input("Fixed overheads (₹)", min_value=0.0, value=200000.0)
varcost = st.sidebar.number_input("Variable cost per visit (₹)", min_value=0.0, value=300.0)
equip   = st.sidebar.number_input("Equipment investment (₹)", min_value=0.0, value=500000.0)
recdays = st.sidebar.number_input("Receivable days", min_value=0, value=30)
paydays = st.sidebar.number_input("Payable days", min_value=0, value=45)
rate    = st.sidebar.slider("Debt interest rate (%)", min_value=0.0, max_value=20.0, value=8.5) / 100
equity  = st.sidebar.number_input("Equity injection (₹)", min_value=0.0, value=1000000.0)
scenario = st.sidebar.selectbox("Scenario", ["Base", "Upside (+10%)", "Downside (–10%)"])
factor = {"Base":1, "Upside (+10%)":1.1, "Downside (–10%)":0.9}[scenario]

if st.sidebar.button("Calculate"):
    inp = Inputs(
        avg_patient_visits=int(visits * factor),
        avg_revenue_per_visit=rev * factor,
        monthly_product_sales=prod * factor,
        fixed_overheads=fixed,
        variable_cost_per_visit=varcost,
        equipment_purchase=equip,
        receivable_days=int(recdays),
        payable_days=int(paydays),
        debt_interest_rate=rate,
        equity_injection=equity
    )
    proj = build_projection(inp)
    cf = build_cashflow(inp, proj)
    rat = calculate_ratios(inp, proj, cf)

    # -- Main Layout --
    tabs = st.tabs(["Projection", "Cash Flow", "Ratios", "Dashboard"])
    with tabs[0]:
        st.header("Revenue & Profit Projection")
        st.dataframe(proj, use_container_width=True)
        csv = proj.to_csv(index=False).encode('utf-8')
        st.download_button("Download Projection CSV", csv, "projection.csv", "text/csv")
    with tabs[1]:
        st.header("Statement of Cash Flows")
        st.dataframe(cf, use_container_width=True)
        csv = cf.to_csv(index=False).encode('utf-8')
        st.download_button("Download Cash Flow CSV", csv, "cashflow.csv", "text/csv")
    with tabs[2]:
        st.header("Key Financial Ratios")
        st.table(rat)
        csv = rat.to_csv(index=False).encode('utf-8')
        st.download_button("Download Ratios CSV", csv, "ratios.csv", "text/csv")
    with tabs[3]:
        st.header("Dashboard Charts")
        st.line_chart(proj.set_index('Month')['Total Revenue'])
        st.line_chart(cf.set_index('Month')['Closing Cash'])
