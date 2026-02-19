def parse_amounts_from_line(line):
    """
    Find all dollar amounts in a line of text. More strict pattern.
    Handles formats like: $1,234.56  1234.56  (1,234.56) for negatives.
    Returns a list of floats.
    """
    amounts = []
    # More precise pattern: optional $, optional (, then digits with optional commas and optional .dd
    for m in re.finditer(r'\$?\s*\(?([\d]{1,3}(?:,[\d]{3})*(?:\.\d{2})?)\)?', line):
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
    Find the first date in a line, preferring dates at the beginning.
    Supports MM/DD/YYYY and YYYY-MM-DD.
    Returns empty string if no date found.
    """
    # First, check if line starts with a date (common in transaction listings)
    m = re.search(r'^(\d{1,2}/\d{1,2}/\d{2,4})', line.strip())
    if m:
        return m.group(1)
    # Otherwise, find any date
    m = re.search(r'\d{1,2}/\d{1,2}/\d{2,4}', line)
    if m:
        return m.group(0)
    m = re.search(r'\d{4}-\d{2}-\d{2}', line)
    if m:
        return m.group(0)
    return ""

def extract_flagged_transactions(full_text):
    """
    Parse the full statement text, find lines with dollar amounts, and flag
    those that meet Medicaid review criteria (large amount or keyword).
    Returns a list of dicts: date, description, amount, flag_reason.
    """
    flagged = []
    # Split into lines and skip completely empty lines
    lines = [ln.strip() for ln in full_text.splitlines() if ln.strip()]
    
    # Skip summary lines that aren't individual transactions
    skip_patterns = ["beginning balance", "ending balance", "total", "withdrawals", "deposits and additions"]
    
    for line in lines:
        # Skip summary lines
        line_lower = line.lower()
        if any(pattern in line_lower for pattern in skip_patterns):
            continue
            
        amounts = parse_amounts_from_line(line)
        if not amounts:
            continue

        reasons = get_flag_reasons(line, amounts)
        if not reasons:
            continue

        # Use the largest absolute amount on the line as the "transaction" amount
        main_amount = max(amounts, key=abs)
        date = parse_date_from_line(line)
        description = line if len(line) <= 200 else line[:197] + "..."

        flagged.append({
            "Date": date,
            "Description": description,
            "Amount": main_amount,
            "Flag reason": "; ".join(reasons),
        })

    return flagged