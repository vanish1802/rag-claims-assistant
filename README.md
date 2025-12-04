# RAG Claims Assistant

Natural-language analytics on health insurance claim data using ETL, RAG, and Text-to-SQL.

## Overview

This project lets non-technical users query insurance claims data in plain English.

Core idea:

- Normalize raw claims from multiple insurers into a unified schema.
- Represent each claim as both structured data and a natural-language sentence.
- Support two query modes:
  - Text-to-SQL for exact numeric/aggregated answers.
  - RAG (Retrieval-Augmented Generation) for semantic, explanation-style answers.

***

## Architecture

Data flow:

1. ETL pipeline (src/etl.py)
   - Bronze: raw CSVs from multiple insurers.
   - Silver: single normalized table (claims_normalized).
   - Gold: Silver + a text_representation column per claim.

2. Storage
   - Silver/Gold in CSV (or a SQL/duckdb table).
   - Gold text + metadata embedded and stored in Chroma (vector DB).

3. Query pipelines
   - Text2SQL (src/text2sql_pipeline.py):
     - NL question + Silver schema → LLM → SQL.
     - Run SQL on Silver → exact numeric/aggregated result.
   - RAG (src/rag_pipeline.py):
     - NL question → embedding → Chroma similarity search.
     - Retrieve top-k claim texts + metadata.
     - Send context + question to LLM → grounded answer.

4. Application (app.py)
   - Streamlit UI:
     - Mode switch: RAG vs Text2SQL.
     - Input box for questions.
     - Displays answer + underlying claims / result tables.

***

## ETL: Bronze → Silver → Gold

Bronze (input):

- insurance_company_1_claims.csv
- insurance_company_2_claims.csv

Schema differences (examples):

- claim_id vs claim_number
- patient_id vs subscriber_id
- claim_amount vs billed_amount
- claim_status (Approved/Denied) vs status (PAID/REJECTED)
- service_date vs date_of_service
- provider_specialty vs specialty
- patient_name ("First Last") vs patient_full_name ("Last, First")

Silver (normalized):

- Unified columns, e.g.:
  - claim_id, patient_id, patient_name
  - diagnosis, icd_code
  - procedure_name, procedure_code
  - claim_amount
  - status (approved/denied, normalized)
  - denial_reason
  - service_date (single date format)
  - provider_specialty
  - source_insurer

Gold (master):

- Same structured columns as Silver.
- Additional text_representation column, e.g.:

  Claim AET-20022: Patient Sarah Johnson (ID: SUB60219) received Echocardiogram for Coronary Artery Disease on 2024-09-18. Amount: $980. Status: denied. Denial reason: Duplicate billing. Specialty: Cardiology.

This text_representation is what gets embedded and stored in Chroma for RAG.

***

## RAG Pipeline

Offline:

- Read Gold CSV.
- For each row:
  - Embed text_representation using a sentence-transformer (e.g. all-MiniLM-L6-v2).
  - Store embedding + text + metadata in Chroma.

Online (per RAG query):

- Embed user question.
- Perform similarity search in Chroma (top-k claim texts).
- Build prompt: context (retrieved claims) + question.
- Send to LLM (Groq) to generate an answer constrained to context.
- Return:
  - Answer text.
  - Table of retrieved claims (for transparency).

Best for:

- “What are common denial reasons for diabetes in Q3?”
- “Summarize recent high-value denied cardiology claims.”

***

## Text-to-SQL Pipeline

Mode: quantitative / exact questions.

Flow:

- Inject Silver schema and business rules into a prompt.
- LLM generates a SQL query over claims_normalized.
- Execute SQL (e.g., via DuckDB or pandas/SQLite) to get exact results.
- Return:
  - Result table.
  - Optionally, the generated SQL and a natural-language paraphrase.

Best for:

- “How many denied cardiology claims in Q4 2024?”
- “Average claim_amount by insurer for diabetes claims in 2023.”

***

## Tech Stack

- Language model: Groq-hosted LLM (Llama family).
- Embeddings: SentenceTransformers (e.g. all-MiniLM-L6-v2).
- Vector DB: Chroma.
- Data processing: pandas.
- Storage: CSVs; can be backed by DuckDB/SQLite.
- UI: Streamlit.

***

## Setup

1. Clone and create environment

   ```bash
   git clone https://github.com/vanish1802/rag-claims-assistant.git
   cd rag-claims-assistant
   python3 -m venv .venv
   source .venv/bin/activate  # macOS/Linux
   # On Windows: .venv\Scripts\activate
   ```

2. Install dependencies

   ```bash
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

3. Set environment variables

   Create a .env file (or export in shell):

   ```bash
   export GROQ_API_KEY="your_groq_api_key_here"
   ```

***

## Running the Pipeline

1. Generate or prepare Bronze data

   Bronze CSVs should be in:

   - data/bronze/insurance_company_1_claims.csv
   - data/bronze/insurance_company_2_claims.csv

   If needed:

   ```bash
   python generate_insurance_data.py
   ```

2. Run ETL (Bronze → Silver → Gold)

   From the project root:

   ```bash
   python src/etl.py
   ```

   Outputs:

   - data/silver/claims_normalized.csv
   - data/gold/claims_master.csv

3. Run the app

   ```bash
   streamlit run app.py
   ```

   Then open the URL shown (typically http://localhost:8501).

***

## Usage

- Choose mode in the UI:
  - RAG mode: semantic, explanation-style answers using Gold + Chroma.
  - Text2SQL mode: exact numeric answers over Silver via auto-generated SQL.

- Type a question, e.g.:
  - RAG: “Explain main reasons why diabetes claims are denied.”
  - Text2SQL: “What is the total amount of denied endocrinology claims in 2024?”

- Inspect:
  - Answer text.
  - Source claims / result table.
  - (Optional) the SQL used in Text2SQL mode.

***
