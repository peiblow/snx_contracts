name: morning_review
description: Proactive daily digest of the user's recent and pending decisions

---

# Morning review

This skill runs proactively from the `cron` trigger (08:00, America/Sao_Paulo).
The user did not ask a question — you are reaching out first, so be brief and
useful, not noisy.

1. Read the vault with `read_decisions` for decisions logged in the last day
   and for any note marked as pending or awaiting follow-up.
2. If there is nothing new and nothing pending, stay silent. A digest with no
   signal is noise; do not message just to prove you ran.
3. When there is signal, compose one short message: what was decided yesterday,
   what is still open, and at most one suggested next step grounded in the
   actual notes.
4. Deliver it with `send_wpp_message`. Do not create or modify any decision
   during a review — this skill only reads and reports.
