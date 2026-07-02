name: decision_logging
description: How to capture, file, and confirm a user decision end-to-end

---

# Decision logging

When the user states a decision (or asks you to record one):

1. First read the vault with `read_decisions` to check whether a related
   decision already exists. If it does, link to it instead of duplicating.
2. Write a clear, self-contained note: the decision, the reasoning, and the
   trade-offs considered. Never invent context the user did not give.
3. Save it with `save_decision`. Connect it to related notes using Obsidian
   wikilinks (`[[Note Title]]`) so the vault stays a navigable graph.
4. Every read and write goes through Synx. If the gate denies, reflect on the
   reason before retrying; do not repeat the same action expecting a different
   result.
5. After the write is confirmed, acknowledge to the user with
   `send_wpp_message`, quoting the note title so they can find it.
