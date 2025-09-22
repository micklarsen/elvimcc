+++
title = "Building your own LLM with Python - Legal AI Assistant (Part 2: Getting Answers)"
date = 2025-09-21
taxonomies.categories = ["LLM"]
+++

# Building a Legal AI Assistant with Python and Danish laws  
### Part 2: Getting Answers

> ⚠️ **Disclaimer (read this before you hyperventilate about “AI”)**  
>  
> I am *not* an “AI influencer”, I don’t spend my days asking coding assistants to write my for-loops, and I am frankly nauseated by the hype train around this stuff.  
>  
> Every corporate pitch deck these days seems to think “AI” is a magic solution that must be applied to everything from **paper clips to lotion**. It’s not.  
>  
> This technology is still immature, it makes mistakes, and it’s not magic fairy dust. But — if you actually use your head while working with it — it *can* be a useful tool. That’s the spirit of this series: not hype, not marketing, but a practical experiment.  

---

## Recap from Part 1

In [Part 1](../part-1-tools-and-setup/), we:  

- Installed **Ollama** to run a language model locally.  
- Installed **Chroma** to store and search text by meaning.  
- Split the Danish Housing Act into clean, overlapping **chunks** for easier retrieval.  

Now it’s time to put it all together and actually **ask questions**.  

---

## Step 1: Ingesting the Law into Chroma

The ingestion script (`ingest.py`) loads all our prepared `.md` files and adds them into a Chroma collection.  

Each chunk is stored with:  

- **Text** (the law paragraph or slice of it)  
- **Metadata**: source file, paragraph number (§), chapter, and chunk index  

This lets us later retrieve not only the text but also know *where it came from*.  

Example output when running `python ingest.py`:

```

✅ Indekseret 147 tekst-chunks fra 131 filer i db/

```

---

## Step 2: Asking a Question

The `ask.py` script is where the magic happens:  

1. You type a question.  
2. It converts your question into an **embedding** (vector).  
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

---

## Example 1: Rehousing

**Question:**  

```bash
python ask.py "If a tenant must be rehoused, what type of replacement housing must be provided?"
````

**Answer (summarized from §85 and §86a):**

* **Temporary housing** must be suitable, usually in the same municipality (can be a hotel or similar).
* **Permanent housing** must have appropriate size, location, and facilities.
* The landlord must cover reasonable moving costs.

**Cited sources:**

* §085.md
* §086a.md

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

---

## Final Thoughts

With less than 200 lines of Python, we built a system that:

* Splits and cleans a law text.
* Stores it in a vector database.
* Retrieves the right chunks for a question.
* Uses an LLM to explain the result with citations.

It’s not perfect and it's certainly not magic. But it can be **useful**.  
And in my book, that beats AI-hype slides about “machine learning for milk cartons” any day 🍺.
