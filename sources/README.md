# Sources

Raw sources (immutable) live in `raw/`. Their summary pages live directly in this folder, one per source, named `Source - YYYY-MM-DD - <Title>.md` per `[[CLAUDE]]` §3.1.

**To ingest a new source:** drop the file into `raw/` and ask the LLM agent to ingest. The agent will read it, discuss takeaways with you, write a summary page here, integrate facts into `wiki/`, update `[[index]]`, and append `[[log]]`.

Rule: never edit anything in `raw/`. Treat it like an archive.
