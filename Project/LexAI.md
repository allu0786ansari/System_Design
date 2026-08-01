# LexAI Study Notes — Part 1

## Header 1: Foundations &nbsp;·&nbsp; Header 2: Ingestion Pipeline

> These notes map directly to the LexAI codebase (`allu0786ansari/LexAI`). Every topic is explained conceptually first, then tied to the exact file/function where it's implemented, with a "why this and not the obvious alternative" section.

---

# HEADER 1 — FOUNDATIONS

## 1. What is RAG (Retrieval-Augmented Generation) and why it exists

### The concept
An LLM like Gemini or Llama only "knows" what was in its training data, frozen at some cutoff date. Ask it "What is the punishment under Section 302 IPC?" and it will answer from memory — which might be right, might be subtly wrong, and *cannot be verified* because there's no source attached to the answer.

RAG fixes this by splitting the job into two stages instead of one:

1. **Retrieve** — before the LLM answers, search a trusted document collection (here: Indian legal PDFs) for the passages most relevant to the question.
2. **Augment + Generate** — hand those retrieved passages to the LLM as context, and instruct it to answer *using only that context*, not its own memory.

The LLM becomes a "reader that summarizes evidence" rather than an "oracle that recites memorized facts."

### Why this is necessary
- **Hallucination control**: LLMs generate the statistically likely next token, not verified facts. Without grounding, a model can invent a plausible-sounding but wrong section number — dangerous in a legal context.
- **Freshness**: A law can be amended after the LLM's training cutoff. RAG lets you update the *document corpus* without retraining the model.
- **Verifiability**: Because the answer is built from specific retrieved chunks, you can show citations (source file, page, section) — the user can check the primary source. This is exactly what `Citation` objects in `schemas.py` and the `CitationsPanel.jsx` component are for.
- **Domain narrowing**: A general-purpose LLM knows a little about everything. RAG lets a general model behave like a narrow expert system without any fine-tuning.

### How it maps to LexAI
```
PDF laws  →  chunk + embed  →  FAISS/BM25 index   (ingestion — offline, one-time)
User question → retrieve top chunks → LLM answers using only those chunks   (query — online, per-request)
```
This is literally the flow in `qa_service.py::stream_answer()`: HyDE → hybrid retrieval → rerank → generation, with the `SYSTEM_PROMPT` explicitly saying *"Answer the user's question using ONLY the retrieved context below... Do not fabricate section numbers, case names, or citations that are not in the context."*

### Real-world analogy
Think of a law student writing an exam answer. A bad student answers purely from memory — fast, but risks misremembering a section number. A good student flips open the bare act, finds the relevant section, quotes it, and *then* explains it in their own words. RAG forces the LLM to be the "good student."

### Why RAG over the alternatives
| Approach | Problem |
|---|---|
| Fine-tuning the LLM on legal text | Expensive, slow to update, doesn't give per-answer citations, and can still hallucinate between facts it half-learned. |
| Bigger LLM context window (paste entire corpus into the prompt) | Indian legal corpus (IPC + CrPC + Constitution + case law + reports) is far larger than any context window can hold cheaply; also wastes tokens on irrelevant law for every single question. |
| Plain keyword search (no LLM) | Returns raw text, no synthesis, no natural-language answer. |
| **RAG (retrieval + generation)** | Cheap to keep current (just re-run ingestion), gives citations, scales to large corpora, keeps the LLM "on topic." |

---

## 2. Text embeddings — vector representations, cosine similarity vs. dot product

### The concept
An embedding model converts a piece of text into a fixed-length list of numbers (a **vector**), e.g. 768 numbers for `nomic-embed-text`. The vector is positioned in a high-dimensional space such that texts with **similar meaning end up close together**, regardless of the exact words used.

Example: `"the punishment for theft"` and `"penalty for stealing"` use completely different words but should land near each other in embedding space, because an embedding model captures *meaning*, not just character overlap.

### Measuring "closeness": cosine similarity vs. dot product (inner product)
- **Dot product** of two vectors `a · b = Σ(aᵢ × bᵢ)` — this is sensitive to vector *magnitude*. A longer vector can score "more similar" even if its direction is less aligned.
- **Cosine similarity** = dot product divided by the product of the magnitudes: `cos(θ) = (a · b) / (‖a‖ × ‖b‖)`. This only measures the *angle* between vectors, ignoring length — exactly what you want when comparing meaning, not text length.

**The trick used in LexAI**: if you normalize every vector to unit length (magnitude = 1) *before* storing/searching, then dot product and cosine similarity become mathematically identical, because `‖a‖ = ‖b‖ = 1` cancels out the denominator. This is why `vector_store.py` normalizes every vector:

```python
@staticmethod
def _normalize(vectors: np.ndarray) -> np.ndarray:
    norms = np.linalg.norm(vectors, axis=1, keepdims=True)
    norms[norms == 0] = 1e-10
    return vectors / norms
```
and then builds a plain **inner-product** FAISS index (`faiss.IndexFlatIP`) rather than a cosine-specific index — because after normalization, inner product *is* cosine similarity, and inner product is the faster, simpler primitive FAISS supports natively.

### Why this is necessary
Without embeddings, a computer can only compare text by exact string/keyword overlap (like `Ctrl+F`). Embeddings let the system understand that "murder" and "homicide" are related concepts even with zero shared characters — this is the entire basis of *semantic* (dense) search, as opposed to *lexical* (keyword) search.

### Real-world analogy
Imagine plotting every word/sentence in a 2D map where similar meanings cluster together, like a "concept map" — "murder," "homicide," and "killing" sit close together in one corner, while "contract," "agreement," and "consideration" cluster in another corner. An embedding model draws this map, just in hundreds of dimensions instead of 2.

### Why this approach over alternatives
| Approach | Limitation |
|---|---|
| Exact keyword match | Misses synonyms, paraphrases, different phrasing of the same legal concept |
| TF-IDF vectors (classic, no neural network) | Still just word-overlap statistics, no real semantic understanding |
| **Neural embeddings (dense vectors)** | Captures semantic meaning learned from massive text corpora — retrieves conceptually relevant text even with zero shared words |

This is also *why LexAI doesn't rely on embeddings alone* — dense embeddings are weak at exact matches like a specific IPC section number, which is why BM25 (Header 5) is paired alongside it.

---

## 3. Tokenization basics — why BM25 and embeddings tokenize differently

### The concept
"Tokenization" is the process of breaking text into discrete units a model or algorithm can work with. There are (at least) two very different tokenization needs in this project, doing very different jobs:

1. **Embedding-model tokenization** (internal to the embedding model, e.g. `nomic-embed-text`) — breaks text into subword pieces the *neural network* was trained on, then produces one dense vector per piece, ultimately pooled into a single sentence/chunk vector. This is invisible to LexAI's own code — it happens inside the embedding model.
2. **BM25 tokenization** — a much simpler, explicit process the project *writes itself*, in `bm25_store.py`:

```python
_TOKEN_PATTERN = re.compile(r"[a-z0-9]+")

def tokenize(text: str) -> list[str]:
    return [t for t in _TOKEN_PATTERN.findall(text.lower()) if t not in _STOPWORDS]
```

This lowercases the text, extracts alphanumeric runs as tokens, and drops common **stopwords** (`the`, `and`, `shall`, `of`, etc.).

### Why they must be handled differently
- Embedding models need rich sub-word tokenization because they're doing *meaning-based* comparison — they need to understand grammar and context.
- BM25 is doing pure **term-frequency statistics** — it counts how often exact tokens appear. If you tokenize the query differently from how you tokenized the documents at indexing time, the token sets won't match up, and BM25 will silently return garbage scores. This is exactly why the code comment in `bm25_store.py` says:

> *"The tokenizer here MUST be the one used at query time too... BM25 scoring is meaningless if the index and the query aren't tokenized the same way."*

### Real-world analogy
Imagine a library's card-catalogue index (BM25) versus a reference librarian who actually understands the subject (embeddings). The card catalogue only works if you file and search using the *exact same* indexing rules — file a book under "Criminal-Law" but search for "criminal law" (different casing/punctuation) and a naive catalogue would miss it. The librarian (embedding model), by contrast, understands you mean the same thing regardless of how you phrase the request.

### Why this matters for the project
Legal text is full of section numbers (`Section 302`, `Article 21`) that must match **exactly** — an embedding model might treat "Section 302" and "Section 304" as "close enough" because they look nearly identical, but for law, these are completely different offences. BM25's *exact* token matching is what makes the hybrid system able to nail these precise lookups (see Header 5 — Hybrid Retrieval).

---

## 4. LLM basics: chat models, roles, and streaming generation

### The concept
Modern LLMs used for conversation aren't called with a single raw string — they're called with a structured list of **messages**, each tagged with a **role**:

- `system` — instructions that set the model's behavior/persona for the whole conversation (in LexAI: the `SYSTEM_PROMPT` that forces grounding in retrieved context).
- `user`/`human` — what the person asked.
- `assistant`/`ai` — what the model previously replied (used to give the model conversational memory across turns).

LexAI builds exactly this structure in `qa_service.py`:
```python
messages = [SystemMessage(content=SYSTEM_PROMPT.format(context=context_text))]
messages.extend(history.messages[-8:])   # prior turns
messages.append(HumanMessage(content=question))
```

### Streaming generation
Rather than waiting for the LLM to finish the *entire* answer before showing anything (which can take many seconds for a long legal explanation), the model can **stream** its output **token by token** (or in small text chunks) as it's generated. The API sends each piece to the frontend the instant it's produced, so the user sees the answer "typing out" live — dramatically improving perceived responsiveness even though total generation time is unchanged.

LexAI implements this with `self._llm.astream(messages)`, an async generator that yields chunks as the Ollama backend produces them (see `LocalOllamaChatStream` in `providers.py`, which reads newline-delimited JSON chunks from Ollama's streaming HTTP response).

### Why this is necessary
- **Roles** let the same "conversation" API represent multi-turn dialogue with distinct instructions vs. history vs. new input — without this structure the model can't tell "the rule I must follow" from "something the user said."
- **Streaming** is a UX necessity for anything that takes more than ~1 second to generate — nobody wants to stare at a blank screen while a legal explanation is composed sentence by sentence server-side.

### Real-world analogy
Roles are like a courtroom transcript that clearly labels who's speaking — Judge's instructions (system), what the lawyer asked (user), what the witness said before (assistant history). Streaming is like watching someone type a live chat message versus waiting for them to hit "send" on a whole essay — you get to start reading immediately.

### Why streaming over "wait for full response"
| Approach | Trade-off |
|---|---|
| Wait, then return full answer | Simple to implement, but the user waits with zero feedback — feels broken/slow, especially over web latency |
| **Stream tokens as generated (SSE)** | More complex plumbing (needs the SSE protocol, see Header 9), but perceived latency drops to "time to first token" instead of "time to last token" |

---

## 5. What "chunking" means and why documents can't be embedded whole

### The concept
Embedding models and LLMs both have **context/input limits** — you can't feed a 200-page PDF of the Indian Penal Code into an embedding call and get back one meaningful vector; the meaning would be so diluted/averaged across hundreds of unrelated topics that the resulting vector would be useless for retrieval (a "murder" question and a "theft" question would both retrieve the same, single, all-purpose IPC vector).

**Chunking** solves this by splitting a document into smaller, semantically coherent pieces — small enough that each chunk's embedding captures one focused idea, and small enough that a handful of chunks can fit comfortably in the LLM's generation context window later.

### Why documents can't just be split every N characters (naively)
A crude "every 1000 characters" splitter can cut a sentence — or a legal clause — right in half, destroying meaning at the boundary and producing chunks that don't correspond to complete legal ideas. This is precisely the motivation behind LexAI's custom **semantic chunker** (fully covered in Header 2, topic 9) — splitting where meaning actually shifts, not at an arbitrary character count.

### Why chunk size matters both ways
- **Too large** → dilutes the embedding (mixes multiple topics into one vector, hurting retrieval precision) and burns LLM context budget unnecessarily.
- **Too small** → loses surrounding context a legal clause needs to be understood (e.g., a chunk with just "...shall be punished with imprisonment for life" with no indication of *which offence* this refers to).

LexAI balances this with `chunk_min_chars` (400) and `chunk_max_chars` (1800) settings in `settings.py`, plus a hard-overlap safety net (`chunk_hard_overlap = 220`) so information isn't cleanly severed at a forced split boundary.

### Real-world analogy
Think of highlighting a textbook for exam revision. You don't highlight the entire chapter as one giant block (too broad to be useful when you skim later), and you don't highlight single disconnected words (too fragmented to make sense). You highlight complete, self-contained ideas — a full sentence or short paragraph — which is exactly what a well-tuned chunk is.

### Why this approach over alternatives
| Approach | Problem |
|---|---|
| Embed the whole document | Vector is an unusable average of everything in the document; can't retrieve one specific clause |
| Fixed-size character/token splitting | Fast and simple, but cuts sentences/clauses mid-thought at arbitrary boundaries |
| **Semantic chunking (LexAI's approach)** | Splits where the *meaning* changes (detected via embedding distance between sentences), producing chunks that are topically coherent — see Header 2 for the full algorithm |

---

# HEADER 2 — INGESTION PIPELINE (`backend/ingestion/`)

This is the **offline, one-time (or re-run-when-corpus-changes)** process that turns raw PDF law books into a searchable index. Entry point: `python -m ingestion.ingest`, orchestrated in `ingest.py`.

## 6. PDF text extraction (`pypdf`) — pages, malformed PDFs, blank-page filtering

### The concept
A PDF is a binary/visual format, not plain text — extracting text requires a parser that understands the PDF's internal structure (fonts, text-positioning operators, etc.) and reconstructs a readable string. LexAI uses **`pypdf`**, a pure-Python PDF library, via `PdfReader`.

### Implementation (`ingest.py::load_pdf_pages`)
```python
reader = PdfReader(str(pdf_path))
for page_number, page in enumerate(reader.pages, start=1):
    text = page.extract_text() or ""
    if len(text.strip()) < MIN_PAGE_CHARS:
        continue
    pages.append(Document(page_content=text, metadata={"source_file": pdf_path.name, "page": page_number}))
```

Three defensive details worth noting:
1. **`try/except PdfReadError`** around the whole file open — a single corrupted PDF shouldn't crash the entire ingestion run across the whole corpus.
2. **Per-page `try/except`** around `extract_text()` — pypdf can throw on individual malformed pages even within an otherwise-healthy PDF; one bad page shouldn't lose the rest of a 200-page statute.
3. **`MIN_PAGE_CHARS` blank-page filtering** — scanned image pages with no OCR text layer, or blank separator pages, extract to near-empty strings. Feeding these downstream would create meaningless embeddings/chunks that pollute the index with noise, so they're dropped early.

Each page becomes its own `Document` object (a LangChain type: `page_content` + `metadata` dict), tagged with its **1-indexed page number** — this is what lets the final answer cite `"IPC 1860.pdf, p.45"` all the way at the end of the pipeline.

### Why this is necessary
Everything downstream (chunking, embedding, retrieval, citation) depends on having clean, page-attributed plain text to work with. Get this step wrong (e.g., silently drop pages on error) and the whole knowledge base is missing content with no visible warning.

### Real-world analogy
This is like a paralegal scanning a stack of paper statutes into a searchable digital library — skipping truly blank pages, flagging any page that's too smudged to read, but not throwing out the entire book because one page was torn.

### Why `pypdf` and page-level `try/except` over the naive approach
| Approach | Problem |
|---|---|
| One `try/except` around the whole PDF, fail-fast | A single malformed page loses the entire document (all other clean pages included) |
| No blank-page filtering | Empty/near-empty "documents" get chunked and embedded anyway — wasted embedding-API calls and noisy search results |
| **Per-page error isolation + blank filtering (LexAI's approach)** | Maximizes usable content extracted per PDF while keeping the index clean |

---

## 7. Document metadata modeling — dataclasses, structured metadata design

### The concept
Every chunk that eventually reaches the vector store needs to carry **structured metadata** — not just raw text — so the system can later: (a) show a real citation, and (b) optionally filter by `law_type`. LexAI models this with Python's `@dataclass`:

```python
@dataclass(frozen=True)
class DocumentMetadata:
    source_file: str
    law_type: LawType
    year: int | None = None
    extra: dict = field(default_factory=dict)
```

`frozen=True` makes instances **immutable** — once a document's metadata is computed, it can't be accidentally mutated somewhere downstream, which matters because this same object is reused across every chunk of that document (`build_document_metadata()` in `metadata.py` runs once per file, not once per chunk).

`field(default_factory=dict)` is a dataclass idiom: you can't use a mutable default (`extra: dict = {}`) directly, because Python would share *one* dict instance across every `DocumentMetadata` object ever created (a classic Python gotcha). `default_factory` tells the dataclass to call `dict()` fresh for each new instance.

### Why this is necessary
Without structured metadata, a retrieved chunk is just an anonymous blob of text — you could never tell the user *where* an answer came from, which is the entire point of RAG's citation/verifiability guarantee (Header 1, topic 1). Structured metadata is also what lets `qa_service.py::_build_citations()` mechanically assemble a `Citation` object per chunk, and what would let the retriever filter by `law_type` if a future query needed to (e.g., "only search case law, not statutes").

### Real-world analogy
It's the difference between a pile of loose photocopied pages (just text) and a properly catalogued law library where every shelf is labeled with the Act name, year, and section — you can find and cite things precisely instead of just "somewhere in this pile."

### Why dataclasses over plain dicts
| Approach | Problem |
|---|---|
| Plain `dict` for metadata everywhere | No type safety/autocomplete, typos in keys fail silently at runtime, no immutability guarantee |
| **`@dataclass(frozen=True)`** | Type-checked fields, IDE autocomplete, immutability prevents accidental shared-state bugs, still converts cleanly to a plain dict when needed (`tag_chunk_metadata` returns a dict for storage, since the final per-*chunk* metadata needs extra per-chunk fields dataclasses aren't used for) |

---

## 8. Regex for structured extraction (section/article/year parsing)

### The concept
Legal text has predictable surface patterns even though the surrounding language is unstructured — "Section 302", "Sec. 34", "Article 21", "Art. 19(1)(a)" all follow recognizable shapes. **Regular expressions (regex)** are the standard tool for pulling structured tokens like this out of free text without needing a full NLP parser.

### Implementation details (`metadata.py`)
**Section/Article extraction:**
```python
_SECTION_PATTERN = re.compile(
    r"\b(?:Section|Sec\.?|Article|Art\.?)\s+\d+[A-Za-z]*(?:\(\d+\))?(?:\([a-zA-Z]\))?",
    re.IGNORECASE,
)
```
This deliberately matches broadly — the code comment explains the philosophy: *"false positives are harmless extra metadata, false negatives just mean no section-level filter is possible for that chunk, which degrades gracefully."* This is an important design principle: when a slightly wrong extra tag costs nothing but a missed tag costs a feature, bias the regex toward over-matching.

**Year extraction — a subtle regex bug avoided:**
```python
_YEAR_PATTERN = re.compile(r"(?<!\d)(1[5-9]\d{2}|20\d{2})(?!\d)")
```
The code explicitly explains *why* `\b` (word boundary) was rejected in favor of lookaround assertions: `\b` only detects a boundary between a word character and a *non*-word character. But an underscore (`_`) counts as a word character in regex — so in a filename like `criminal_procedure,_1973`, `\b1973\b` would **fail to match**, because there's no boundary between `_` and `1` (both are "word" characters). Using `(?<!\d)...(?!\d)` ("not preceded/followed by another digit") sidesteps this entirely by checking the *actual* condition needed — "this is a clean 4-digit number, not part of a longer digit run" — rather than relying on `\b`'s looser, sometimes-wrong definition of a boundary.

### Why this is necessary
Section numbers extracted at ingestion time become searchable/filterable metadata (`sections: list[str]` in the `Citation` schema) and are shown directly in the frontend (`CitationsPanel.jsx` renders them as badges) — letting a user see at a glance which exact provisions a chunk discusses, without reading the whole chunk.

### Real-world analogy
This is like a law clerk skimming a document with a highlighter, specifically trained to flag any "Section ___" or "Article ___" reference for a quick index — not reading for full comprehension, just pattern-spotting known shapes.

### Why regex over an NLP/ML approach here
| Approach | Trade-off |
|---|---|
| Named Entity Recognition (NER) model | Overkill for a fixed, well-known surface pattern; adds a model dependency and inference cost for something regex solves in microseconds |
| **Regex with deliberately permissive matching** | Fast, deterministic, zero dependencies, and the "graceful degradation on miss" design means it doesn't need to be perfect |

---

## 9. Semantic chunking algorithm (`chunker.py`) — the core ingestion innovation

### The concept, step by step
This is LexAI's most involved custom algorithm, and it directly implements the theory from topic 5 above ("split where meaning changes, not at an arbitrary character count"). The docstring lays out the algorithm in seven steps — here's what each one does and *why*:

**Step 1 — Split into sentences.**
```python
_SENTENCE_SPLIT_REGEX = re.compile(r"(?<=[.?!])\s+(?=[A-Z(\"'\u201c])")
```
Splits after a sentence-ending punctuation mark, but only where it's followed by whitespace and then a capital letter/quote — avoiding false splits on abbreviations like "Art." mid-sentence (a naive `text.split(".")` would incorrectly break "Art. 21" into two fragments).

**Step 2 — Combine each sentence with its neighbours (buffer window).**
```python
def _combine_with_buffer(self, sentences):
    b = self.config.buffer_size   # default: 2
    for i in range(n):
        lo, hi = max(0, i - b), min(n, i + b + 1)
        windows.append("".join(sentences[lo:hi]))
```
A single short sentence in isolation ("This shall not apply.") is nearly meaningless to an embedding model on its own. By embedding each sentence *together with its 2 neighbours on each side*, the embedding captures local context — this is a windowing technique.

**Step 3 — Embed every window in one batch call.** (Efficiency: one API/model call for all windows in a document, not N separate calls.)

**Step 4 — Compute cosine distance between consecutive window embeddings.**
```python
sims = np.sum(unit[:-1] * unit[1:], axis=1)   # cosine similarity of unit-normalized vectors
return (1.0 - sims).tolist()                   # distance = 1 - similarity
```
If sentence *i* and sentence *i+1* are about very different topics, their windows' embeddings will point in noticeably different directions — a large cosine *distance*. This is the numeric signal for "the topic just shifted here."

**Step 5 — Threshold with a percentile, not a fixed number.**
```python
threshold = float(np.percentile(distances, self.config.breakpoint_percentile))  # default: 90th percentile
breakpoints = {i for i, d in enumerate(distances) if d > threshold}
```
Using a **percentile** (rather than a fixed distance value like "0.3") makes the algorithm self-calibrating per document — legal text density and topic-shift frequency vary a lot between, say, a single dense IPC section vs. a narrative case judgment. A percentile-based threshold means "split at the most significant ~10% of topic shifts in *this specific document*," which adapts automatically instead of requiring hand-tuned thresholds per document type.

**Step 6 — Merge undersized chunks.** A chunk under `min_chunk_chars` (400) gets absorbed into its neighbour — prevents fragments like a single leftover sentence becoming its own useless chunk.

**Step 7 — Force-split oversized chunks.** A chunk over `max_chunk_chars` (1800) — legal statutes can have very long unbroken clauses with no natural sentence-level breakpoint the algorithm caught — gets forced through a plain `RecursiveCharacterTextSplitter` as a safety net, so no chunk can ever blow the LLM's context budget regardless of how the semantic pass behaved.

### Graceful degradation
If the embedding call fails entirely (rate limit, API down), the whole method doesn't crash — it falls back to the plain recursive splitter:
```python
except Exception as exc:
    logger.warning("semantic_chunking_fallback_used", ...)
    return self._apply_size_safety_net(self._safety_net.split_text(text))
```
A slightly worse (but working) chunking strategy beats a completely failed ingestion run.

### Why this is necessary
Chunk quality directly determines retrieval quality — if a chunk boundary cuts a legal clause in half, the retriever will either miss it (query doesn't match the truncated half it needed) or return an incomplete answer to the LLM. Semantic chunking maximizes the odds that each chunk is a complete, coherent legal idea.

### Real-world analogy
Imagine editing a documentary from raw interview footage. A lazy editor cuts a new clip every exactly 30 seconds, sometimes mid-sentence. A good editor watches for natural pauses — where the speaker shifts to a new point — and cuts there instead. The percentile-based breakpoint detection is the algorithmic version of "watching for the natural pause."

### Why this over `langchain_experimental.SemanticChunker`
The docstring is explicit about this: `langchain_experimental` is officially unmaintained (linked GitHub issue confirms it). Taking a hard dependency on a sunset package for a *core pipeline component* is a fragility risk that shouldn't survive review — so the project owns a compact, well-tested reimplementation of the same underlying idea instead, using only actively-maintained building blocks (`numpy`, `langchain_core.embeddings.Embeddings` interface, `langchain_text_splitters`).

| Approach | Trade-off |
|---|---|
| Fixed-size character splitting | Simple, fast, but ignores meaning entirely — can sever a clause mid-thought |
| `langchain_experimental.SemanticChunker` | Same underlying idea, but an unmaintained dependency — a liability for a core pipeline component |
| **Custom semantic chunker (LexAI)** | Same algorithmic benefit, fully owned/testable code, no dependency risk, tuned defaults for legal text (`min/max_chunk_chars`) |

---

## 10. Chunk-size safety nets — `RecursiveCharacterTextSplitter`, overlap strategy

### The concept
`RecursiveCharacterTextSplitter` (from `langchain_text_splitters`, an actively maintained package) is used in **two** places in the pipeline as a fallback/backstop, not as the primary chunking strategy:
1. Inside `SemanticChunker._apply_size_safety_net()` — force-splits any chunk that survived semantic chunking but is still too large.
2. As the entire chunking strategy during `--dry-run` mode (see topic 12) — since dry-run explicitly avoids calling the embedding API, semantic chunking (which *requires* embeddings to detect breakpoints) isn't available, so ingestion swaps to this simpler splitter.

It works "recursively" through a list of separators, from most to least meaningful:
```python
separators=["\n\n", "\n", ". ", " ", ""]
```
It tries to split on paragraph breaks first; if a resulting piece is still too big, it tries single line breaks; then sentence-ending periods; then spaces; and only as an absolute last resort, mid-word. This ordering means it *tries* to respect natural text boundaries even without doing any semantic analysis — hence "recursive."

### `chunk_overlap` — why chunks share a bit of text with their neighbours
```python
self._safety_net = RecursiveCharacterTextSplitter(
    chunk_size=self.config.max_chunk_chars,
    chunk_overlap=self.config.hard_split_overlap,   # 220 chars (chunker) / passed from settings.chunk_hard_overlap
    ...
)
```
When a chunk is force-split, without overlap, information right at the cut boundary can lose its surrounding context — a sentence like "...the accused shall be liable" split right after "liable" tells you nothing about liable *for what*. A small overlap (~220 characters) means the tail of one chunk repeats as the head of the next, so a boundary cut doesn't fully sever the meaning on either side.

### Why this is necessary
This is the final guarantee that **no chunk can ever exceed the size budget**, regardless of what happens upstream — protecting the LLM's context window from being blown by one abnormally long, unbroken legal clause (which does happen — dense statutory language with almost no punctuation for hundreds of characters).

### Real-world analogy
Overlap is like how a good multi-page photocopy of a document repeats the last line of the previous page at the top of the next page — so if you're reading page 2 first, you're not missing the sentence that started on page 1.

### Why layered safety nets over "just trust the semantic chunker"
| Approach | Risk |
|---|---|
| Trust semantic chunking alone, no size cap | An unusual document with almost no sentence-level distance variation could produce one giant chunk, silently blowing the LLM context budget at query time |
| **Semantic chunking + hard size safety net + overlap** | Best-effort semantic quality, with a hard mathematical guarantee on maximum chunk size no matter what |

---

## 11. Batch processing with retry/backoff — exponential backoff, quota/rate-limit handling

### The concept
Calling an embedding API (or any external API) for hundreds/thousands of chunks will inevitably hit **transient failures** — network blips, server 5xx errors, or (very commonly on free-tier APIs) **429 rate-limit / quota-exceeded** errors. A production-grade pipeline shouldn't crash the entire multi-hour ingestion run over one bad request — it should **retry intelligently**.

### Batching (`ingest.py::batched`)
```python
def batched(items: list, batch_size: int):
    for i in range(0, len(items), batch_size):
        yield items[i : i + batch_size]
```
Rather than one API call per chunk (slow, expensive in request overhead) or one call for the *entire* corpus (a single call might exceed API payload limits, and a single failure would lose all progress), chunks are embedded in batches (default 50 at a time, `embedding_batch_size`).

### Two flavors of retry logic in this codebase, with a key distinction
`ingest.py::build_retrying_embed_fn` distinguishes **retryable transient errors** from **quota errors**, because they need *different wait strategies*:
```python
def _is_retryable_error(exc):
    return isinstance(exc, (APIError, TimeoutError, ConnectionError)) or "resource_exhausted" in message or "429" in message

def _is_quota_error(exc):
    return "resource_exhausted" in message or "429" in message or "quota" in message
```
- A generic transient error (timeout, connection drop) can often be retried almost immediately.
- A **quota** error (you've used up your per-minute/per-day free-tier allotment) will fail again instantly if retried right away — you must wait for the quota window to reset. The code even tries to parse the API's own suggested wait time out of the error message:
```python
def _extract_retry_delay_seconds(exc, settings):
    match = re.search(r"retry in\s+([0-9.]+)s", message, re.IGNORECASE)
    if match:
        return max(settings.embedding_quota_wait_seconds, math.ceil(float(match.group(1))))
    return settings.embedding_quota_wait_seconds if _is_quota_error(exc) else int(settings.embedding_retry_min_seconds)
```
This is confirmed directly by a unit test (`test_ingestion_resume.py::test_build_retrying_embed_fn_waits_minute_after_quota`), which simulates a `"429 RESOURCE_EXHAUSTED. Please retry in 12s."` error and asserts the code waits the full **60-second** quota window (`embedding_quota_wait_seconds`), not just the API-suggested 12 seconds — a deliberately conservative floor, since free-tier quota windows are commonly per-minute.

### Exponential backoff (in the *semantic chunker's* embedding retries — `chunker.py`)
```python
wait_seconds = min(
    self.config.embedding_retry_max_seconds,
    self.config.embedding_retry_min_seconds * (2 ** (attempt - 1)),
)
```
Each retry attempt doubles the wait time (1s → 2s → 4s → 5s-capped), up to a max. This is the classic **exponential backoff** pattern: back off aggressively so repeated retries don't hammer an already-struggling server, but cap it so you're not waiting absurdly long for a merely-transient hiccup.

### Quota exhaustion doesn't abort the whole run
```python
except Exception as exc:
    if _is_quota_error(exc):
        logger.warning("embedding_quota_exhausted_skipping_faiss_build", ...)
        vectors = []
    else:
        raise
```
If embeddings ultimately can't be produced after all retries, the ingestion run still completes and saves the **BM25 index** (which needs no embeddings at all) plus a stats file — rather than losing all progress on a multi-hour run because the FAISS half failed. Partial functionality beats total failure.

### Why this is necessary
Ingesting a real legal corpus means hundreds of embedding API calls. On a free-tier API key, hitting rate limits is not an edge case — it's the *expected* common case. Without this retry/backoff machinery, ingestion would fail unpredictably and require manually restarting from scratch every time.

### Real-world analogy
This is like calling a busy customer service line. A naive caller hangs up and immediately redials the instant they're disconnected — annoying and often getting the same "still busy" result. A smart caller waits progressively longer between attempts (backoff), and if told "call back after 12pm" (quota reset), respects that specific wait time instead of guessing.

### Why retry/backoff over "just fail and let the user re-run manually"
| Approach | Cost |
|---|---|
| No retry — abort on first API error | A single transient blip near the end of a multi-hour ingestion run wastes all prior progress |
| Immediate retry, no backoff | Hammers a rate-limited API immediately again, likely fails again, wastes time and quota |
| **Backoff + quota-aware wait + graceful partial degradation** | Maximizes the odds of a completed run despite realistic API flakiness, without wasting already-completed work |

---

## 12. Idempotent pipelines — checkpointing, resumability, backup-before-overwrite

### The concept
An **idempotent** operation produces the same safe outcome whether you run it once or accidentally run it multiple times — critical for any pipeline that might be interrupted (crash, manual `Ctrl+C`, machine restart) partway through a long-running job.

LexAI's ingestion pipeline implements this with three complementary mechanisms:

**1. Checkpointed resume (`select_pdf_files_to_process`)**
```python
def select_pdf_files_to_process(data_dir, checkpoint_path=None):
    pdf_files = sorted(...)
    if checkpoint_path is None or not checkpoint_path.exists():
        return pdf_files
    payload = json.load(...)
    completed = set(payload.get("completed_files", []))
    return [p for p in pdf_files if p.name not in completed]
```
If a checkpoint file records which PDFs were already fully processed, a re-run skips them and only processes what's left — so an interrupted 70-document ingestion run doesn't have to start over from document #1. (Confirmed directly by `test_select_pdf_files_to_process_skips_completed` in the test suite.)

**2. Backup-before-overwrite (`backup_existing_database`)**
```python
def backup_existing_database(database_dir):
    if not database_dir.exists() or not any(database_dir.iterdir()):
        return None
    timestamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
    backup_dir = database_dir.parent / f"{database_dir.name}_backup_{timestamp}"
    shutil.move(str(database_dir), str(backup_dir))
    return backup_dir
```
Before writing a fresh index, the *existing working* `Database/` directory is renamed (not deleted) to a timestamped backup folder. If the new ingestion run fails partway or produces a worse index, the previous working index is still sitting right there, recoverable — "a bad run never silently destroys a working index," as the module docstring states. `--no-backup` opts out for cases where disk space or speed matters more (e.g. CI).

**3. Fail-before-touching-storage guard**
```python
if not all_chunks:
    raise RuntimeError("Ingestion produced zero chunks across all source documents — aborting before touching Database/.")
```
This ordering matters: the check happens *before* any backup/overwrite logic runs. If something went catastrophically wrong upstream (e.g., every PDF failed to parse), the pipeline refuses to touch the existing database at all — rather than backing up a good index and then writing an empty, broken one over it.

### `--dry-run` mode as a safe validation path
```python
parser.add_argument("--dry-run", action="store_true", ...)
```
Dry-run validates the entire pipeline — PDF loading, metadata tagging, chunk-count stats — **without spending any embedding API quota** and without writing a FAISS index, by swapping in the plain `RecursiveCharacterTextSplitter` (topic 10) instead of the embedding-dependent semantic chunker. This is explicitly designed for CI / "no API key available" smoke tests: you can verify the pipeline *runs end-to-end without errors* on every commit, without needing real API credentials in CI.

### Why this is necessary
Ingestion over a real corpus can take a long time and cost real API quota. Any of these can go wrong mid-run: a crash, a killed process, an exhausted API quota, a bad PDF causing an unexpected exception. Without idempotency safeguards, *any* interruption means either losing all progress (no checkpoint) or — much worse — silently corrupting a previously-working search index that the live API depends on (no backup).

### Real-world analogy
This is standard "safe deployment" practice borrowed from software engineering generally: never overwrite the only copy of production data in place. Keep the old version until the new version is verified good (like blue-green deployments), and keep a record of exactly how far you got (checkpoint) so an interrupted multi-hour job can resume instead of restarting from zero.

### Why this layered approach over "just re-run the whole thing from scratch every time"
| Approach | Risk/Cost |
|---|---|
| No backup, overwrite in place | One bad ingestion run (bug, bad data) destroys the only working index with no way back |
| No checkpoint, always full re-run | Wastes hours of API calls/time re-processing already-successfully-embedded documents after any interruption |
| **Checkpoint + backup + fail-fast guard + dry-run** | Interruptions cost minutes, not hours; a bad run is always recoverable; CI can validate pipeline correctness with zero API cost |

---

*End of Part 1 (Headers 1–2).*

---

# HEADER 3 — EMBEDDINGS & VECTOR STORAGE (`backend/app/services/vector_store.py`, `providers.py`)

## 13. Embedding providers — local (Ollama) vs. hosted (Google Gemini) APIs, task types

### The concept
An "embedding provider" is whatever service actually runs the neural network that turns text into vectors. LexAI is designed to support two kinds:

- **Local (Ollama)** — a model (e.g. `nomic-embed-text`) runs entirely on your own machine via the Ollama runtime, no internet/API key required, no per-call cost, but you need enough local compute (RAM/CPU/GPU) and the model quality is generally smaller/weaker than large hosted models.
- **Hosted (Google Gemini embeddings)** — a much larger, higher-quality model runs on Google's servers, called over HTTPS with an API key, but costs money past free-tier limits and is subject to rate limits/quotas (this is exactly what Header 2, topic 11's retry/backoff logic exists to handle).

`settings.py` exposes this as a simple config switch:
```python
llm_provider: str = Field(default="ollama", description="Provider for chat generation: google or ollama")
embedding_provider: str = Field(default="ollama", description="Provider for embeddings: google or ollama")
ollama_base_url: str = Field(default="http://localhost:11434", ...)
```

**Important thing to notice while studying this repo**: `providers.py`'s `build_llm_client()` and `build_embeddings_client()` functions currently *only* construct `LocalOllamaChat` / `LocalOllamaEmbeddings` regardless of what `llm_provider`/`embedding_provider` is set to — the Google branch referenced in settings/docstrings/README isn't actually wired up in the current code (confirmed by reading `providers.py` end-to-end). This is a good example of "config exists, but the implementation behind one branch of it isn't there yet" — worth knowing before you assume the Google path works, in case you're asked to implement it.

### Task types — why the *same* text embeds differently depending on its role
```python
class LocalOllamaEmbeddings:
    def __init__(self, settings, task_type: str = "semantic_similarity"):
```
Called with three different `task_type` values across the codebase:
- `"retrieval_document"` — used at **ingestion time**, when embedding chunks that will be *stored* in the index.
- `"retrieval_query"` — used at **query time**, when embedding the user's *question* (see `retriever.py`: `self._query_embeddings = build_embeddings_client(settings, task_type="retrieval_query")`).
- `"semantic_similarity"` — used by the semantic chunker (Header 2, topic 9), comparing sentence windows *to each other* rather than query-to-document.

Some embedding model families (notably Google's Gemini embeddings, referenced in the docstrings) are **asymmetric**: they use a different internal representation depending on whether the text is "a thing to be found" (document) or "a thing doing the searching" (query) — even though both end up as vectors in the same space. The `retriever.py` module docstring explicitly warns about this:

> *"Using the wrong task_type silently degrades retrieval quality rather than erroring, so this is easy to get wrong without noticing."*

This is a subtle but important bug class: the code will *run fine* with the wrong task_type, just retrieve worse results — no exception is raised, so it's easy to ship a quietly-degraded system.

### Why this is necessary
Different deployment contexts have different needs: a developer without an API key or with privacy/cost concerns needs the local path to work end-to-end; a production deployment wanting the highest retrieval quality might prefer a larger hosted model. Task-type awareness exists because embedding models genuinely encode queries and documents differently under the hood for models that support it — ignoring this quietly loses accuracy.

### Real-world analogy
Think of a translator who translates slightly differently depending on whether they're translating "a question someone is asking" vs. "an encyclopedia entry someone might look up" — both end up in the same target language, but a good translator phrases a question-translation to be *search-friendly*, and an entry-translation to be *complete and standalone*. Mixing the two up doesn't cause an error, it just makes the translation subtly less useful for its purpose.

### Why a provider abstraction over hardcoding one embedding backend
| Approach | Trade-off |
|---|---|
| Hardcode one provider (e.g., always Google) | No local/offline option, forces every developer to have an API key just to run the project |
| **`build_embeddings_client()` factory function switched by settings** | Lets deployment context (local dev vs. hosted production) pick the right trade-off between cost, privacy, and quality without touching calling code |

---

## 14. FAISS fundamentals — `IndexFlatIP`, vector normalization, inner product ≈ cosine similarity

### The concept
**FAISS** (Facebook AI Similarity Search) is a library, written by Meta, purpose-built for **fast nearest-neighbor search over large collections of vectors** — exactly the "find the most similar chunks to this query vector" operation retrieval needs.

`vector_store.py` uses the simplest FAISS index type:
```python
self._index = faiss.IndexFlatIP(dimension)
```
- **`Flat`** means it's an exact, brute-force index — every query vector is compared against *every* stored vector, no approximation. This guarantees perfectly correct nearest-neighbor results (no recall loss), at the cost of search time scaling linearly with corpus size. For a legal corpus of a few thousand to tens of thousands of chunks, this is fast enough that the accuracy/simplicity trade-off clearly favors "flat."
- **`IP`** = Inner Product. As covered in topic 2, once vectors are L2-normalized to unit length, inner product search *is* cosine-similarity search — so `IndexFlatIP` on normalized vectors gives you exact cosine-similarity nearest-neighbor search.

### Normalization enforced at every entry/query point
```python
def add_documents(self, documents, embeddings):
    vectors = np.vstack([...])
    self._index.add(self._normalize(vectors))   # normalized before storage

def similarity_search_by_vector(self, query_embedding, k=4):
    query = self._normalize(np.array([...]))     # normalized before search
```
Both the stored document vectors *and* the incoming query vector are normalized — consistency here matters exactly the same way tokenizer-consistency mattered for BM25 (topic 3): if one side is normalized and the other isn't, the resulting "cosine similarity" scores would be mathematically wrong.

### The `_coerce_vector` defensive dimension handling
```python
def _coerce_vector(self, embedding, dimension=None):
    vector = np.asarray(embedding, dtype=np.float32).reshape(-1)
    target_dim = self.dimension if dimension is None else dimension
    if vector.size < target_dim:
        padded = np.zeros(target_dim, dtype=np.float32)
        padded[: vector.size] = vector
        return padded
    return vector[:target_dim]
```
This pads a too-short vector with zeros or truncates a too-long one to match the index's fixed dimension — a safety net against a provider mismatch (e.g., swapping embedding models between ingestion and query time, which would otherwise produce a hard crash deep inside FAISS's C++ layer with a cryptic error). It doesn't *fix* the underlying mismatch (mixing embedding models is still a real bug — the vectors wouldn't be comparable even at the same dimension), but it prevents an ungraceful crash.

### Persistence — `save_local` / `load_local`
```python
def save_local(self, directory):
    faiss.write_index(self._index, str(directory / self._INDEX_FILENAME))
    with open(directory / self._DOCSTORE_FILENAME, "wb") as f:
        pickle.dump({"dimension": self.dimension, "docstore": self._docstore}, f)
```
Two files are saved side by side: FAISS's own native binary format for the vector index itself (fast to load, opaque binary), and a separate **pickle** file holding the parallel list of `Document` objects (the actual text + metadata) plus the vector dimension. FAISS only stores vectors and returns *integer indices* on search — it has no concept of "documents" — so the code must maintain its own `self._docstore: list[Document]` where position `i` in the list corresponds exactly to vector `i` in the FAISS index, and re-associate `ScoredDocument(document=self._docstore[idx], score=...)` after every search.

### Why this is necessary
Comparing a query vector against every document vector one-by-one in pure Python would be extremely slow at scale. FAISS is written in optimized C++ with SIMD vectorization specifically for this operation, making even brute-force ("Flat") search over thousands of vectors take milliseconds.

### Real-world analogy
`IndexFlatIP` is like a librarian who, given a description of what you're looking for, checks *every single book* against that description and returns the best matches, guaranteed correct — versus a faster but approximate librarian (an `IVF`/`HNSW`-style FAISS index, not used here) who only checks a promising subset of shelves for speed, occasionally missing the true best match. For a corpus this size, LexAI chooses the always-correct librarian since it's still fast enough.

### Why a custom thin wrapper over `langchain_community.vectorstores.FAISS`
The module docstring is explicit and worth internalizing as a general engineering lesson:

> *"the `langchain-community` package was officially sunset (repository archived, no further maintenance)... Building a project's core retrieval index on an archived package is not a decision that should survive contact with an interviewer asking 'why did you pick that dependency.'"*

| Approach | Trade-off |
|---|---|
| `langchain_community.vectorstores.FAISS` | Convenient pre-built wrapper, but built on an archived, unmaintained package — a liability for a core, load-bearing component |
| **Thin custom wrapper directly on `faiss` + `langchain_core.documents.Document`** | A few dozen extra lines of code, but only depends on actively-maintained foundations, with full control over the exact persistence format |

---

## 15. Building a vector store from scratch — serialization design (`vector_store.py`)

### The concept
"Building a vector store from scratch" here doesn't mean reimplementing nearest-neighbor search math (FAISS already does that) — it means designing the **wrapper class** around FAISS that turns "a raw similarity-search primitive" into "a store that understands documents, persists to disk, and exposes a clean API the rest of the app can depend on." This is a good case study in API design around a lower-level library.

### The class's public surface (deliberately small)
```python
class FaissVectorStore:
    def __init__(self, dimension): ...
    def __len__(self): ...
    def add_documents(self, documents, embeddings): ...
    def similarity_search_by_vector(self, query_embedding, k=4): ...
    def save_local(self, directory): ...
    @classmethod
    def load_local(cls, directory): ...
    @property
    def documents(self): ...
```
Every other detail (normalization, index type, pickle format) is an internal implementation detail hidden behind this surface — `retriever.py` never touches FAISS directly, it only calls `self.vector_store.similarity_search_by_vector(embedding, k=...)`. This is the **encapsulation** principle: callers depend on a stable, meaningful interface, not on internal storage mechanics that might later change (e.g., swapping `IndexFlatIP` for a different FAISS index type later would require zero changes to `retriever.py`).

### Validation at the boundary
```python
def add_documents(self, documents, embeddings):
    if len(documents) != len(embeddings):
        raise ValueError(f"documents ({len(documents)}) and embeddings ({len(embeddings)}) counts must match.")
```
Fails loudly and immediately with a clear message if the caller passes mismatched lists — rather than letting a silent off-by-one corrupt the `docstore ↔ index` alignment (which would cause every future search to return the *wrong document* for a given vector match — a very hard bug to notice, since the code would keep running, just silently returning wrong citations).

### Fail-loud on missing index files
```python
@classmethod
def load_local(cls, directory):
    if not index_path.exists() or not docstore_path.exists():
        raise FileNotFoundError(f"No FAISS store found at {directory}. ... Run the ingestion pipeline first.")
```
A clear, actionable error message rather than a cryptic FAISS internal exception if someone tries to start the API server before ever running ingestion.

### Why this is necessary
A raw FAISS index alone can't answer "which document does result #47 correspond to, and what page/law_type does it have?" — that mapping has to be maintained by *someone*. Wrapping it in a purpose-built class with an explicit `docstore` list keeps that responsibility in one clearly-owned place instead of leaking FAISS index/document alignment logic into every caller.

### Real-world analogy
This is like a library's physical card catalogue: the catalogue drawer (FAISS index) only stores call numbers in searchable order, but the librarian's separate master list (docstore) is what tells you the call number `#47` actually corresponds to "IPC 1860, page 45, Section 302." You need *both*, kept perfectly in sync, or the catalogue is useless.

### Why this design over "just use raw FAISS calls everywhere retrieval is needed"
| Approach | Problem |
|---|---|
| Call `faiss.IndexFlatIP` / `.add()` / `.search()` directly wherever needed | FAISS-specific details (normalization, dimension handling, docstore alignment) leak into every caller, error-prone and hard to change later |
| **Encapsulated `FaissVectorStore` class** | One place owns correctness (normalization, dimension safety, docstore sync); callers get a simple, document-aware API |

---

# HEADER 4 — SPARSE RETRIEVAL (`backend/app/services/bm25_store.py`)

## 16. The BM25 algorithm — term frequency, inverse document frequency, why it beats plain keyword search

### The concept
BM25 ("Best Matching 25") is a **ranking function** for scoring how relevant a document is to a query, based purely on the words they share — no neural network, no "understanding" of meaning, just careful statistics. It's the direct mathematical descendant of the older TF-IDF idea, refined to handle real-world text better. Two core statistical ideas combine to produce a BM25 score:

**1. Term Frequency (TF)** — how many times a query word appears in a document. Intuition: a document mentioning "murder" five times is probably more about murder than one mentioning it once. But BM25 uses a *saturating* function of term frequency (not linear) — going from 1 mention to 2 mentions matters a lot, but going from 20 mentions to 21 barely moves the needle. This prevents a document that just repeats a word many times from dominating rankings unfairly.

**2. Inverse Document Frequency (IDF)** — how *rare* a word is across the whole corpus:
```
IDF(word) = log((N - n + 0.5) / (n + 0.5))
```
where `N` is the total number of documents and `n` is how many documents contain that word. A word that appears in almost every document (like "shall", "section", "the") is a weak, common signal and gets a *low* (or even negative) IDF weight. A word that appears in only a handful of documents (like "arbitration" or a specific case name) is a strong, distinguishing signal and gets a *high* IDF weight. This is the statistical formalization of "rare words tell you more than common words."

The final BM25 score for a document, given a query, sums TF-weighted-by-IDF across every query word present in that document — implemented via the `rank_bm25` library's `BM25Okapi` class:
```python
self._bm25 = BM25Okapi(self._corpus_tokens) if self._corpus_tokens else None
...
scores = self._bm25.get_scores(query_tokens)
```

### Why plain keyword search (e.g., `if query_word in document`) isn't good enough
A naive boolean keyword match treats every matching word equally and gives no ranking signal beyond "does it appear at all" — it can't tell you that a document mentioning your query term 8 times in a focused way is more relevant than one that mentions it once in an unrelated aside, and it can't tell you that a rare, specific term (a case citation) is a stronger match signal than a common word (that happens to also be in the query).

### Why this is necessary alongside embeddings
BM25 is *exact* — it only "sees" surface word forms it's told to. This is a weakness for synonyms/paraphrases (Header 1, topic 2 covers why embeddings solve that), but it's a *strength* for **exact term matching**: a specific IPC section number, a defined legal term of art, or a case citation name. Embeddings can be "fuzzy" in exactly the situation where legal search needs precision (Section 302 vs. Section 304 look nearly identical to an embedding model but mean completely different offences). This complementary strength/weakness pairing is the entire justification for the **hybrid retrieval** architecture covered in Header 5.

### Real-world analogy
BM25 is like `Ctrl+F` with a brain — it still only finds exact words (no understanding of synonyms), but it's smart about *how much* a match should count: repeated hits of a rare, specific term count for a lot; a single hit of an extremely common word barely registers.

### Why BM25 over plain TF-IDF (the older cousin)
| Approach | Limitation |
|---|---|
| Plain boolean keyword match | No relevance ranking at all — every match is equally "relevant" |
| TF-IDF (raw cosine similarity of frequency vectors) | Doesn't saturate term-frequency, so word repetition can be gamed/over-weighted; doesn't handle document-length normalization as well |
| **BM25** | Saturating term-frequency curve + tunable document-length normalization, empirically the strongest classical (non-neural) sparse ranking function — the standard baseline modern hybrid RAG systems still pair with dense retrieval |

---

## 17. Stopword filtering and why it matters for small/domain-specific corpora

### The concept
**Stopwords** are extremely common words (`the`, `and`, `of`, `shall`, `is`) that carry little distinguishing meaning on their own. `bm25_store.py` defines an explicit stopword list and filters them out during tokenization:
```python
def tokenize(text: str) -> list[str]:
    return [t for t in _TOKEN_PATTERN.findall(text.lower()) if t not in _STOPWORDS]
```

### Why this matters *especially* for this corpus (not just "always good practice")
BM25's IDF formula theoretically already down-weights very common words automatically (since `n` — the document-count containing the word — approaches `N` for stopwords). But the code comment explains a specific edge case where this safety net breaks down:

> *"BM25's classic IDF... goes negative and gets epsilon-floored for terms in >50% of documents... lets common connector words ('the', 'under', 'shall', 'of') dominate rankings on any corpus small enough, or repetitive enough, for those words to appear in most documents."*

Legal statute text is unusually repetitive in its structural vocabulary — words like "shall," "under," "section," and "provided that" appear in nearly *every* chunk of a penal code, not just "most" documents in general English. On a large, generic corpus (millions of diverse web pages), automatic IDF down-weighting works fine because these words really are rare relative to content words on average. But on a **small, domain-specific corpus** (a few dozen legal PDFs where structural legal vocabulary repeats constantly), the IDF formula's own floor-clamping behavior for extremely common terms breaks down, letting stopword noise creep back into rankings.

Critically, the docstring notes this was **verified empirically, not just theorized**:
> *"removing this caused an irrelevant IT-Act chunk to outrank the actual murder-punishment chunk for the query 'what is the punishment for murder', purely because of stopword term-frequency noise."*

This is a great example of "don't just trust the theory — test the actual behavior on your actual data," and it's worth remembering as a general lesson: an algorithm's asymptotic/typical-case guarantees (IDF naturally handling common words) can quietly fail on smaller or more specialized datasets than the algorithm was originally validated against.

### Why this is necessary
Without stopword filtering, on this particular small legal corpus, BM25 rankings can be dominated by structural connector words shared across nearly every document rather than the actual distinguishing legal content words — directly degrading answer quality, since the retriever might surface an irrelevant chunk from an unrelated Act purely because it happens to share more common words with the query.

### Real-world analogy
Imagine grading essays by simply counting shared words with a model answer, without discounting "the," "a," "is." Every essay uses those words constantly — they tell you nothing about which essay is actually closer to the model answer's *content*. You have to explicitly ignore the "glue words" to see the real signal.

### Why an explicit stopword list over "just trust BM25's IDF weighting"
| Approach | Result on this corpus (empirically tested) |
|---|---|
| Rely on BM25's IDF alone, no stopword list | Structural legal words dominate rankings on this small, repetitive corpus — verified to cause wrong top results |
| **Explicit stopword removal before BM25 indexing/querying** | Removes the noise source entirely, letting genuine content words drive the ranking |

---

## 18. `rank_bm25` library usage — indexing, searching, and consistency requirements

### The concept
`rank_bm25` is a small, focused Python library implementing the BM25Okapi variant. LexAI's `Bm25Store` class wraps it the same way `FaissVectorStore` wraps FAISS (topic 15) — encapsulation around a lower-level library, exposing a clean, document-aware API.

### Building the index (`__init__`)
```python
def __init__(self, documents: list[Document]):
    self._documents = documents
    self._corpus_tokens = [tokenize(doc.page_content) for doc in documents]
    self._bm25 = BM25Okapi(self._corpus_tokens) if self._corpus_tokens else None
```
Every document's text is tokenized *once*, up front, using the shared `tokenize()` function — and the resulting list-of-token-lists (`self._corpus_tokens`) is what actually gets passed to `BM25Okapi`, which precomputes term frequencies and document-length statistics across the whole corpus at construction time. This precomputation is why building the index once and reusing it for many queries is efficient — the expensive corpus-wide statistics are computed a single time, not per-query.

The `if self._corpus_tokens else None` guard handles the empty-corpus edge case gracefully (e.g., ingestion produced zero chunks) — `search()` then simply returns `[]` rather than crashing on a `None` BM25 object.

### Searching (`search`)
```python
def search(self, query: str, k: int = 4) -> list[ScoredDocument]:
    if self._bm25 is None or not self._documents:
        return []
    query_tokens = tokenize(query)
    if not query_tokens:
        return []
    scores = self._bm25.get_scores(query_tokens)
    top_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)[:k]
    return [ScoredDocument(document=self._documents[i], score=float(scores[i])) for i in top_indices if scores[i] > 0]
```
`get_scores()` returns one score *per document in the entire corpus* (not just matches) — the code then sorts all indices by score descending and takes the top `k`. The final `if scores[i] > 0` filter drops zero-score documents (documents sharing *no* tokens with the query at all) even if fewer than `k` results remain — returning a smaller, honest result set is better than padding it out with completely irrelevant documents just to hit a fixed count.

### The consistency requirement, revisited
This is the same principle from topic 3, worth reinforcing here concretely: `self._documents[i]` and `self._corpus_tokens[i]` (used to build the index) must stay in the same order/alignment — exactly the same "parallel list alignment" discipline `FaissVectorStore` needed between its FAISS index positions and `self._docstore`. Both the dense and sparse stores independently solve the same underlying problem: a low-level search library returns *indices*, and the wrapper is responsible for faithfully mapping those indices back to real `Document` objects.

### Persistence
```python
def save_local(self, directory):
    with open(directory / self._FILENAME, "wb") as f:
        pickle.dump({"documents": self._documents}, f)

@classmethod
def load_local(cls, directory):
    ...
    return cls(documents=payload["documents"])
```
Notice this **only pickles the raw `documents` list**, not the tokenized corpus or the `BM25Okapi` object itself — `load_local` re-runs `__init__`, which re-tokenizes and rebuilds the `BM25Okapi` index from scratch on load. This is a deliberate simplicity trade-off: re-tokenizing on load costs a little startup time, but avoids depending on `BM25Okapi` objects being reliably picklable/version-stable across `rank_bm25` library upgrades — storing only plain data (`Document` objects) and *rebuilding* derived structures on load is a more robust persistence strategy than pickling the derived structures themselves.

### Why this is necessary
Every design choice here echoes lessons already introduced elsewhere: shared tokenizer consistency (topic 3), encapsulation around a lower-level library (topic 15), and a persistence strategy that favors rebuilding derived state over pickling it directly — reinforcing that these aren't one-off tricks but consistent engineering principles applied throughout the whole codebase.

### Real-world analogy
Storing only the raw documents and rebuilding the BM25 statistics on load is like keeping the original ingredients list for a recipe rather than freezing the fully-cooked dish — reheating (rebuilding) from fresh ingredients each time is slightly slower than microwaving a frozen meal, but it's far more reliable across a kitchen renovation (a library version upgrade) than hoping the frozen dish (a pickled `BM25Okapi` object) survives unchanged.

### Why "rebuild on load" over "pickle the whole BM25Okapi object"
| Approach | Risk |
|---|---|
| Pickle the constructed `BM25Okapi` object directly | Fragile across `rank_bm25` version changes; pickled objects can silently fail to unpickle, or unpickle into a subtly incompatible internal state, after a library upgrade |
| **Pickle only plain `Document` data, rebuild `BM25Okapi` on load** | Slightly slower startup, but robust — plain data formats don't break across library version changes the way pickled class instances can |

---

*End of Part 2 (Headers 3–4).*

---

# HEADER 5 — HYBRID RETRIEVAL & FUSION (`backend/app/services/retriever.py`, HyDE in `qa_service.py`)

This is where Headers 3 (dense/FAISS) and 4 (sparse/BM25) get combined into a single retrieval signal, plus a third technique — HyDE — layered on top to improve the *query side* before either search even runs.

## 19. Why dense-only or sparse-only retrieval each fail in different ways

### The concept
By this point you've seen each retrieval method's individual weakness:
- **Dense (FAISS/embeddings) alone** is great at *meaning* but blurs fine-grained lexical distinctions — it can't reliably tell "Section 302" from "Section 304" apart, because they look almost identical as text even though they refer to legally unrelated offences (murder vs. culpable homicide not amounting to murder).
- **Sparse (BM25) alone** is great at *exact* term matches but has zero understanding of meaning/paraphrase — a query for "penalty for stealing" will completely miss a chunk that only ever says "punishment for theft," because there's no literal token overlap, even though they're the same legal concept.

Neither is "better" in general — they fail on **different kinds of queries**, which is precisely why combining them (rather than picking one) is the right engineering call for a legal corpus, where users ask both conceptual questions ("what counts as cruelty under this law?") and precise citation lookups ("what does Section 498A say?").

### Implementation — running both searches per query
```python
dense_lists: list[list[Document]] = []
for embedding in dense_embeddings:
    scored = self.vector_store.similarity_search_by_vector(embedding, k=self.settings.retrieval_k_dense)
    dense_lists.append([s.document for s in scored])

sparse_results = self.bm25_store.search(query, k=self.settings.retrieval_k_sparse)
sparse_list = [s.document for s in sparse_results]
```
`HybridRetriever.retrieve()` in `retriever.py` runs *both* searches for every single query, unconditionally — there's no attempt to classify "is this a lexical query or a semantic query" up front and route accordingly. That classification problem is itself hard and error-prone; running both and *fusing the results* (topic 20) sidesteps needing to solve it at all.

### Why this is necessary
A legal research assistant needs to handle the full spectrum of question types a real user asks — from vague conceptual questions to precise citation lookups — without the user having to know or care which retrieval method would work best for their specific phrasing.

### Real-world analogy
This is like a research assistant who, given any question, always checks *both* the subject-matter index (organized by topic/meaning) *and* the alphabetical/citation index (organized by exact term) of a law library — rather than gambling on guessing which single index would happen to contain the answer for this particular question.

### Why "run both, always" over "classify the query type first"
| Approach | Trade-off |
|---|---|
| Classify query as "lexical" or "semantic" first, route to one method | Adds a fragile, error-prone extra classification step; a wrong classification means total retrieval failure for that query |
| **Always run both dense and sparse, fuse afterward (LexAI's approach)** | Slightly more compute per query (two searches instead of one), but robust — never bets everything on a single method being right |

---

## 20. Reciprocal Rank Fusion (RRF) — the formula, why it's rank-based not score-based

### The concept
Once you have two *independent* ranked lists of candidate chunks (one from FAISS, one from BM25), you need a principled way to merge them into a single ranking. The core difficulty: **FAISS similarity scores and BM25 relevance scores are not on the same numeric scale** — a cosine similarity of 0.82 and a BM25 score of 14.3 aren't directly comparable numbers, so you can't just average or sum the raw scores together and expect a meaningful result.

**Reciprocal Rank Fusion** sidesteps this entirely by throwing away the raw scores and using only each document's **rank position** (1st, 2nd, 3rd...) within each list:

```python
def reciprocal_rank_fusion(ranked_lists: list[list[Document]], k: int) -> list[FusedDocument]:
    scores: dict[tuple, float] = {}
    for list_index, ranked_list in enumerate(ranked_lists):
        is_sparse_list = list_index == len(ranked_lists) - 1
        for rank, doc in enumerate(ranked_list, start=1):
            identity = _chunk_identity(doc)
            scores[identity] = scores.get(identity, 0.0) + 1.0 / (k + rank)
            ...
    fused = [...]
    fused.sort(key=lambda f: f.rrf_score, reverse=True)
    return fused
```

The formula is `RRF(d) = Σ 1 / (k + rank(d))`, summed across every ranked list the document `d` appears in. A document ranked **#1** in a list contributes a large term (`1/(k+1)`); ranked **#50**, a tiny one (`1/(k+50)`) — the contribution decays smoothly as rank gets worse. `k` (a small constant, e.g. 60, from `settings.retrieval_rrf_k`) softens the curve so the difference between rank #1 and rank #2 isn't wildly more dramatic than the difference between rank #40 and rank #41.

**The key emergent property**: a document that appears in *both* lists — even at a middling rank in each — accumulates *two* fractional scores and can outrank a document that's #1 in only *one* list. This is exactly the desired behavior: **agreement between two independent, differently-biased signals (semantic + lexical) is stronger evidence of true relevance than excellence on just one signal.**

### The `_chunk_identity` matching problem — why the same chunk needs to be recognized across lists
```python
def _chunk_identity(doc: Document) -> tuple:
    source = doc.metadata.get("source_file")
    chunk_index = doc.metadata.get("chunk_index")
    if source is not None and chunk_index is not None:
        return (source, chunk_index)
    return ("__content_hash__", hash(doc.page_content))
```
FAISS and BM25 each return `Document` objects independently — even if both happen to retrieve "the exact same chunk," Python doesn't automatically know these are "the same" object for scoring purposes. `_chunk_identity()` gives every chunk a **stable identity key** — `(source_file, chunk_index)` from ingestion metadata — so RRF can correctly recognize "chunk 7 of the IPC PDF returned by FAISS" and "chunk 7 of the IPC PDF returned by BM25" as the *same* document to be merged, not double-counted as two separate entries in the final ranking. The content-hash fallback handles the edge case of documents that somehow bypassed the ingestion pipeline's metadata tagging.

### Why the ordering convention matters (`sparse MUST be last`)
```python
all_lists = dense_lists + [sparse_list]  # sparse MUST be last — reciprocal_rank_fusion relies on this order
```
`reciprocal_rank_fusion` identifies which list is "the sparse one" purely by *position* (`list_index == len(ranked_lists) - 1`), used to populate the diagnostic `matched_dense`/`matched_sparse` flags on each `FusedDocument`. This is a small but real fragility — reordering the list-building code without noticing the comment could silently corrupt those diagnostic flags (the actual fusion *ranking* would still work correctly regardless of order, since RRF itself is order-agnostic in how it sums scores — only the dense/sparse *labeling* depends on position).

### Why this is necessary
Fusion is the mechanism that actually delivers on the promise of Header 5's opening premise ("hybrid beats either alone") — without a principled way to combine two differently-scaled ranked lists, you'd be stuck arbitrarily picking one method's results over the other's, defeating the entire purpose of running both.

### Real-world analogy
Imagine two expert judges independently ranking contestants in a competition — one judge scores on technical precision, the other on artistic impression, using totally different scoring scales that can't be directly compared. RRF is like converting both judges' opinions to "how many people did I rank above/below" (i.e., pure rank position) before combining — a contestant both judges place in their top 3 clearly deserves to win overall, even if their raw numeric scores from each judge aren't comparable.

### Why RRF over alternatives
| Approach | Problem |
|---|---|
| Normalize and sum raw scores (e.g. min-max scale both to 0–1, add) | Score distributions differ wildly between methods (BM25 scores are unbounded and corpus-dependent; cosine similarity is bounded [-1,1]) — normalization choices become an arbitrary, hard-to-justify tuning knob |
| Pick whichever method's top result "looks better" (heuristic/LLM judge per query) | Expensive per-query, adds latency and unpredictability |
| **Reciprocal Rank Fusion (rank-based, not score-based)** | Scale-independent by construction, no normalization needed, simple, well-established in information retrieval literature, rewards cross-method agreement |

---

## 21. HyDE (Hypothetical Document Embeddings) — generating a hypothetical answer to improve retrieval

### The concept
This is the cleverest technique in the pipeline, and it's worth understanding the core insight carefully: **a user's question and the passage that answers it are often phrased very differently**, even in the same language about the same topic. A question is phrased as a *question* ("What is the punishment for theft?"); the actual statutory text answering it is phrased as a *declarative legal statement* ("Whoever commits theft shall be punished with imprisonment..."). Even a good embedding model can only partially bridge this "question-vs-answer" phrasing gap — questions and their answers don't always embed as close neighbors, purely because of *how differently they're structured as text*, independent of whether they're topically related.

**HyDE's trick**: instead of embedding the raw question and hoping it lands near the answer passages in vector space, first ask an LLM to *hallucinate* a plausible, well-formed hypothetical answer to the question — then embed *that hypothetical answer* instead (or alongside the original question). Because the hypothetical answer is written in the same declarative, statutory register as the real source documents, its embedding tends to land much closer to the *actual* relevant chunks than the raw question's embedding would.

### Implementation (`qa_service.py`)
```python
HYDE_PROMPT = """You are a legal research assistant. Write a short hypothetical passage \
(3-5 sentences) that would be a strong, ideal answer to the legal question below, as it might \
appear in an Indian statute, judgment, or legal commentary. Do not hedge, do not mention that \
this is hypothetical - just write the passage itself, in a neutral legal-drafting register.

Question: {question}

Hypothetical passage:"""

async def _generate_hyde_document(self, question: str) -> str | None:
    try:
        response = await self._hyde_llm.ainvoke(HYDE_PROMPT.format(question=question))
        text = response.content if isinstance(response.content, str) else str(response.content)
        return text.strip() or None
    except Exception as exc:
        logger.warning("hyde_generation_failed", error=str(exc))
        return None
```
Note the prompt explicitly instructs the model to write in a **"neutral legal-drafting register"** — mimicking the actual style of the source corpus is what makes the resulting embedding land near real statutory text. The prompt also says **"do not hedge, do not mention that this is hypothetical"** — a hedged, uncertain-sounding passage ("this might possibly relate to...") would embed differently (less confidently declarative) than a real statute, undermining the whole technique.

### HyDE augments the query, it doesn't replace it
```python
fused, trace = self.retriever.retrieve(
    query=question,
    extra_query_texts=[hyde_text] if hyde_text else None,
)
...
dense_query_texts = [query] + [t for t in (extra_query_texts or []) if t and t.strip()]
dense_embeddings = self._query_embeddings.embed_documents(dense_query_texts)
```
Both the **original question's embedding** *and* the **HyDE hypothetical answer's embedding** are searched separately against FAISS, producing two separate dense ranked lists — both of which then feed into the same RRF fusion (topic 20) as if they were independent retrieval signals, exactly like the dense/sparse fusion. This is a deliberate hedge: if the LLM's hypothetical answer happens to be subtly off-topic or wrong, the *original* question's embedding is still in the mix contributing its own signal — HyDE augments retrieval, it doesn't gamble everything on the hallucinated passage being accurate.

### Graceful degradation — HyDE failure never blocks the whole request
```python
except Exception as exc:
    logger.warning("hyde_generation_failed", error=str(exc))
    return None
```
If the HyDE LLM call fails (rate limit, timeout), `_generate_hyde_document` returns `None` rather than raising — `stream_answer()` then simply proceeds with retrieval using *only* the original question, no HyDE augmentation. This mirrors the same "best-effort enhancement, not a hard dependency" philosophy seen in the ingestion pipeline's quota handling (Header 2, topic 11) and the semantic chunker's fallback (Header 2, topic 9) — a recurring, deliberate design pattern throughout this codebase: **every enhancement layer is additive and optional; its failure degrades quality, never availability.**

### Why this is necessary
Without HyDE, retrieval quality is capped by how closely a user's *natural question phrasing* happens to embed near the *source document's* declarative phrasing — a gap that's often larger than people intuitively expect, especially for terse or colloquially-phrased questions ("can I sue my landlord for this?") against formally-worded statutory text.

### Real-world analogy
Imagine trying to find a matching quote in a book of famous speeches, but all you have is a *casual paraphrase* of what you're looking for. You'd likely have much better luck if you first mentally rephrased your paraphrase into the *formal, rhetorical style* the speeches themselves use, then searched for *that* — because now you're searching "like speaks to like." HyDE automates exactly this mental rephrasing step, using the LLM itself to bridge the style gap before search even happens.

### Why HyDE over just improving the embedding model
| Approach | Trade-off |
|---|---|
| Try to fine-tune/pick a "better" embedding model for question-answer matching | Expensive (fine-tuning) or may not exist off-the-shelf for a narrow legal domain; doesn't fix the fundamental question-vs-answer phrasing asymmetry |
| No HyDE, embed the raw question only | Simpler, faster (no extra LLM call), but retrieval quality suffers on questions phrased very differently from how the source text reads |
| **HyDE: generate + embed a hypothetical answer, fuse with the original query (LexAI's approach)** | One extra LLM call of added latency per query, but meaningfully closes the phrasing gap, with graceful degradation if it fails, and never fully replaces the original query's own signal |

---

# HEADER 6 — RERANKING (`backend/app/services/reranker.py`)

## 22. Bi-encoders vs. cross-encoders — why FAISS/BM25 are "retrieve broad" and CrossEncoder is "rerank narrow"

### The concept
There are two fundamentally different architectures for computing "how relevant is this document to this query," and the difference matters enormously for both **speed** and **accuracy**:

- **Bi-encoder** (what FAISS/dense embeddings use): the query and each document are embedded **completely independently**, each producing its own standalone vector, with *no interaction between them* during encoding — only afterward is a simple similarity function (cosine/inner product) applied between the two separately-computed vectors. Crucially, this means **every document's vector can be precomputed once, ahead of time, at ingestion** — at query time, you only need to embed the *query* (one embedding call) and then do fast vector math against thousands of already-computed document vectors. This is *why* FAISS search is fast.
- **Cross-encoder** (what the reranker uses): the query and a *specific candidate document* are fed **together, jointly**, into the neural network in a single pass — the model can directly attend to interactions between query words and document words while computing relevance, producing a much richer, more accurate relevance judgment. But this means a cross-encoder score **cannot be precomputed** — it has to be freshly computed for every single (query, document) pair, at query time, which is far too slow to run across an entire corpus of thousands of chunks for every user question.

### The two-stage "retrieve broad, then rerank narrow" pipeline
This speed/accuracy trade-off is exactly why LexAI (like most production RAG systems) uses **both**, in sequence, rather than picking one:
1. **Stage 1 (broad, fast, less precise)**: FAISS + BM25 fusion (Headers 3–5) quickly narrows an entire corpus of possibly thousands of chunks down to a manageable shortlist — e.g. the top ~20 RRF-fused candidates (`settings.retrieval_fused_top_n`).
2. **Stage 2 (narrow, slow, more precise)**: The CrossEncoder reranker then does the expensive, high-accuracy joint (query, document) scoring — but only on that small shortlist of ~20, which is fast enough to run per-request, producing a final, much more precise ordering (`settings.rerank_top_n`, the number actually sent to the LLM as context).

This is explicitly documented in the reranker module's own docstring: *"the top ~20 RRF-fused candidates are re-scored by a CrossEncoder... which is more accurate than either retrieval signal alone but too slow to run over the full corpus — hence 'retrieve broad, rerank narrow' as a two-stage pipeline."*

### Why this is necessary
Running a cross-encoder over an entire corpus for every query would be computationally prohibitive (a full neural network forward pass jointly encoding the query with each of thousands of chunks, for *every single question asked*) — but skipping the cross-encoder entirely and relying only on the faster bi-encoder/BM25 fusion leaves real accuracy on the table, since neither of those methods lets the model directly reason about query-document *interactions*, only compare independently-computed representations.

### Real-world analogy
This mirrors how a hiring pipeline typically works: first, an initial resume screen (bi-encoder-like — fast, independent scoring of each resume against a job description keyword list) narrows thousands of applicants down to, say, 20 candidates. Then, a much more expensive and thorough step — actual interviews (cross-encoder-like — the interviewer and candidate directly interact, producing a far richer judgment) — is only run on that narrowed shortlist, because you could never afford to individually interview every single applicant who submitted a resume.

### Why two-stage over cross-encoder-only or bi-encoder-only
| Approach | Problem |
|---|---|
| Cross-encoder over the *entire* corpus for every query | Far too slow — computationally infeasible at any real scale |
| Bi-encoder (FAISS/BM25 fusion) only, no reranking | Fast, but leaves accuracy on the table — misses the more nuanced query-document interaction signal a cross-encoder captures |
| **Two-stage: bi-encoder retrieves broad, cross-encoder reranks narrow (LexAI's approach)** | Gets both: retrieval stays fast at corpus scale, while the final small candidate set gets the more accurate, more expensive scoring treatment |

---

## 23. `sentence-transformers` CrossEncoder usage — model choice, lazy import, precomputed rank output

### The concept and implementation
LexAI uses the `sentence-transformers` library's `CrossEncoder` class, loaded with `BAAI/bge-reranker-v2-m3` (`settings.reranker_model`) — a well-regarded, publicly available, multilingual reranking model from Hugging Face:

```python
class CrossEncoderReranker:
    def __init__(self, settings: Settings):
        from sentence_transformers import CrossEncoder   # lazy import
        logger.info("loading_reranker_model", model=settings.reranker_model)
        self._model = CrossEncoder(settings.reranker_model)
        logger.info("reranker_ready", model=settings.reranker_model)

    def rerank(self, query: str, documents: list[Document], top_n: int) -> list[RerankedDocument]:
        if not documents:
            return []
        texts = [doc.page_content for doc in documents]
        ranked = self._model.rank(query, texts, top_k=min(top_n, len(documents)), return_documents=False)
        return [
            RerankedDocument(document=documents[entry["corpus_id"]], rerank_score=float(entry["score"]))
            for entry in ranked
        ]
```

Three implementation details worth understanding:

**1. Lazy import, explained directly in the code comment:**
```python
# Imported lazily so importing this module doesn't require torch
# to be installed (e.g. when `reranker_enabled=False` and the
# caller never constructs this class — see qa_service.py).
```
`sentence-transformers` pulls in `torch` as a dependency — a large (hundreds of MB), heavyweight library. The module docstring's own opening line notes this pulls in *"a few hundred MB, and the model is downloaded from Hugging Face on first load"* — a real operational cost, especially on resource-constrained free-tier hosting. By deferring the `from sentence_transformers import CrossEncoder` line to *inside* `__init__` (rather than a top-of-file import), simply *importing* the `reranker.py` module elsewhere in the codebase doesn't force `torch` to load unless a `CrossEncoderReranker` instance is actually constructed — which `qa_service.py` only does conditionally: `self.reranker = CrossEncoderReranker(settings) if settings.reranker_enabled else None`.

**2. `.rank()` returns pre-sorted, structured results** — the `sentence-transformers` API does the sorting internally and returns a list of dicts like `{"corpus_id": 3, "score": 8.21}`, where `corpus_id` is the *index into the original `texts` list* you passed in. This is why the reranker's return line indexes back into the original `documents` list by `entry["corpus_id"]` — reconnecting the model's numeric output back to the actual `Document` objects (with their metadata) that were passed in.

**3. Model loaded once at process startup, not per-request** — `get_qa_service()`'s docstring explicitly calls this out: *"Building this loads the FAISS index, the BM25 index, and (if enabled) downloads/loads the CrossEncoder model — all real startup cost, which is why this is built once at app startup... rather than per-request."* Re-loading a neural network model from disk (or worse, re-downloading it from Hugging Face) on every single API request would make response latency completely unacceptable — a singleton pattern (fully covered in Header 9) ensures this heavy initialization happens exactly once.

### Why this is necessary
Reranking only delivers its accuracy benefit (topic 22) if it's actually fast enough to run inline, per-request, without becoming the new bottleneck — the combination of lazy-loading (avoiding unnecessary `torch` cost when disabled) and load-once-at-startup (avoiding repeated model-loading cost when enabled) is what makes that practical.

### Real-world analogy
This is like a specialist consultant a hospital keeps on retainer, available the instant a case needs them (loaded once, kept ready) — rather than hiring and onboarding a brand-new consultant from scratch for every single patient (reloading the model per request), and also not paying that consultant's retainer at all on days when the hospital has decided it doesn't need that specialty on staff (lazy import when disabled).

### Why `sentence-transformers` + a public reranker model over training a custom one
| Approach | Trade-off |
|---|---|
| Train/fine-tune a custom cross-encoder on Indian legal data | Requires labeled training data and ML training infrastructure — a large undertaking disproportionate to the project's scope |
| **Off-the-shelf `sentence-transformers` CrossEncoder (`bge-reranker-v2-m3`)** | Strong, well-tested general-purpose reranking quality out of the box, zero training required, integrates in a few lines of code |

---

## 24. Two-stage retrieval pipeline design — latency vs. accuracy tradeoff, and graceful degradation

### The concept, tying Headers 5 and 6 together
By this point you've seen the *entire* retrieval pipeline as it actually executes inside `stream_answer()`, end to end:

```
question
   │
   ├─(optional)→ HyDE: LLM generates a hypothetical answer passage  [topic 21]
   │
   ▼
Hybrid retrieval: FAISS (dense, ×N query embeddings) + BM25 (sparse) → RRF fusion  [topics 19–20]
   │  → top ~20 fused candidates
   ▼
CrossEncoder reranking (query, chunk) scored jointly, on just those ~20  [topics 22–23]
   │  → top ~5 (settings.rerank_top_n) final context chunks
   ▼
LLM generation, grounded only in those final chunks, streamed via SSE
```

Every stage narrows the candidate set further while spending progressively *more* compute per remaining candidate — cheap-and-broad first (thousands of chunks → ~20), expensive-and-narrow last (~20 → ~5). This is a general pattern worth recognizing beyond just this project: **whenever you have a cheap-but-imprecise filter and an expensive-but-precise filter, apply the cheap one first to shrink the problem, then the expensive one only on what's left** — never the reverse.

### Graceful degradation is threaded through every single stage
This is one of the most consistent design principles across the *entire* LexAI codebase, and it's worth naming explicitly, because it recurs everywhere:

```python
# HyDE fails → proceed without it
except Exception as exc:
    logger.warning("hyde_generation_failed", error=str(exc))
    return None

# Reranking fails → fall back to RRF order, unranked-by-cross-encoder but still usable
except Exception as exc:
    logger.warning("rerank_failed_falling_back_to_rrf_order", error=str(exc))
    context_documents = fused[: self.settings.rerank_top_n]

# Retrieval itself fails → this one CAN'T degrade further, so it's a hard error
except Exception as exc:
    logger.error("retrieval_failed", session_id=session_id, error=str(exc), exc_info=True)
    yield {"event": "error", "data": {"message": "Retrieval failed. Please try again."}}
    return
```
Notice the pattern: **enhancement layers** (HyDE, reranking) fail *softly* — a lesser-quality answer beats no answer. But **retrieval itself** (the one thing there's genuinely no fallback for — you cannot answer a question with zero retrieved context and still call it "grounded") fails *hard*, with an explicit, honest error message rather than silently returning an ungrounded/hallucinated answer. This same reasoning appears again in Header 2 (ingestion's quota-exhaustion handling — the BM25 half can still succeed even if FAISS embedding fails) and will appear again in Header 9 (the API layer's global exception handler).

### Why this is necessary
A multi-stage pipeline with several LLM/ML calls (HyDE's LLM call, the reranker's model, the final generation LLM call) has several independent points where something can go wrong — an API rate limit, a network blip, a timeout. Treating every failure as fatal would make the whole system fragile in proportion to how many stages it has; instead, each stage's failure mode is deliberately reasoned about individually: *"if this fails, what's the best we can still do?"*

### Real-world analogy
This mirrors how an experienced surgeon has a clear mental hierarchy of "which steps of a procedure have a safe fallback if a tool fails, versus which steps are truly critical-path with no safe alternative" — worked out and planned *in advance*, not improvised in the moment.

### Why explicit multi-stage design over a single monolithic "chain"
The `qa_service.py` module docstring directly explains this architectural choice, referencing the older `ConversationalRetrievalChain` LangChain abstraction it replaced: *"every stage here (HyDE, retrieval, rerank) is a custom component with its own error-handling and timing needs, so composing them explicitly is more readable than hiding that behind `RunnableLambda | RunnableLambda | ...` without gaining anything."*

| Approach | Trade-off |
|---|---|
| Single opaque chain abstraction (e.g. LangChain's `\|`-pipe / `ConversationalRetrievalChain`) | Concise to write, but hides each stage's individual error-handling and timing needs behind a generic interface — harder to reason about *and* to customize per-stage failure behavior |
| **Explicit, composed Python pipeline (LexAI's approach)** | More lines of code, but each stage's specific failure mode, timing (`hyde_ms`, `retrieval_ms`, `rerank_ms` are all individually measured and logged), and degrade-vs-error decision is visible and independently reasoned about |

---

*End of Part 3 (Headers 5–6).*

---

# HEADER 7 — LLM ORCHESTRATION & PROMPTING (`backend/app/services/providers.py`, prompts in `qa_service.py`)

## 25. Prompt engineering — system prompts, grounding instructions, anti-hallucination constraints

### The concept
"Prompt engineering" is the practice of carefully wording the instructions given to an LLM to reliably steer its behavior — what tone to use, what it's allowed/not allowed to do, and how to handle uncertainty. For a RAG system specifically, the single most important prompt-engineering job is **grounding**: forcing the model to answer *from the retrieved context*, not from its own possibly-outdated or hallucinated training knowledge.

### Implementation — the system prompt, line by line
```python
SYSTEM_PROMPT = """You are LexAI, a legal research assistant specialised in Indian law \
(the Constitution, IPC, CrPC, IT Act, and related statutes, judgments, and reports).

Answer the user's question using ONLY the retrieved context below. Follow these rules strictly:
- If the context does not contain enough information to answer confidently, say so plainly \
instead of guessing or relying on outside knowledge.
- Cite the source inline where relevant, e.g. "(IPC 1860.pdf, p.45)".
- Be precise and concise. This is a professional legal-research tool, not a casual chat.
- Do not fabricate section numbers, case names, or citations that are not in the context.

Retrieved context:
{context}"""
```
Each rule targets a *specific, known LLM failure mode*:
- **"using ONLY the retrieved context... not... outside knowledge"** — the core grounding instruction (topic 1's whole premise), explicitly overriding the model's instinct to answer from its parametric memory.
- **"If the context does not contain enough information... say so plainly instead of guessing"** — directly targets the single most dangerous LLM failure mode for a legal tool: confidently answering a question it actually can't answer from the given material. An LLM's default behavior, unconstrained, is to always attempt a fluent-sounding answer even when it should say "I don't know" — this instruction explicitly permits and requires abstention.
- **"Cite the source inline... e.g. '(IPC 1860.pdf, p.45)'"** — a concrete *format example* rather than an abstract instruction ("cite your sources") — LLMs follow concrete examples of desired output format far more reliably than abstract descriptions of the format.
- **"Do not fabricate section numbers, case names, or citations"** — names the *specific* hallucination risks most damaging in a legal context (an invented section number or case citation is far more harmful than an invented adjective), rather than a generic "don't make things up."

### How the context actually gets inserted (`_format_context`)
```python
def _format_context(documents) -> str:
    blocks = []
    for i, item in enumerate(documents, start=1):
        doc = item.document
        meta = doc.metadata
        header = f"[{i}] {meta.get('source_file', 'unknown')}"
        if meta.get("page") is not None:
            header += f", p.{meta['page']}"
        if meta.get("law_type"):
            header += f" ({meta['law_type']})"
        blocks.append(f"{header}\n{doc.page_content}")
    return "\n\n".join(blocks)
```
Every retrieved chunk is prefixed with a numbered, structured header (`[1] IPC 1860.pdf, p.45 (statute)`) *before* its actual text — this gives the LLM everything it needs to satisfy the system prompt's citation instruction without having to guess or reconstruct source information itself. This is a form of prompt engineering too: making the *right* answer the *easy* answer by pre-formatting the information the model needs to cite correctly, right next to the content it's citing.

### Why this is necessary
An ungrounded, unconstrained LLM prompt in a legal context is actively dangerous — a fluent, confident-sounding but fabricated section number is worse than no answer at all, because it's *indistinguishable* from a correct one to a non-expert user without independently checking. Careful prompt engineering is the primary lever available (short of fine-tuning) to push the model's behavior toward honesty and traceability.

### Real-world analogy
This is exactly the standard you'd hold a junior paralegal to: "only cite what's actually in the case file in front of you; if the file doesn't have the answer, say so — don't guess or make something up to look thorough." A good manager gives that instruction explicitly and with concrete examples of the expected citation format, rather than assuming the junior will intuit the standard on their own.

### Why explicit, example-driven prompting over a vague/generic system prompt
| Approach | Trade-off |
|---|---|
| Generic prompt ("You are a helpful legal assistant. Cite your sources.") | Vague — leaves too much room for the model to fall back on its own training-data knowledge or invent plausible-looking citations |
| **Specific, rule-by-rule prompt with concrete citation-format examples (LexAI's approach)** | Longer prompt, but each rule targets a named, specific failure mode and gives the model an unambiguous, example-anchored standard to follow |

---

## 26. LangChain core primitives — `Document`, message types (`SystemMessage`/`HumanMessage`/`AIMessage`)

### The concept
LangChain is a framework providing common data structures and interfaces so different pieces of an LLM pipeline (loaders, chunkers, embedders, vector stores, LLM clients) can interoperate through shared types, rather than every component inventing its own incompatible representation. LexAI deliberately uses only `langchain_core` (the actively-maintained foundation package) — not the higher-level, more opinionated `langchain` or the archived `langchain_community`/`langchain_experimental` packages (a distinction covered fully in topic 28).

### The two primitives used throughout the codebase

**1. `Document`** (`langchain_core.documents.Document`) — the universal "a piece of text plus its metadata" container, used identically from ingestion all the way through to generation:
```python
Document(page_content=text, metadata={"source_file": pdf_path.name, "page": page_number})
```
Every stage of the pipeline — the PDF loader, the chunker, the vector store, the BM25 store, the retriever, the reranker — passes `Document` objects around using this same shared shape. This consistency is exactly what lets, say, `reciprocal_rank_fusion()` (Header 5) accept ranked lists of `Document` objects from *both* FAISS and BM25 interchangeably, even though those two systems are otherwise completely unrelated implementations — they both speak `Document`.

**2. Message types** (`langchain_core.messages`: `SystemMessage`, `HumanMessage`, `AIMessage`) — structured wrappers around the role-tagged conversation turns introduced conceptually in topic 4:
```python
messages = [SystemMessage(content=SYSTEM_PROMPT.format(context=context_text))]
messages.extend(history.messages[-8:])
messages.append(HumanMessage(content=question))
```
Rather than passing around raw `{"role": "system", "content": "..."}` dicts everywhere, these typed classes give you a `.content` attribute, a `.type` attribute (used by `providers.py::_message_to_dict` to map back to Ollama's expected role strings), and integrate cleanly with `InMemoryChatMessageHistory` (Header 8), which stores exactly these message objects.

### Why this is necessary
Without a shared vocabulary of types, every pair of components in the pipeline (loader → chunker, chunker → embedder, retriever → reranker, memory → LLM client) would need custom glue code to translate between each other's bespoke data shapes. Standardizing on `langchain_core`'s `Document` and message types means new components can be swapped in (a different vector store, a different LLM provider) as long as they speak the same shared interface.

### Real-world analogy
This is like a shipping industry standardizing on the intermodal container — once every port, ship, truck, and crane is built to handle the same container shape, cargo can move between completely different transport systems without needing custom handling equipment at every single transfer point. `Document` and the message types play that same "standard container" role for text and conversation data moving through LexAI's pipeline.

### Why `langchain_core` primitives over inventing custom equivalents
| Approach | Trade-off |
|---|---|
| Custom in-house classes for documents/messages | Full control, zero dependency, but reinvents a well-established wheel and loses interoperability with any LangChain-ecosystem tool |
| **`langchain_core.documents.Document` + `langchain_core.messages.*` (LexAI's approach)** | Interoperable, well-documented, actively-maintained foundation types — while still avoiding the *higher-level*, more opinionated (and in places unmaintained) LangChain abstractions built on top of them |

---

## 27. Why this project avoids deprecated LangChain abstractions — dependency risk awareness

### The concept
This is one of the most valuable *engineering judgment* lessons in the whole codebase, and it's called out explicitly and repeatedly rather than left implicit. LangChain as an ecosystem has several packages, with very different maintenance statuses:
- `langchain_core` — actively maintained foundation types (topic 26).
- `langchain_text_splitters` — actively maintained (used for `RecursiveCharacterTextSplitter`, Header 2 topic 10).
- `langchain_community` — **archived/unmaintained**, historically bundled many third-party integrations (including a `FAISS` vectorstore wrapper).
- `langchain_experimental` — **unmaintained**, historically included `SemanticChunker`.
- The `ConversationalRetrievalChain` — a *deprecated* high-level orchestration abstraction.

LexAI's actual dependency choices deliberately track this split:

| Feature | Deprecated/unmaintained path avoided | LexAI's actual choice |
|---|---|---|
| Vector store (Header 3, topic 15) | `langchain_community.vectorstores.FAISS` | Custom `FaissVectorStore` built directly on `faiss` + `langchain_core.documents.Document` |
| Semantic chunking (Header 2, topic 9) | `langchain_experimental.text_splitter.SemanticChunker` | Custom `SemanticChunker` reimplementation |
| RAG orchestration (this header) | `ConversationalRetrievalChain` | Explicit, hand-composed async pipeline in `qa_service.py` |

### Why this matters as a general engineering principle
The `qa_service.py` module docstring states the reasoning plainly: *"This replaces the old `ConversationalRetrievalChain` (deprecated, and its `[INST]`-tagged prompt template was written for a Llama model, not [the target model]) with an explicit async pipeline."* Two separate problems are named here:
1. **Deprecation risk** — a deprecated abstraction can be removed in a future library version, breaking the project with no warning beyond a changelog entry easy to miss.
2. **Hidden, wrong defaults** — `ConversationalRetrievalChain`'s *built-in default prompt template* was tuned for a specific model family (Llama-style `[INST]` instruction tags), which is simply the wrong prompt format for a different target model — a subtle correctness bug that's easy to inherit silently just by using the "convenient" high-level abstraction without reading deeply into what it does internally.

This is a genuinely mature piece of engineering judgment: **choosing to write more code yourself (the explicit pipeline) in exchange for full visibility into — and control over — exactly what's happening at each step**, rather than trusting a convenient black-box abstraction whose internal assumptions (a specific model's prompt format) may not match your actual use case, and whose long-term maintenance status is questionable.

### Why this is necessary
For a **portfolio project meant to demonstrate production-engineering judgment** (which is explicitly how you've described this project, per its role in your interview prep), taking on unnecessary dependency risk in *core* pipeline components — the parts an interviewer is most likely to probe — is exactly the kind of decision that should be actively defended, not stumbled into by taking the path of least resistance.

### Real-world analogy
This is like a contractor choosing to build a load-bearing wall with well-understood, individually-sourced materials they can inspect and vouch for, rather than a prefabricated "convenient" wall panel from a supplier who's gone out of business and stopped answering support calls — even though the prefab panel would have been faster to install initially.

### Why explicit code over convenient-but-risky abstractions, as a general rule
| Approach | Trade-off |
|---|---|
| Use the popular, convenient high-level abstraction (`ConversationalRetrquestionChain`, `langchain_community.FAISS`, etc.) | Less code to write initially, but inherits that abstraction's internal assumptions (sometimes wrong for your use case), maintenance status, and future breakage risk |
| **Build explicit, owned code on top of only actively-maintained foundation packages (LexAI's approach)** | More code to write and test yourself, but full transparency into behavior, no dependency on packages that could be abandoned or removed, and defensible design choices under scrutiny |

---

## 28. Async generators in Python (`AsyncIterator`, `async for`) — used for streaming

### The concept
A **generator** in Python is a function that produces a sequence of values *lazily*, one at a time, using `yield` instead of `return` — the caller can start consuming values before the generator has finished producing all of them, which is fundamentally different from a normal function that must compute its entire result before returning anything. An **async generator** is the same idea, but for asynchronous code — it can `await` between yields (e.g., waiting on a network response) without blocking the rest of the program while it waits.

### Why this is exactly the right tool for LLM streaming
`stream_answer()` in `qa_service.py` is declared as:
```python
async def stream_answer(self, question: str, session_id: str) -> AsyncIterator[dict]:
    ...
    yield {"event": "citations", "data": citations}
    ...
    async for chunk in self._llm.astream(messages):
        ...
        yield {"event": "token", "data": {"text": text}}
    ...
    yield {"event": "done", "data": {...}}
```
This function doesn't compute a full answer and then return it — it **yields a sequence of SSE events over time**, as each becomes available: citations first (as soon as retrieval/rerank finish), then individual answer tokens one by one (as the LLM produces them), then a final "done" summary event. The API layer (Header 9) then iterates this async generator with `async for event in service.stream_answer(...)`, forwarding each yielded event to the client immediately as an SSE message — this is the actual mechanism that makes token-by-token streaming (topic 4) possible at the code level.

### Why `async` specifically (not just a plain synchronous generator)
The LLM call itself (`self._llm.astream(messages)`, hitting a network endpoint — Ollama locally, or a hosted API) is an I/O-bound operation: the program is mostly *waiting* on a network response, not doing CPU work. A synchronous, blocking implementation would tie up the entire request-handling thread/process while waiting for each token — meaning the FastAPI server couldn't handle *any other* incoming request concurrently during that wait. `async`/`await` lets the Python event loop switch to handling other requests during these I/O waits, which is essential for a server expected to handle multiple concurrent users (rate-limited per-client via `slowapi`, per Header 9, but still needing to serve multiple *different* clients concurrently).

### Why this is necessary
Without generators (async or otherwise), the only way to "stream" would be to buffer the entire answer, then send it all at once — defeating the entire purpose of streaming (topic 4's UX argument: showing the user progress immediately rather than making them wait for the complete response).

### Real-world analogy
This is the difference between a chef who serves each course of a meal to the table the moment it's ready (a generator — the diner starts eating the appetizer while the main course is still being prepared) versus a chef who insists on plating the entire multi-course meal before bringing any of it out (a normal function that returns only once, fully computed) — diners obviously prefer the first approach when a full meal takes a while to prepare.

### Why async generators over threads/callbacks for streaming
| Approach | Trade-off |
|---|---|
| Multi-threading with callbacks | Much more complex to reason about (shared state, thread safety, callback-based control flow), higher overhead per concurrent request |
| **`async`/`await` + async generators (`AsyncIterator`, LexAI's approach)** | Python's modern, idiomatic tool for this exact I/O-bound-concurrency problem — cooperative, single-threaded concurrency with straightforward, linear-looking code (`async for` reads just like a normal `for` loop) |

---

# HEADER 8 — SESSION/MEMORY MANAGEMENT (`backend/app/services/memory.py`)

## 29. Conversation history management — `InMemoryChatMessageHistory`

### The concept
An LLM call is, by default, **stateless** — each call to the model has no memory of prior calls unless you explicitly re-supply the previous conversation as part of the new prompt. To support natural, multi-turn conversation (a user asking a follow-up question like "what about for a repeat offender?" without re-stating the original context), the *application* — not the model — has to track and re-supply conversation history on every request.

### Implementation
LexAI uses `langchain_core.chat_history.InMemoryChatMessageHistory` — described in the module docstring as *"the direct modern equivalent of the old `ConversationBufferMemory`"* (another example of topic 27's "avoid deprecated LangChain abstractions" principle: `ConversationBufferMemory` is itself an older, now-superseded API):
```python
@dataclass
class _SessionEntry:
    history: InMemoryChatMessageHistory = field(default_factory=InMemoryChatMessageHistory)
    last_active: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
```
Each conversation session gets one `InMemoryChatMessageHistory` object — essentially a growing list of `HumanMessage`/`AIMessage` objects (topic 26) in order. At the end of every turn, `qa_service.py` appends both sides of the exchange:
```python
history.add_user_message(question)
history.add_ai_message(answer)
```
And on the *next* request in the same session, that history is read back and spliced into the prompt (Header 7, topic 25's message-building code): `messages.extend(history.messages[-8:])`.

### Why only the last 8 messages, not the full history
```python
messages.extend(history.messages[-8:])  # last 4 turns of prior context, kept small deliberately
```
An unboundedly growing conversation history would eventually exceed the LLM's context window, and — even before hitting that hard limit — would waste an increasing amount of the context budget on old, possibly now-irrelevant turns, at the expense of the space available for the actual retrieved legal context (which is the part that actually needs to be large and detailed for a good answer). Capping at the last 8 messages (4 user/assistant turn pairs) is a deliberate, explicit trade-off: enough history for natural short-range follow-up questions, without letting conversation length silently degrade answer quality over a long session.

### Why this is necessary
Without conversation history, every single question in a session would need to be a fully self-contained, standalone query — a real usability problem for a research tool where users naturally ask follow-ups ("what about under the amended version?", "does that apply retroactively?") that only make sense in light of the immediately preceding exchange.

### Real-world analogy
This is like a human legal researcher who remembers the last few things a colleague asked in the current conversation (so "what about the amended version?" is understood in context) but doesn't try to recall — or drag into every new thought — the entirety of a months-long working relationship's worth of prior conversations, which would be both impractical and mostly irrelevant to the question actually being asked right now.

### Why `InMemoryChatMessageHistory` + a capped window over alternatives
| Approach | Trade-off |
|---|---|
| No conversation history at all | Simplest, but breaks natural follow-up questions entirely |
| Full, unbounded history every request | Natural conversation continuity, but eventually blows the context budget and dilutes the space available for actual retrieved legal content |
| **`InMemoryChatMessageHistory`, capped to the last 8 messages (LexAI's approach)** | Supports natural short-range follow-ups while keeping a predictable, bounded prompt size regardless of session length |

---

## 30. TTL-based in-memory stores, thread safety (`threading.Lock`), eviction strategies

### The concept
`SessionStore` in `memory.py` is a small, purpose-built in-process key-value store — session ID → conversation history — with two production-relevant concerns layered on top of a plain Python `dict`: **automatic expiry** and **thread safety**.

### TTL (time-to-live) eviction
```python
def _evict_expired(self) -> None:
    now = datetime.now(timezone.utc)
    with self._lock:
        expired = [sid for sid, entry in self._sessions.items() if now - entry.last_active > self._ttl]
        for sid in expired:
            del self._sessions[sid]
    if expired:
        logger.info("sessions_expired", count=len(expired))
```
Every session tracks a `last_active` timestamp, updated on every access (`get_history()` refreshes it: `entry.last_active = datetime.now(timezone.utc)`). `_evict_expired()` — called at the start of both `get_history()` and the `active_session_count` property, i.e. lazily on access rather than via a separate background timer/thread — sweeps out any session inactive longer than the configured TTL (`settings.session_ttl_minutes`).

**Why lazy eviction (on-access) rather than a background sweep thread**: it's simpler — no separate thread lifecycle to manage, start, or gracefully shut down — and "correct enough" for this use case, since stale sessions consume a small, bounded amount of memory regardless of exactly *when* they get cleaned up; there's no urgency requiring a dedicated background timer.

### Why TTL eviction is necessary at all
Without eviction, `_sessions` would grow **without bound** for as long as the server process runs — every unique visitor, including ones who never return, permanently occupies memory. For a public-facing demo/portfolio deployment expected to run continuously, this is a genuine, if slow-building, memory leak. TTL eviction bounds the store's size by the number of sessions active within any single `ttl_minutes` window, not the cumulative number of visitors ever.

### Thread safety — `threading.Lock`
```python
def __init__(self, ttl_minutes: int):
    ...
    self._lock = threading.Lock()

def create_session(self) -> str:
    ...
    with self._lock:
        self._sessions[session_id] = _SessionEntry()
    ...
```
Every method that reads or mutates `self._sessions` wraps the operation in `with self._lock:`. The class docstring is precise about *why* this matters and its actual scope: *"Thread-safe (for a single-process asyncio app)"* — FastAPI's async request handling is largely single-threaded cooperative concurrency (Header 7, topic 28), but certain operations (e.g. requests dispatched to a thread pool for blocking work) could still touch this shared dict from more than one OS thread. The lock protects against a real, if narrow, race-condition risk: two concurrent operations both reading-then-writing the dict at the same moment could corrupt its internal state or silently lose one of the two updates, without a lock to serialize access.

### Honest, documented limitations — not hidden
The module docstring states plainly:
> *"LIMITATION (documented, not hidden): this is a single-process in-memory store. It does not survive a restart and is not shared across multiple uvicorn workers. That's the correct tradeoff for a single-container... deployment — a multi-worker production deployment would swap this for a Redis-backed store behind the same `get_history()` / `touch()` interface."*

This is a valuable habit worth calling out on its own: rather than silently having a scaling limitation an interviewer might later "catch," the docstring **states it upfront, explains exactly why it's an acceptable trade-off for the actual deployment target, and names the specific upgrade path** (Redis, behind the same interface) if requirements ever changed. This demonstrates the design was a *deliberate, scoped choice* — not an oversight.

### Why this is necessary
A production (or production-adjacent, portfolio-demonstrating) service needs to bound its own resource usage over time, and be honest about its own architectural boundaries — both of which this small module does explicitly.

### Real-world analogy
TTL eviction is like a hotel automatically releasing a guest's room reservation after a defined period of no check-in activity — rather than permanently reserving every room ever booked, forever, which would eventually leave no rooms available for anyone. The `threading.Lock` is like a "one person in the filing cabinet at a time" rule — preventing two people from grabbing the same file drawer simultaneously and leaving it in an inconsistent, half-updated state.

### Why an in-memory TTL store (with a documented upgrade path) over building Redis-backed persistence immediately
| Approach | Trade-off |
|---|---|
| Build Redis-backed session storage from day one | More "production-grade" on paper, but adds real infrastructure complexity (a Redis dependency, connection handling, serialization) disproportionate to a single-container deployment's actual needs |
| **In-memory `dict` + TTL + lock, with the upgrade path explicitly named (LexAI's approach)** | Minimal, correct for the actual deployment target, and — critically — the limitation is documented and the future upgrade path is designed for (`get_history()`/`touch()` as the stable interface a Redis-backed implementation could drop in behind) rather than requiring a rewrite later |

---

*End of Part 4 (Headers 7–8).*

---

# HEADER 9 — BACKEND API ENGINEERING (FastAPI) (`backend/app/main.py`, `backend/app/api/routes.py`, `backend/app/config/settings.py`)

## 31. FastAPI basics — routers, dependency injection (`Depends`), Pydantic models/schemas

### The concept
**FastAPI** is a modern Python web framework built around Python type hints — it uses your function signatures to automatically validate incoming requests, generate interactive API documentation, and handle serialization, with minimal boilerplate. Two of its core mechanisms show up throughout `routes.py`:

**1. Routers** — `APIRouter()` lets you define a group of related endpoints (`/query`, `/session`, `/health`) in their own module, then mount them onto the main `app` with a shared prefix:
```python
router = APIRouter()
# ... route definitions ...
app.include_router(router, prefix="/api")
```
This keeps `main.py` (app-wide concerns: CORS, error handling, startup) cleanly separated from `routes.py` (the actual business endpoints) — a standard pattern for keeping a growing API organized rather than one giant file mixing infrastructure and business logic.

**2. Dependency Injection via `Depends`** — rather than each route handler constructing its own dependencies (like the `LegalQAService`), it *declares what it needs* as a parameter, and FastAPI resolves it automatically:
```python
@router.post("/query")
async def query(request: Request, body: QueryRequest, qa_service: LegalQAService = Depends(get_qa_service)):
    ...
```
`Depends(get_qa_service)` tells FastAPI: "before calling this route function, call `get_qa_service()` and pass its result in as `qa_service`." Because `get_qa_service()` is itself `@lru_cache`-wrapped (Header 8/9's singleton pattern, topic 33), every route that declares this dependency transparently receives the *same* already-built `LegalQAService` instance — without each route needing to know or care how that instance is constructed, or where it comes from.

**3. Pydantic models for request/response validation** — `body: QueryRequest` in the route signature means FastAPI automatically parses the incoming JSON body, validates it against the `QueryRequest` schema (defined in `app/models/schemas.py`), and rejects malformed requests with a clear `422 Unprocessable Entity` error *before* your route function's code even runs — you never have to manually check "did the client send a `question` field, and is it actually a string?"

### Why this is necessary
Without a framework handling request validation and dependency wiring, every route would need repetitive, error-prone boilerplate: manually parsing/validating the request body, manually constructing (or globally importing) the QA service, manually checking types. FastAPI's approach (declare what you need via type hints, let the framework handle the mechanics) keeps route handlers focused purely on their actual logic.

### Real-world analogy
Dependency injection here is like a hospital's operating room having a standard "requisition" system — a surgeon just states "I need the anesthesiologist and this specific set of instruments for this procedure," and the hospital's logistics system ensures the right (already-prepared, already-available) resources show up, rather than the surgeon personally sourcing and preparing every tool themselves before every single operation.

### Why FastAPI + Pydantic over manually validating requests
| Approach | Trade-off |
|---|---|
| Manually parse/validate JSON in each route (e.g. with plain Flask) | More boilerplate per route, validation logic duplicated or inconsistent across endpoints, no automatic interactive docs |
| **FastAPI + Pydantic schemas + `Depends` (LexAI's approach)** | Type hints double as both documentation and runtime validation; auto-generated OpenAPI docs at `/docs`; dependencies are declared, not manually wired |

---

## 32. Server-Sent Events (SSE) — protocol format, streaming responses, why not WebSockets

### The concept
**Server-Sent Events (SSE)** is a simple, standard, HTTP-based protocol for a server to push a *stream* of events to a client over a single, long-lived HTTP response — the client just keeps reading the response body as more data arrives, rather than the connection closing after one reply. This is the wire-level mechanism underlying the token-by-token streaming introduced conceptually back in topic 4, and the code-level async-generator mechanism from Header 7 topic 28.

### The wire format
```python
def _sse_format(event: dict) -> str:
    """Wire-format a single SSE frame: `event: <type>\ndata: <json>\n\n`."""
    return f"event: {event['event']}\ndata: {json.dumps(event['data'])}\n\n"
```
Each SSE "frame" is plain text: an `event:` line naming the event type, a `data:` line carrying a JSON payload, and a blank line (`\n\n`) marking the end of that frame — this double-newline terminator is what lets a client (Header 10, topic 39) know where one complete event ends and the next begins, even though the underlying TCP/HTTP stream delivers bytes in arbitrary-sized chunks with no inherent message boundaries of its own.

### Wiring the async generator (Header 7's `stream_answer`) into an HTTP response
```python
async def _sse_stream(qa_service: LegalQAService, question: str, session_id: str):
    async for event in qa_service.stream_answer(question, session_id):
        yield _sse_format(event)

@router.post("/query")
async def query(request: Request, body: QueryRequest, qa_service: LegalQAService = Depends(get_qa_service)):
    session_id = body.session_id or qa_service.sessions.create_session()
    return StreamingResponse(
        _sse_stream(qa_service, body.question, session_id),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```
FastAPI's `StreamingResponse` takes any async generator/iterable and streams its yielded values to the client as they're produced, rather than buffering the whole response first — this is the direct HTTP-layer continuation of the async-generator pattern established in `qa_service.py`. Two response headers matter specifically for reliable streaming: `Cache-Control: no-cache` prevents any intermediate cache from buffering/replaying a stale stream, and `X-Accel-Buffering: no` (an nginx-specific header, noted in the code comment "disable nginx response buffering, if fronted by nginx") prevents a reverse proxy from silently accumulating the whole response before forwarding it — which would completely defeat the purpose of streaming, delivering the "streamed" tokens all at once anyway.

### Why POST + hand-rolled SSE instead of the browser's native `EventSource`
This is directly explained in the frontend's `apiClient.js` docstring (fully covered in Header 10, topic 40) but is worth noting from the backend side too: the browser's native `EventSource` API can *only* issue `GET` requests with no custom request body — but this endpoint needs to receive a JSON body (`{question, session_id}`). Since `EventSource` structurally can't do that, both sides of this project use `POST` + a hand-rolled SSE parser instead.

### Why SSE over WebSockets
WebSockets provide a full-duplex (bidirectional) persistent connection — the client can send messages to the server at any time over the same open connection, not just receive them. LexAI's actual communication pattern is **one request in, then a stream of responses out** — the client never needs to send anything mid-stream (it just waits for the current answer to finish before it could conceivably send a follow-up, in `handleSend`'s design). SSE is the simpler, HTTP-native tool that exactly matches this one-directional-after-the-initial-request pattern, without taking on WebSockets' additional protocol complexity (a separate handshake, a stateful bidirectional connection to manage, different infrastructure/proxy compatibility considerations) for capability the application doesn't actually need.

### Why this is necessary
Server-side generation of a legal answer can take several seconds; SSE is what lets that latency be surfaced to the user progressively (as tokens arrive) rather than as one long, silent wait — the direct payoff of the UX argument introduced in topic 4, now implemented concretely at the transport layer.

### Real-world analogy
SSE is like a sports commentator giving a live, continuous play-by-play over the radio (one-way broadcast, always flowing from broadcaster to listener) — versus a WebSocket, which is more like a two-way radio/walkie-talkie conversation where both parties can transmit at any moment. For a play-by-play broadcast, you don't need — or want the complexity of — a walkie-talkie.

### Why SSE over WebSockets, summarized
| Approach | Trade-off |
|---|---|
| WebSockets | Full bidirectional capability, but more complex protocol/infrastructure for a use case that's actually one-directional after the initial request |
| **SSE over a `StreamingResponse` (LexAI's approach)** | Simpler, plain-HTTP, exactly matches the actual "one request → stream of responses" communication pattern needed here |

---

## 33. Rate limiting (`slowapi`) and CORS configuration

### Rate limiting — the concept and implementation
Rate limiting caps how many requests a single client can make in a given time window, protecting the service from being overwhelmed (whether by a malicious actor, a buggy client stuck in a retry loop, or just an unexpectedly popular demo link) and — for a project with real LLM/embedding API costs behind it — from runaway usage-based billing.

```python
limiter = Limiter(key_func=get_remote_address)

@router.post("/query")
@limiter.limit(get_settings().rate_limit_query)   # "20/minute"
async def query(...): ...

@router.post("/session")
@limiter.limit(get_settings().rate_limit_session)  # "10/minute"
async def create_session(...): ...
```
`slowapi` (a FastAPI-oriented port of the popular `Flask-Limiter`) is configured with `key_func=get_remote_address` — meaning limits are tracked *per client IP address*, so one heavy user can't degrade service for everyone else, but different users aren't affected by each other's usage. The specific limit strings (`"20/minute"`, `"10/minute"`) are sourced from `Settings` (topic 34), not hardcoded — keeping them tunable without a code change. Wiring this in also requires the app-level middleware and exception handler seen in `main.py`:
```python
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
app.add_middleware(SlowAPIMiddleware)
```

### CORS — the concept and implementation
**CORS (Cross-Origin Resource Sharing)** is a *browser-enforced* security mechanism: by default, a web page served from one origin (e.g. `http://localhost:5173`, the Vite dev server) is *blocked by the browser itself* from making API calls to a different origin (e.g. `http://localhost:8000`, the FastAPI backend), unless the server explicitly opts in by sending specific response headers granting permission. This is a browser-side protection against malicious cross-site request patterns — it has nothing to do with server-to-server calls, only browser-originated ones.

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "OPTIONS"],
    allow_headers=["*"],
)
```
Since LexAI's frontend (served from a Vite dev server or a separate static host) and backend (a separate FastAPI process/port) are, by definition, different origins from the browser's perspective, this middleware is what makes cross-origin calls between them possible at all — `allow_origins` is deliberately an explicit **allowlist** (sourced from `settings.cors_allowed_origins_raw`, defaulting to a set of known local dev ports) rather than a wildcard `"*"`, which matters specifically because `allow_credentials=True` is set — browsers actually forbid combining a wildcard origin with credentialed requests, and even where technically allowed, an open wildcard would let *any* website's JavaScript call this API on a visitor's behalf.

### Why both of these are necessary
Rate limiting protects service availability and cost under real or accidental load; CORS configuration is what makes the browser-based frontend able to talk to the backend at all, while still constraining *which* origins are allowed to do so — both are standard, expected hardening for any public-facing API, and their absence would be an immediate, obvious gap in a technical review.

### Real-world analogy
Rate limiting is like a ticket counter enforcing "one group served at a time, no more than N transactions per minute per customer" so no single customer can monopolize the counter and leave everyone else waiting indefinitely. CORS is like a building's front desk checking a visitor's badge against an approved list before letting them through to a restricted floor — even though the visitor is a legitimate person, the building only lets in badges it's been explicitly told to trust.

### Why explicit configuration over permissive defaults
| Approach | Risk |
|---|---|
| `allow_origins=["*"]` (wildcard CORS) | Any website could call this API using a visitor's browser session; incompatible with credentialed requests anyway |
| No rate limiting | A single client (malicious or just buggy) can degrade the service for everyone, or run up real API costs |
| **Explicit origin allowlist + per-route, per-IP rate limits (LexAI's approach)** | Constrained to known-trusted origins, usage bounded per client, both externally tunable via settings |

---

## 34. Centralized settings management — `pydantic-settings`, `.env` files, `lru_cache` singletons

### The concept
Scattering `os.environ.get("SOME_VAR")` calls throughout a codebase is a well-known maintainability anti-pattern: it's hard to know what configuration a project actually needs just by reading it, defaults get duplicated or drift inconsistently between call sites, and there's no validation that a given environment variable is actually the right *type* (a string where a number was expected, silently causing bugs downstream).

LexAI centralizes **every single configurable value** — for both the ingestion pipeline *and* the API service — into one `Settings` class (`pydantic-settings`' `BaseSettings`), with an explicit, enforced rule stated in the module docstring: *"Do not read `os.environ` directly anywhere else in the codebase — add a field here instead."*

### Implementation highlights
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=str(BACKEND_ROOT / ".env"),
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    llm_temperature: float = Field(default=0.3, ge=0.0, le=2.0)
    chunk_min_chars: int = Field(default=400, gt=0)
    retrieval_rrf_k: int = Field(default=50, gt=0, description="The k constant in RRF(d) = sum(1/(k+rank(d))).")
    ...
```
Every field is a typed, validated `Field` — `llm_temperature` can never silently become the string `"0.3"` or an out-of-range value like `5.0` (the `ge=0.0, le=2.0` constraint rejects it at startup, loudly, rather than causing a confusing bug deep inside an LLM call later). `pydantic-settings` automatically sources values from environment variables (or a `.env` file, loaded via `env_file=...`) matching each field name, falling back to the declared default if unset — meaning `backend/.env.example` (referenced in `require_google_api_key()`'s error message) effectively serves as **self-documenting configuration**, since every possible setting corresponds directly and traceably to one `Field` declaration in this one file.

### The `@lru_cache` singleton pattern for settings
```python
@lru_cache(maxsize=1)
def get_settings() -> Settings:
    """Cached settings singleton — import and call this, don't instantiate Settings() directly."""
    return Settings()
```
`@lru_cache(maxsize=1)` on a **zero-argument** function is a common, idiomatic Python trick for a lazy singleton: the *first* call to `get_settings()` actually constructs a `Settings()` instance (parsing environment variables, running all field validation); every *subsequent* call, anywhere in the codebase, returns that exact same cached instance instantly, without re-parsing anything. This same exact pattern is reused for `get_qa_service()` in `qa_service.py` (Header 5–8's `LegalQAService`, whose construction is genuinely expensive — loading FAISS/BM25 indexes and the reranker model) — a `Settings()` object is cheap to build, but the *pattern itself* (module-wide, consistently-accessed singleton, resolved lazily and cached) is applied identically for a fundamentally different reason there: to guarantee the *entire application* shares exactly one instance of expensive, stateful resources (one FAISS index in memory, one loaded reranker model), never accidentally rebuilding them.

### `require_google_api_key()` — "fail loudly and early"
```python
def require_google_api_key(self) -> str:
    """Fail loudly and early instead of letting a 401 surface deep in a request."""
    if not self.google_api_key:
        raise RuntimeError("GOOGLE_API_KEY is not set. Add it to backend/.env (see .env.example) before running ingestion or starting the API.")
    return self.google_api_key
```
This is a deliberate, named design principle worth internalizing generally: when a piece of required configuration is genuinely missing, **fail immediately, at startup, with a clear, actionable error message** — rather than letting the program start "successfully" and only discover the problem much later, deep inside a request, as a confusing, hard-to-diagnose authentication error surfacing from a completely unrelated part of the code.

### Why this is necessary
A project with this many independently-tunable knobs (chunking parameters, retrieval `k` values, RRF's constant, rate limits, TTLs, model names) needs one authoritative, browsable, type-checked place to see and adjust *all* of them — without this, understanding "what can this project even be configured to do differently" would require grepping the entire codebase for scattered `os.environ` calls.

### Real-world analogy
This is like a building having one single, authoritative circuit-breaker panel, clearly labeled, rather than individual light switches scattered and hidden throughout the building with no central map of what controls what — if something needs adjusting, or something's misconfigured, there's exactly one place to look.

### Why centralized, validated settings over scattered `os.environ` calls
| Approach | Problem |
|---|---|
| `os.environ.get(...)` calls scattered throughout the codebase | No single source of truth for what's configurable, no type validation, easy for defaults to drift/duplicate inconsistently between call sites |
| **One centralized, typed, validated `Settings` class + a cached singleton accessor (LexAI's approach)** | Every configurable value is discoverable in one file, validated at startup (fail loudly, early), and guaranteed consistent everywhere it's used |

---

# HEADER 10 — FRONTEND (REACT) (`frontend/src/`)

## 35. React fundamentals — hooks (`useState`, `useEffect`, `useCallback`, `useRef`), component composition

### The concept
React builds UIs out of **components** — reusable functions that return what should be rendered, automatically re-running (re-rendering) whenever their inputs (props) or internal state change. **Hooks** are React's mechanism for giving a plain function component access to state, side effects, and other React features, without needing to write a class.

### The hooks actually used, and *why* each one specifically, in `ChatInterface.jsx`
- **`useState`** — holds values that change over time and should trigger a re-render when they do: `sessions`, `activeSessionId`, `isStreaming`, `errorMessage`, `statusMessage`. Each of these directly drives what's shown on screen (the message list, the "Streaming answer..." status text, an error banner), so React needs to know to re-render when they change.
- **`useRef`** — holds a mutable value that does *not* trigger a re-render when changed, used for two different jobs here:
  - `bottomRef` — a reference to a DOM element (an empty div at the bottom of the message list), used purely to call `.scrollIntoView()` on it — this doesn't need to be React "state" because it's not something the UI displays; it's an imperative escape hatch to directly manipulate the DOM.
  - `abortRef` — holds the current `AbortController` for an in-flight SSE stream, so it can be cancelled later (e.g., on component unmount) — again, not something that should ever cause a re-render, just a value that needs to persist across renders and be mutated imperatively.
- **`useEffect`** — runs side effects (code with an external effect beyond just computing a render) in response to specific dependency changes:
  ```js
  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth", block: "end" });
  }, [sessions, activeSessionId]);   // re-run whenever messages or the active session change
  
  useEffect(() => {
    return () => abortRef.current?.abort();   // cleanup function: abort any in-flight stream on unmount
  }, []);   // empty dependency array = runs once, on mount, with cleanup on unmount
  ```
- **`useCallback`** — memoizes a function definition so it isn't recreated fresh on every single render, which matters specifically because these functions (`persist`, `updateActiveMessages`, `handleSend`, etc.) are both called from event handlers *and* passed down as props to child components (`Sidebar`, `InputBox`) — without memoization, every parent re-render would hand those children a *brand-new* function reference, which can cause unnecessary re-renders of the children or break other memoization further down the tree.

### Component composition
```jsx
export function ChatInterface() {
  return (
    <div className="flex h-screen">
      <Sidebar sessions={sessions} activeSessionId={activeSessionId} onSelectSession={handleSelectSession} onNewChat={handleNewChat} />
      <div>
        <MessageList messages={activeMessages} bottomRef={bottomRef} />
        <InputBox onSend={handleSend} disabled={isStreaming} />
      </div>
    </div>
  );
}
```
`ChatInterface` is the single "smart" component holding all state and logic; `Sidebar`, `MessageList`, `InputBox`, and `CitationsPanel` are comparatively "dumb" presentational components, each receiving exactly the data and callback functions they need as **props**, with no direct knowledge of the SSE streaming machinery, session persistence, or any of the state management happening above them. This is a standard, deliberate separation: state and business logic concentrated in one place, display logic distributed into small, focused, independently-understandable pieces.

### Why this is necessary
Hooks are what let a plain, simple function (not a class) fully participate in React's rendering and lifecycle model — reacting to state changes, running side effects at the right times, and avoiding unnecessary work — while component composition keeps a genuinely complex UI (streaming chat, session history, citations, error states) organized into pieces small enough to reason about individually.

### Real-world analogy
Component composition here is like an assembly line where each station (`Sidebar`, `MessageList`, `InputBox`) does one focused job and only receives exactly the materials/instructions it needs for that job — nobody at the `InputBox` station needs to understand how session persistence or SSE parsing works; they just need "here's a `onSend` function to call when the user submits a question."

### Why hooks + composition over a class-based or monolithic approach
| Approach | Trade-off |
|---|---|
| One giant component handling everything, no sub-components | Simpler initially, but becomes unreadable and hard to modify as complexity grows |
| Class components (React's older API) | More verbose lifecycle methods (`componentDidMount`, etc.) for the same behavior hooks express more concisely |
| **Function components + hooks + focused sub-components (LexAI's approach)** | Concise, each piece of logic/state colocated with why it exists, and presentational components stay simple and reusable |

---

## 36. Consuming SSE from the browser with `fetch` + `ReadableStream` — why not `EventSource`

### The concept, and the constraint that rules out the "obvious" tool
As established from the backend side (topic 32), the browser's native `EventSource` API is the standard, built-in way to consume SSE — but it can **only** issue `GET` requests with no custom body. LexAI's `/api/query` endpoint needs a `POST` with a JSON body (`{question, session_id}`), so `EventSource` is structurally ruled out entirely — not a stylistic choice, a hard capability mismatch. The `apiClient.js` docstring documents this reasoning explicitly, including that a third-party alternative (`axios-sse`) was *considered and rejected* for being a low-adoption, single-maintainer package not worth taking a dependency risk on (echoing the exact same dependency-risk reasoning from Header 7, topic 27) — landing on a small, fully-owned, hand-rolled implementation instead.

### The hard part: SSE framing across arbitrary network chunk boundaries
```js
export async function* streamQuery(question, sessionId, { signal } = {}) {
  const response = await fetch(`${API_BASE}/api/query`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ question, session_id: sessionId ?? null }),
    signal,
  });
  ...
  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    const frames = buffer.split("\n\n");
    buffer = frames.pop() ?? "";  // last segment may be incomplete - held back for the next read
    for (const frame of frames) {
      const event = parseSSEFrame(frame);
      if (event) yield event;
    }
  }
  ...
}
```
`response.body.getReader()` gives raw access to the HTTP response as a stream of binary chunks — critically, **a single network "chunk" delivered by `reader.read()` can end anywhere**: mid-line, mid-JSON-value, or mid-frame, entirely independent of where the *logical* SSE frame boundaries (`\n\n`) actually fall, because TCP/HTTP delivers bytes with no awareness of the application-level SSE framing on top of it. The code's own docstring names this as *"the one thing this MUST get right."*

The fix: accumulate incoming text into a running `buffer`, split on `\n\n` to find *complete* frames, but — critically — **hold back** the last (possibly incomplete) segment via `frames.pop()` rather than processing it immediately, appending more data to it on the *next* read until it eventually does contain a complete `\n\n`-terminated frame. Only ever-complete frames are parsed and yielded; a frame straddling two separate `reader.read()` calls is correctly reassembled before being processed, rather than being incorrectly parsed as two broken half-frames.

**Verification, not just assumption**: the docstring notes this exact scenario was *"verified with a dedicated test that deliberately splits a single SSE frame's JSON payload across multiple simulated reads"* — turning a subtle, easy-to-get-wrong edge case (that might only manifest occasionally, unpredictably, depending on real-world network timing) into something explicitly, deterministically tested.

### Malformed-frame resilience
```js
try {
  return { event: eventType, data: JSON.parse(dataLines.join("\n")) };
} catch {
  return null; // malformed/partial JSON - drop this frame rather than throw and kill the whole stream
}
```
`parseSSEFrame` swallows a JSON parse failure and returns `null` rather than throwing — because throwing here would propagate up and terminate the *entire* async generator, killing the whole streaming connection over what might be a single, recoverable, malformed frame. Dropping one bad frame and continuing to process the rest of the stream is a far better failure mode for a user mid-conversation than the whole answer stream abruptly dying.

### Why this is necessary
Without correct frame reassembly across chunk boundaries, the chat interface would intermittently, unpredictably fail to parse tokens correctly whenever a network chunk happened to split mid-frame — a bug that would be maddeningly hard to reproduce and debug later if not handled correctly (and tested for) from the start, since it depends on real-world network chunking behavior that's largely out of the application's control.

### Real-world analogy
This is like receiving a long fax transmission in separate physical page-feeds that don't necessarily line up with where sentences or paragraphs actually end on the original document — you have to hold onto an unfinished sentence from the bottom of one page and correctly stitch it onto the next page's continuation before you can actually read that sentence, rather than trying to interpret each physical page-feed in complete isolation.

### Why hand-rolled `fetch`-based SSE parsing over the alternatives
| Approach | Problem |
|---|---|
| Native `EventSource` | Cannot send a POST request with a JSON body at all — structurally incompatible with this endpoint |
| Third-party SSE library (`axios-sse`) | Low-adoption, single-maintainer package — a dependency-risk concern (same reasoning as Header 7, topic 27) for something this small to just own directly |
| **Hand-rolled `fetch` + `ReadableStream`, with explicit buffer-based frame reassembly (LexAI's approach)** | Full control, no dependency risk, and — because the hard part (chunk-boundary framing) is both understood and explicitly tested — genuinely robust despite being "hand-rolled" |

---

## 37. Handling network chunk boundaries / partial JSON buffering

*(This topic is covered in full detail as part of topic 36 above — the buffering logic in `streamQuery` and the malformed-frame handling in `parseSSEFrame` are precisely the mechanism for this. Summarized here for completeness of the syllabus, cross-referenced rather than repeated.)*

The core principle, worth restating on its own because it generalizes well beyond just SSE parsing: **any time you're processing a stream of data delivered in chunks whose boundaries don't necessarily align with your data's logical structure, you must buffer incomplete trailing data and wait for more input, rather than assuming a single read() call corresponds to one complete logical unit.** This same principle applies equally to parsing streamed JSON, reading a file line-by-line from a network source, or (going back to Header 2) reading PDF pages — any time "how the data arrives" and "how the data is logically structured" aren't guaranteed to match up, this buffering pattern is the correct general-purpose fix.

---

## 38. `localStorage` for client-side persistence

### The concept
`localStorage` is a simple, synchronous, browser-provided key-value string store, persisted on the user's device across page reloads and browser restarts (until explicitly cleared) — but scoped to a single browser, on a single device; it isn't backed by any server and isn't shared across devices or browsers.

### Why LexAI needs this at all — filling a genuine backend gap
The `apiClient.js` docstring explains the *architectural reason* this exists on the frontend rather than being a backend feature:
> *"The backend keeps server-side memory per session_id (30-min TTL, see app/services/memory.py) but has no endpoint to list past sessions or replay their transcripts — it's a memory store, not a chat history database. So 'chat history sidebar' is implemented client-side."*

This is a precise, important distinction: `SessionStore` (Header 8) exists purely to give the *LLM* short-term conversational context for follow-up questions — it was never designed to be a durable, listable, replayable chat-history database, and (per its own documented, deliberate limitation) doesn't even survive a server restart. The **sidebar's chat history list**, by contrast, is a completely different feature with different requirements — and rather than building out backend infrastructure for a feature the core RAG system doesn't actually need, it's implemented entirely client-side.

### Implementation
```js
const SESSION_STORAGE_KEY = "lexai_sessions_v1";

export function loadSessionList() {
  try {
    const raw = localStorage.getItem(SESSION_STORAGE_KEY);
    return raw ? JSON.parse(raw) : [];
  } catch {
    return [];
  }
}

export function saveSessionList(sessions) {
  try {
    localStorage.setItem(SESSION_STORAGE_KEY, JSON.stringify(sessions));
  } catch {
    /* localStorage unavailable (private browsing, quota) - degrade to in-memory only for this tab */
  }
}
```
The entire list of sessions — each with its `sessionId`, a derived `title`, and its *full message transcript* — is serialized to JSON and stored under one key. Every mutation (`persist()`/`updateActiveMessages()` in `ChatInterface.jsx`) both updates React state *and* calls `saveSessionList()`, keeping the two in sync. Both functions are wrapped in `try/catch` — `localStorage` can throw (private/incognito browsing modes restrict it, or a quota can be exceeded) and the code deliberately degrades to "in-memory only for this tab" rather than crashing the whole app over a non-critical persistence failure — yet another instance of the graceful-degradation philosophy first named explicitly back in Header 6, topic 24.

### Reconnecting to server-side memory
```js
/* ... reuses the same session_id when you click back into one (continuing the same server-side memory,
   if it hasn't expired - if it has, the backend transparently starts a fresh one,
   per SessionStore.get_history's documented fallback). */
```
When a user clicks an old conversation in the sidebar, the frontend re-sends that same `session_id` on the next `/api/query` call. If the backend's 30-minute TTL (Header 8, topic 30) hasn't expired that session yet, the LLM genuinely still has that conversation's context. If it *has* expired, `SessionStore.get_history()`'s documented fallback transparently creates a fresh, empty history under that same ID — the frontend's displayed message transcript (from `localStorage`) and the backend's actual conversational memory can silently diverge in that case, but the UX consequence is graceful: worst case, a follow-up question loses server-side context and gets a slightly less contextually-aware answer, not an error.

### Why this is necessary
Without any persistence at all, refreshing the browser tab would wipe out the entire visible conversation history — an unacceptable UX regression for a research tool where a user might want to revisit an earlier session's answer, even after closing and reopening the browser.

### Real-world analogy
This is like keeping your own personal notebook of past conversations with a consultant, separate from whatever internal notes the consultant's own office keeps — your notebook lets you look back at what was discussed even long after the consultant's own working notes for that specific conversation have been filed away or discarded.

### Why client-side `localStorage` over building backend chat-history infrastructure
| Approach | Trade-off |
|---|---|
| Build a backend chat-history database (persisted transcripts, list/replay endpoints) | Real infrastructure (a database, new endpoints, migrations) for a feature that's purely about the *frontend's* display convenience, not the RAG system's actual retrieval/generation needs |
| **Client-side `localStorage`, decoupled from the backend's short-TTL LLM-context memory (LexAI's approach)** | Zero backend changes needed, degrades gracefully if unavailable, correctly scoped to what's actually a client-side UX concern |

---

## 39. Markdown rendering (`react-markdown`, `remark-gfm`)

### The concept
LLM-generated legal answers naturally include structured formatting — numbered lists of conditions, bold emphasis on key terms, sometimes tables comparing provisions. If rendered as raw text, all of this Markdown syntax (`**bold**`, `1. item`, `| table | cells |`) would show up as ugly literal characters instead of the intended formatting. `react-markdown` parses Markdown text and renders it as proper React elements (real `<strong>`, `<ol>`, `<table>` tags), and `remark-gfm` (GitHub-Flavored Markdown) extends the base Markdown support with commonly-needed extras — tables, strikethrough, autolinks — that plain Markdown doesn't include by default.

### Why this is necessary
An LLM instructed (Header 7, topic 25) to be *"precise and concise"* about legal information will naturally reach for structured formatting — a numbered list of statutory conditions is genuinely clearer than the same conditions crammed into one run-on paragraph. Properly rendering that formatting (rather than showing raw asterisks and pound signs) is a basic, expected polish item for any AI chat interface displaying LLM output.

### Real-world analogy
This is like the difference between reading a lawyer's typed brief with its numbered clauses, bold defined terms, and proper indentation rendered as an actual formatted document — versus reading the exact same content as an unformatted plain-text email where all that structural signal has been stripped out, leaving you to mentally reconstruct which parts were meant to be a list or emphasized.

### Why `react-markdown` + `remark-gfm` over rendering raw LLM text
| Approach | Problem |
|---|---|
| Render the LLM's raw text output directly (`{content}` in JSX) | Markdown syntax characters (`**`, `#`, `-`) show up literally instead of being interpreted as formatting |
| **`react-markdown` + `remark-gfm` (LexAI's approach)** | LLM output renders as properly structured, readable formatting — lists, emphasis, tables — matching what the model actually intended |

---

## 40. Component design — Tailwind CSS utility classes, shadcn/ui-style components

### The concept
**Tailwind CSS** is a "utility-first" CSS framework — instead of writing custom named CSS classes (`.chat-header { ... }`) in a separate stylesheet, you compose a component's styling directly in its markup using small, single-purpose utility classes: `className="flex items-center gap-2 border-b border-ink-3 pb-4"` (from `ChatInterface.jsx`'s header) means "flex layout, center-align items, gap of 2 units between them, a bottom border in a custom `ink-3` color, bottom padding of 4 units" — all composed inline, with no separate CSS file to cross-reference.

### Custom design tokens, not default Tailwind colors
Notice the project doesn't use Tailwind's generic default palette (`bg-gray-900`, `text-blue-500`) — it uses **custom, semantically-named color tokens**: `bg-ink`, `text-text-ink`, `text-muted-ink`, `border-ink-3`, `text-brass`. This means the color palette was deliberately designed (likely a warm, "legal library"-styled ink/brass/parchment theme, matching the "Legal Research Desk" framing seen in the header text) and configured once in Tailwind's config, then referenced consistently everywhere by meaningful name — rather than reaching for whatever default blue or gray Tailwind ships with out of the box. This is a real signal of deliberate visual design rather than accepting an unstyled, generic-looking default.

### Why this is necessary
A chat interface for a *professional legal research tool* (as the system prompt itself frames it — Header 7, topic 25) benefits from a considered, purpose-built visual identity rather than looking like an unstyled or generically-templated AI chat demo — first impressions matter for a portfolio piece meant to be reviewed by an interviewer.

### Real-world analogy
Utility-first CSS is like furnishing a room by directly placing labeled, purpose-built pieces exactly where you want them (this utility class governs spacing, this one governs color, this one governs layout) versus writing out a separate, freeform interior-design document elsewhere that someone then has to cross-reference against the room to understand what's actually being styled and why.

### Why Tailwind + custom design tokens over plain CSS or default utility colors
| Approach | Trade-off |
|---|---|
| Plain, hand-written CSS files | More verbose, requires context-switching between markup and a separate stylesheet, easier for styles to drift out of sync with markup over time |
| Tailwind with default/generic color palette | Fast to build, but looks like every other unstyled Tailwind demo — no distinctive visual identity |
| **Tailwind + custom, semantically-named design tokens (LexAI's approach)** | Fast utility-first workflow, *and* a deliberate, consistent, on-brand visual identity applied everywhere through meaningfully-named tokens |

---

*End of Part 5 (Headers 9–10).*

---

# HEADER 11 — EVALUATION & BENCHMARKING (`backend/evaluation/`)

Every technique covered so far (HyDE, hybrid fusion, reranking) is a *claim* that it improves answer quality. This header covers how LexAI actually **measures** whether that claim holds — turning "I think this is better" into a number you can point to.

## 45. Retrieval metrics — precision, recall, nDCG@10, character-span scoring

### The concept
Before you can measure whether the *whole* pipeline gives a good answer, you first need to isolate and measure just the **retrieval** step in isolation — did the system find the right source material at all, independent of how well the LLM then wrote about it? `run_benchmark.py` does exactly this, following the format of the public **LegalBench-RAG** benchmark (`docstring: "verified against https://github.com/zeroentropy-cc/legalbenchrag's benchmark_types.py, not guessed"` — a deliberate note that the schema compatibility was actually checked against the real upstream source, not assumed from memory).

Three metrics are computed, all at the **character-span level** rather than whole-document level — meaning ground truth isn't just "which document contains the answer" but "which exact character range within that document":

- **Precision** — of all the characters your retrieved chunks cover, what fraction actually fall inside a true relevant span? (Are you retrieving mostly-relevant material, or a lot of noise alongside it?)
- **Recall** — of all the characters in the true relevant span(s), what fraction did your retrieval actually cover? (Did you find the *whole* answer, or just part of it?)
- **nDCG@10** (Normalized Discounted Cumulative Gain) — LexAI's own addition on top of the upstream metrics (explicitly noted: *"the upstream repo only computes precision/recall"*), which additionally accounts for **ranking order**: a relevant chunk retrieved at position #1 should count for more than the same chunk retrieved at position #9, since a user (or the LLM downstream) is much more likely to actually benefit from what's near the top of the results.

```python
def precision_recall(retrieved: list[OffsetChunk], ground_truth: list[GroundTruthSnippet]) -> tuple[float, float]:
    """Exact character-overlap formula from legalbenchrag's run_benchmark.py QAResult.precision/.recall."""
    total_retrieved_len = sum(c.end - c.start for c in retrieved)
    total_relevant_len = sum(s.end - s.start for s in ground_truth)
    relevant_retrieved_len = 0
    for chunk in retrieved:
        for gt in ground_truth:
            if chunk.file_path == gt.file_path:
                relevant_retrieved_len += char_overlap(chunk.start, chunk.end, gt.start, gt.end)
    precision = relevant_retrieved_len / total_retrieved_len if total_retrieved_len else 0.0
    recall = relevant_retrieved_len / total_relevant_len if total_relevant_len else 0.0
    return precision, recall
```

### Why a *separate*, simpler chunker just for this benchmark
```python
def chunk_corpus_with_offsets(corpus: dict[str, str]) -> list[OffsetChunk]:
    """
    Simple fixed-size chunking with tracked character offsets. Deliberately
    NOT the production SemanticChunker: this benchmark isolates the
    retrieval algorithm as the variable under test... using one simple,
    consistent chunker across all three configs keeps chunking strategy
    from confounding the comparison.
    """
```
This is a genuinely important experimental-design principle, worth internalizing on its own: **when you want to isolate the effect of one variable (here: the retrieval algorithm — baseline vs. hybrid vs. full), hold every other variable constant across the comparison.** If the benchmark used the production semantic chunker (which produces variable-sized, meaning-based chunks) for one config but a different chunker for another, any measured difference in precision/recall could be caused by the *chunking* difference rather than the *retrieval algorithm* difference you actually set out to measure — an experimental design flaw called a **confound**. Using one simple, fixed-offset chunker uniformly across all three configurations eliminates that confound, at the (acceptable, deliberate) cost of not testing with the exact chunking strategy production actually uses.

### The three-way ablation comparison
```python
results.append(run_pipeline_config("baseline_faiss_only", tests, index, lambda q: index.retrieve_baseline(q, k=5)))
results.append(run_pipeline_config("hybrid_faiss_bm25_rrf", tests, index, lambda q: index.retrieve_hybrid(q, k=5)))
results.append(run_pipeline_config("full_hybrid_plus_reranker", tests, index, lambda q: index.retrieve_full(q, k=5, reranker=reranker)))
```
This directly and empirically tests the central architectural claims made throughout Headers 5–6: does hybrid fusion actually beat dense-only retrieval? Does reranking actually improve on hybrid alone? Rather than assuming yes based on the general literature/intuition, this produces an actual measured table (`Precision@5`, `Recall@5`, `nDCG@10` per configuration) specific to *this* corpus and *these* queries — turning Header 5–6's design arguments into checkable, falsifiable claims.

### Why this is necessary
"We used hybrid retrieval and reranking because they're generally considered better techniques" is a much weaker, less defensible engineering claim than "we measured a +X precision, +Y recall improvement from adding hybrid fusion, and a further +Z from reranking, on this specific corpus" — the second is something you can actually show and defend under technical questioning, and it's also how you'd notice if a technique that's *supposed* to help actually didn't, for this particular domain/corpus.

### Real-world analogy
This is exactly the scientific-experiment principle of a **controlled comparison**: if you're testing whether a new fertilizer improves crop yield, you don't also switch the soil type or watering schedule between your test plot and control plot — you change only the one variable you're actually trying to measure the effect of, holding everything else identical, so any yield difference you observe can be confidently attributed to the fertilizer and nothing else.

### Why character-span, ranked, ablation-style evaluation over "eyeballing a few example answers"
| Approach | Problem |
|---|---|
| Manually read a handful of example Q&A pairs and judge "looks good" | Subjective, doesn't scale, easy to unconsciously cherry-pick favorable examples, no way to quantify improvement from one design choice to the next |
| **Systematic benchmark: character-span precision/recall/nDCG, controlled ablation across configs (LexAI's approach)** | Objective, reproducible, quantifies exactly how much each architectural addition (hybrid fusion, reranking) actually helps on this corpus |

---

## 46. RAGAS-style metrics — faithfulness, context precision/recall, answer relevancy

### The concept
Retrieval metrics (topic 45) only test whether the *right source material* was found — they say nothing about whether the *final LLM-generated answer* is actually good: is it faithful to the retrieved context (not hallucinating beyond what the sources actually say), does it actually answer the question asked, and so on? **RAGAS** (Retrieval-Augmented Generation Assessment) is a well-known framework defining exactly this family of end-to-end metrics:

- **Faithfulness** — does the generated answer's content actually follow from / is supported by the retrieved context, or does it contain claims not grounded in what was retrieved (i.e., hallucination)?
- **Context Precision** — of the contexts retrieved, how many were actually useful/relevant for answering this specific question?
- **Context Recall** — did the retrieved contexts, collectively, cover everything actually needed to produce a complete correct answer?
- **Answer Relevancy** — does the generated answer actually address what was asked, rather than being generically plausible-sounding but off-target?

### An important, worth-knowing discrepancy — the documented plan vs. the actual implementation
This is genuinely worth understanding carefully, in the same spirit as the embedding-provider discrepancy flagged in Header 3 (topic 13) — it's a good habit to notice when a project's *documentation of intent* and its *actual running code* have diverged.

`_ragas_compat.py` documents, in real detail, a genuine upstream dependency conflict in the actual `ragas` library (a `langchain_community.chat_models.vertexai` import that no longer exists in current `langchain-community`), and provides a working monkey-patch/stub fix for it — clearly written by someone who *did* attempt a real `ragas`-library integration and *did* hit and diagnose this specific bug.

However, `run_ragas.py`'s actual `run_ragas_eval()` function, as it stands, does **not** call the real `ragas` library or the `_ragas_compat` shim at all:
```python
def run_ragas_eval(golden_pairs: list[dict]) -> dict:
    # We intentionally avoid upstream ragas/judge dependencies and instead
    # compute lightweight, local approximations of the requested metrics
    # using the collected samples. This keeps evaluation offline and avoids
    # any external LLM quotas.
    ...
    def f1_overlap(a: str, b: str) -> float:
        ...
        common = sum(1 for t in a_tok if t in b_tok)
        prec = common / len(a_tok)
        rec = common / len(b_tok)
        return 2 * (prec * rec) / (prec + rec)

    faithfulness_scores.append(f1_overlap(response, reference))       # faithfulness ~ token-overlap F1 vs. reference
    answer_relevancies.append(f1_overlap(response, reference))         # same formula reused for relevancy
    ...
```
Instead, it computes **simple token-overlap-based approximations** (an F1 score between the generated answer's tokens and a reference answer's tokens, and set-based overlap between retrieved-context tokens and reference-answer tokens) — genuinely simpler, offline, dependency-free proxies for the real RAGAS metrics, explicitly chosen to *"keep evaluation offline and avoid any external LLM quotas"* (real RAGAS metrics are typically computed by using another LLM as a judge, which costs API calls and quota).

**The lesson**: `_ragas_compat.py` represents real, valuable diagnostic work that's still sitting in the repo, ready to be wired in — but the currently *active* evaluation path took a pragmatic, simpler, zero-additional-cost approach instead. This is worth explicitly understanding *before* discussing "how does RAGAS evaluation work in this project" in an interview — the honest, accurate answer is "the project implements lightweight local approximations of RAGAS's metrics, and separately contains a working fix for a real bug in the actual `ragas` library, ready to swap in if a judge-LLM-based evaluation is later wanted" — not simply "it uses RAGAS."

### Faithfulness gating — turning metrics into a pass/fail check
```python
GATE_THRESHOLDS = {"faithfulness": 0.75, "context_precision": 0.65}
...
if args.gate:
    failures = [m for m, threshold in GATE_THRESHOLDS.items() if scores.get(m, 0.0) < threshold]
    if failures:
        logger.error("ci_gate_failed", failed_metrics=failures, thresholds=GATE_THRESHOLDS)
        sys.exit(1)
```
The `--gate` flag turns this evaluation script into a genuine **quality gate**: if faithfulness or context precision fall below a defined threshold, the script exits with a non-zero status code — exactly the kind of check a CI pipeline (Header 12) could run automatically on every change, failing the build if a code change (e.g., a prompt tweak, a retrieval parameter change) measurably degrades answer quality, rather than only catching such a regression much later via manual spot-checking.

### Why this is necessary
Retrieval quality alone (topic 45) doesn't guarantee the final user-facing answer is good — the LLM could still retrieve the perfect context and then hallucinate anyway, or answer a slightly different question than what was actually asked. End-to-end metrics are what actually validate the thing the user experiences: the generated answer itself.

### Real-world analogy
This is like the difference between grading a research assistant purely on "did they pull the right source documents from the archive" (retrieval metrics) versus grading the actual final memo they wrote based on those documents (end-to-end/RAGAS-style metrics) — a research assistant could pull exactly the right sources and still write a memo that misrepresents or goes beyond what those sources actually say.

### Why lightweight local approximations over a full judge-LLM-based RAGAS run, as an engineering trade-off
| Approach | Trade-off |
|---|---|
| Full `ragas` library, LLM-as-judge metrics | More accurate, semantically-aware scoring (an LLM judge can recognize paraphrased-but-faithful content that token overlap would miss), but costs real API calls/quota per evaluation run, and requires navigating the documented upstream dependency bug |
| **Lightweight local token-overlap approximations (LexAI's currently active approach)** | Free, fast, fully offline, good enough to catch gross regressions and support a CI gate — with the real fix for the upstream bug already written and available as a documented, ready upgrade path |

---

## 47. Golden dataset construction for QA evaluation

### The concept
Both the retrieval benchmark (topic 45) and the RAGAS-style evaluation (topic 46) need **ground truth** — a set of real questions paired with known-correct answers/relevant-source-spans — to actually score anything against. Building this "golden dataset" well is its own distinct engineering task, and `build_golden_dataset.py` assembles it from **three deliberately different sources**, mixed together:

```python
"""
- 100 pairs from the Kaggle "LLM Fine-Tuning Dataset of Indian Legal Texts"...
- 100 pairs sampled from IndicLegalQA V2..., 20 each across factual / interpretive /
  procedural / contextual / predictive categories.
- 100 synthetic pairs generated from the actual Data/ PDF corpus via RAGAS's TestsetGenerator.
"""
```

### Why three different sources, deliberately mixed
Relying on just one source risks a systematic blind spot in the evaluation itself:
- **Public pre-existing QA datasets** (Kaggle, IndicLegalQA) test general Indian-legal-domain knowledge and question phrasing diversity, but may not be about content that's actually *in* LexAI's specific ingested PDF corpus — a model could score well here just from general legal knowledge, without the retrieval system actually working.
- **Synthetic pairs generated directly from the project's own `Data/` corpus** guarantee the questions are actually answerable *from what was ingested* — directly testing the retrieval + generation pipeline against its own real knowledge base — but synthetic, LLM-generated questions can be systematically easier or more formulaic than genuine human-asked questions.

Combining all three gives a more balanced, harder-to-game evaluation than any single source alone would.

### Honesty about data provenance — no fabricated ground truth
```python
"""
Download the first two datasets yourself first... This script does NOT
invent QA pairs from a dataset it can't see — every non-synthetic pair
here is read from files you point it at.
...
Any source can be omitted... the script assembles whatever it's given and
logs exactly how many pairs came from where, rather than silently padding
the shortfall.
"""
```
This is a deliberate, important integrity principle for building an evaluation dataset specifically: **the script refuses to fabricate placeholder ground-truth data to hit a target count.** If a source dataset isn't available (e.g., a Kaggle download hasn't been done yet), the resulting golden set is simply smaller and the shortfall is explicitly logged — never silently padded with made-up or lower-quality substitute data. This matters enormously for evaluation specifically, because a benchmark quietly built on fabricated or mismatched ground truth produces metric numbers that *look* legitimate but are actually meaningless — arguably worse than having no benchmark at all, since it creates false confidence.

### Flexible column-name handling for messy real-world data
```python
_KAGGLE_QUESTION_CANDIDATES = ["question", "instruction", "prompt", "query"]
_KAGGLE_ANSWER_CANDIDATES = ["answer", "output", "response", "completion"]

def _pick_column(fieldnames, candidates, override):
    if override:
        ...
    for candidate in candidates:
        if candidate in fieldnames:
            return candidate
    raise ValueError(...)
```
The code comment notes *"public instruction-tuning datasets on Kaggle are inconsistent about naming"* — this small utility pragmatically tries a list of common column-name variants before giving up and asking the user to specify explicitly via `--kaggle-question-col`, rather than being brittle and failing outright the moment a dataset happens to name its question column `"prompt"` instead of `"question"`.

### Why this is necessary
Every metric computed in topics 45–46 is only as trustworthy as the ground truth it's measured against — careful, honest, multi-source golden-dataset construction is the unglamorous but essential foundation that makes the rest of the evaluation pipeline's numbers actually mean something.

### Real-world analogy
This is like a school setting an exam using questions pulled from several different, independent textbooks and question banks, rather than only from the exact same textbook the students studied from — a exam built solely from the study material risks testing rote memorization of that one specific source rather than genuine understanding, while a multi-source exam gives a fairer, harder-to-game picture of actual competence.

### Why multi-source, honestly-assembled ground truth over a single convenient source
| Approach | Risk |
|---|---|
| Use only synthetic, LLM-generated QA pairs from the project's own corpus | Convenient (no external downloads needed), but synthetic questions can be systematically easier/more formulaic, and the evaluation would only ever test the system against questions the *system itself* (or a similar LLM) generated |
| Silently pad missing data sources with placeholder/fabricated pairs to hit a target count | Produces metric numbers that look complete but are measured against meaningless ground truth — actively misleading |
| **Three genuinely different sources, honestly-reported when a source is missing (LexAI's approach)** | Harder to game, more representative of real-world question diversity, and any gaps in coverage are transparent rather than hidden |

---

# HEADER 12 — TESTING & ENGINEERING PRACTICES (`backend/tests/`)

## 48. `pytest` fundamentals, `monkeypatch`, fixtures (`tmp_path`)

### The concept
**`pytest`** is Python's most widely-used testing framework. Rather than manually running a script and eyeballing whether the output looks right, tests are written as functions (conventionally prefixed `test_`) containing `assert` statements — `pytest` discovers and runs every such function automatically, reporting exactly which ones passed or failed, and why.

Two of `pytest`'s built-in **fixtures** (reusable, automatically-injected pieces of test setup, requested simply by naming them as a test function's parameter) do essential work in `test_ingestion_resume.py`:

**`tmp_path`** — provides a fresh, automatically-cleaned-up temporary directory unique to each test run:
```python
def test_select_pdf_files_to_process_skips_completed(tmp_path: Path) -> None:
    data_dir = tmp_path / "Data"
    data_dir.mkdir()
    for name in ["a.pdf", "b.pdf", "c.pdf"]:
        (data_dir / name).write_bytes(b"pdf")
    checkpoint = tmp_path / "ingestion_checkpoint.json"
    checkpoint.write_text('{"completed_files": ["a.pdf", "c.pdf"]}', encoding="utf-8")
    pending = select_pdf_files_to_process(data_dir, checkpoint)
    assert [path.name for path in pending] == ["b.pdf"]
```
This test directly verifies the checkpoint-resume logic from Header 2, topic 12 — it creates three fake `.pdf` files and a checkpoint claiming two are already done, then asserts that exactly the *remaining one* (`"b.pdf"`) is returned as pending. `tmp_path` means this test can freely create and write real files on disk to genuinely exercise the file-scanning logic, *without* ever touching the real project's actual `Data/` or risking leaving test debris behind — `pytest` automatically provides a clean, isolated directory per test and cleans up afterward.

**`monkeypatch`** — lets a test temporarily replace (patch) a piece of code's behavior for the duration of that single test only, automatically reverting afterward:
```python
def test_build_retrying_embed_fn_waits_minute_after_quota(monkeypatch) -> None:
    sleep_calls: list[float] = []
    monkeypatch.setattr(ingest.time, "sleep", lambda seconds: sleep_calls.append(seconds))
    ...
```
This is essential here for an obvious practical reason: the real retry/backoff logic (Header 2, topic 11) genuinely calls `time.sleep(60)` on a quota error — a real test that actually waited 60 real seconds every time it ran would make the test suite unbearably slow to run repeatedly during development. `monkeypatch.setattr` replaces the *real* `time.sleep` with a fake version that just *records* what it would have slept for (into `sleep_calls`), letting the test assert on the *intended waiting behavior* (`assert sleep_calls == [60.0]`) instantly, without the test suite actually pausing for a minute.

### Why this is necessary
Automated tests are what let a codebase evolve with confidence — a change to the retry logic, the checkpoint format, or the embeddings client can be verified *not to have broken existing, expected behavior* in seconds, by running the test suite, rather than requiring a human to manually re-verify every previously-working code path by hand after every change.

### Real-world analogy
`tmp_path` is like a chemistry teacher giving each student their own disposable, disconnected test-tube setup for an experiment — free to make a mess and get real, physical reactions happening, without any risk of contaminating the shared classroom supply cabinet, and it's simply thrown away and replaced fresh for the next experiment. `monkeypatch`'s time-mocking is like a flight simulator that can instantly fast-forward through hours of simulated flight time in seconds, letting a trainee pilot practice and verify their response to a long, slow-developing scenario without literally waiting for real hours to pass.

### Why `pytest` + fixtures over manual/ad-hoc verification
| Approach | Trade-off |
|---|---|
| Manually running the ingestion script and eyeballing the output after each change | Slow, easy to forget to re-check every previously-working scenario, no permanent record of what's actually been verified |
| **`pytest` with `tmp_path`/`monkeypatch` fixtures (LexAI's approach)** | Fast (seconds, not minutes), isolated (no real filesystem/time side effects), repeatable, and serves as living, always-current documentation of exactly what behavior is guaranteed |

---

## 49. Mocking/dummy classes for isolating units in tests

### The concept
A **unit test** aims to verify one specific piece of logic *in isolation* — but that logic often depends on other, more expensive or less-predictable components (an embedding API call, a FAISS index, a BM25 store). **Mocking/dummy objects** are stand-ins for those dependencies: minimal fake implementations that satisfy just the interface the code under test actually needs, without doing any of the real, expensive/unpredictable work.

### Implementation — `test_run_ingestion_reports_stats_when_embedding_is_skipped`
This test verifies a specific, important piece of Header 2's graceful-degradation behavior (topic 11: *"quota exhaustion doesn't abort the whole run"*) — that ingestion still completes and reports stats correctly even when embedding fails entirely. To test *just* that specific behavior, without needing a real PDF, a real embedding API, or a real FAISS/BM25 build, the test substitutes minimal dummy classes for every dependency:

```python
class DummyChunker:
    def __init__(self, *args, **kwargs) -> None:
        pass
    def split_documents(self, pages):
        return [Document(page_content="chunk", metadata={"page": 1})]

class DummyVectorStore:
    def __init__(self, dimension: int = 0) -> None:
        self.dimension = dimension
    def add_documents(self, chunks, vectors) -> None:
        pass
    def save_local(self, path) -> None:
        pass
    def __len__(self) -> int:
        return 0

class DummyBm25Store:
    def __init__(self, docs) -> None:
        self.docs = docs
    def add_documents(self, chunks) -> None:
        pass
    def save_local(self, path) -> None:
        pass

monkeypatch.setattr(ingest, "SemanticChunker", DummyChunker)
monkeypatch.setattr(ingest, "FaissVectorStore", DummyVectorStore)
monkeypatch.setattr(ingest, "Bm25Store", DummyBm25Store)
monkeypatch.setattr(ingest, "embed_all", lambda texts, settings: (_ for _ in ()).throw(RuntimeError("quota")))
```
Each dummy class implements *only* the specific methods the real `ingest.run_ingestion()` function actually calls on it (`split_documents`, `add_documents`, `save_local`, `__len__`) — with trivial, predictable bodies (do nothing, return a fixed value) instead of the real, complex, possibly-network-dependent logic. Critically, `embed_all` is patched to **deliberately raise a `RuntimeError("quota")`** — directly and deterministically forcing the exact failure scenario being tested (an embedding quota failure), rather than hoping to naturally trigger a real rate-limit by chance, which would make the test flaky and slow.

The test then asserts the ingestion pipeline still completes and reports the correct chunk count *despite* this forced embedding failure:
```python
stats = ingest.run_ingestion(data_dir, database_dir, dry_run=False, no_backup=True, settings=settings)
assert stats["total_chunks"] == 1
```

### Why this is necessary
Without mocking, testing "does ingestion gracefully handle a quota failure" would require either (a) actually exhausting a real API's quota on demand (impractical, slow, costs real money/quota, and not reliably reproducible), or (b) not testing this important failure-handling behavior at all, leaving it unverified and at risk of silently breaking in a future code change. Dummy classes let the test isolate and deterministically trigger *exactly* the scenario it needs, on demand, in milliseconds.

### Real-world analogy
This is like a flight simulator training a pilot for "what do you do if engine 2 fails mid-flight" — you don't need to actually damage a real engine on a real plane to meaningfully test and verify the correct emergency procedure; a simulator that faithfully reproduces *just the specific failure being tested*, with everything else behaving normally, is both safer and more practical, and lets you test the exact failure scenario reliably, on demand, as many times as needed.

### Why dummy classes/mocking over integration-testing every dependency for real
| Approach | Trade-off |
|---|---|
| Test against real PDFs, real embedding APIs, real FAISS/BM25 builds every time | Slow, costs real API quota, hard to reliably trigger specific failure scenarios (like a quota error) on demand, and a test failure could be caused by *any* dependency, making it unclear what actually broke |
| **Minimal dummy classes implementing just the needed interface, deliberately forcing specific failure scenarios (LexAI's approach)** | Fast, free, deterministic, and isolates exactly the one behavior (graceful quota-failure handling) actually under test |

---

## 50. Docker/CI-CD basics (deployment stack)

### The concept
This final topic ties back to the deployment stack referenced in your own project history (the "full Docker/CI/CD deployment stack" phase of LexAI's build) — while the specific Docker/CI configuration files weren't part of the code read in this pass through the repository, it's worth understanding *why* this layer exists and what problem it solves, as the natural capstone to everything covered so far.

### Why containerization (Docker) matters here specifically
Every piece of this project has real, specific runtime dependencies: a particular Python version, `torch` + `sentence-transformers` for the reranker (Header 6, topic 23), FAISS's compiled bindings, Node.js + a build toolchain for the React frontend (Header 10), and (if running Ollama locally) a whole separate local-model runtime. **"Works on my machine" is a genuinely common failure mode** for projects with this many moving pieces — a reviewer or teammate cloning the repo could easily hit a dependency-version mismatch, a missing system library, or an OS-specific quirk that has nothing to do with the actual application logic.

**Docker** solves this by packaging the application together with its *exact* runtime environment — the OS base image, system libraries, the precise Python/Node versions, all pinned dependencies — into a single, portable, reproducible unit (a container image) that runs identically regardless of what's installed on the host machine. This directly extends the same underlying philosophy seen throughout this project's dependency choices (Header 7, topic 27: avoiding fragile, drifting dependencies) — Docker is the deployment-level version of "make behavior predictable and reproducible, not dependent on whatever happens to already be installed on whichever machine runs this."

### Why CI/CD matters here specifically
**Continuous Integration (CI)** means automatically running checks (the `pytest` suite from topic 48, linting, potentially the evaluation gate from topic 46) on *every* code change, before it's merged — catching regressions immediately, at the moment they're introduced, rather than discovering much later that some earlier change quietly broke the checkpoint-resume logic or degraded answer faithfulness. **Continuous Deployment (CD)** extends this by automatically building and shipping a new container image once those checks pass, removing manual, error-prone, "did I remember every step" deployment procedures.

This is exactly where topic 46's `--gate` flag and topic 12's `--dry-run` flag earn their keep in practice: a CI pipeline can run `pytest` (fast, no API cost), the ingestion pipeline's `--dry-run` mode (validates the pipeline runs end-to-end without needing real API credentials in CI, per Header 2, topic 12), and `run_ragas.py --gate` (fails the build if answer quality measurably regresses) — all automatically, on every single change, forming a genuine automated quality safety net rather than relying purely on manual review.

### Why this is necessary
For a portfolio project specifically meant to demonstrate production-engineering competence (not just "does the demo work when I personally run it"), showing that the project can be reliably built, tested, and deployed by *anyone*, on *any* machine, via a repeatable automated process — rather than depending on tribal knowledge of exactly how to get it running — is itself a significant, concrete signal of engineering maturity to a technical reviewer.

### Real-world analogy
Docker is like shipping a fully-furnished, self-contained modular room (walls, wiring, plumbing, and furniture all included and pre-fitted together) rather than shipping a floor plan and a shopping list and hoping the receiving site happens to have compatible materials and skilled labor to assemble it identically. CI/CD is like a factory's automated quality-control line that inspects every single unit as it comes off the line, before it ships to a customer — catching a defect the moment it's introduced, rather than relying on an occasional manual spot-check that might miss it until a customer complains.

### Why containerized, CI/CD-driven deployment over manual setup-and-deploy
| Approach | Risk |
|---|---|
| Manual setup instructions ("install Python 3.11, run `pip install -r requirements.txt`, install Node 20, ...") + manual deployment | Fragile, environment-dependent, easy for instructions to go stale, no automated safety net against regressions |
| **Docker (reproducible runtime environment) + CI/CD (automated test/eval gates on every change, automated deployment) (the project's stated approach)** | Consistent, reproducible across machines, and regressions in correctness *or* answer quality are caught automatically, before they ever reach a deployed environment |

---

*End of Part 6 (Headers 11–12). This completes the full 50-topic LexAI study syllabus (Headers 1–12).*

## Closing note

You now have a complete, code-grounded study document covering every layer of LexAI: the theoretical foundations (Header 1), the offline ingestion pipeline (Header 2), the storage layer (Headers 3–4), the retrieval and reranking logic that's the technical heart of the project (Headers 5–6), the orchestration and conversational-memory layer (Headers 7–8), the API and frontend surfaces (Headers 9–10), and how the whole thing is measured and kept trustworthy (Headers 11–12).

A good next step, when you're ready: pick a handful of the "why this over alternatives" comparison tables scattered throughout and rehearse explaining them out loud, unprompted — that's the exact form most interview follow-up questions about a portfolio project take ("why did you choose X over Y here?"), and every one of them in this document is answered directly from what's actually implemented in the code, not from generic best-practice assumptions.
