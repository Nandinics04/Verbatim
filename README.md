# Verbatim

Search over a written policy and a meeting, with a citation for every claim.

A teammate asks: "Can we refund $400 on an enterprise R-14 ticket, and who owns the follow-up?" The wiki only says a refund over $200 needs a manager. The exception was spoken in a meeting. Verbatim has to return both, name who owns the follow-up, and refuse a code that never appears.

## Scenario

Northwind Support, meeting `2026-03-12-refund-review`.

| When | Who | What |
| --- | --- | --- |
| 00:03:12 | Nandini | An enterprise ticket with code R-14 can be refunded up to $500 without a manager. |
| | Omar | Finance agrees with exception. |
| | Mark | Who owns the follow-up? |
| | Nandini | Sam is the owner of the follow-up. |

The wiki is `data/scenario/refund-policy.md`. The meeting is `data/scenario/meeting.json`. Records point at real turn ids. A record that cites a missing turn is rejected.

## What works now

The walkthrough is `notebook/01_scenario.ipynb`.

- Meeting turns and wiki sections become chunks. A turn chunk keeps the speaker and timestamp on the sentence. Topic windows are context, not the unit that gets searched.
- BM25 keyword search ranks turn, record, and section chunks.
- The query `enterprise R-14 refund owner` returns the wiki section, the decision on `t1`, Nandini's turn, the action record, and turn `t4`.
- The query `Z-99` returns no hits.

`refund` and `refunded` are different tokens, so the wiki outranks the spoken exception on the word "refund". The meeting still matches through `enterprise` and `R-14`.

## Run

Requires [uv](https://docs.astral.sh/uv/) and Python 3.13.

```powershell
uv sync
```

Open `notebook/01_scenario.ipynb` and choose the project kernel `.venv`. Run the cells from the top.

## Layout

```text
data/scenario/     wiki page and hand-written meeting
notebook/          step-by-step walkthrough
pyproject.toml     dependencies managed by uv
```

## Still ahead

Citations and abstention, vector search with reciprocal rank fusion, reranking, and a LangGraph search flow traced in LangSmith. Speech comes after that: faster-whisper for the transcript and pyannote for speakers. The searchable record stays the one this notebook already builds.
