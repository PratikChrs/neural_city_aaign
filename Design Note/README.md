# 🌾 Talk to Government Data — Design Note

## Dataset
**Crop Yield in Indian States (1997–2020)**
Source: Kaggle (originally Ministry of Agriculture & Farmers Welfare, Govt. of India)
Link: https://www.kaggle.com/datasets/akshatgupta7/crop-yield-in-indian-states-dataset
Columns: State, Crop, Year, Season, Area (hectares), Production (tonnes), Yield (tonnes/hectare)

---

## 1. Stack & Decisions
- **LLM:** Gemini 1.5 Flash via LangChain — free, fast, good at structured output
- **Data layer:** Pandas — simple, auditable, every operation is a readable line of code
- **UI:** Gradio — launches inside Colab in one line, gives a public share link
- **Framework:** LangChain `with_structured_output()` with a JSON schema

**Key design decision — Structured JSON, not exec():**
I constrain Gemini to output a JSON plan like:
`{"operation": "top_n", "Crop": "Wheat", "Year": 2015, "group_by": "State"}`
Then my own trusted Python code runs that against pandas.

The alternative is asking Gemini to write Python and running exec() on it.
I rejected that because: model-generated code running on a server is a security risk —
the model could hallucinate a destructive operation, an infinite loop, or leak data.
With JSON, Gemini can only do what my executor explicitly allows.

Trade-off: JSON schema covers ~80% of natural questions well.
Very complex multi-hop questions ("compare wheat yield growth rate vs rice across decades")
may need richer schema. With more time I'd expand the operation types.

---

## 2. Correctness & Trust
- Every number comes from a pandas operation on the real CSV — never from model memory
- Every answer shows "How computed" — the exact steps that ran (filter → group → sort → top-N)
- To prove correctness to a sceptical government officer:
  show them the raw CSV rows that produced the number (I can add df[mask] printout)
- Out-of-scope questions are refused before any computation happens
- Evaluation set includes one hand-verified number checked manually against the CSV

---

## 3. Government Deployment
- **Data residency:** Deploy on-premise or government-approved cloud; CSV never leaves the server
- **Audit trails:** Log every question + JSON plan + result + timestamp + user ID to a database
- **Access control:** Role-based auth — a district officer sees only their district's data
- **Generated code risk:** Solved by design — we use JSON, not exec(). No model-generated code runs.
- **What changes for production:**
  Replace Colab with a proper API server (FastAPI), add auth middleware,
  database-backed immutable audit log, rate limiting, input sanitisation

---

## 4. Scaling
1. **Many tables:** Build a schema registry so Gemini knows which table answers which question,
   then route the query to the right table automatically
2. **Millions of rows:** Replace pandas with DuckDB — same syntax, columnar speed,
   handles hundreds of millions of rows on a single machine
3. **Slow responses:** Cache answers for repeated identical questions;
   pre-aggregate common queries (yearly totals by state) at load time

---

## 5. Validation
- Show 10 questions to non-technical stakeholders (farmers, district officers) — check if answers match domain knowledge
- Watch for: wrong year filters silently returning empty data, NaN in Production treated as zero,
  state name mismatches (e.g. "Orissa" vs "Odisha"), aggregation over duplicates
- Build a regression test suite from the evaluation set; run it on every code change

---

## 6. Honest Limitations
- Dataset ends at 2020 — questions about 2021+ are refused
- JSON schema handles common patterns well; unusual phrasings can confuse Gemini's output
- No authentication on Gradio link — anyone with the URL can query
- Charts are static matplotlib — not interactive or downloadable
- Fuzzy state/crop matching may occasionally pick the wrong match
