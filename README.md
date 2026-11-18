# Medium Search Engine with Pinecone and Airflow

DATA 226 homework (week 13). The Airflow DAG `Medium_to_Pinecone` builds a semantic search engine over Medium articles:

```
download_data ──► preprocess_data ──┐
                                    ├──► generate_embeddings_and_upsert ──► test_search_query
create_pinecone_index ──────────────┘
```

1. **download_data:** downloads `medium_data.csv` (2,498 articles) from S3.
2. **preprocess_data:** builds `title + " " + subtitle` metadata and a string `id` for each article.
3. **create_pinecone_index:** (re)creates the serverless index `semantic-search-fast` (384 dimensions, `dotproduct`, AWS `us-east-1`).
4. **generate_embeddings_and_upsert:** embeds each article with `all-MiniLM-L6-v2` and upserts it in batches of 100.
5. **test_search_query:** embeds *"what is ethics in AI"* and prints the top 5 matches.

The full write-up with screenshots is in [`Rahil_Shah_020755137_Pinecone_Report.pdf`](Rahil_Shah_020755137_Pinecone_Report.pdf).

## Changes from the class files

The first commit is the unmodified class `docker-compose.yaml` and `week13/build_pinecone_search.py`, so each change is its own commit:

| File | Change | Why |
|---|---|---|
| `docker-compose.yaml` | Added `sentence-transformers pinecone numpy==1.26.4 --extra-index-url https://download.pytorch.org/whl/cpu` to `_PIP_ADDITIONAL_REQUIREMENTS` | The DAG needs both libraries. The numpy pin keeps the image's pandas 2.1.4 working. The CPU index avoids the CUDA build of torch. |
| `dags/build_pinecone_search.py` line 168 | `index.upsert(upsert_data)` → `index.upsert(vectors=upsert_data)` | In pinecone 10.x, `Index.upsert` takes keyword arguments only. |

## Run it

```bash
echo "AIRFLOW_UID=50000" > .env      # optional, silences the AIRFLOW_UID warning
docker compose up -d                 # first start pip-installs the packages (a few minutes)
docker compose ps                    # wait for airflow to show (healthy)
```

1. Open http://localhost:8081 and log in as `airflow` / `airflow`.
2. **Admin → Variables → +**: key `pinecone_api_key`, value = your Pinecone API key. Never commit the key.
3. Turn the `Medium_to_Pinecone` toggle on. This starts one scheduled run by itself, so don't also press Trigger.
4. When all 5 tasks are green, open `test_search_query` → **Logs** for the search results. Turn the toggle off afterwards.

## Result

All 5 tasks succeeded in 2 min 48 s. Top 5 results for *"what is ethics in AI"*:

| Rank | ID | Score | Title + subtitle (first 50 characters) |
|---|---|---|---|
| 1 | 1 | 0.741106 | Ethics in AI: Potential Root Causes for Biased Alg… |
| 2 | 1634 | 0.740668 | Ethics in AI: Potential Root Causes for Biased Alg… |
| 3 | 1326 | 0.717081 | The ethical implications of AI in design It's time… |
| 4 | 661 | 0.666769 | Ethical Considerations In Machine Learning Project… |
| 5 | 1627 | 0.651915 | Navigating the Ethical Contours of AI Copy Generat… |

The model outputs unit-length vectors, so these `dotproduct` scores are cosine similarities. IDs 1 and 1634 are the same article, which appears twice in the dataset.

## Stack

Airflow 2.10.1 (`apache/airflow:2.10.1`, Python 3.12), Postgres 13, pinecone 10.3.0, sentence-transformers 6.3.0, torch 2.15.0+cpu, numpy 1.26.4, pandas 2.1.4.
