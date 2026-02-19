# intro-data-capstone-biodiversity

import streamlit as st
from pypdf2 import PdfReader
from io import BytesIO

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
            else:
                st.warning("No text could be extracted from this PDF (it may be scanned or image)")
        except Exception as e:
            st.error(f"Error reading PDF: {e}")
else:
    st.info("Upload a PDF to see its text content.")