# aganitha-project
poetry new paper_fetcher
cd paper_fetcher
poetry add requests click pandas
paper_fetcher/fetch.py
bio.Entrez
get-papers-list = "paper_fetch.cli:main"

import argparse
import csv
import re
from Bio import Entrez
from datetime import datetime
Entrez.email = "your_email@bhuvanadidgi.com"
COMPANY_KEYEORDS=['pharma','biotech','inc','ltd','gmbh','corp','llc']
def is_non_academic(affiliatiuon):
    if not affliation:
       return false
    affiliation = affiliation.lower()
    return any(keyword in affliation for keyword in COMPANY_KEYWORDS)
def fetch_pubmed_ids(query, debug=False):
    if debug:
        print(f"[DEBUG] searching PubMed for query: {query}")
    handle = Entrez.esearch(db="pubmed", term=query, retmax=50)
    results=Entrez.read(handle)
    handle.close()
    return results["IdList"]

def fetch_pubmed_details(pubmed_ids, debug=False):
    ids_str = ",".join(pubmed_ids)
    if debug:
        print(f"[DEBUG] Fetching details for IDs: {ids_str}")
    handle = Entrez.efetch(db="pubmed", id=ids_str, rettype="medline", retmode="text")
    records = handle.read()
    handle.close()
    return records

def parse_records(records_text, debug=False):
    from Bio import Medline
    from io import StringIO

    records = list(Medline.parse(StringIO(records_text)))
    results = []

    for record in records:
        pubmed_id = record.get("PMID", "")
        title = record.get("TI", "")
        date = record.get("DP", "")
        authors = record.get("AU", [])
        affiliations = record.get("AD", "")
        email = ""

        if isinstance(affiliations, list):
            affiliations = " ".join(affiliations)

        # Try to find email
        email_matches = re.findall(r"[\w\.-]+@[\w\.-]+", affiliations)
        if email_matches:
            email = email_matches[0]

        # Check for company affiliations
        non_academic_authors = []
        company_affiliations = []

        if isinstance(authors, list):
            for author in authors:
                if is_non_academic(affiliations):
                    non_academic_authors.append(author)
                    company_affiliations.append(affiliations)

        if company_affiliations:
            results.append({
                "PubMedID": pubmed_id,
                "Title": title,
                "Publication Date": date,
                "Non-academic Author(s)": "; ".join(non_academic_authors),
                "Company Affiliation(s)": "; ".join(set(company_affiliations)),
                "Corresponding Author Email": email
            })

    return results

def write_to_csv(data, filename=None):
    fieldnames = ["PubMedID", "Title", "Publication Date", "Non-academic Author(s)",
                  "Company Affiliation(s)", "Corresponding Author Email"]
    if filename:
        with open(filename, "w", newline='', encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            writer.writeheader()
            writer.writerows(data)
    else:
        writer = csv.DictWriter(sys.stdout, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(data)

def main():
    parser = argparse.ArgumentParser(description="Fetch PubMed papers with company-affiliated authors.")
    parser.add_argument("query", nargs="?", help="PubMed search query")
    parser.add_argument("-f", "--file", help="Output CSV file")
    parser.add_argument("-d", "--debug", action="store_true", help="Enable debug output")

    args = parser.parse_args()

    if not args.query:
        parser.print_help()
        return

    pubmed_ids = fetch_pubmed_ids(args.query, args.debug)
    records_text = fetch_pubmed_details(pubmed_ids, args.debug)
    results = parse_records(records_text, args.debug)

    if results:
        write_to_csv(results, args.file)
        if args.debug:
            print(f"[DEBUG] Fetched {len(results)} results.")
    else:
        print("No matching papers found.")

if __name__ == "__main__":
    main()
