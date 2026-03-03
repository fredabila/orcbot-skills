---
name: arxiv
description: >
  Use this skill whenever an agent needs to interact with arXiv in any way.
  Covers searching and discovering papers, downloading full PDFs, extracting
  structured metadata (authors, abstracts, citations), summarizing papers,
  comparing multiple papers, extracting key findings and methods, and saving
  or organizing references. Trigger this skill whenever the user mentions
  arXiv, research papers, academic papers, finding papers, reading PDFs,
  literature review, or anything research/paper-related — even casually.
  Don't try to wing arXiv tasks from memory; always consult this skill first.
---

# arXiv Skill

A complete guide for agents to search, retrieve, read, and analyze arXiv papers using the **arXiv API** (no API key required) and the `arxiv` Python library.

---

## 1. Setup

### Install the arxiv Python library

```bash
pip install arxiv --break-system-packages
```

### Optional: install PDF reading support

```bash
pip install pymupdf --break-system-packages   # for reading PDFs (fast, reliable)
```

---

## 2. Searching for Papers

### Basic search

```python
import arxiv

client = arxiv.Client()

search = arxiv.Search(
    query="attention is all you need transformers",
    max_results=10,
    sort_by=arxiv.SortCriterion.Relevance
)

for paper in client.results(search):
    print(paper.title)
    print(paper.entry_id)       # e.g. https://arxiv.org/abs/1706.03762
    print(paper.published)
    print(paper.summary[:300])
    print("---")
```

### Search by field

```python
# Search in title only
search = arxiv.Search(query="ti:diffusion models", max_results=5)

# Search in abstract only
search = arxiv.Search(query="abs:reinforcement learning from human feedback", max_results=5)

# Search by author
search = arxiv.Search(query="au:Hinton", max_results=5)

# Combine fields
search = arxiv.Search(query="ti:language model abs:in-context learning", max_results=10)
```

### Search by category

```python
# Filter to a specific arXiv category
search = arxiv.Search(
    query="large language models",
    max_results=20,
    sort_by=arxiv.SortCriterion.SubmittedDate,  # get newest first
)

# Common categories:
# cs.AI  - Artificial Intelligence
# cs.LG  - Machine Learning
# cs.CL  - Computation and Language (NLP)
# cs.CV  - Computer Vision
# cs.RO  - Robotics
# stat.ML - Statistics / ML
# q-bio  - Quantitative Biology
# quant-ph - Quantum Physics
```

### Fetch a paper by arXiv ID

```python
# From a URL like https://arxiv.org/abs/2303.08774
paper = next(client.results(arxiv.Search(id_list=["2303.08774"])))
print(paper.title)
print(paper.authors)
print(paper.summary)
```

---

## 3. Extracting Structured Metadata

### Full metadata available on each result

```python
for paper in client.results(search):
    print({
        "id": paper.entry_id,                        # full URL
        "arxiv_id": paper.get_short_id(),            # e.g. "1706.03762v5"
        "title": paper.title,
        "authors": [a.name for a in paper.authors],
        "abstract": paper.summary,
        "published": paper.published.isoformat(),
        "updated": paper.updated.isoformat(),
        "categories": paper.categories,              # e.g. ['cs.CL', 'cs.AI']
        "primary_category": paper.primary_category,
        "pdf_url": paper.pdf_url,
        "doi": paper.doi,                            # may be None
        "journal_ref": paper.journal_ref,            # may be None
        "comment": paper.comment,                    # may be None
    })
```

---

## 4. Downloading PDFs

### Download a single PDF

```python
import arxiv, os

client = arxiv.Client()
paper = next(client.results(arxiv.Search(id_list=["1706.03762"])))

# Downloads to current directory as e.g. "1706.03762v5.Attention_Is_All_You_Need.pdf"
paper.download_pdf(dirpath="./papers/")
print("Downloaded:", paper.title)
```

### Download multiple PDFs from search results

```python
import os
os.makedirs("./papers", exist_ok=True)

search = arxiv.Search(query="chain of thought prompting", max_results=5)

for paper in client.results(search):
    try:
        path = paper.download_pdf(dirpath="./papers/")
        print(f"✅ {paper.title}")
    except Exception as e:
        print(f"❌ Failed: {paper.title} — {e}")
```

---

## 5. Reading PDF Content

### Extract full text from a PDF using PyMuPDF

```python
import fitz  # pymupdf

def read_pdf(path: str) -> str:
    doc = fitz.open(path)
    text = ""
    for page in doc:
        text += page.get_text()
    return text

text = read_pdf("./papers/1706.03762v5.pdf")
print(text[:2000])  # first 2000 chars
```

### Extract text by section (heuristic)

```python
def extract_sections(text: str) -> dict:
    """Rough section extraction based on common paper headings."""
    import re
    sections = {}
    # Common section headers in ML papers
    markers = ["abstract", "introduction", "related work", "background",
               "method", "methodology", "approach", "experiment",
               "results", "discussion", "conclusion", "references"]
    
    lower = text.lower()
    positions = []
    for m in markers:
        idx = lower.find(f"\n{m}")
        if idx != -1:
            positions.append((idx, m))
    
    positions.sort()
    for i, (pos, name) in enumerate(positions):
        end = positions[i+1][0] if i+1 < len(positions) else len(text)
        sections[name] = text[pos:end].strip()
    
    return sections

sections = extract_sections(text)
print(sections.get("abstract", "Not found"))
print(sections.get("conclusion", "Not found"))
```

---

## 6. Summarizing Papers

### Summarize using Claude (recommended pattern)

```python
def summarize_paper(paper_text: str, title: str) -> str:
    """Pass to Claude for summarization."""
    # Truncate if too long (keep intro + conclusion most useful)
    max_chars = 12000
    excerpt = paper_text[:max_chars] if len(paper_text) > max_chars else paper_text
    
    prompt = f"""You are a research assistant. Summarize the following paper.

Title: {title}

Paper text:
{excerpt}

Provide:
1. **One-line summary** (what this paper does in plain English)
2. **Problem** being solved
3. **Key method/approach**
4. **Main findings/results**
5. **Limitations or open questions**
"""
    return prompt  # pass this to your LLM call

# Or summarize from abstract alone (faster, no PDF needed)
def summarize_from_abstract(paper) -> str:
    return f"""
Title: {paper.title}
Authors: {', '.join(a.name for a in paper.authors[:5])}
Published: {paper.published.strftime('%B %Y')}
Abstract: {paper.summary}
"""
```

---

## 7. Comparing Multiple Papers

### Build a comparison table

```python
def compare_papers(papers: list) -> str:
    """Generate a structured comparison across multiple papers."""
    rows = []
    for p in papers:
        rows.append({
            "title": p.title,
            "year": p.published.year,
            "authors": ", ".join(a.name for a in p.authors[:3]),
            "categories": ", ".join(p.categories[:2]),
            "abstract_snippet": p.summary[:300] + "...",
        })
    
    # Format as markdown table
    header = "| Title | Year | Authors | Categories | Summary |\n"
    divider = "|-------|------|---------|------------|---------|"
    table = header + divider + "\n"
    for r in rows:
        table += f"| {r['title'][:50]} | {r['year']} | {r['authors']} | {r['categories']} | {r['abstract_snippet'][:100]} |\n"
    return table
```

### Prompt for deep comparison

When passing multiple papers to an LLM for comparison, use this structure:

```python
def comparison_prompt(papers_text: list[dict]) -> str:
    papers_block = "\n\n---\n\n".join(
        f"PAPER {i+1}: {p['title']}\n\n{p['text'][:4000]}"
        for i, p in enumerate(papers_text)
    )
    return f"""Compare the following {len(papers_text)} research papers:

{papers_block}

Address:
1. What problem does each paper solve?
2. How do their approaches differ?
3. What are the relative strengths and weaknesses?
4. Which paper's contributions are most significant and why?
5. How do they relate to or build on each other?
"""
```

---

## 8. Saving & Organizing References

### Save references as JSON

```python
import json
from datetime import datetime

def save_references(papers, filepath="references.json"):
    refs = []
    for p in papers:
        refs.append({
            "arxiv_id": p.get_short_id(),
            "title": p.title,
            "authors": [a.name for a in p.authors],
            "abstract": p.summary,
            "published": p.published.isoformat(),
            "pdf_url": p.pdf_url,
            "categories": p.categories,
            "saved_at": datetime.now().isoformat(),
        })
    
    # Load existing if file exists
    try:
        with open(filepath) as f:
            existing = json.load(f)
    except FileNotFoundError:
        existing = []
    
    # Deduplicate by arxiv_id
    existing_ids = {r["arxiv_id"] for r in existing}
    new_refs = [r for r in refs if r["arxiv_id"] not in existing_ids]
    combined = existing + new_refs
    
    with open(filepath, "w") as f:
        json.dump(combined, f, indent=2)
    
    print(f"Saved {len(new_refs)} new references ({len(combined)} total) to {filepath}")
    return combined
```

### Save as BibTeX (for academic use)

```python
def to_bibtex(paper) -> str:
    arxiv_id = paper.get_short_id().replace(".", "_")
    first_author = paper.authors[0].name.split()[-1] if paper.authors else "Unknown"
    year = paper.published.year
    key = f"{first_author}{year}_{arxiv_id}"
    
    authors_str = " and ".join(a.name for a in paper.authors)
    
    return f"""@article{{{key},
  title   = {{{paper.title}}},
  author  = {{{authors_str}}},
  year    = {{{year}}},
  journal = {{arXiv preprint arXiv:{paper.get_short_id()}}},
  url     = {{{paper.entry_url}}},
}}"""
```

---

## 9. Best Practices for Agents

- **Search broadly, then filter** — use 20–50 results and filter by date/category rather than a narrow query
- **Prefer abstracts for speed** — reading the abstract is usually enough to decide relevance; only download PDFs you'll actually analyze
- **Respect rate limits** — add `time.sleep(1)` between bulk downloads to avoid getting throttled
- **Use `id_list` when you know the paper** — much faster than a text search
- **Truncate PDFs intelligently** — for LLM context, the abstract + intro + conclusion is usually more useful than the full text
- **Always record `arxiv_id`** — it's the stable identifier; titles can have typos and URLs can change

---

## 10. Troubleshooting

| Problem | Fix |
|--------|-----|
| `ModuleNotFoundError: arxiv` | `pip install arxiv --break-system-packages` |
| `ModuleNotFoundError: fitz` | `pip install pymupdf --break-system-packages` |
| PDF download fails | Check disk space; try again (transient network issue) |
| Search returns 0 results | Simplify query; avoid special characters |
| Rate limited by arXiv | Add `time.sleep(1)` between requests; use `arxiv.Client(delay_seconds=1)` |
| Paper ID not found | Double-check the ID format — use `"2303.08774"` not `"arxiv:2303.08774"` |
