# Unstructured Data ML Pipeline - Portfolio

This is a Python ML pipeline that transforms raw community interview transcripts (from Otter.ai) into structured, queryable analysis data. Built for People's Budget Birmingham, a civic engagement initiative studying whether city budgets are worth community organizing around. The pipeline processes ~3,400 utterances from 17 door-to-door interviews through sentiment analysis, emotion detection, unsupervised topic modeling, and keyword extraction, then uploads results to BigQuery for SQL-based research queries and semantic search.

---

## Context

This is an applied data science project built for a real civic engagement organization. The architecture reflects that context in several ways: the pipeline prioritizes **cost efficiency** (all local ML is free; the full BigQuery ML phase costs $3.74 total), **reproducibility** (any researcher can re-run the pipeline on new interview data), and **interpretability** (topic names are human-curated, not raw cluster labels). The 7-phase design lets collaborators enter at any phase - a community organizer can skip to the SQL analysis phase and query the data without understanding the ML internals.

---

## Skills Demonstrated

| Skill | Where It Appears |
|-------|-----------------|
| NLP / Transformer Models (HuggingFace) | `src/ml/sentiment.py` - Loads and runs RoBERTa and DistilRoBERTa for sentiment and emotion classification with GPU-aware device selection |
| Unsupervised Topic Modeling (BERTopic) | `src/ml/topics.py` - Discovers latent topics from interview text using sentence embeddings + clustering, with custom stop word filtering for conversational data |
| Text Parsing with Regex | `src/parsers/otter_parser.py` - Converts unstructured Otter.ai transcript files into structured speaker/timestamp/utterance records using regex pattern matching |
| Google Cloud / BigQuery Integration | `src/bigquery/schema.py`, `uploader.py`, `bqml/semantic_search.py` - Defines typed schemas, uploads DataFrames, and builds parameterized SQL for cosine-similarity search over embeddings |
| Data Pipeline Orchestration | `scripts/run_ml_analysis.py` - Coordinates four ML stages (sentiment, emotion, topics, keywords) with speaker-role filtering, word-count thresholds, and result merging |
| Keyword Extraction (YAKE + TF-IDF) | `src/ml/keywords.py` - Multi-method extractor with aggressive conversational-filler filtering, optimized for interview data rather than formal text |
| Semantic Search (Local + Cloud) | `scripts/local_semantic_search.py` - Offline vector search using cached Vertex AI embeddings with sentence-transformers query encoding and cosine distance ranking |
| Domain-Driven Data Curation | `scripts/topic_names.py` - Manual classification of 23 discovered topics into substantive, meta, and questionable categories with descriptions and exclusion lists |
| Configuration Management | `src/utils/config.py` - Centralized config using `python-dotenv` with path resolution, model name constants, and startup validation |

---

## Architectural Decisions

### 1. Dual-Model Sentiment Analysis for Ambiguity Detection

**Problem:** Single-model sentiment classifiers miss subtle criticism in conversational data. A resident saying "I guess the budget process is fine, I just don't really understand it" reads as neutral to most models, but contains implicit negative sentiment about transparency.

**Why this approach:** Rather than fine-tuning a single model (which requires labeled training data we don't have), running two off-the-shelf models in parallel and flagging disagreements surfaces ambiguous cases for human review - a pragmatic choice for a small-dataset civic project.

```python
# src/ml/sentiment.py:290-328
def _check_disagreement(self, result1: Dict, result2: Dict) -> tuple[bool, str]:
    label1 = result1['label'].lower()
    label2 = result2['label'].lower()

    # Case 1: Direct contradiction (positive vs negative)
    if (label1 == 'positive' and label2 == 'negative') or \
       (label1 == 'negative' and label2 == 'positive'):
        return True, f"Direct disagreement: {label1} vs {label2}"

    # Case 2: Model 1 says neutral, but Model 2 detects strong negative
    if label1 == 'neutral' and label2 == 'negative' and result2['score'] > 0.9:
        return True, f"Model 1 neutral, Model 2 strong negative"

    # Case 4: Both models' negative scores differ significantly
    if 'negative' in result1['all_scores'] and 'negative' in result2['all_scores']:
        neg1 = result1['all_scores']['negative']
        neg2 = result2['all_scores']['negative']
        if neg1 > 0.1 and neg2 > 0.1:
            neg_diff = abs(neg1 - neg2)
            if neg_diff > self.disagreement_threshold:
                return True, f"Negative scores differ: {neg1:.2f} vs {neg2:.2f}"

    return False, None
```

### 2. Speaker-Role Filtering with Word-Count Thresholds

**Problem:** Interview transcripts contain two types of noise for ML analysis: (1) interviewer questions, which reflect the researcher's framing, not community sentiment; and (2) short filler responses ("yeah", "okay", "uh huh") that dominate topic models with meaningless clusters.

**Why this approach:** A two-stage filter is simpler and more transparent than training a classifier to distinguish substantive from non-substantive speech. The 10-word threshold was tuned empirically by examining which utterances produced meaningful topic assignments.

```python
# scripts/run_ml_analysis.py:53-99
# Stage 1: Filter to interviewee utterances only
interviewee_df = df[df['speaker_role'] == 'interviewee'].copy()

# Stage 2: For topic modeling, further filter to substantive utterances
substantive_mask = interviewee_df['word_count'] >= 10
substantive_df = interviewee_df[substantive_mask].copy()
substantive_texts = substantive_df['text'].tolist()

# Sentiment and emotion run on ALL interviewee utterances (even short ones)
# Topic modeling runs ONLY on substantive utterances (10+ words)
topic_modeler = TopicModeler(min_topic_size=6)
topics, topic_probs = topic_modeler.fit(substantive_texts)
```

### 3. Hybrid Local/Cloud Architecture

**Problem:** Running ML inference and semantic search entirely in the cloud is expensive and requires constant connectivity. Running everything locally limits query flexibility and doesn't scale to collaborative research.

**Why this approach:** The pipeline splits work by cost: free local models handle the compute-heavy inference (sentiment, emotion, topics, keywords), while BigQuery handles storage, SQL aggregation, and embedding-based search. A local semantic search fallback (`local_semantic_search.py`) uses cached Vertex AI embeddings with sentence-transformers queries for offline exploration at ~70-85% of cloud quality.

```python
# scripts/local_semantic_search.py:71-81
def cosine_distance(vec1: np.ndarray, vec2: np.ndarray) -> float:
    """Calculate cosine distance (1 - cosine similarity)."""
    dot_product = np.dot(vec1, vec2)
    norm1 = np.linalg.norm(vec1)
    norm2 = np.linalg.norm(vec2)

    if norm1 == 0 or norm2 == 0:
        return 2.0  # Maximum distance

    cosine_sim = dot_product / (norm1 * norm2)
    return 1 - cosine_sim
```

---

## Code Snippets (Core Competencies)

### 1. Transcript Parser with Regex State Machine
**Skills:** regex pattern matching, state machine parsing, data normalization

Converts unstructured Otter.ai `.txt` files into structured records. The parser uses a regex to detect speaker header lines (which have a distinctive `Name  MM:SS` format with double-spacing), then accumulates text lines until the next speaker header. A nested `save_utterance` closure handles the emit-and-reset logic cleanly.

```python
# src/parsers/otter_parser.py:26-200
class OtterTranscriptParser:
    SPEAKER_PATTERN = re.compile(r'^(.+?)\s{2,}(\d{1,2}:\d{2}(?::\d{2})?).*$')

    def parse(self) -> List[Dict]:
        with open(self.file_path, 'r', encoding='utf-8') as f:
            lines = f.readlines()

        current_speaker = None
        current_timestamp = None
        current_text_lines = []

        def save_utterance():
            if current_speaker and current_text_lines:
                text = self.clean_text(' '.join(current_text_lines))
                if text:
                    self.utterances.append({
                        'utterance_id': self.generate_utterance_id(
                            current_speaker, current_timestamp, text
                        ),
                        'interview_id': self.filename,
                        'speaker_name': current_speaker,
                        'speaker_role': self.identify_speaker_role(current_speaker),
                        'timestamp': current_timestamp,
                        'timestamp_seconds': self.timestamp_to_seconds(current_timestamp),
                        'text': text,
                        'word_count': len(text.split()),
                        'char_count': len(text),
                    })

        for line in lines:
            line = line.rstrip()
            match = self.SPEAKER_PATTERN.match(line)
            if match:
                save_utterance()
                current_speaker = match.group(1).strip()
                current_timestamp = match.group(2).strip()
                current_text_lines = []
            elif line.strip() and current_speaker:
                current_text_lines.append(line.strip())

        save_utterance()
        return self.utterances
```

### 2. GPU-Aware Batch Inference with HuggingFace Transformers
**Skills:** PyTorch, HuggingFace transformers, hardware-aware ML inference, batch processing

Loads a pre-trained RoBERTa model and routes computation to the best available accelerator (Apple Silicon MPS, NVIDIA CUDA, or CPU). Processes texts in configurable batches with softmax normalization to produce per-class probability distributions, not just top-1 labels.

```python
# src/ml/sentiment.py:42-51, 119-176
# Device selection: Apple Silicon → NVIDIA → CPU
if torch.backends.mps.is_available():
    self.device = 'mps'
elif torch.cuda.is_available():
    self.device = 'cuda'
else:
    self.device = 'cpu'

self.model.to(self.device)
self.model.eval()

def analyze_batch(self, texts: List[str], batch_size: int = 32) -> List[Dict]:
    results = []
    for i in tqdm(range(0, len(texts), batch_size)):
        batch_texts = texts[i:i + batch_size]
        inputs = self.tokenizer(
            batch_texts,
            return_tensors='pt',
            truncation=True,
            max_length=512,
            padding=True
        )
        inputs = {k: v.to(self.device) for k, v in inputs.items()}

        with torch.no_grad():
            outputs = self.model(**inputs)
            predictions = torch.nn.functional.softmax(outputs.logits, dim=-1)

        scores_batch = predictions.cpu().numpy()
        for scores in scores_batch:
            predicted_class_id = np.argmax(scores)
            results.append({
                'label': self.id2label[predicted_class_id],
                'score': float(scores[predicted_class_id]),
                'all_scores': {self.id2label[i]: float(scores[i]) for i in range(len(scores))}
            })
    return results
```

### 3. BERTopic with Custom Conversational Stop Words
**Skills:** unsupervised learning, BERTopic, sentence embeddings, sklearn vectorization

Configures BERTopic for conversational interview data by extending sklearn's English stop words with domain-specific fillers ("yeah", "okay", "um") and high-frequency non-discriminative words. The `max_df=0.85` threshold removes words appearing in more than 85% of documents, preventing common terms from dominating cluster representations.

```python
# src/ml/topics.py:25-69
class TopicModeler:
    def __init__(self, n_gram_range: Tuple[int, int] = (1, 2), min_topic_size: int = 10):
        self.embedding_model = SentenceTransformer('all-MiniLM-L6-v2')

        custom_stop_words = [
            'yeah', 'okay', 'oh', 'um', 'uh', 'hey', 'like', 'just',
            'know', 'think', 'mean', 'guess', 'right', 'well',
            'don', 'didn', 'doesn', 'isn', 'wasn', 'weren',
            'thing', 'things', 'stuff', 'lot', 'lots', 'kind', 'way',
            'people', 'person', 'time', 'year', 'years', 'day', 'want'
        ]

        self.vectorizer_model = CountVectorizer(
            ngram_range=n_gram_range,
            stop_words=list(
                set(CountVectorizer(stop_words='english').get_stop_words())
                | set(custom_stop_words)
            ),
            min_df=1,
            max_df=0.85
        )

        self.model = BERTopic(
            embedding_model=self.embedding_model,
            vectorizer_model=self.vectorizer_model,
            min_topic_size=min_topic_size,
            calculate_probabilities=True,
            verbose=True
        )
```

### 4. BigQuery Schema Definition with Clustering and Partitioning
**Skills:** BigQuery, schema design, data warehouse optimization, cloud infrastructure

Defines typed BigQuery schemas with `REPEATED` mode for array columns (keywords, emotions), nullable ML fields for rows that skip analysis (interviewer utterances), and a table creation function that supports time-based partitioning and clustering for query performance.

```python
# src/bigquery/schema.py:10-49, 113-161
UTTERANCES_SCHEMA = [
    bigquery.SchemaField("utterance_id", "STRING", mode="REQUIRED"),
    bigquery.SchemaField("interview_id", "STRING", mode="REQUIRED"),
    bigquery.SchemaField("speaker_name", "STRING", mode="REQUIRED"),
    bigquery.SchemaField("speaker_role", "STRING", mode="REQUIRED"),
    bigquery.SchemaField("text", "STRING", mode="REQUIRED"),
    # ML fields nullable - interviewer rows have no ML analysis
    bigquery.SchemaField("sentiment_label", "STRING", mode="NULLABLE"),
    bigquery.SchemaField("sentiment_score", "FLOAT", mode="NULLABLE"),
    bigquery.SchemaField("topic_id", "INTEGER", mode="NULLABLE"),
    bigquery.SchemaField("keywords", "STRING", mode="REPEATED"),  # Array column
    bigquery.SchemaField("created_at", "TIMESTAMP", mode="REQUIRED"),
]

def create_table_if_not_exists(client, dataset_id, table_id, schema,
                                clustering_fields=None, partition_field=None):
    table_ref = f"{client.project}.{dataset_id}.{table_id}"
    try:
        return client.get_table(table_ref)  # Already exists
    except Exception:
        pass

    table = bigquery.Table(table_ref, schema=schema)
    if partition_field:
        table.time_partitioning = bigquery.TimePartitioning(
            field=partition_field,
            type_=bigquery.TimePartitioningType.DAY
        )
    if clustering_fields:
        table.clustering_fields = clustering_fields

    return client.create_table(table)
```

### 5. Multi-Method Keyword Extraction with Conversational Filtering
**Skills:** keyword extraction (YAKE), TF-IDF, NLP preprocessing, domain adaptation

Combines YAKE (which works well on individual short documents) with TF-IDF (which leverages corpus-wide statistics) in a hybrid mode. The `_is_valid_keyword` filter rejects fragments, pure numbers, repeated words, and an expanded conversational stop word list - necessary because standard NLP stop word lists are designed for written text, not transcribed speech.

```python
# src/ml/keywords.py:62-87, 191-213
def _is_valid_keyword(self, keyword: str) -> bool:
    keyword_lower = keyword.lower().strip()
    if len(keyword_lower) < 3:
        return False
    words = keyword_lower.split()
    if any(w in self.custom_stop_words for w in words):
        return False
    if not re.search(r'[a-zA-Z]', keyword):
        return False
    if keyword_lower.replace(' ', '').isdigit():
        return False
    if len(words) > 1 and len(set(words)) == 1:  # "yeah yeah"
        return False
    return True

# Hybrid mode: merge top results from both methods, deduplicated
elif self.method == 'hybrid':
    combined = []
    for yake_kw, tfidf_kw in zip(yake_results, tfidf_results):
        all_kw = {}
        for kw, score in yake_kw[:3]:
            all_kw[kw.lower()] = (kw, score)
        for kw, score in tfidf_kw[:3]:
            if kw.lower() not in all_kw:
                all_kw[kw.lower()] = (kw, score)
        combined.append(list(all_kw.values())[:n_keywords])
    return combined
```

---

## Clever Implementations

### 1. Topic Taxonomy with Substantive/Meta Classification

Instead of treating all discovered topics equally, the project manually classifies each of the 23 BERTopic clusters into three tiers: **substantive** (14 topics about actual budget issues), **meta** (6 topics capturing interview logistics like scheduling and conversational fragments), and **questionable** (2 topics needing human review). This classification is encoded as simple Python lists that downstream scripts use to filter visualizations and reports.

What makes this non-obvious: most topic modeling tutorials stop at "here are your topics." In real interview data, a large fraction of discovered topics are artifacts of the interview process itself, not the subject matter. Without this separation, aggregate statistics like "top 5 topics across all interviews" would include "Meeting Coordination" alongside "Budget Accountability" - misleading for policy researchers.

```python
# scripts/topic_names.py:13-85
TOPIC_NAMES = {
    # CORE SUBSTANTIVE TOPICS - Use these for analysis
    0: "Budget Accountability & Transparency",
    1: "Neighborhood Safety & Maintenance",
    4: "Civic Disengagement & Trust Issues",
    # ...14 total substantive topics

    # USELESS/META TOPICS - Exclude from substantive analysis
    2: "[META] Interview Logistics",
    7: "[META] Meeting Coordination",
    9: "[META] Conversational Fragments",
    19: "[META] Workshop Feedback",

    # QUESTIONABLE TOPICS - Review before including
    11: "[QUESTIONABLE] Internal Initiatives",
    17: "[QUESTIONABLE] Social Dynamics",
}

EXCLUDE_FROM_ANALYSIS = [2, 7, 9, 19]
REVIEW_BEFORE_USE = [11, 17]
SUBSTANTIVE_TOPICS = [0, 1, 3, 4, 5, 6, 8, 10, 12, 13, 14, 15, 16, 18]
```

### 2. Parameterized Semantic Search with Pre/Post Filter Splitting

The BigQuery semantic search function builds a CTE-based SQL query that separates filters into **pre-filters** (sentiment, topic - applied before the expensive cosine distance computation) and **post-filters** (similarity thresholds - applied after). This matters because computing `ML.DISTANCE` against every row is the bottleneck; filtering by sentiment or topic first reduces the comparison set.

What makes this non-obvious: naively putting all `WHERE` clauses together would work, but the query planner may not push filters before the distance computation. By structuring the CTE to apply categorical filters in the inner query and similarity thresholds in the outer query, the approach guarantees the optimization regardless of the query planner's behavior.

```python
# src/bigquery/bqml/semantic_search.py:57-117
# Split filters: categorical filters go BEFORE distance computation,
# similarity thresholds go AFTER
pre_filters = [f for f in filters if 'similarity_score' not in f]
post_filters = [f for f in filters if 'similarity_score' in f]

sql = f"""
WITH query_embedding AS (
  SELECT ml_generate_embedding_result AS embedding
  FROM ML.GENERATE_EMBEDDING(
    MODEL `{Config.GCP_PROJECT_ID}.{Config.BIGQUERY_DATASET}.embedding_model`,
    (SELECT '''{query}''' AS content),
    STRUCT(TRUE AS flatten_json_output, 'SEMANTIC_SIMILARITY' AS task_type)
  )
),
results_with_distance AS (
  SELECT u.*, ML.DISTANCE(u.text_embedding, q.embedding, 'COSINE') AS similarity_score
  FROM `...utterance_embeddings` u, query_embedding q
  {where_clause}  -- pre-filters applied here (before distance calc)
)
SELECT * FROM results_with_distance
{"WHERE " + " AND ".join(post_filters) if post_filters else ""}
ORDER BY similarity_score ASC
LIMIT {limit}
"""
```
