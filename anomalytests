import streamlit as st
import pandas as pd
import numpy as np
import altair as alt
import openpyxl

st.set_page_config(page_title="🧠 Outlier Dashboard", layout="wide")

# Inject custom styles
st.markdown("""
    <style>
        html, body, [class*="css"]  {
            font-family: 'wpp', Arial, sans-serif;
            background-color: #f5f5f5;
        }
        h1, h2, h3, .stApp > header {
            color: #0A2756;
        }
        .main > div:first-child {
            background-color: white;
            padding: 1rem;
            border-radius: 0.5rem;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
        }
        .block-container {
            padding-top: 0;
        }
    </style>
""", unsafe_allow_html=True)

# Display WPP logo

st.title("📊 Campaign Performance Analyzer v2")

# --- Upload Excel ---
uploaded_file = st.file_uploader("Upload raw Excel file (Calcs tab required)", type=["xlsx"])
if uploaded_file:
    df = pd.read_excel(uploaded_file, sheet_name="Calcs")

    # --- Data Prep ---
    df["Date"] = pd.to_datetime(df["Date"], errors="coerce")
    df = df.dropna(subset=["CTR", "CVR", "Revenue", "Costs", "Orders", "Retailer", "Line Item", "Date"])
    df["ASP"] = df.apply(lambda x: x["Revenue"] / x["Orders"] if x["Orders"] > 0 else np.nan, axis=1)
    df["CPO"] = df.apply(lambda x: x["Costs"] / x["Orders"] if x["Orders"] > 0 else np.nan, axis=1)

    # --- Z-Scores ---
    metric_cols = ["CTR", "CVR", "Revenue", "ASP", "CPO"]
    def detect_outliers(data):
        chunks = []
        for r, g in data.groupby("Retailer"):
            g = g.copy()
            for col in metric_cols:
                mean, std = g[col].mean(), g[col].std()
                g[f"{col}_z"] = (g[col] - mean) / std
                g[f"{col}_flag"] = g[f"{col}_z"].apply(lambda z: "High" if z > 2 else "Low" if z < -2 else None)
            chunks.append(g)
        return pd.concat(chunks)

    df = detect_outliers(df)

    # --- Broken Metrics + Suggested Fixes ---
    def label_issues(row):
        return ", ".join([f"{m}+" if row[f"{m}_flag"] == "High" else f"{m}-" for m in metric_cols if row[f"{m}_flag"]])

    def suggest_fix(issues):
        if not issues: return None
        if "CTR+" in issues and "Revenue-" in issues: return "High engagement, low conversion. Check targeting."
        if "ASP+" in issues and "CPO+" in issues: return "High pricing/cost inefficiency. Consider promo adjustments."
        if "CTR-" in issues: return "Poor engagement. Refresh creative."
        if "CVR-" in issues: return "Low conversion. Audit checkout or product appeal."
        return "Check channel-level strategy."

    df["Broken_Metrics"] = df.apply(label_issues, axis=1)
    df["Needs_Review"] = df["Broken_Metrics"].apply(lambda x: bool(x))
    df["Suggested_Fix"] = df["Broken_Metrics"].apply(suggest_fix)

    flagged = df[df["Needs_Review"] == True]

    # --- Sidebar Filters ---
    st.sidebar.header("🎛️ Filters")
    retailers = st.sidebar.multiselect("Retailers", df["Retailer"].unique(), default=df["Retailer"].unique())
    date_range = st.sidebar.date_input("Date Range", [df["Date"].min(), df["Date"].max()])
    metric_focus = st.sidebar.selectbox("Focus Metric", ["All"] + metric_cols)
    search = st.sidebar.text_input("Line Item Search")

    f = flagged.copy()
    f = f[(f["Retailer"].isin(retailers)) & (f["Date"] >= pd.to_datetime(date_range[0])) & (f["Date"] <= pd.to_datetime(date_range[1]))]
    if metric_focus != "All":
        f = f[f["Broken_Metrics"].str.contains(metric_focus[:3], na=False)]
    if search:
        f = f[f["Line Item"].str.contains(search, case=False)]

    # --- Tabs Layout ---
    tab1, tab2, tab3, tab4 = st.tabs(["📈 Overview", "📉 Metric Breakdown", "🧠 Drilldown", "🧰 Fixes"])

    with tab1:
        st.subheader("💰 Revenue Pareto")
        selected_retailer = st.selectbox("Select Retailer", df["Retailer"].unique())
        rev = df[df["Retailer"] == selected_retailer].groupby("Retailer")["Revenue"].sum().reset_index().sort_values("Revenue", ascending=False)
        rev["Cumulative %"] = rev["Revenue"].cumsum() / rev["Revenue"].sum()
        pareto = alt.Chart(rev).mark_bar(color="#0582CA").encode(
            x=alt.X("Retailer", sort="-y"), y="Revenue", tooltip=["Retailer", "Revenue"]
        ) + alt.Chart(rev).mark_line(point=True, color="#FF9C00").encode(
            x="Retailer", y=alt.Y("Cumulative %", axis=alt.Axis(format="%"))
        )
        st.altair_chart(pareto, use_container_width=True)

        st.subheader("📊 Count of Broken Metric Types")
        def combo_counts(vals):
            return ", ".join(sorted(set(vals.split(", ")))) if pd.notna(vals) else None
        f["Issue Combo"] = f["Broken_Metrics"].apply(combo_counts)
        combo_counts = f["Issue Combo"].value_counts().reset_index()
        combo_counts.columns = ["Issue Type", "Count"]
        chart = alt.Chart(combo_counts).mark_bar(color="#56319F").encode(
            x="Count", y=alt.Y("Issue Type", sort="-x")
        )
        st.altair_chart(chart, use_container_width=True)

    with tab2:
        st.subheader("📈 Metric Z-Scores by Retailer")
        z_melt = df.melt(
            id_vars=["Retailer"],
            value_vars=[f"{m}_z" for m in metric_cols],
            var_name="Metric", value_name="Z"
        )
        z_melt["Metric"] = z_melt["Metric"].str.replace("_z", "")
        chart = alt.Chart(z_melt).mark_circle(size=30).encode(
            x="Retailer", y="Z", color="Metric", tooltip=["Retailer", "Metric", "Z"]
        ).properties(height=400)
        st.altair_chart(chart, use_container_width=True)

    with tab3:
        st.subheader("🧠 Campaigns That Need Fixing")
        st.dataframe(f[["Date", "Retailer", "Line Item", "Broken_Metrics", "Suggested_Fix", "CTR", "CVR", "ASP", "CPO", "Revenue"]], use_container_width=True)
        st.download_button("📥 Download Fix List", f.to_csv(index=False), file_name="flagged_campaigns.csv")

    with tab4:
        st.subheader("📉 Moving Averages + Anomalies")
        selected_metric = st.selectbox("Select Metric to Smooth", metric_cols)
        ma_df = df[["Date", selected_metric]].sort_values("Date").copy()
        ma_df = ma_df.dropna()
        ma_df["MA"] = ma_df[selected_metric].rolling(window=7).mean()
        ma_df["MAD"] = ma_df[selected_metric].rolling(window=7).apply(lambda x: np.median(np.abs(x - np.median(x))))
        ma_df["Anomaly"] = abs(ma_df[selected_metric] - ma_df["MA"]) > (3 * ma_df["MAD"])

        base = alt.Chart(ma_df).encode(x="Date")
        line = base.mark_line(color="#00B5B1").encode(y=selected_metric)
        smooth = base.mark_line(color="#FF2AD4").encode(y="MA")
        points = base.mark_circle(color="#FF9C00", size=60).encode(y=selected_metric).transform_filter("datum.Anomaly == true")
        st.altair_chart(line + smooth + points, use_container_width=True)

else:
    st.info("👈 Upload your Excel file to get started.")
