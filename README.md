# youtube-hybrid-rag-llamaindex
Hybrid RAG chatbot for YouTube videos using LlamaIndex. Fetches the transcript with a self-written loader, searches ChromaDB Cloud and a Neo4j graph, fuses results with Reciprocal Rank Fusion, reranks with a cross-encoder, and answers in a CLI chat with PostgreSQL memory. Includes Phoenix tracing and cost tracking. Hybrid version of a RAG series.
