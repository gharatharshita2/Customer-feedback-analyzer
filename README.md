# Customer Feedback Insights Analyzer

**StarvoxLabs · AI/ML Intern technical assessment · Streamlit MVP**

A small deployable application that reads **customer reviews and customer-authored support messages** from CSV exports, cleans them, classifies topic and sentiment, flags urgent feedback, groups uncertain topics, and presents a weekly complaint summary with clickable **original review/message IDs and quotes**. It compares a traditional text classifier to a semantic embedding-based classifier using the *same* independently reviewed test set.

For a one-link deployment, the owner configures a cloud LLM API key privately; reviewers need no installations or keys. The bundled demo contains **720 synthetic reviews and 48 distinct customer support messages**, plus example agent replies and one duplicate to test cleaning. Use **public or synthetic data only**, never confidential real customer messages.

## 1. Start the dashboard (Windows PowerShell)

**Extract the ZIP first.** Open a PowerShell terminal in the extracted folder **that contains `app.py` and `requirements.txt`**. To verify:

```powershell
dir app.py, requirements.txt
```

Then:

```powershell
python -m pip install -r requirements.txt
python -m streamlit run app.py
```

Open `http://localhost:8501` if it does not open automatically. The dashboard automatically loads both bundled demo sources. The default **TF-IDF baseline** works without an external API or LLM download.

For an isolated environment, optionally create a venv first: `python -m venv .venv`, then `.\.venv\Scripts\Activate.ps1` (PowerShell policy permitting). Python **3.11 or 3.12** is a conservative choice for the optional deep-learning packages. The code has also passed headless Python 3.13 tests.

## 2. Upload customer reviews and support messages

The sidebar has **one `Upload Feedback File` control**. Upload any one CSV containing:

- **Customer reviews:** a `text`, `review`, `review_text`, `feedback`, `content`, or `body` column; optional `id` and `date`.
- **Customer support messages:** a `message`, `message_body`, `message_text`, `body`, `content`, or `text` column. Optional `message_id`, `ticket_id`, `sender_role`, `channel`, `created_at` and related export fields help preserve message context.
- **Both in the same CSV:** include a `source_type` column with `Review` or `Support message` on each row, plus a common `text` column (see `data/sample_feedback_upload.csv`). You may use `feedback_type` or `record_type` instead of `source_type`.

The application detects review vs support-message formats automatically when there is no `source_type` field. Once a CSV is uploaded, the bundled demo is replaced by that file. With no upload, both synthetic demo datasets are loaded automatically. This single upload control does not require changing models or filters.

Sample review CSV:

```csv
id,date,review
R-1,2026-10-12,"The product is excellent, but the parcel arrived late."
R-2,2026-10-12,"The delivery was quick and the product is great."
```

Sample support-message CSV:

```csv
message_id,ticket_id,created_at,sender_role,channel,message
M-1,T-102,2026-10-12 10:30,customer,email,"My refund has not arrived."
M-2,T-102,2026-10-12 10:45,agent,email,"We are checking your refund."
M-3,T-103,2026-10-13 11:00,customer,live_chat,"The charger overheated and could cause injury."
```

Agent, bot and system responses are **excluded** when marked in the `sender_role` field; only customer-authored messages contribute to feedback counts. Unknown/missing sender roles are included (export customer messages where possible). Repeated text across **different support tickets** is counted as multiple customer complaints; repeated text within the **same ticket** is deduplicated. Each prediction retains its source type, message ID, ticket ID and channel. You can filter by feedback source and open quoted evidence to inspect its original record.

CSV intake is designed for structured exports, **not** direct connections to private inboxes or customer support platforms. Do not upload confidential personal information; use synthetic or public data for this assessment.

## 3. Run the embedding approach

Install the optional pre-trained encoder:

```powershell
python -m pip install -r requirements-embeddings.txt
```

Switch the dashboard's **Classification method** to **MiniLM embeddings**. The first run downloads the free `sentence-transformers/all-MiniLM-L6-v2` encoder and caches it. This may use significant RAM and requires internet **on first use**. Both classifiers use logistic regression on top of their respective features.

| Approach | Text representation | Classifier |
| --- | --- | --- |
| Classic ML | TF-IDF word + bigram features | Logistic Regression |
| Embedding ML | 384-dimensional pretrained MiniLM sentence embeddings | Logistic Regression |

The baseline stays available when the embedding package is not installed in a local environment; the hosted requirements install both approaches for the reviewer.

## 4. Generate weekly LLM summaries (works on the hosted website)

The app now sends evidence-limited prompts to a **Groq-hosted LLM**, using the `openai/gpt-oss-20b` model, instead of asking reviewers to install Ollama. The app owner needs a Groq API key (a limited free tier is available); **reviewers require no key or installation**.

For the hosted app, store the key privately in **Streamlit Community Cloud > Advanced settings > Secrets**:

```toml
GROQ_API_KEY = "your_real_key_here"
```

For local development, either set `GROQ_API_KEY` as an environment variable or create a private `.streamlit/secrets.toml` (which is gitignored). Do **not** commit your key or paste it in the chat.

Inside **Weekly issues & quotes**, select a week and click **Generate LLM weekly summary**. Source IDs, the quoted messages, and computed topic counts are supplied to the LLM. Every proposed topic, evidence ID and claim is validated; unsupported claims automatically fall back to the safer extractive preview. Without an API key, the button is disabled and the preview is explicitly marked extractive. **Never call an extractive preview an LLM-generated summary.**

Only use synthetic or public feedback for the interview demo: selected customer quotes are sent to an external API. API free-tier quotas and model access can change.

## 5. CSV cleaning and business categories

The feedback uploader accepts common header variations (including spaced headers such as `Message Body`), and create a normalized `text` column. The app removes blank messages, cleans HTML/whitespace, handles repeated/missing IDs, and reports counts for duplicates, invalid rows, agent/bot replies excluded and missing dates. Missing dates are allowed (weekly charts require dates). Broken or empty CSVs produce a clear error. Source IDs are prefixed `REV:` or `SUP:` to keep citation links unambiguous across feedback types.

The topics are: **Delivery, Product Quality, Pricing, Refunds, Customer Support, Packaging, Website/App, Payments, Returns, Other**. Sentiments are **positive / neutral / negative**. Urgency uses transparent keywords for serious safety and legal concerns; these flags are **not** automated emergency or legal decisions.

**Mixed feedback:** A transparent clause-level rule shows both aspects when detected (e.g., positive quality and negative delivery). For complaint tracking, the negative aspect is assigned as the primary topic/sentiment, and the app shows the explanation. This rule is deliberately limited; it is not a general aspect-based neural model.

## 6. REQUIRED human evaluation (not optional for submission)

The file `data/evaluation_to_verify.csv` contains **120 distinct AI-drafted labels** with `human_verified=no`. The author of this package has **not** represented them as personally human-labelled. The candidate must review at least 100 rows and correct the topic/sentiment labels as needed; then mark `human_verified=yes` only for rows they personally checked. See `docs/LABELLING_GUIDE.md`.

After reviewing, run:

```powershell
python -m pip install -r requirements-embeddings.txt
python -m src.evaluate
```

The script refuses to report formal scores with fewer than 100 verified rows. It checks the evaluation schema, duplicate IDs, allowed labels and text overlap, and removes matching evaluation text from training. **Both classifiers are evaluated on the exact same verified review IDs.** It creates:

- `outputs/evaluation_results.json`: topic and sentiment accuracy / macro F1 and four confusion matrices (two methods × two tasks).
- `outputs/predictions_topic_tfidf.csv`, `outputs/predictions_topic_minilm.csv`, and equivalent sentiment files.
- `outputs/mistakes_to_analyse.csv`: real incorrect predictions with a blank field for your root-cause explanations. Explain at least **five** errors after validating the labels.

The **Model evaluation** tab reads the JSON and presents a score table and confusion matrices. No numerical accuracy/F1 is included here before actual human verification. `docs/MODEL_FAILURE_ANALYSIS.md` gives five **provisional examples** from draft-labelled data as a starting point; they are not final human-verified results.

## 7. Tests

```powershell
python -m pip install -r requirements-dev.txt
python -m pytest -q
```

Tests cover: cleaning, duplicates, urgent detection, invalid CSV, positive-not-a-complaint, mixed feedback, evidence grounding, forged LLM citations, model baseline, clustering, customer-only support intake, ticket-level deduplication, linked support quotes, and enforcement of human verification. LLM network behavior is tested with a mocked hosted API response (no claim that a live external model ran during automated tests).

## 8. Project structure

```text
app.py                          Streamlit dashboard
src/core.py                     CSV validation, cleaning, source IDs
src/intake.py                   Review/support CSV adapters and customer-role filter
src/aspects.py                  Mixed-review handling/explanation
src/models.py                   TF-IDF / MiniLM, scores, K-Means clusters
src/summaries.py                Hosted LLM + evidence validation/fallback
src/evaluate.py                 Leakage-safe held-out evaluation
 data/reviews.csv               720 synthetic reviews (demo/training)
 data/support_messages.csv      59 synthetic message rows (48 distinct customer messages, 10 agent replies, 1 duplicate)
 data/evaluation_to_verify.csv  120 independent draft-labelled reviews
 tests/                         Automated unit tests
 docs/LABELLING_GUIDE.md        Human annotation instructions
 docs/MODEL_FAILURE_ANALYSIS.md Provisional error examples
 docs/SHIP_DECISION.md          One-page shipping decision note
 docs/DEMO_SCRIPT.md            3–5-minute video walkthrough
 docs/SUBMISSION_CHECKLIST.md   Required final candidate actions
 docs/DEPLOYMENT.md             Hosted Streamlit Cloud deployment instructions
 requirements.txt               Core demo
 requirements-embeddings.txt    Optional MiniLM dependency
```

## 9. One-link hosted deployment (recommended)

Follow `docs/DEPLOYMENT.md` to publish the application on Streamlit Community Cloud. Add your Groq API key to **private app secrets**. Streamlit installs the Python dependencies and the MiniLM model downloads on the **server**, so reviewers can open one URL and use the feedback upload, dashboards, both classifiers, and hosted weekly LLM summaries without installing Ollama or Python.

The app still runs locally for developer testing. A cloud inference connection and a successful Streamlit deployment must be tested with your own account: this package does not contain a live URL or active API key.

## 10. Scope / limits / technical trade-offs

1. **Dataset:** synthetic reviews and support messages, not proof of real-world accuracy. The 720 training examples contain repeated writing patterns; 120 evaluation items are more varied and independent. An honest out-of-distribution comparison is more useful than a near-perfect training score.
2. **Model comparison:** same held-out human-verified IDs, two different text representations, same downstream logistic regression family. The classifier outputs are evaluated *before* mixed-review post-processing to keep the comparison fair.
3. **Grounding:** computed counts and direct source quotes are reliable; the LLM's descriptive headline is lexically constrained but still merits human review for subtle implication errors. Unaccepted claims fall back to extractive text.
4. **Emerging issues:** K-Means of 'Other'/uncertain records is only an exploratory grouping. It is not an automatically verified new taxonomy.
5. **Cloud demo:** reviewers use one link. The app owner supplies a hosted-LLM API key; rate limits and external-data transfer require care, and MiniLM uses server resources.

## 11. Submission & ownership

Submit a GitHub repository, working local instructions **or** hosted demo URL, a personally recorded 3–5-minute video, a verified evaluation set, scores with 4 confusion matrices, >=5 analysed mistakes, and the one-page shipping note. Review the checklist before sending the recruitment email.

**AI assistance disclosure:** Application code, synthetic text and draft labels were generated with AI assistance. The candidate should test and understand the implementation, correct/verify their annotations, record their own modifications and truthfully describe their individual contribution. Do not claim otherwise.
