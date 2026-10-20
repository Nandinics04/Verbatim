# Verbatim

Search over a written policy and a meeting, with a citation for every claim.

A teammate asks: "Can we refund $400 on an enterprise R-14 ticket, and who owns the follow-up?" The wiki only says a refund over $200 needs a manager. The exception was spoken in a meeting. Verbatim has to return both, name who owns the follow-up, and refuse a code that never appears.

Audio, a human transcript, and a downloaded meeting transcript all become the same chunks. Search does not see the microphone.

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

The walkthrough is `notebook/01_scenario.ipynb`. Numbers are in `notes/eval.md`.

- Meeting turns, decisions, actions, and wiki sections become chunks. A turn chunk keeps the speaker and timestamp on the sentence. Topic windows are context, not the unit that gets searched.
- BM25 ranks turn, record, and section chunks. The query `enterprise R-14 refund owner` returns the wiki section, the decision on `t1`, Nandini's turn, the action record, and turn `t4`. The query `Z-99` returns no hits. `refund` and `refunded` are different tokens, so the wiki outranks the spoken exception on the word "refund". The meeting still matches through `enterprise` and `R-14`.
- An answer is stitched only from retrieved chunks. Each sentence keeps a citation: who spoke, when, or which wiki section. If the evidence is missing, the answer abstains. A decision or action is kept only when its numbers and long words appear in the turns it cites.
- The same chunks are embedded with `sentence-transformers/all-MiniLM-L6-v2`. Keyword rank and vector rank are merged with reciprocal rank fusion, then reranked with `BAAI/bge-reranker-base`.
- LangGraph runs that path as fuse, rerank, answer, then a citation check. The checker rejects an invented chunk id and a number or long token that is not in the cited text. The answer stays extractive unless `OPENAI_API_KEY` is set. The cited refund answer can be read aloud with `pyttsx3`.
- faster-whisper (`base`, CPU, int8) turns `Recording.wav` into the same meeting shape. Word error rate against the hand-written transcript is 20% on this clip. Pauses longer than 700 ms start a new turn. The file is 48 kHz. In 5-second pieces, resampled to 16 kHz, the first text returns in 1.39 seconds. A fixed window splits the R-14 sentence; the pause rule is what keeps the turns apart.
- pyannote `speaker-diarization-3.1` returns speaker segments. A turn takes the segment it overlaps most. The recording is one voice, so every turn is `SPEAKER_00`.
- AMI meeting `ES2004a` is loaded from the manual word files, not from audio. The human action uses `prepare` and `components`, which are not in the transcript, so that sentence is not stored as a record.
- One EnterpriseRAG-Bench question, `qst_0004`, is searched over its gold Fireflies transcript plus 200 other Redwood Inference documents. Document recall@5 is 1.0, and all five hits are sections of that same transcript. The summary states the recommendation. The GCP speaker's own recommendation is not in the top 5.

On the ten Northwind questions, BM25 and fusion both reach recall@5 of 0.67 on the refund question. Fusion moves the first gold chunk from rank 3 to rank 2. Neither list is empty for an unanswerable question that shares ordinary words with the corpus. The bare query `Z-99` still returns nothing.

## Run

Requires [uv](https://docs.astral.sh/uv/) and Python 3.13.

```powershell
uv sync
```

Open `notebook/01_scenario.ipynb` and choose the project kernel `.venv`. Run the cells from the top.

Speech cells need extra setup. `HF_TOKEN` goes in `.env`, which is gitignored. Accept the pyannote model pages for `speaker-diarization-3.1`, `segmentation-3.0`, and `speaker-diarization-community-1` on the same Hugging Face account. PyAV is pinned to 18.1.0 so faster-whisper can open the wav. Diarization is given the waveform in memory, because torchcodec needs FFmpeg's shared DLLs on Windows.

### AMI meeting annotations

`data/ami/` is gitignored. The notebook reads one meeting from the AMI manual annotations, not from the audio. The dataset is **AMI Corpus manual annotations v1.6.2** (CC BY 4.0), about 22 MB.

1. Download [ami_public_manual_1.6.2.zip](http://groups.inf.ed.ac.uk/ami/AMICorpusAnnotations/ami_public_manual_1.6.2.zip).
2. Unzip it so the word files are at `data/ami/ami_public_manual_1.6.2/words/`.
3. The notebook uses meeting `ES2004a` only: `ES2004a.A.words.xml` through `ES2004a.D.words.xml`, and the summary `data/ami/ami_public_manual_1.6.2/abstractive/ES2004a.abssumm.xml`.

Do not download the AMI headset recordings. The words and the abstractive summary are enough.

### EnterpriseRAG-Bench

Questions and documents come from Hugging Face dataset `onyx-dot-app/EnterpriseRAG-Bench` (MIT). The notebook loads the `questions` split, then streams the `documents` split and keeps the gold Fireflies document for `qst_0004` plus 200 other documents. The document file is about 1.3 GB and is cached by Hugging Face on first run. It does not load all 511,962 documents.

## Layout

```text
data/scenario/     wiki, hand-written meeting, and Recording.wav
data/ami/          AMI annotations, gitignored
notebook/          step-by-step walkthrough
notes/eval.md      recall, abstention, AMI, and the Redwood slice
pyproject.toml     dependencies managed by uv
.env               Hugging Face token, not committed
```

## Limits

The citation check can still see every chunk in the index, not only the reranked set. Abstention fails when an unanswerable question shares words such as `refund` or `the` with the sources. A long transcript can fill the top 5 with its own sections. LangSmith tracing stays optional, through environment variables.
