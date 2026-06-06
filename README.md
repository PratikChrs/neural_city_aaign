## Decisions Log — Trade-offs & Where I Got Stuck

### Decision 1: JSON schema vs Pydantic
Chose raw JSON schema (like the Samsung review example) over Pydantic class.
Reason: simpler to write, output is already a plain dict, no .model_dump() needed.
Trade-off: Pydantic gives better IDE autocomplete and type safety.

### Decision 2: Gemini Flash vs Pro
Chose Flash — it's free tier, fast enough, and structured output works well.
Pro would handle more ambiguous phrasings better but costs money.

### Decision 3: Fuzzy matching for state/crop names
Added `.lower() in s.lower()` matching so "west bengal" matches "West Bengal".
Trade-off: could match wrong state if two state names share a substring.
Better fix: use rapidfuzz for proper fuzzy matching.

### Decision 4: Gradio over Streamlit
Gradio launches faster in Colab, share=True gives instant public URL.
Streamlit needs a separate tunnel setup in Colab.

### Where I Got Stuck
- Gemini occasionally returns null for group_by on trend questions → fixed by
  adding explicit rule in system prompt: "trend questions → group_by=Year"
- Some crop names in dataset have inconsistent casing → fixed in cleaning step
- Gradio image output needs PIL Image not matplotlib figure → added io.BytesIO conversion
