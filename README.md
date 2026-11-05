# 🔀 YouTube Hybrid RAG Chatbot (LlamaIndex)

A CLI chatbot that lets you chat with a YouTube video through its transcript, using two kinds of search at the same time: similarity search over embeddings in ChromaDB Cloud, and a knowledge graph in Neo4j. The first time you run it, it fetches the video's captions, splits them into overlapping chunks, embeds the chunks into ChromaDB Cloud, and asks Gemini to extract relationships from every chunk and store them in Neo4j. From then on, for every question the LLM first writes two extra variants, all three versions of the question are searched in both stores, and the results are merged with Reciprocal Rank Fusion. The merged results are reranked with a cross-encoder, and the best ones are passed to Gemini, which is instructed to answer strictly from what the video says. The conversation is saved in PostgreSQL, so it carries over between runs. This is the **Hybrid RAG** version, where context is found by combining the vector search and the graph search. The Vector version lives in [youtube-vector-rag-llamaindex](https://github.com/viochris/youtube-vector-rag-llamaindex), and the Graph version in [youtube-graph-rag-llamaindex](https://github.com/viochris/youtube-graph-rag-llamaindex).

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?style=flat&logo=python&logoColor=white)
![LlamaIndex](https://img.shields.io/badge/Framework-LlamaIndex-6E56CF?style=flat)
![ChromaDB](https://img.shields.io/badge/VectorDB-ChromaDB%20Cloud-FF6F00?style=flat)
![Neo4j](https://img.shields.io/badge/GraphDB-Neo4j-008CC1?style=flat&logo=neo4j&logoColor=white)
![Gemini](https://img.shields.io/badge/LLM-Google%20Gemini-4285F4?style=flat&logo=googlegemini&logoColor=white)
![LiteLLM](https://img.shields.io/badge/Routing-LiteLLM-black?style=flat)
![Reranker](https://img.shields.io/badge/Reranker-Cross--Encoder-yellowgreen?style=flat)
![PostgreSQL](https://img.shields.io/badge/Memory-PostgreSQL%20(AsyncPG)-336791?style=flat&logo=postgresql&logoColor=white)
![Phoenix](https://img.shields.io/badge/Tracing-Arize%20Phoenix-purple?style=flat)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

## 📌 Overview

This repository contains one CLI script, `Hybrid-self-make-youtube-reader-cmd.py`, that turns a YouTube video into something you can ask questions. You give it a list of video URLs. It downloads each transcript and splits the text into overlapping chunks. On the vector side, the chunks are embedded with Google's Gemini embedding model and stored in ChromaDB Cloud. On the graph side, an LLM extracts relationships from every chunk, written as subject, relation, and object, and they are stored in Neo4j as a property graph. The script then opens an interactive chat loop in the terminal, where each answer is built from the passages retrieved from both stores for that question.

The transcript loader is written inside the script itself (`load_youtube_transcripts`) instead of using a ready-made LlamaIndex reader. For each URL, it reads the video id, calls `youtube-transcript-api` to get the caption snippets, joins the snippets into one continuous text, and wraps that text in a LlamaIndex `Document`. The video id and the URL are stored in the document's metadata, under the keys `video_id` and `url`.

The two stores are checked separately. The script asks whether the Chroma collection `youtube_hybrid_docs` has any vectors, and whether the Neo4j database has any node at all. If at least one of them is empty, the transcripts are fetched once. Then each side is handled on its own: a side that is empty is built from the fetched documents, and a side that already has data is loaded straight from its store. If both stores already have data, nothing is downloaded, embedded, or extracted, which saves time and LLM calls.

For every question, the script does not search with the question alone. The retrievers are wrapped in LlamaIndex's `QueryFusionRetriever`, set to `num_queries=3` and `mode="reciprocal_rerank"`. With this setting, the LLM writes 2 extra variants of the question, and the original question is kept, so 3 queries are searched. Each query goes to both the vector retriever and the graph retriever, which makes 6 searches. Their result lists are merged with Reciprocal Rank Fusion, the top 5 merged results are kept, and a cross-encoder reranker narrows them to the top 3 that reach the LLM. The answer is produced by a context-mode chat engine that also uses the PostgreSQL memory.

This design means that a single question can use several LLM calls. With the settings in the script and LlamaIndex's defaults, one call writes the question variants, the graph retriever asks the LLM for related keywords once for each of the 3 queries, and one more call writes the answer. That is about five LLM calls per question. The token and cost numbers printed after each answer add up the LLM calls recorded during that turn. They do not include the LLM calls made while the graph is built, because the counters are reset when the chat session starts.

The graph index also uses the Gemini embedding model, which is configured in the script. By LlamaIndex's default for a graph index, the graph nodes are embedded as well, so a question can be matched to the graph by meaning and not only by keywords. The script does not change this default.

The answers are meant to come from the retrieved passages and not from the model's own memory. The system prompt tells the model to answer strictly from the context retrieved from the vector store and the graph, and to say so when the answer is not there. This is an instruction and not a hard check, so the printed sources are there to let you verify each answer. After every answer, the script prints the source nodes with their scores, the token usage, and the estimated cost of that turn.

This repository is the Hybrid version of a three-architecture comparison: [Vector RAG](https://github.com/viochris/youtube-vector-rag-llamaindex), [Graph RAG](https://github.com/viochris/youtube-graph-rag-llamaindex), and Hybrid RAG. See [Three RAG Architectures](#-three-rag-architectures) for how they differ and where each one lives.

Here is what happens when you run the script:

1. Load `.env` and stop with a plain error message if `GOOGLE_API_KEY`, `GROQ_API_KEY`, or `PHOENIX_API_KEY` is missing. This check runs before the error handling in step 13 starts.
2. Configure the LLM (Gemini through LiteLLM), the embedding model, the chunking, the token counter, and the debug handler that is later used to calculate the cost.
3. Register Phoenix tracing, so every run is recorded as a trace that you can inspect in the Phoenix web interface.
4. Create the cross-encoder reranker. Its model weights are downloaded the first time this happens.
5. Connect to ChromaDB Cloud, open the `youtube_hybrid_docs` collection or create it if it does not exist yet, and check whether it has vectors.
6. Connect to Neo4j with the credentials from `.env`, and count the nodes in the database. On connection, the graph store also checks the database version and creates the constraints and the vector index it needs, if they do not exist yet.
7. If the Chroma collection is empty or the Neo4j database has no nodes, fetch the transcripts from YouTube once.
8. Vector side: if the collection is empty, chunk the documents, embed the chunks, and store the vectors in ChromaDB Cloud. Otherwise, load the index straight from the vector store.
9. Graph side: if the database has no nodes, let the LLM extract relationships from every chunk, and store the nodes, relationships, and embeddings in Neo4j. Otherwise, load the index straight from the existing graph.
10. Create the PostgreSQL memory, the vector retriever, the graph retriever, the fusion retriever, and the context chat engine with the reranker, which ties together retrieval, fusion, reranking, memory, and the LLM.
11. Run an interactive loop in the terminal until you type an exit word. Empty input is rejected with a warning.
12. For every question, print the answer, the source nodes with their scores, the token usage, and the estimated cost.
13. If anything fails after that, print the traceback and a categorized summary, then exit with code 1.

### ✨ Key Features
* 🎬 **Self-written transcript loader.** Fetches captions with `youtube-transcript-api` and builds the LlamaIndex documents itself, with the video id and the URL stored as metadata.
* 🔀 **Two searches, one answer.** Every question is searched in a vector store (ChromaDB Cloud) and in a knowledge graph (Neo4j), and the results are combined before they reach the LLM.
* ✍️ **Question variants.** The LLM writes 2 extra variants of each question (`num_queries=3`), and all 3 queries are searched in both stores, so the search is not limited to the exact wording of the question.
* 🧮 **Reciprocal Rank Fusion.** The result lists of all the searches are merged into one list by rank, so a passage that ranks well in several lists rises to the top. The top 5 merged results are kept.
* 🧠 **Semantic retrieval over the transcript.** The text is split into 2048-token chunks with a 200-token overlap and embedded with `models/gemini-embedding-2`. Questions are matched to chunks by meaning, so a question does not need to reuse the speaker's exact words.
* 🕸️ **LLM-built knowledge graph.** Each chunk is sent to Gemini in its own call, and up to 15 relationships are extracted from it. They are stored in Neo4j as a property graph. With `include_text=True`, each item that the graph side retrieves combines the matched relationships with the source text they came from.
* ♻️ **Build once, for each store.** The Chroma collection and the Neo4j database are checked separately. A store that already has data is loaded and not rebuilt, which keeps later runs fast and cheap.
* 🎯 **Fuse, then rerank.** The top 5 merged results are narrowed to the top 3 with a `cross-encoder/ms-marco-MiniLM-L-2-v2` reranker, so only the best passages reach the LLM.
* 💬 **Persistent conversational memory.** Chat history is stored in PostgreSQL through `Memory` and AsyncPG, so a follow-up question still makes sense after you restart the script.
* 🧱 **Grounded by instruction.** A strict system prompt tells the model to answer only from the retrieved context and to admit when it does not have the information.
* 🔍 **Transparent answers.** The retrieved source nodes are printed with their scores and text after every response, so each answer can be checked against the passages behind it.
* 📊 **Token and cost tracking.** Token counts come from LlamaIndex's callback handler, and the cost of each turn is estimated through LiteLLM. A single question uses several LLM calls, and the numbers add up the calls recorded during the turn.
* 🛰️ **Phoenix tracing.** Every run sends its traces to the Arize Phoenix project `youtube-hybrid-project`, where each step of a question can be inspected.
* 🛡️ **Categorized error handling.** Failures print the raw traceback for debugging, then a readable summary such as a quota, authentication, database, or access error.

---

## 🎯 Context & Problem Statement

### 📖 The Problem

Video is where a lot of knowledge lives today, and it is hard to work with. You cannot press `Ctrl+F` on something that was spoken, and scrubbing through an hour of video to find one explanation is slow. Pasting a transcript into a general chatbot brings its own problems. Four of them show up when you try to build something better.

1. **A video cannot be searched like text, and one way of searching may not be enough.** Finding the part that explains why something works, or how two ideas connect, means watching or skimming. Searching for similar passages finds text that resembles the question. Searching a graph follows the relationships between ideas. A question may be easier to answer with one of them than with the other. Keyword search would not help much either, because a question phrased differently from the speaker's words can match nothing.
2. **A general chatbot answers from memory, not from the video.** Asked about a specific video, a plain LLM may rely on old training data, mix in other sources, or guess. It also has no duty to say "that was not in the video", so you cannot tell which parts of its answer came from the video you care about.
3. **Naive pipelines are wasteful and forgetful.** A basic fetch-and-ask script downloads and embeds the same video on every run, and for a graph it would also pay for the relationship extraction again each time. It also loses the whole conversation when the process exits, so every session starts from zero.
4. **It is hard to see what an answer was based on and what it cost.** Without printed sources and token counts, you cannot check an answer against the video, and you cannot tell how many tokens a question used or what it cost.

### 💡 The Solution

Each problem maps to one piece of the pipeline, and each piece has a known gap.

| Problem | How this repo handles it | What is still missing |
| :--- | :--- | :--- |
| 1. Video cannot be searched, and one search may not be enough | The transcript is searched in two ways at once, by meaning in ChromaDB and through relationships in Neo4j. The LLM also writes extra variants of the question, and all the results are merged with Reciprocal Rank Fusion | The relationships come from an LLM, so the graph can be incomplete or wrong. Only the captions are used, so videos without captions cannot be read, and the transcript is requested in the default language, which is English |
| 2. Answers from memory | The merged results are reranked by a cross-encoder, a strict system prompt limits the answer to them, and the source nodes are printed so the answer can be checked | The limit is an instruction to the model and not a hard check on its output, so the printed sources remain the way to verify |
| 3. Wasteful and forgetful | Each store is only built when it is empty, and the chat history lives in PostgreSQL, so repeated use stays cheap and continuous | The stores are checked separately, the Neo4j check counts every node in the database and not only the ones from this script, and the session id is fixed in the script, so every run continues the same conversation |
| 4. Hidden sources and cost | Source nodes, token counts, and a per-turn cost estimate are printed, and Phoenix records a trace of every run | Token counts use an OpenAI tokenizer, so they are approximate for Gemini, the cost covers LLM calls only, and the calls made while building the graph are not included |

---

## 🧠 Core Concepts Explained

If RAG, graphs, or rank fusion are new to you, this table explains the terms used in the rest of the README.

| Concept | Plain-language explanation | Where you see it |
| :--- | :--- | :--- |
| **RAG** | Retrieval-Augmented Generation. First find the passages that are relevant to the question, then give them to the LLM together with the question, so the answer is based on your source and not only on what the model remembers | The whole script |
| **Transcript** | The captions of a video as text. Here it is fetched with `youtube-transcript-api` and joined into one continuous text per video | `load_youtube_transcripts` |
| **Document** | A LlamaIndex object that holds text and metadata. Each video becomes one document, with its `video_id` and `url` stored as metadata | `Document(...)` |
| **Chunking** | Splitting long text into smaller pieces, because search, extraction, and prompting work better with short passages. `chunk_size=2048` is counted in tokens, and `chunk_overlap=200` repeats the last 200 tokens at the start of the next chunk, so a sentence cut at the boundary keeps its context | `SentenceSplitter` |
| **Embedding** | A list of numbers that represents the meaning of a text. Texts with similar meaning get similar numbers, which is what makes search by meaning possible | `models/gemini-embedding-2` |
| **Chroma collection** | A named container inside ChromaDB that holds vectors, much like a table in a database | `youtube_hybrid_docs` |
| **Vector store and index** | The vector store is where chunks and their embeddings are kept. The index is the LlamaIndex object that searches it | `ChromaVectorStore`, `VectorStoreIndex` |
| **Knowledge graph** | A way to store knowledge as things and the relationships between them, instead of as running text. For example, "Gemini" and "embedding model" can be two nodes with a relationship between them | The Neo4j database |
| **Relationship (triplet)** | One fact written as subject, relation, object. The extractor asks the LLM for these, and each one becomes a relationship between two nodes in the graph | `SimpleLLMPathExtractor` |
| **Property graph** | A graph whose nodes and relationships can carry properties, such as a name or text. It is the graph model that LlamaIndex stores in Neo4j here | `PropertyGraphIndex` |
| **Neo4j** | A database made for graphs. It stores nodes and relationships directly and is queried with Cypher | `Neo4jPropertyGraphStore` |
| **Cypher** | Neo4j's query language. The script uses one Cypher query, `MATCH (n) RETURN COUNT(n) as total`, to count the nodes and decide whether the graph has to be built | `graph_store.structured_query(...)` |
| **Vector retriever and top-k** | Finding the chunks whose embeddings are closest to the question's embedding. `similarity_top_k=5` means "fetch the 5 closest" | `vector_index.as_retriever(similarity_top_k=5)` |
| **Graph retriever** | Finding the parts of the graph that relate to the question. By default, LlamaIndex combines two lookups for a graph index: the LLM is asked for keywords and synonyms of the question, which are matched against the graph, and the question's embedding is matched against the embedded graph nodes. `similarity_top_k=5` is passed on to the retrievers. These are LlamaIndex defaults that the script does not override, and they can change between LlamaIndex versions | `graph_index.as_retriever(include_text=True, similarity_top_k=5)` |
| **`include_text`** | An option that makes the graph retriever return the source text the relationships came from. By LlamaIndex's default, each returned item then holds a short preamble, the relationships found in that chunk, and the text of the chunk | `include_text=True` |
| **Question variants** | Extra versions of the question that the LLM writes before the search, because one phrasing may match less than another. With `num_queries=3`, the LLM writes 2 variants, and the original question is kept, so 3 queries are searched | `QueryFusionRetriever(..., num_queries=3)` |
| **Reciprocal Rank Fusion (RRF)** | A way to merge several ranked lists into one. An item gets `1 / (rank + 60)` from every list it appears in, with the rank counted from 0, and the scores are added up, so an item that ranks well in several lists rises to the top. It uses ranks and not raw scores, so lists with different kinds of scores can be merged. Items with identical content count as one item | `mode="reciprocal_rerank"` |
| **`QueryFusionRetriever`** | A LlamaIndex retriever that writes the question variants, sends each query to every retriever it wraps, merges all the result lists with the chosen fusion mode, and keeps the top `similarity_top_k` | `hybrid_retriever` |
| **Reranker (cross-encoder)** | A second, more careful scoring step. The searches are fast and approximate. The cross-encoder then reads the question and each merged item together and re-scores them, keeping only the best `top_n` | `SentenceTransformerRerank` (`top_n=3`) |
| **Source node** | An item that the retriever returned and that reached the LLM. It is typically a plain transcript chunk from the vector side, or a chunk together with its relationships from the graph side. The score printed next to it is the reranker's score | `ai_response.source_nodes` |
| **Context chat engine** | A chat engine that, for every message, retrieves context, adds it to the prompt, and remembers the exchange. The script creates it directly with `ContextChatEngine.from_defaults`, so that the fused retriever can be plugged in | `ContextChatEngine` |
| **System prompt** | A standing instruction given to the LLM before the conversation starts. Here it says to answer only from the retrieved context | `ai_system_prompt` |
| **`Memory`** | LlamaIndex's conversation memory. With `async_database_uri`, the history is stored in a database table, and `token_limit` caps how much history is kept | `Memory.from_defaults(...)` |
| **LiteLLM** | A library that offers one interface to many LLM providers. Here it calls Gemini and also provides the cost calculation | `LiteLLM`, `litellm.completion_cost` |
| **Trace** | A step-by-step record of what happened inside a run. Arize Phoenix collects traces and shows them in a web interface | `register(...)`, `LlamaIndexInstrumentor` |
| **`TokenCountingHandler`** | A LlamaIndex callback that counts the tokens sent to and received from the LLM | `token_counter` |
| **`LlamaDebugHandler`** | A LlamaIndex callback that records LLM inputs and outputs. The script reads its events to calculate the cost | `debug_handler` |
| **`nest_asyncio`** | Allows async code to run in environments where an event loop is already running | `nest_asyncio.apply()` |

---

## 🧩 Tech Stack Highlights

| Layer | Tool | Role |
| :--- | :--- | :--- |
| **Transcript source** | `youtube-transcript-api` | Fetches the captions of each video |
| **Orchestration** | LlamaIndex core (`Settings`, `Document`, `VectorStoreIndex`, `PropertyGraphIndex`, `StorageContext`, `SentenceSplitter`) | Global configuration, chunking, and index construction |
| **Text embeddings** | `GoogleGenAIEmbedding` (`models/gemini-embedding-2`) | Embeds the transcript chunks, the graph nodes, and each query |
| **Vector storage** | ChromaDB Cloud (`chromadb.CloudClient`, `ChromaVectorStore`) | Stores the vectors in the `youtube_hybrid_docs` collection and searches them |
| **Graph extraction** | `SimpleLLMPathExtractor` (up to 15 relationships per chunk, 5 workers) | Uses the LLM to turn each chunk into relationships |
| **Graph storage** | Neo4j (`Neo4jPropertyGraphStore`) | Stores the nodes, relationships, and embeddings, and serves the graph retrieval |
| **Retrieval fusion** | `QueryFusionRetriever` (`num_queries=3`, `mode="reciprocal_rerank"`, `similarity_top_k=5`) | Writes the question variants, searches both stores, and merges the results with Reciprocal Rank Fusion |
| **Reranking** | `SentenceTransformerRerank` (`cross-encoder/ms-marco-MiniLM-L-2-v2`) | Rescores the merged results and keeps the top 3, running locally on the CPU build of `torch` that `requirements.txt` installs |
| **LLM** | LiteLLM with `gemini/gemini-2.5-flash` (temperature 0.3) | Relationship extraction, question variants, keyword expansion, and answer generation |
| **Chat engine** | LlamaIndex `ContextChatEngine` | Retrieves context on every turn and injects it into the prompt |
| **Conversation memory** | `Memory` + AsyncPG (PostgreSQL) | Persistent chat history across runs |
| **Observability** | Arize Phoenix (`phoenix.otel`, OpenInference LlamaIndex instrumentor) | Traces of every run |
| **Metrics** | `TokenCountingHandler` with `tiktoken`, `LlamaDebugHandler`, `litellm.completion_cost` | Token counts and the cost estimate |
| **Config & runtime** | `python-dotenv`, `nest-asyncio`, `asyncio` | Loads secrets from `.env` and runs the async loop |

---

## ⚙️ Configuration Reference

Every tunable value lives directly in `Hybrid-self-make-youtube-reader-cmd.py`, so changing a setting means editing the script.

| Setting | Value | Effect |
| :--- | :--- | :--- |
| **Target videos** | `youtube_urls = ["https://www.youtube.com/watch?v=i3OYlaoj-BM"]` | The videos that are fetched and indexed on a fresh run |
| **Transcript language** | The default of `youtube-transcript-api` (English) | The transcript is requested without a language option, so a video needs an English transcript |
| **Chunking** | `chunk_size=2048`, `chunk_overlap=200` | How the transcript is split before embedding and extraction |
| **Embedding model** | `models/gemini-embedding-2`, `embed_batch_size=50` | Embeds the chunks, the graph nodes, and each query, 50 texts per request |
| **Graph extractor** | `SimpleLLMPathExtractor(llm=Settings.llm, max_paths_per_chunk=15, num_workers=5)` | Up to 15 relationships are extracted from each chunk, with 5 parallel workers |
| **Vector retriever** | `vector_index.as_retriever(similarity_top_k=5)` | Number of chunks fetched by the vector side for each query |
| **Graph retriever** | `graph_index.as_retriever(include_text=True, similarity_top_k=5)` | `similarity_top_k=5` is passed on to the graph retrievers, and each retrieved item combines the relationships with the source text they came from |
| **Fusion retriever** | `QueryFusionRetriever(retrievers=[vector, graph], similarity_top_k=5, num_queries=3, mode="reciprocal_rerank")` | Writes 2 variants of the question, searches 3 queries in both retrievers, merges the lists with Reciprocal Rank Fusion, and keeps the top 5. The LLM is the global one |
| **Reranker** | `cross-encoder/ms-marco-MiniLM-L-2-v2`, `top_n=3` | Number of merged items (source nodes) that reach the LLM |
| **LLM** | `gemini/gemini-2.5-flash`, `temperature=0.3` | The model for extraction, question variants, and answers, and its randomness (a lower value gives more focused answers) |
| **LLM fallback** | A commented-out block in the script | Model fallback and retries are disabled. The block lists a Gemini model and a Groq model as fallbacks |
| **Chat mode** | `ContextChatEngine.from_defaults(...)` | Retrieves context on every message |
| **Chroma collection** | `youtube_hybrid_docs` | Where the transcript vectors are stored |
| **Vector build check** | `chroma_collection.count() > 0` | If the collection has any vector, the vector side is loaded and not rebuilt |
| **Graph store** | `Neo4jPropertyGraphStore` with `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_URI`, `NEO4J_DATABASE` | Where the graph is stored and read from |
| **Store setup** | Done by `Neo4jPropertyGraphStore` when it connects | Reads the schema (which needs APOC), checks the database version, and creates uniqueness constraints and a vector index named `entity` if they do not exist. The vector index is only created on Neo4j 5.23 or newer |
| **Graph build check** | `MATCH (n) RETURN COUNT(n) as total`, then `total > 0` | If the database has any node at all, the graph side is loaded and not rebuilt |
| **Chat memory** | `session_id="youtube_hybrid_session"`, `token_limit=40000`, `table_name="youtube_hybrid_memory"` | How the chat history is stored and trimmed |
| **Phoenix project** | `youtube-hybrid-project` | The project that receives the traces |
| **Phoenix endpoint** | `PHOENIX_COLLECTOR_ENDPOINT`, with a hardcoded default | Where traces are sent. The default points to the author's Phoenix space |
| **Token counting** | `tiktoken` with `cl100k_base` | The tokenizer behind the token numbers |
| **Cost format** | Six decimal places | The estimated cost of each turn, for example `$0.010026 USD` |
| **Debug trace** | `LlamaDebugHandler(print_trace_on_end=True)` | Prints a trace in the terminal after each run |
| **Logging** | The root logger and the `httpx`, `google_genai`, and `litellm` loggers are set to `WARNING` | Keeps the terminal readable |
| **Checked at startup** | `GOOGLE_API_KEY`, `GROQ_API_KEY`, `PHOENIX_API_KEY` | The script stops with an error if any of them is missing |
| **Not checked at startup** | `CHROMA_API_KEY`, `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE`, `ASYNCPG_DATABASE_URL` | They are passed straight to their clients |
| **Exit commands** | `exit`, `close`, `out`, `break`, `quit`, `x` | Ends the session |

---

## 💡 Questions to Try

The first three questions are the ones used in the [benchmark repository](https://github.com/viochris/rag-token-benchmark-vector-graph-hybrid), in increasing order of difficulty. They are generic enough to ask about any video.

| Question | What it shows |
| :--- | :--- |
| `What is this video mainly about?` | A simple question that needs the overall topic of the video |
| `What are the main points or steps explained in this video, and how do they relate to each other?` | An analytical question that needs several parts of the transcript and how they connect |
| `Based on everything discussed in this video, what conclusion or key takeaway can be drawn, and what supporting details from different parts of the video lead to that conclusion?` | A multi-hop question that has to connect details from different parts of the video |
| `Explain that in simpler terms`, right after any answer | The memory at work, because the follow-up only makes sense with the previous answer |
| A question the video never covers | The grounding instruction at work, where the model should say it does not have the information |

---

## 🖥️ Example Console Output

An illustrative first-run session with both stores empty, based on the messages the script prints. Text in angle brackets stands for values that change from run to run.

```text
⚙️ [SYSTEM] Configuring LlamaIndex Settings (LLM & Embeddings)...
📊 [SYSTEM] Initializing Phoenix Observability & Tracing...
⚙️ [SYSTEM] Initializing Cross-Encoder Reranker...
⏳ [SYSTEM] Connecting to ChromaDB Cloud Client...
🔌 [SYSTEM] Attaching ChromaVectorStore adapter...
⏳ [SYSTEM] Connecting to Neo4j Graph Database...
⏳ [SYSTEM] Extraction triggered. Fetching transcripts from YouTube...
📥 [SYSTEM] Fetching YouTube transcripts...
⚙️ [SYSTEM] Vector DB empty. Embedding and indexing to Chroma...
🧠 [SYSTEM] Graph DB empty. Extracting triplets via LLM to Neo4j...
<progress bars while the graph is built>
✅ [SYSTEM] Both Vector and Graph indexes are successfully loaded!
🧠 [SYSTEM] Initializing Hybrid Chat Engine and PostgreSQL memory...
✅ [SYSTEM] Hybrid Chat Engine successfully integrated and ready for interaction.

======================================================================
💬 [SYSTEM] Interactive Hybrid Agent Session Started. Type 'exit' to stop.
======================================================================

🗣️ [USER] You: <your question about the video>
🧠 [AI] Querying both Vector and Graph databases...
----------------------------------------------------------------------
🤖 [AI RESPONSE]:
<the answer, based on the retrieved context>
----------------------------------------------------------------------

🔍 [SOURCE NODES] Retrieved <number> relevant nodes from fused databases:
--- Node 1 (score: <rerank score>) ---
📄 Text Context: <the source node text: a transcript chunk, or a chunk with a short preamble and its relationships>

📊 [LLAMAINDEX CALLBACK TOKEN OVERVIEW]:
Model Used        : <model>
Prompt Tokens     : <number>
Completion Tokens : <number>
Total LLM Tokens  : <number>
======================================================================

💰 [COST ANALYSIS]:
Total Estimated Cost: $<amount> USD
======================================================================
```

On later runs, when both stores already have data, the ingestion lines are replaced by `📂 [SYSTEM] Both databases contain data. Bypassing extraction phase.`, followed by `📂 [SYSTEM] Vector DB has data. Loading existing Chroma index...` and `📂 [SYSTEM] Graph DB has data. Loading existing Neo4j index...`. If only one store is empty, the transcripts are still fetched, and only the empty side is built while the other side is loaded.

---

## 📁 Repository Structure

```text
youtube-hybrid-rag-llamaindex/
├── Hybrid-self-make-youtube-reader-cmd.py
├── LICENSE
├── README.md
└── requirements.txt
```

| Path | Description |
| :--- | :--- |
| `Hybrid-self-make-youtube-reader-cmd.py` | The full pipeline in one script: transcript loading, vector and graph indexing, fused retrieval, reranking, chat engine, and the CLI loop |
| `requirements.txt` | All Python dependencies, with notes inside the file that explain why each one is needed |
| `LICENSE` | MIT License |
| `README.md` | You're reading it |

---

## 🏗️ Architecture & Data Flow

### 🔄 End-to-End System Flowchart

```mermaid
flowchart TD
    A["YouTube URLs from the youtube_urls list"] --> B{"Is the Chroma collection youtube_hybrid_docs empty?"}
    A --> M{"Does the Neo4j database have no nodes?"}

    B -- "No" --> H["VectorStoreIndex.from_vector_store loads existing data"]
    M -- "No" --> H2["PropertyGraphIndex.from_existing loads the stored graph"]

    subgraph VectorBuild ["Vector side, built only if the Chroma collection is empty"]
        C["Documents from load_youtube_transcripts, fetched once if either store needs data"] --> D["SentenceSplitter chunks the text, 2048 size and 200 overlap"]
        D --> E["GoogleGenAIEmbedding produces vectors"]
        E --> F["ChromaDB Cloud collection youtube_hybrid_docs"]
    end

    subgraph GraphBuild ["Graph side, built only if Neo4j has no nodes"]
        C2["The same documents"] --> D2["SentenceSplitter chunks the text, 2048 size and 200 overlap"]
        D2 --> E2["SimpleLLMPathExtractor asks the LLM for up to 15 relationships per chunk"]
        E2 --> E3["GoogleGenAIEmbedding embeds the graph nodes"]
        E3 --> F2["Neo4j stores nodes, relationships, and embeddings"]
    end

    B -- "Yes" --> C
    M -- "Yes" --> C2

    F --> G["VectorStoreIndex"]
    H --> G
    F2 --> G2["PropertyGraphIndex"]
    H2 --> G2

    Q["Question in the terminal"] --> V["The LLM writes 2 variants of the question, 3 queries in total"]
    V --> R1["Vector retriever, similarity_top_k 5, for each query"]
    V --> R2["Graph retriever with source text, similarity_top_k 5, for each query"]
    G --> R1
    G2 --> R2
    R1 --> RRF["Reciprocal Rank Fusion merges the result lists and keeps the top 5"]
    R2 --> RRF
    RRF --> J["Cross-encoder reranker keeps the top 3"]
    J --> K["Context chat engine"]
    L["PostgreSQL memory through AsyncPG"] --> K
    P["Gemini LLM through LiteLLM"] --> K
    K --> N["Answer based on the retrieved context"]
    N --> O["Answer, source nodes, tokens, and cost are printed"]
```

> **Note.** On every run, the script checks the two stores separately. If the Chroma collection has any vector, the vector index is loaded. If the Neo4j database has any node, the graph index is loaded. The transcripts are fetched once, only when at least one of the two stores needs to be built, and the same documents feed both sides. Phoenix records a trace for each run, and the token counter and debug handler observe every LLM call. The numbers printed after each answer are reset at the start of the chat session, so the LLM calls made during extraction are not part of them.

### 🔎 What Happens to One Question

Here is one question followed through the pipeline, from the moment you press Enter.

1. **The question reaches the chat engine.** The context chat engine asks the fusion retriever for context, using your message as it is.
2. **The LLM writes 2 variants.** With `num_queries=3`, the LLM is asked for 2 other versions of the question. If it returns more lines than needed, the extra ones are dropped. Together with your original question, that makes 3 queries.
3. **Each query is searched in the vector store.** The vector retriever compares the embedding of the query with the chunk embeddings stored in ChromaDB Cloud, and returns the closest chunks (`similarity_top_k=5`).
4. **Each query is searched in the graph.** By LlamaIndex's defaults, the graph retriever asks the LLM for keywords and synonyms of the query and matches them against the graph. It also matches the embedding of the query against the embedded graph nodes. With `include_text=True`, each result combines the matched relationships with the source text they came from.
5. **The result lists are merged.** The 3 queries and the 2 retrievers give 6 ranked lists. Reciprocal Rank Fusion merges them into one list, adds up the scores of items that appear more than once, and the top 5 are kept.
6. **The reranker narrows the list.** The cross-encoder reads your original question together with each merged item, scores them again, and keeps the best 3.
7. **The LLM writes the answer.** The 3 items are placed in the prompt together with the system prompt and the earlier messages from the PostgreSQL memory, and the LLM writes the answer from them.
8. **The script prints the result.** It prints the answer, the source nodes with their scores, the token usage, and the estimated cost, and the exchange is added to the memory.

Counted together, the LLM is called about five times for one question: once for the variants, once for each of the 3 keyword expansions in step 4, and once for the answer. The token and cost lines add up these calls. Embedding calls are not part of them.

---

## 🆚 Three RAG Architectures

This repository implements **Hybrid RAG**. The same task of answering questions about a video can also be done with plain similarity search over embeddings, with a knowledge graph, or with both approaches combined. That makes three versions of the same chatbot, plus a repository that measures what each one costs. This section explains how they differ, where each one lives, and what the benchmark found.

### 📍 Where Each Architecture Lives

| Architecture | Approach | Location |
| :--- | :--- | :--- |
| **Vector RAG** | Similarity search over embeddings | [youtube-vector-rag-llamaindex](https://github.com/viochris/youtube-vector-rag-llamaindex) |
| **Graph RAG** | Search over a knowledge graph | [youtube-graph-rag-llamaindex](https://github.com/viochris/youtube-graph-rag-llamaindex) |
| **Hybrid RAG** | Vector search and graph search combined | This repository (`Hybrid-self-make-youtube-reader-cmd.py`) |
| **Token and cost benchmark** | Compares the three on the same video and questions | [rag-token-benchmark-vector-graph-hybrid](https://github.com/viochris/rag-token-benchmark-vector-graph-hybrid) |

### 🔍 What Differs

The Graph and Hybrid columns follow the setup used in the benchmark repository.

| Aspect | [Vector RAG](https://github.com/viochris/youtube-vector-rag-llamaindex) | [Graph RAG](https://github.com/viochris/youtube-graph-rag-llamaindex) | Hybrid RAG (this repo) |
| :--- | :--- | :--- | :--- |
| How context is found | The question is compared with the chunk embeddings, and the closest chunks are used | The text is turned into relationships (subject, relation, object) kept in a graph, and the question is answered from that graph together with the source text | Both retrievers run, and their results are merged with Reciprocal Rank Fusion. The LLM also writes extra variants of the question before the search |
| Storage | ChromaDB Cloud | Neo4j | ChromaDB Cloud and Neo4j |
| Extra LLM work when indexing | None, only embeddings | An LLM extracts the relationships from every chunk | The same as Graph RAG |
| Reranking | Cross-encoder, top 5 to top 3 | The same cross-encoder | The same cross-encoder |
| Moving parts | Fewest | More | Most |

### 📊 About the Benchmark Repository

[rag-token-benchmark-vector-graph-hybrid](https://github.com/viochris/rag-token-benchmark-vector-graph-hybrid) builds the three architectures end to end on the same YouTube transcript and asks each of them the same three questions, the simple, analytical, and multi-hop ones listed under [Questions to Try](#-questions-to-try). Every turn reports prompt, completion, and total tokens plus an estimated cost in USD. All runs use `gemini-2.5-flash` with fallback disabled, so the model, the source, and the questions stay the same across the three.

Results from that repository, shown as total tokens and estimated cost:

| Question | Vector RAG | Graph RAG | Hybrid RAG |
| :--- | :---: | :---: | :---: |
| Q1 (Simple) | 5,858 tokens, $0.002618 | 4,959 tokens, $0.004638 | 9,208 tokens, $0.010026 |
| Q2 (Analytical) | 7,318 tokens, $0.005876 | 7,989 tokens, $0.010416 | 10,408 tokens, $0.010901 |
| Q3 (Multi-hop) | 8,754 tokens, $0.005338 | 9,836 tokens, $0.008787 | 12,871 tokens, $0.012668 |

In estimated cost, the order was Vector, then Graph, then Hybrid on every question. In total tokens, the same order held for the analytical and multi-hop questions. On the simple question, Graph used fewer total tokens than Vector (4,959 against 5,858) but still cost more, because its completions were longer.

These numbers come from one video, three questions, and one run per question, with costs estimated through LiteLLM. The benchmark measures tokens and cost only. It does not measure how good the answers are. The Hybrid column comes from the benchmark repository's runs, so it is a reference for this version and not a measurement made with this script alone.

### 🧭 Which One Fits Your Case

* If you want the simplest setup and the lowest cost, Vector RAG is the place to start, and the [Vector repository](https://github.com/viochris/youtube-vector-rag-llamaindex) shows how it works end to end.
* Graph RAG, in the [Graph repository](https://github.com/viochris/youtube-graph-rag-llamaindex), adds a Neo4j database and an LLM step at indexing time, so that retrieval can follow explicit relationships.
* Hybrid RAG, the version in this repository, runs both searches and merges them, and it adds the question variants and the keyword expansion on top. That means more moving parts and more LLM calls per question than the Vector version, and the benchmark shows what it costs per question.
* Whether Graph or Hybrid RAG answers better than Vector RAG was not measured, so the choice between them still needs a quality comparison.

---

## 💻 Installation & Reproduction Steps

### 📋 Prerequisites
* **Python 3.10 or newer** is required.
* **Accounts and keys** you will need: a Google AI Studio API key for Gemini, a [Chroma Cloud](https://www.trychroma.com/) API key, an [Arize Phoenix](https://phoenix.arize.com/) API key, a PostgreSQL database for the conversation memory, and a Neo4j database for the graph. A hosted PostgreSQL database, for example one on Supabase, works with its connection string. The script also requires `GROQ_API_KEY` to be set, because its startup check looks for it. See the note under the variable table below.
* **A Neo4j database, version 5.23 or newer, with the APOC plugin enabled.** The graph store reads the schema through APOC procedures, and it creates its vector index only on Neo4j 5.23 or newer. The database can be hosted or local.
* **No GPU is needed.** `requirements.txt` installs the CPU-only build of `torch`, and the cross-encoder reranker runs on your machine on every turn.
* **Internet access**, because the script calls YouTube, Gemini, ChromaDB Cloud, Neo4j, Phoenix, and PostgreSQL. On the first run it also downloads the reranker weights from Hugging Face and the tokenizer file that `tiktoken` needs, so the first start takes longer than later ones.

### 🛠️ CLI Installation & Execution

#### 1. Clone the Repository
```bash
git clone https://github.com/viochris/youtube-hybrid-rag-llamaindex.git
cd youtube-hybrid-rag-llamaindex
```

#### 2. Create and Activate a Virtual Environment
A virtual environment keeps this project's packages separate from the rest of your system.
```bash
# macOS/Linux
python3 -m venv venv
source venv/bin/activate

# Windows
python -m venv venv
venv\Scripts\activate
```

#### 3. Install Dependencies
```bash
pip install --upgrade pip
pip install -r requirements.txt
```
> `requirements.txt` asks pip for the CPU-only build of `torch` (pinned to version 2.14.1) from the PyTorch CPU index, so the large NVIDIA CUDA wheels are not downloaded. The first installation can still take a while, because the LlamaIndex ecosystem ships as many small, fast-moving sub-packages. If you hit a resolver conflict, try installing into a clean virtual environment and letting `pip` resolve every package from `requirements.txt` in a single command rather than one at a time, since that gives it the full picture needed to pick mutually compatible versions.
>
> To check which build of `torch` was installed, run `pip show torch` and look at the `Version` line. A CPU build ends with `+cpu`. If you want a GPU build instead, remove the `--extra-index-url` line from `requirements.txt`, and pip will take `torch` from PyPI.

#### 4. Configure Environment Variables
Create a `.env` file in the project root.
```env
GOOGLE_API_KEY=your_google_ai_studio_key
GROQ_API_KEY=your_groq_key
CHROMA_API_KEY=your_chroma_cloud_key
PHOENIX_API_KEY=your_phoenix_key
PHOENIX_COLLECTOR_ENDPOINT=https://app.phoenix.arize.com/s/your-space/v1/traces
NEO4J_URI=neo4j+s://your-instance.databases.neo4j.io
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_neo4j_password
NEO4J_DATABASE=neo4j
ASYNCPG_DATABASE_URL=postgresql+asyncpg://user:password@host:5432/dbname
```

| Variable | Purpose |
| :--- | :--- |
| `GOOGLE_API_KEY` | Gemini key. The embedding model receives it directly, and LiteLLM finds its Gemini key in the environment for the LLM calls |
| `GROQ_API_KEY` | Only checked at startup. The Groq model appears only in the commented-out fallback block, so no active call uses it |
| `CHROMA_API_KEY` | Connects to ChromaDB Cloud, where the vectors are stored |
| `PHOENIX_API_KEY` | Authenticates the trace upload to Phoenix |
| `PHOENIX_COLLECTOR_ENDPOINT` | Optional. Where traces are sent |
| `NEO4J_URI` | Connection address of the Neo4j database, for example a `neo4j+s://...` address of a hosted instance or `bolt://localhost:7687` for a local one |
| `NEO4J_USERNAME` | Neo4j user name |
| `NEO4J_PASSWORD` | Neo4j password |
| `NEO4J_DATABASE` | Name of the Neo4j database to use, usually `neo4j` |
| `ASYNCPG_DATABASE_URL` | Async connection string of the PostgreSQL database that holds the chat memory table |

> The script has a default value for `PHOENIX_COLLECTOR_ENDPOINT` that points to the author's own Phoenix space. Set this variable to your own space, otherwise your traces will not appear in your account.
>
> Only `GOOGLE_API_KEY`, `GROQ_API_KEY`, and `PHOENIX_API_KEY` are checked at startup, and the check only looks for a non-empty value. The Chroma, Neo4j, and PostgreSQL variables are not checked, so a mistake there only shows up when that client is first used. Keep `.env` out of version control.

#### 5. Set the Target Videos
The video URLs are set directly in the script.
```python
youtube_urls = ["https://www.youtube.com/watch?v=i3OYlaoj-BM"]
```
Edit this list before the first run. The video needs captions, and the transcript is requested in the default language, which is English.

#### 6. Run the Application
```bash
python Hybrid-self-make-youtube-reader-cmd.py
```
Type your questions at the `🗣️ [USER] You:` prompt. Type `exit`, `close`, `out`, `break`, `quit`, or `x` to end the session. On the first run, both stores are built before the chat starts. This takes a while, because the chunks are embedded and the LLM is called once for every chunk to extract relationships.

#### 7. Look at the Graph (Optional)
After the first run, you can see part of the graph in the Neo4j Browser, or in the query tool of your Neo4j provider, with this Cypher query:
```cypher
MATCH (n)-[r]->(m) RETURN n, r, m LIMIT 25
```

#### 8. Switch to Another Video or Start Over
Each store is only built when it is empty, so changing `youtube_urls` has no effect while a store still holds data. To switch videos, reset **both** stores. If you reset only one, that side is rebuilt from the new list while the other side keeps describing the old video. The script below deletes the Chroma collection and empties the Neo4j database, and the next run builds both from the new list. The chat history stays in PostgreSQL, so also change `session_id` in the script, or clear the `youtube_hybrid_memory` table, if you do not want the old conversation to carry over.
```python
import os
import chromadb
from dotenv import load_dotenv
from neo4j import GraphDatabase

load_dotenv()

# 1. Delete the vector collection
chroma_client = chromadb.CloudClient(api_key=os.getenv("CHROMA_API_KEY"))
chroma_client.delete_collection("youtube_hybrid_docs")

# 2. Empty the graph database
with GraphDatabase.driver(
    os.getenv("NEO4J_URI"),
    auth=(os.getenv("NEO4J_USERNAME"), os.getenv("NEO4J_PASSWORD")),
) as driver:
    driver.execute_query("MATCH (n) DETACH DELETE n", database_=os.getenv("NEO4J_DATABASE"))
```
> The Neo4j part deletes **every** node and relationship in that database, including data from any other project that uses the same database. The same Cypher query also works in the Neo4j Browser.

---

## 🛠️ Troubleshooting & Error Messages

### 🚨 Error Categories Printed by the Script

When something fails after the startup key check, the script prints the traceback and then one of these summaries, and exits with code 1. The checks run from top to bottom, and the first match is used. A missing `GOOGLE_API_KEY`, `GROQ_API_KEY`, or `PHOENIX_API_KEY` is raised before this handler starts, so it appears as a normal Python error with its message and no summary.

| Message prefix | Matches when |
| :--- | :--- |
| `🚨 [API ERROR]` | The error type is `ResourceExhausted`, or the message contains `quota` or `429` |
| `🚨 [DATA ERROR]` | A `ValueError` whose message contains `empty` |
| `🚨 [AUTH ERROR]` | The error type is `InvalidArgument`, or the message contains `api_key` |
| `🚨 [DATABASE ERROR]` (ChromaDB) | The error type is `OperationalError`, `DatabaseError`, or `InvalidCollectionException`, or the message contains `chroma` |
| `🚨 [DATABASE ERROR]` (Neo4j) | The message contains `neo4j`, `serviceunavailable`, or `auth` |
| `🚨 [DATABASE ERROR]` (PostgreSQL) | The message contains `asyncpg` or `postgres` |
| `🚨 [ACCESS ERROR]` | A `PermissionError` |
| `🚨 [UNKNOWN ERROR]` | Anything that does not match the categories above |

The database checks match on the type or the text of the message, in the order of the table. A message that contains `chroma` is reported as a ChromaDB error even when the cause is elsewhere. The Neo4j check runs before the PostgreSQL check, so a PostgreSQL message that contains `auth`, such as a failed password authentication, is reported as a Neo4j error. A Neo4j connection error whose message has none of the three words is reported as `UNKNOWN ERROR`. In all these cases, the printed traceback shows the real cause.

### 🧰 Common Problems

| Symptom | Likely cause | What to do |
| :--- | :--- | :--- |
| `GOOGLE_API_KEY is missing`, `GROQ_API_KEY is missing`, or `PHOENIX_API_KEY is missing` at startup | The variable is missing from `.env`, or the file is not in the project root | Add the variable to `.env` in the project root and run the script again |
| LiteLLM reports a missing Gemini key | LiteLLM did not find the key in the environment | Also set `GEMINI_API_KEY` in `.env` with the same value as `GOOGLE_API_KEY` |
| `UNKNOWN ERROR` mentioning a transcript | The video has no captions, captions are disabled, or no English transcript exists | Choose a video that has English captions |
| `UNKNOWN ERROR` mentioning an invalid YouTube URL | The URL does not contain `youtube` or `youtu.be`, and has no `v=` parameter | Use a standard `youtube.com` or `youtu.be` link |
| YouTube refuses the request | YouTube can block requests from some cloud provider and VPN IP addresses | Run the script from a regular home or office connection |
| The answers come from an old video, or the two sides seem to disagree | One store still holds the old video while the other was rebuilt from the new list, or both still hold the old video, so the new URLs are ignored | Reset both stores as shown in step 8 |
| The script says a store has data, but you never built it, or the answers lack context | The Neo4j database already has nodes from another project, so the graph build is skipped | Empty the database as shown in step 8, or use a separate Neo4j database for this project |
| The first run stopped halfway, and the next run does not build everything | Each store is checked on its own, so a store that was filled before the failure is loaded and not rebuilt. A graph that was partly written also counts as built, because the check only counts nodes | Check what each store holds before running again, and reset the incomplete one. If you reset only one store, the two sides can end up describing different videos, so resetting both as shown in step 8 is the safe choice |
| `API ERROR` (quota or `429`) during the first run | The graph build makes one LLM call per chunk, with 5 workers in parallel, which can reach the rate limit of your Gemini plan | Wait for the limit to reset and run again, or check the plan's limits |
| `API ERROR` (quota or `429`) while chatting | Every question uses several LLM calls, about five with the current settings, so the limit is reached sooner than with a single call per question | Wait for the limit to reset. Lowering `num_queries` in the script reduces the calls, and `num_queries=1` turns the question variants off |
| A Neo4j error mentioning `apoc` or `db.index.vector` | The database does not have the APOC plugin, or it is older than 5.23, so the vector index is not created | Use Neo4j 5.23 or newer with APOC enabled |
| Neo4j authentication or connection error | `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, or `NEO4J_DATABASE` is wrong, or the database is not running | Check the values in `.env` against the connection details of your Neo4j instance |
| `UNKNOWN ERROR` (`TypeError`) saying `CloudClient()` is missing `tenant` and `database` | An old `chromadb` version is installed, and it requires both arguments | Run `pip install --upgrade "chromadb>=1.1.0"` or reinstall from `requirements.txt` |
| `ImportError` asking for `sentence-transformers` or `torch` | The reranker dependencies are not installed | Run `pip install -r requirements.txt` in the active virtual environment |
| `pip` downloads very large NVIDIA CUDA packages | The `--extra-index-url` line was removed from `requirements.txt`, or pip could not reach the PyTorch CPU index and took `torch` from PyPI instead | Keep the line, check your connection, and confirm with `pip show torch` that the version ends with `+cpu` |
| The first run is slow before the chat starts | The reranker weights are downloaded from Hugging Face, the chunks are embedded, and the graph is built with LLM calls | Wait for all of it to finish. Later runs reuse the weights and the stored data |
| Traces do not show up in your Phoenix account | `PHOENIX_COLLECTOR_ENDPOINT` still points to the default space | Set it to your own Phoenix endpoint |
| Error about the async driver | `asyncpg` is not installed, or `ASYNCPG_DATABASE_URL` does not start with `postgresql+asyncpg://` | Run `pip install -r requirements.txt` and check the URL |
| The chat remembers an old conversation | The memory uses a fixed `session_id` | Change `session_id` in the script, or clear the `youtube_hybrid_memory` table |
| Long trace output appears after each answer | `LlamaDebugHandler` is created with `print_trace_on_end=True` | Set it to `False` in the script |
| A database error appears for a problem that is not about the database | Any message containing `chroma` is reported as a ChromaDB error, and any message containing `neo4j`, `serviceunavailable`, or `auth` as a Neo4j error. A PostgreSQL authentication failure can be one of them | Read the printed traceback to find the real cause |

---

## 📝 Conclusion

This project set out to solve four problems: a video that cannot be searched like text, and for which one way of searching may not be enough, a general chatbot that answers from memory instead of from the video, a naive pipeline that downloads and forgets everything on every run, and answers whose sources and cost stay hidden. The script addresses them directly. It fetches the transcript with its own loader, builds a vector store in ChromaDB Cloud and a knowledge graph in Neo4j from it, lets the LLM write extra variants of each question, searches both stores, merges the results with Reciprocal Rank Fusion, reranks what it keeps, tells the model to answer only from that context, and keeps the stores and the conversation so repeated use stays cheap and continuous. Every answer is printed with its source nodes, its token usage, and its estimated cost.

The design choices hold up reasonably well on their own terms. Checking each store separately avoids paying for the same download, embeddings, and relationship extraction twice. Merging the two searches by rank lets lists with different kinds of scores be combined. Reranking with a small cross-encoder before the LLM is a deliberate quality step, because it filters the merged results before they take up space in the prompt. Keeping the whole flow in one readable script makes it easy to follow, run, and modify.

It is also not a finished product. The video list is hardcoded, changing it requires resetting both stores by hand, the transcript carries no timestamps, the graph quality depends on what the LLM extracts, every question uses several LLM calls, and nothing measures how good the answers are. The sections below go through these gaps and lay out the next steps.

---

## ⚠️ System Limitations

### 🏗️ Architectural Limitations
* **Hardcoded target videos.** The URLs are set by editing the `youtube_urls` list in the script, not passed in as an argument, so switching videos means opening the code.
* **Two separate build checks.** The vector side is built only when the Chroma collection is empty, and the graph side only when the Neo4j database has no nodes at all. Changing the list without resetting the stores does not trigger a new build. If only one store is empty, only that side is built from the current list while the other keeps its old data, so the two sides can describe different videos, without any warning.
* **The graph check counts every node.** The check looks at all the nodes in the Neo4j database, not only the ones from this script. A database that holds nodes from another project makes the script skip the graph build and answer from the wrong graph.
* **One shared collection, one shared graph, and one fixed conversation.** Every video goes into the same collection and the same graph, and the memory uses one `session_id` and one table name. Every run continues the same conversation, even after you switch videos, so old topics can mix into new answers.
* **Any error ends the session.** After the startup key check, the whole pipeline, including the chat loop, sits inside one top-level `try` block. A transient failure on a single turn, such as an API rate limit, is reported but still stops the script, and you have to start it again.
* **Settings are hardcoded.** The collection name, model names, chunk settings, extractor limits, retrieval depth, number of question variants, fusion mode, Phoenix project name, and memory settings are fixed values in the script.
* **Startup validation does not match the real needs.** The script requires `GROQ_API_KEY` even though no active code uses it, because the fallback that would use it is commented out. The Chroma, Neo4j, and PostgreSQL variables, which the script does need, are not checked, so a mistake there only shows up when that client is first used.
* **Model fallback is disabled.** The fallback and retry options are commented out, so a failed model call is not retried and is not sent to another model. This also applies to the calls made while the graph is built and to the extra calls made for every question.
* **Several LLM calls for every question.** The LLM writes the question variants, and the graph retriever asks it for related keywords once for each query, before the answer is written. With the current settings, that is about five LLM calls for one question, which is more than a search that only compares embeddings would use.
* **Dependence on external services.** The script needs live access to YouTube, Gemini, ChromaDB Cloud, Neo4j, Phoenix, and PostgreSQL. There is no offline mode.
* **Specific Neo4j requirements.** The graph store needs the APOC plugin and, to create its vector index, Neo4j 5.23 or newer, so not every Neo4j instance will work.
* **The cost of building the stores is not reported.** The LLM calls that extract the relationships are made before the chat starts, and the token and cost counters are reset when the session starts. The printed numbers therefore describe the chat turns only.
* **CLI-only interface.** All interaction happens in a terminal. There is no web page, no visual chat history, and no built-in view of the graph.

### 🔬 Model & Domain Limitations
* **Captions only.** A video without captions cannot be read, and automatic captions can contain mistakes that end up in the vectors, in the graph, and in the answers. The transcript is requested in the default language, which is English.
* **No timestamps.** The loader joins the caption snippets into plain text, so answers cannot point to a moment in the video.
* **Simple URL parsing.** The video id is taken from the URL with string rules, so unusual link formats may fail or give a wrong id.
* **The graph is only as good as the extraction.** An LLM reads each chunk and writes up to 15 relationships with a simple prompt. The script defines no entity or relationship types, so the graph holds whatever the LLM wrote, it can miss facts, and the same idea can appear under different names. If the extractor cannot read the LLM's reply for a chunk, LlamaIndex's default is to log the error and continue, so that chunk adds no relationships while the script carries on.
* **The question variants are not shown.** The LLM writes the variants, and the script does not print them. A variant can shift the meaning of the question and bring in less relevant passages, and the reranker is the step that filters them.
* **Fusion looks at ranks only.** Reciprocal Rank Fusion uses the position of each item in each list and ignores how strong the match was. Items are treated as the same only when their content is identical, so a chunk that both sides find can appear twice when the graph side adds its relationships to the chunk text.
* **Retrieval on every message.** The context chat mode retrieves for every message, including greetings and thanks, and each retrieval runs the question variants, so even a short message can use several LLM calls.
* **Grounding relies on the prompt.** Staying within the retrieved context is enforced by the system prompt and the retrieval setup, not by a hard check on the output, so the model can still go beyond the retrieved passages.
* **Fixed retrieval settings.** Chunk size, overlap, `similarity_top_k`, the number of question variants, and the reranker's `top_n` are fixed. A broad question that needs many parts of a long video is limited by only three source nodes reaching the LLM.
* **Local reranking.** The cross-encoder runs on your machine, so it needs `torch`, a one-time weight download, and some compute on every turn.
* **Token counts are approximate.** `TokenCountingHandler` uses the `cl100k_base` tokenizer from OpenAI, while the model is Gemini, so the numbers can differ from what the provider counts.
* **The cost covers LLM calls only.** Embedding calls are not included, and the estimate depends on the response data that LiteLLM receives.
* **No quality evaluation.** Retrieval and answer quality were not measured against a labeled question set, so there is no number for how often the script retrieves the right passage or answers correctly, and no comparison with the Vector and Graph versions.

---

## 🚀 Future Work
* **Take the video URLs as an argument.** Accept one or more URLs on the command line, so the code does not need editing to switch videos.
* **One collection, one graph partition, and one session per video.** Name the collection and the memory session after the video id, and keep each video's graph separate, for example in its own database or by filtering on the `video_id` property that the extractor copies from the document metadata, so switching videos needs no manual cleanup and old conversations do not mix in.
* **Keep the two stores in sync.** Record which videos each store holds, and rebuild a store when it does not match the current list, instead of only checking whether it is empty.
* **Check for this project's data only.** Decide whether to build the graph by looking for the nodes this script created, instead of counting every node in the database.
* **Report the cost of building the stores.** Keep the token and cost numbers of the extraction step, so the full cost of a video is visible and not only the cost of the chat turns.
* **Show the question variants.** Print the variants that the LLM writes, for example by turning on the `verbose` option of the fusion retriever, so the effect of each variant on the answer can be inspected.
* **Keep timestamps.** Store the start time of each caption in the metadata, so answers can point to the moment in the video.
* **Support more transcript languages.** Let the user choose the language, or fall back to another one when English is missing.
* **Make the chat loop resilient.** Move error handling inside the loop, so a failed turn prints the error and lets you try again instead of ending the session.
* **Enable model fallback.** Turn on the retry and fallback options through the LiteLLM configuration, so a temporary model failure does not stop a turn, and remove the `GROQ_API_KEY` check if no Groq model is used.
* **Guide the extraction with a schema.** Define the entity and relationship types the graph should use, so the extraction is more consistent across chunks.
* **Make the fusion settings configurable.** Take the number of question variants, the fusion mode, and the retrieval depth from environment variables or arguments, and compare different values.
* **Move configuration out of the code.** Take model names, chunk settings, and retrieval depth from environment variables or arguments.
* **Evaluate retrieval and answers.** Build a small labeled question set for a few videos and measure retrieval and answer quality, and compare the Hybrid version with the Vector and Graph versions, instead of relying on fixed defaults.
* **Add a simple interface.** A small Streamlit page would make the answers and their sources easier to read than terminal output.

---

## 📄 License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---
**Author:** [Silvio Christian, Joe](https://github.com/viochris)

*"Ask a video a question, and let its transcript and its knowledge graph answer together."*
