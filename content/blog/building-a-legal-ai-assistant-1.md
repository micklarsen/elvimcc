+++
title = "Building your own LLM with Pyhton - Legal AI Assistant (Part 1: Tools and Setup)"
date = 2025-09-20
draft = false
+++

# Building a Legal AI Assistant with Python  

> ⚠️ **Disclaimer (read this before you hyperventilate about “AI”)**  
>  
> I am *not* an “AI influencer”, I don’t spend my days asking coding assistants to write my for-loops, and I am frankly nauseated by the hype train around this stuff.  
>  
> Every corporate pitch deck these days seems to think “AI” is a magic solution that must be applied to everything from **paper clips to lotion**. It’s not.  
>  
> This technology is still immature, it makes mistakes, and it’s not magic fairy dust. But — if you actually use your head while working with it — it *can* be a useful tool. That’s the spirit of this series: not hype, not marketing, but a practical experiment.  


### Part 1: Tools and Setup

This is part one of a two-part series where I document how I built a small **AI-powered assistant** for the Danish *Almenlejeloven* (Housing Act). You can use these technologies for building RAGs for anything of course.  

- **Part 1** (this post): Introduces the tools and how to prepare the data.  
- **Part 2**: Shows how to connect everything into a working Retrieval-Augmented Generation (RAG) system.  

---

## Why Build This?

Large Language Models (LLMs) like ChatGPT are powerful, but they have two issues when you want to use them with **your own documents**:

1. **They don’t know your data** – unless it was in their training set, they won’t have access to your exact law, manual, or policy.  
2. **They can hallucinate** – making up answers that sound correct but aren’t.<sup>1</sup>

To solve this, we need two things:  
- A way to **store and search our own text data** by meaning.  
- A way to **run a language model locally** and give it just the text it needs.  

That’s where **Chroma** and **Ollama** come in.  

---

## What is Ollama?

[Ollama](https://ollama.com/) is a tool that lets you run open-source LLMs *locally* on your own machine.  

- It works on macOS, Linux, and Windows.  
- It has its own model catalog: `llama2`, `mistral`, `gemma`, etc.  
- It provides a simple API on `http://localhost:11434` so you can send prompts from Python.  

Why use Ollama?  
- **Privacy**: Your questions and documents never leave your machine.  
- **Flexibility**: You can try different models easily.  
- **Cost**: No API keys or cloud bills.  

Example request to Ollama:

```python
import requests

resp = requests.post(
    "http://localhost:11434/api/chat",
    json={
        "model": "llama3.1",
        "messages": [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "What is the capital of Denmark?"}
        ]
    }
)

print(resp.json()["message"]["content"])
# -> "The capital of Denmark is Copenhagen."
````

---

## What is Chroma?

[Chroma](https://www.trychroma.com/) is a **vector database**. Instead of storing text as plain strings, it stores **embeddings**: numerical representations of meaning.

* If you ask about “rehousing” and your text says “erstatningsbolig”, embeddings help the database see that these are *semantically related*.
* You can query Chroma with a piece of text, and it returns the most similar text chunks from your collection.

Example usage:

```python
import chromadb
from chromadb.utils import embedding_functions

# Create a persistent client
client = chromadb.PersistentClient(path="db")

# Use a multilingual embedding model
emb_fn = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="paraphrase-multilingual-MiniLM-L12-v2"
)

# Create collection
coll = client.get_or_create_collection(name="laws", embedding_function=emb_fn)

# Add some documents
coll.add(
    documents=[
        "§ 86a: Temporary replacement housing must be suitable and in the same municipality.",
        "§ 85: A landlord must offer a replacement home if a tenant is terminated."
    ],
    ids=["sec86a", "sec85"]
)

# Query
res = coll.query(query_texts=["rehousing rules"], n_results=2)
print(res["documents"])
# -> returns the most relevant law sections
```

---

## Preparing the Danish Housing Act (Markdown → Chunks)

The Danish Housing Act is available online and can be exported to Markdown format. But the raw file is **too big** and messy for our use. We need to:

1. **Split by § (sections)**
   Each paragraph (§1, §2, §3 …) becomes its own file.
2. **Clean up Markdown**
   Remove links, images, ads, and keep only the text.
3. **Chunk the text**
   Long sections are further split into \~1200-character chunks. This is important because:

   * LLMs have context size limits.
   * Smaller chunks improve retrieval accuracy.
   * Overlap between chunks prevents losing context between cuts.

Example chunking function in Python:

```python
import re

def chunk_text(text, max_chars=1200, overlap=150):
    # Split the input text into paragraphs.
    # We split on blank lines (\n\s*\n) and remove empty ones.
    paras = [p.strip() for p in re.split(r"\n\s*\n", text) if p.strip()]

    # chunks = final list of text pieces
    # buf = temporary buffer we build up until it's "full enough"
    chunks, buf = [], ""

    # Loop through each paragraph
    for p in paras:
        # Try to add this paragraph to the buffer
        cand = (buf + "\n\n" + p).strip() if buf else p

        # If the combined buffer is still small enough, keep it
        if len(cand) <= max_chars:
            buf = cand
        else:
            # Otherwise: the buffer is full, save it as a chunk
            if buf:
                chunks.append(buf)

            # Now handle the current paragraph, which is too long
            # Break it into hard slices of max_chars size
            # (with overlap to avoid cutting sentences too harshly)
            for i in range(0, len(p), max_chars - overlap):
                chunks.append(p[i:i+max_chars])

            # Reset buffer
            buf = ""

    # If there's leftover text in the buffer, save it too
    if buf:
        chunks.append(buf)

    return chunks

```

---

## Diagram: How It Fits Together

Here’s a simple flow of the system we’re building:

![img](/images/llm-mermaid.png)

---

## "What about 'Markov Chains' - are we using that?"

Older text generation methods like **Markov chains** only look at the probability of one word following another (Or several using ngrams). They can generate funny sentences, but they have no true understanding of meaning.

LLMs, by contrast:

* Encode meaning in embeddings.
* Handle long-range dependencies.
* Can answer questions based on provided context.

So while a Markov chain might generate “The tenant pays rent daily,” only an LLM with context from §86a can generate a correct answer about rehousing obligations.

---

## To be continued (Part 2)

In **Part 2**, we’ll connect the dots:

* Retrieve relevant chunks with Chroma.
* Feed them into Ollama.
* Get answers with **citations** to the exact law paragraphs.

This turns the raw Housing Act into a **mini legal AI assistant** you can run on your own machine.

><sup>1</sup> I hate the term **“hallucinate”** here. Models aren’t taking LSD and seeing pink elephants — they’re just **wrong**. Call it “fabricating”, “lying”, or “predicting nonsense if you squint too hard”. Anything but hallucinating.  