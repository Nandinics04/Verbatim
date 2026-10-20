# Northwind search eval

Ten questions over the hand-written meeting and the refund wiki. Recall@5 is the fraction of gold chunks in the top 5. Abstention means the retrieved list was empty. The reranked list holds at most 4 chunks, so that column is the fraction of gold chunks still kept.

The main question is a $400 enterprise R-14 refund and who owns the follow-up. Gold chunks are Nandini's R-14 turn, the wiki section Standard limit, and the turn that names Sam. BM25 recall@5 is 0.67. Fused recall@5 is 0.67. The reranked list also keeps 0.67. Fusion moves the first gold chunk from rank 3 to rank 2 and does not recover the missing chunk.

The other answered questions score 1.0 for BM25 and for fusion. "Who owns the follow-up?" finds Sam's turn at rank 3 in both lists. "Can we refund $50 on a normal ticket?" puts the wiki first for BM25 and second after fusion. Two questions were mistyped (`$4oo`, `supervsior`), so those rows are about the text that was actually searched.

All three unanswerable questions fail on both systems: a $400 refund on ticket Z-99, the weather in Jabalpur, and a $9000 enterprise refund. The list is not empty, because the questions share words with the corpus (`refund`, `ticket`, `the`). The bare query `Z-99` still returns nothing. Empty-list abstention only works when the question shares no words with the sources.

I keep hybrid search plus rerank for questions the corpus can answer, because it improved the rank of the first gold chunk on the R-14 question. It did not improve recall@5. Neither system abstains once an unanswerable question shares ordinary words with the documents. That is the failure this fixture still has.

Speech, measured separately: faster-whisper `base` on `Recording.wav` has word error rate 20% against the hand-written transcript. pyannote labels every turn `SPEAKER_00` because the clip is one voice.