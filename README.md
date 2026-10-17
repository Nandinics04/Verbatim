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

The wiki is `data/scenario/refund-policy.md`. The meeting is `data/scenario/meeting.json`. Records point at real turn ids. A record that cites a missing turn is rejected. The same meeting, read aloud, is `data/scenario/Recording.wav`.

## What works now

The walkthrough is `notebook/01_scenario.ipynb`.

- Meeting turns, decisions, actions, and wiki sections become chunks. A turn chunk keeps the speaker and timestamp on the sentence. Topic windows are context, not the unit that gets searched.
- BM25 ranks turn, record, and section chunks. The query `enterprise R-14 refund owner` returns the wiki section, the decision on `t1`, Nandini's turn, the action record, and turn `t4`. The query `Z-99` returns no hits. `refund` and `refunded` are different tokens, so the wiki outranks the spoken exception on the word "refund". The meeting still matches through `enterprise` and `R-14`.
- An answer is stitched only from retrieved chunks. Each sentence keeps a citation: who spoke, when, or which wiki section. If the evidence is missing, the answer abstains.
- The same chunks are embedded with `sentence-transformers/all-MiniLM-L6-v2`. Keyword rank and vector rank are merged with reciprocal rank fusion, then reranked with `BAAI/bge-reranker-base`.
- LangGraph runs that path as fuse, rerank, answer, then a citation check. The checker rejects an invented chunk id and a number or long token that is not in the cited text. The answer stays extractive unless `OPENAI_API_KEY` is set.
- faster-whisper (`base`, CPU, int8) turns `Recording.wav` into the same meeting shape. Word error rate against the hand-written transcript is 20% on this clip. Pauses longer than 700 ms start a new turn. Speakers from the transcript are `unknown`.
- pyannote `speaker-diarization-3.1` returns speaker segments. A turn takes the segment it overlaps most. A two-segment fixture assigns `SPEAKER_00` and `SPEAKER_01` correctly. The recording is one voice, so those labels are not Nandini, Omar, and Mark.

## Run

Requires [uv](https://docs.astral.sh/uv/) and Python 3.13.

```powershell
uv sync
```

Open `notebook/01_scenario.ipynb` and choose the project kernel `.venv`. Run the cells from the top.

Speech cells need extra setup. `HF_TOKEN` goes in `.env`, which is gitignored. Accept the pyannote model pages for `speaker-diarization-3.1`, `segmentation-3.0`, and `speaker-diarization-community-1` on the same Hugging Face account. PyAV is pinned to 18.1.0 so faster-whisper can open the wav. Diarization is given the waveform in memory, because torchcodec needs FFmpeg's shared DLLs on Windows.

## Layout

```text
data/scenario/     wiki, hand-written meeting, and Recording.wav
notebook/          step-by-step walkthrough
pyproject.toml     dependencies managed by uv
.env               Hugging Face token, not committed
```

## Still ahead

The citation check still sees every chunk, not only the reranked set. A structured summary still has to be written, with a test that an action's words appear in the turns it cites. After that: recall@5, citation accuracy, abstention, and latency for BM25 against hybrid search plus rerank; one AMI meeting and a small EnterpriseRAG-Bench slice; live audio only after those numbers exist. LangSmith tracing stays optional, through environment variables.
