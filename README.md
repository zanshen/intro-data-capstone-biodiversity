import re
import streamlit as st
from PyPDF2 import PdfReader
from io import BytesIO
import pandas as pd

# ---------------------------------------------------------------------------
# Medicaid eligibility: keywords that may indicate unreported income or assets
# (case-insensitive matching)
# ---------------------------------------------------------------------------
MEDICAID_FLAG_KEYWORDS = [
    "life insurance",
    "pension",
    "annuity",
    "dividend",
    "unknown account",
]

# Threshold above which a transaction is flagged (single transaction)
LARGE_AMOUNT_THRESHOLD = 10_000


def parse_amounts_from_line(line):
    """
    Find all dollar amounts in a line of text.
    Handles formats like: $1,234.56  1234.56  (1,234.56) for negatives.
    Returns a list of floats.
    """
    amounts = []
    # Match optional $, optional spaces, optional (, digits with commas or decimals, optional )
    for m in re.finditer(r"\$?\s*\(?([\d,]+\.?\d*)\)?", line):
        num_str = m.group(1).replace(",", "")
        try:
            val = float(num_str)
            # Parentheses usually mean negative (e.g. (500.00))
            if "(" in m.group(0):
                val = -val
            amounts.append(val)
        except ValueError:
            pass
    return amounts


def parse_date_from_line(line):
    """
    Find the first date in a line. Supports MM/DD/YYYY and YYYY-MM-DD.
    Returns empty string if no date found.
    """
    m = re.search(r"\d{1,2}/\d{1,2}/\d{2,4}", line)
    if m:
        return m.group(0)
    m = re.search(r"\d{4}-\d{2}-\d{2}", line)
    if m:
        return m.group(0)
    return ""


def get_flag_reasons(line, amounts):
    """
    Decide why a line/transaction should be flagged for Medicaid review.
    Returns a list of reason strings (empty if not flagged).
    """
    reasons = []
    line_lower = line.lower()

    # Check for large amounts (use absolute value so large debits/credits both count)
    for amt in amounts:
        if abs(amt) >= LARGE_AMOUNT_THRESHOLD:
            reasons.append(f"Amount over ${LARGE_AMOUNT_THRESHOLD:,}")
            break

    # Check for keywords
    for keyword in MEDICAID_FLAG_KEYWORDS:
        if keyword in line_lower:
            reasons.append(f"Keyword: \"{keyword}\"")
            break

    return reasons


def extract_flagged_transactions(full_text):
    """
    Parse the full statement text, find lines with dollar amounts, and flag
    those that meet Medicaid review criteria (large amount or keyword).
    Returns a list of dicts: date, description, amount, flag_reason.
    """
    flagged = []
    # Split into lines and skip completely empty lines
    lines = [ln.strip() for ln in full_text.splitlines() if ln.strip()]

    for line in lines:
        amounts = parse_amounts_from_line(line)
        if not amounts:
            continue

        reasons = get_flag_reasons(line, amounts)
        if not reasons:
            continue

        # Use the largest absolute amount on the line as the "transaction" amount
        # (often the main debit/credit; balance can be smaller or similar)
        main_amount = max(amounts, key=abs)
        date = parse_date_from_line(line)
        # Description is the full line (user can see context); trim for display if very long
        description = line if len(line) <= 200 else line[:197] + "..."

        flagged.append({
            "Date": date,
            "Description": description,
            "Amount": main_amount,
            "Flag reason": "; ".join(reasons),
        })

    return flagged


st.set_page_config(page_title="Hyle AI", layout="wide")
st.title(" Hyle AI - Medicaid Statement Analyzer")

st.write("Upload a bank statement to see flagged transactions.")

uploaded_file = st.file_uploader("Choose a PDF", type="pdf")

if uploaded_file is not None:
    st.success(f"File uploaded: {uploaded_file.name}")
    with st.spinner("Reading PDF..."):
        try:
            pdf = PdfReader(BytesIO(uploaded_file.read()))
            text_parts = []
            for page in pdf.pages:
                text_parts.append(page.extract_text() or "")
            full_text = "\n\n".join(text_parts).strip()

            if full_text:
                st.subheader("Extracted text")
                st.text_area("PDF content", full_text, height=400, label_visibility="collapsed")

                # Parse and flag transactions
                flagged_list = extract_flagged_transactions(full_text)

                st.subheader("Flagged transactions (Medicaid review)")
                if flagged_list:
                    # Build a DataFrame for display and CSV export
                    df = pd.DataFrame(flagged_list)
                    # Display with currency formatting for Amount
                    st.dataframe(
                        df,
                        use_container_width=True,
                        hide_index=True,
                        column_config={
                            "Amount": st.column_config.NumberColumn(
                                "Amount", format="$%.2f"
                            ),
                        },
                    )
                    # CSV export: use the same DataFrame
                    csv_bytes = df.to_csv(index=False).encode("utf-8")
                    st.download_button(
                        label="Download flagged transactions as CSV",
                        data=csv_bytes,
                        file_name="medicaid_flagged_transactions.csv",
                        mime="text/csv",
                    )
                else:
                    st.info("No transactions were flagged. No amounts over ${:,} or keyword matches found.".format(LARGE_AMOUNT_THRESHOLD))
            else:
                st.warning("No text could be extracted from this PDF (it may be scanned or image)")
        except Exception as e:
            st.error(f"Error reading PDF: {e}")
else:
    st.info("Upload a PDF to see its text content.")
