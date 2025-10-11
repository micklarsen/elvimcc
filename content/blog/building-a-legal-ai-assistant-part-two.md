+++
title = "Building your own LLM & RAG with Python (# 2: Getting Answers)"
date = 2025-08-21
draft = false
taxonomies.categories = ["LLM"]
+++

## Part 2: Getting Answers

In [Part 1](../part-1-tools-and-setup/), we:  

- Installed Ollama to run a language model locally.  
- Installed Chroma to store and search text by meaning.  
- Split the Danish Housing Act into clean, overlapping **chunks** for easier retrieval.  

Now it’s time to put it all together and actually ask questions. The final project [is on Github](https://github.com/micklarsen/dklovrag)

---

## Step 1: Ingesting the Law into Chroma

To populate our vector database we will need some code that can ingest all the chunks of markdown we produced in the last post.
The ingestion script (`ingest.py`) loads all our prepared `.md` files and adds them into a Chroma collection.  

Remember, that each chunk is stored with:  

- **Text** (the law paragraph or slice of it)  
- **Metadata**: source file, paragraph number (§), chapter, and chunk index  

This lets us later retrieve not only the text but also know *where it came from*.  

An example of ingesting code could be like this: 

``` python
#!/usr/bin/env python3

import os
from pathlib import Path
import chromadb
from chromadb.config import Settings
from chromadb.utils import embedding_functions

DATA_DIR = Path("data")          # your pre-chunked files live here
DB_DIR = Path("db")              # where Chroma stores data
COLLECTION = "almenlejelov"      # change as you like

# Optional: quiet telemetry (I hate getting nonsense warnings in my terminal - tweak as you like)
os.environ["CHROMADB_DISABLE_TELEMETRY"] = "1"
os.environ["CHROMA_TELEMETRY_ENABLED"] = "false"
os.environ["ANONYMIZED_TELEMETRY"] = "false"

# A multilingual model that handles non-english languages
EMB_MODEL = "paraphrase-multilingual-MiniLM-L12-v2"
emb = embedding_functions.SentenceTransformerEmbeddingFunction(model_name=EMB_MODEL)

def main():
    if not DATA_DIR.exists():
        raise SystemExit(f"Missing data directory: {DATA_DIR.resolve()}")

    # Spin up (persistent) Chroma client and collection
    client = chromadb.PersistentClient(path=str(DB_DIR), settings=Settings(anonymized_telemetry=False))
    coll = client.get_or_create_collection(name=COLLECTION, embedding_function=emb, metadata={"hnsw:space": "cosine"})

    # Collect docs
    docs, ids, metas = [], [], []
    for i, p in enumerate(sorted(DATA_DIR.rglob("*.*")), start=1):
        if p.suffix.lower() not in {".md", ".txt"} or not p.is_file():
            continue
        text = p.read_text(encoding="utf-8", errors="ignore").strip()
        if not text:
            continue

        # Use a stable, readable id: relative path + index
        rid = f"{p.relative_to(DATA_DIR)}::{i}"
        docs.append(text)
        ids.append(rid)
        metas.append({"source": str(p.relative_to(DATA_DIR))})

    if not docs:
        raise SystemExit("No .md/.txt files found under ./data")

    # Write to Chroma
    coll.add(ids=ids, documents=docs, metadatas=metas)
    print(f"✅ Ingested {len(docs)} chunks into '{COLLECTION}' at '{DB_DIR}/'")
    print(f"   Embeddings: {EMB_MODEL}")

if __name__ == "__main__":
    main()

```

Now, when we run our script, we should hopefully see something like:  

```
$ python ingest.py
✅ Ingested 147 chunks into almenlejelov at db/

```

---

## Step 2: Asking a Question
We have now populated our vector database and need to be able to interact with it. This is there the LLM comes into play. 
let's make a script for asking questions - `ask.py`. The idea is:  

1. You type a question.  
2. It converts your question into an *embedding* (vector).  
3. Chroma finds the closest text chunks.  
4. We build a prompt like:  

```

Here are relevant law texts:

§85: \[chunk text...]
§86a: \[chunk text...]

Answer the question using only these.

````

5. This is sent to **Ollama**, which generates a natural-language answer.  
6. The script shows both the **answer** and the **cited sources**.  

An example script for asking could be like this: 

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import os
import sys
import chromadb
from chromadb.config import Settings
from chromadb.utils import embedding_functions

DB_DIR = "db"                   # same dir as used during ingest
COLLECTION = "almenlejelov"     # must match the ingest script

# turn off telemetry for clarity (Like in the ingest.py code)
os.environ["CHROMADB_DISABLE_TELEMETRY"] = "1"
os.environ["CHROMA_TELEMETRY_ENABLED"] = "false"
os.environ["ANONYMIZED_TELEMETRY"] = "false"

# same embedding model as ingest
EMB_MODEL = "paraphrase-multilingual-MiniLM-L12-v2"
emb = embedding_functions.SentenceTransformerEmbeddingFunction(model_name=EMB_MODEL)

def ask(query: str, k: int = 5):
    client = chromadb.PersistentClient(path=DB_DIR, settings=Settings(anonymized_telemetry=False))
    coll = client.get_or_create_collection(name=COLLECTION, embedding_function=emb)
    res = coll.query(query_texts=[query], n_results=k)

    docs = res.get("documents", [[]])[0]
    metas = res.get("metadatas", [[]])[0]

    print(f"🔎 Query: {query}\n")
    for i, (doc, meta) in enumerate(zip(docs, metas), start=1):
        print(f"[{i}] {meta.get('source', '?')}")
        # show first ~200 chars only
        print(doc[:200].replace("\n", " ") + "...\n")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python ask_minimal.py 'your question'")
        sys.exit(1)

    ask(sys.argv[1])

```
---

## Example 1: Rehousing

A question might look like this:  

``` bash
python ask_minimal.py "Hvad siger § 85 om genhusning?"
```

And the answer: 

``` python
"[1] kapitel-10/§085/chunk-001.md"
"Lejeren har krav på en erstatningsbolig, hvis udlejeren opsiger lejemålet på grund af større ombygning...""

"[2] kapitel-10/§086/chunk-001.md"
"Kommunalbestyrelsen kan i særlige tilfælde...""

```
---

## Example 2: A Failure Case

Not every question works perfectly.

**Question:**

```bash
python ask.py "Can my landlord paint my living room pink without asking me?"
```

**Answer:**
The model will probably say something *plausible* about maintenance rules or landlord obligations — but it may not cite the exact relevant law.

This shows the limits: if your question doesn’t map cleanly to how the law is written, retrieval can miss the right chunk.

---

## Why This Works

This approach is called **RAG (Retrieval-Augmented Generation)**.

* Instead of asking the LLM to “know” Danish housing law (it doesn’t), we **ground** it in the actual text.
* Chroma retrieves the most relevant paragraphs.
* Ollama just rephrases those paragraphs into a human-friendly answer.

The assistant isn’t a lawyer — but it saves you from digging through 100 pages of legalese to find §86a.

---

## Reflections

* **Transparency**: Every answer includes citations.
* **Not magic**: If the retrieval step fails, the answer can drift.
* **Reusable pattern**: You can replace the law with manuals, policies, or any other big text corpus.

---

## What’s Next?

Right now, we have a working CLI assistant. Next steps could be:

* Wrapping this in a **FastAPI** or **Flask** web app.
* Adding a search interface with highlighting.
* Expanding with more laws → building a small searchable **legal AI library**.
* Tweaking the model for more/less sensitivity

---

## Final Thoughts

Without using thousands of lines of Python, we built a system that:

* Splits and cleans a law text.
* Stores it in a vector database.
* Retrieves the right chunks for a question.
* Uses an LLM to explain the result with citations.

It’s not perfect and it's certainly not magic. But it can be **useful**.  
And in my book, that beats AI-hype slides about “machine learning for milk cartons” any day 🍺.
