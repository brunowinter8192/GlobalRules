# RAG-Specific Standards

**Model Selection:** Bei Modell-Auswahl (Embedding, Reranker, etc.): IMMER aktuellen Benchmark prüfen (MTEB, etc.) bevor ein Modell in den Plan kommt. Nie aus altem Bead-Research übernehmen ohne Aktualitätscheck.

**GGUF-Dateinamen:** Nach Model-Download `ls models/` prüfen und Pfade im Code anpassen. HuggingFace-Dateinamen können von Erwartung abweichen (CamelCase vs lowercase).

**Schema Extension = Backfill First:** When adding a new column to existing data (e.g., `sparse_embedding`), write a backfill script BEFORE starting full re-index. Full re-index only when content changed. Backfill updates only the new column, skipping expensive recomputation of existing embeddings.

**Load-Test Before Default Change:** When changing defaults that affect system load (batch sizes, model params, candidate counts): test with production-scale data BEFORE changing the default. Example: setting rerank=True without testing that 50 candidates fit in llama-server context.
