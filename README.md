\# AI-Powered Job Matching Pipeline



An n8n workflow that automates job discovery and fit-scoring for the Philippine job market: 

scrapes job boards via Apify, scores each posting against a resume using an LLM (1-100 fit score), 

filters to high-fit matches only, and extracts structured hiring signals (required skills, ATS 

keywords, seniority, years of experience, culture signals) for jobs that pass the threshold.



\## Architecture



1\. \*\*Fetch\*\* — Apify actor scrapes JobStreet Philippines by search query

2\. \*\*Trim \& Batch\*\* — job data condensed and batched into a single LLM call (not N sequential calls)

3\. \*\*Score\*\* — LLM scores all jobs in one batched call, 1-100 fit score

4\. \*\*Filter\*\* — jobs scoring ≥80 continue; others logged separately

5\. \*\*Deep Analysis\*\* — passing jobs get a second LLM pass extracting structured hiring signals

6\. \*\*Dedupe\*\* — checks against previously logged jobs by URL before reprocessing

7\. \*\*Output\*\* — results appended to Google Sheets (Matches / Skipped tabs)



\## Stack



n8n (self-hosted on AWS EC2), Docker, Apify, Google Gemini API, Google Sheets API (service account auth)



\## Key design decisions



\- \*\*Batched LLM calls instead of one-call-per-job\*\* — cut API requests by \~90%, avoiding free-tier 

&#x20; rate limits that sequential per-job scoring hit almost immediately.

\- \*\*Two-tier prompting\*\* — condensed resume summary for the wide first-pass scoring (keeps token 

&#x20; cost low across a large batch), full resume detail reserved for the narrower deep-analysis pass.

\- \*\*Defensive JSON parsing\*\* — handles malformed/truncated LLM output with a repair fallback rather 

&#x20; than failing the whole batch on one bad object.

\- \*\*Dedup by URL\*\* — checks existing Sheet rows before re-scoring previously seen postings.



\## Setup



1\. Import `workflow.json` into your n8n instance

2\. Configure credentials: Apify API token, LLM provider API key (Header Auth), Google Sheets 

&#x20;  (Service Account)

3\. Update the `Resume Text` node with your own resume summary/full text

4\. Adjust `search\_queries` to your target roles



\## Notes



This was built as a learning project to get hands-on with n8n, cloud infrastructure (AWS EC2), 

and LLM API integration patterns — including working through real free-tier rate-limiting behavior 

across multiple providers (Groq, Gemini, OpenRouter, DeepSeek) and designing around it.

