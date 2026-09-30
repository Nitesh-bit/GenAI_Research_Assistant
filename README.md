# 📚 Agentic Research Assistant

An **agentic research assistant** that lets you upload a research paper as a PDF and ask questions about it through a conversational interface.

The project combines **Retrieval-Augmented Generation (RAG)**, **vector search**, **tool routing**, **conversation-aware follow-up handling**, and a lightweight local LLM to produce grounded answers with confidence levels and source-page references.

> **Project format:** The current implementation is developed as a Jupyter/Google Colab notebook. The notebook also generates `backend.py`, `app.py`, and `requirements.txt` for the Streamlit application.

---

## ✨ Features

- Upload a research paper in **PDF** format
- Split documents into chunks and index them with **FAISS**
- Generate embeddings using `sentence-transformers/all-MiniLM-L6-v2`
- Use **Qwen/Qwen2.5-1.5B-Instruct** as the final application LLM
- Route questions between:
  - **DOCUMENT** → search the uploaded research paper
  - **CALCULATOR** → perform mathematical calculations
  - **DIRECT** → answer using general knowledge
- Maintain conversational history
- Resolve follow-up questions such as:
  - “What dataset was used?”
  - “How many instances does it contain?”
- Return source page information for document-based answers
- Provide **High / Medium / Low** confidence
- Apply grounding rules to reduce unsupported answers
- Provide a Streamlit chat interface
- Includes Google Colab support for running the application and optionally exposing it through a Cloudflare tunnel

---

## 🏗️ Architecture

```text
                    ┌──────────────────────┐
                    │   Streamlit UI       │
                    │   PDF Upload + Chat  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Document Processing │
                    │      PyPDFLoader     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Text Chunking      │
                    │  Recursive Splitter  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │    Embeddings        │
                    │ all-MiniLM-L6-v2     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   FAISS Vector Store │
                    │      Retriever       │
                    └──────────┬───────────┘
                               │
                               ▼
                 ┌─────────────────────────────┐
                 │       Agentic Router        │
                 └──────┬────────┬────────────┘
                        │        │
             ┌──────────┘        └─────────────┐
             ▼                                  ▼
      ┌──────────────┐                  ┌──────────────┐
      │   DOCUMENT   │                  │  CALCULATOR  │
      │   Retriever  │                  │   numexpr    │
      └──────┬───────┘                  └──────┬───────┘
             │                                  │
             └──────────────┬───────────────────┘
                            ▼
                 ┌──────────────────────┐
                 │ Grounded Answer Chain │
                 │ Qwen 1.5B + Sources   │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ Answer + Confidence  │
                 │ + Source Pages       │
                 └──────────────────────┘
```

---

## 🔄 How It Works

### 1. Upload a research paper

The Streamlit interface accepts a PDF research paper.

The backend loads the document using:

```python
PyPDFLoader
```

### 2. Split the document

The paper is divided into smaller chunks using:

```python
RecursiveCharacterTextSplitter(
    chunk_size=700,
    chunk_overlap=150
)
```

This makes relevant sections easier to retrieve.

### 3. Create embeddings

Each document chunk is converted into a vector using:

```text
sentence-transformers/all-MiniLM-L6-v2
```

### 4. Build the vector database

The embeddings are stored in a local **FAISS** vector store.

For each question, the retriever searches for the most relevant document chunks.

### 5. Route the question

The application classifies the question into one of three routes:

```text
DOCUMENT
CALCULATOR
DIRECT
```

Examples:

| Question | Route |
|---|---|
| What dataset was used? | DOCUMENT |
| What was the model accuracy? | DOCUMENT |
| What is 125 × 24? | CALCULATOR |
| What is machine learning? | DIRECT |

The final routing implementation uses deterministic keyword/pattern matching for these categories.

### 6. Retrieve evidence

For document questions, the system retrieves relevant chunks from FAISS and attaches source metadata such as the document name and page number.

Example:

```text
[Source: research-paper.pdf, Page: 6]
Relevant passage from the paper...
```

### 7. Generate the answer

The retrieved evidence is passed to the Qwen model with strict instructions to:

- use the provided evidence
- avoid inventing facts
- preserve numerical values and units
- distinguish similar concepts
- prefer experimental results and tables
- return a concise answer

### 8. Return confidence and sources

The intended response format is:

```text
ANSWER: ...
CONFIDENCE: High
SOURCES: 6
```

If the retrieved evidence does not support the answer, the assistant can respond:

```text
ANSWER: I could not find the answer in the provided document.
CONFIDENCE: Low
SOURCES: ...
```

---

## 🧠 Conversational Follow-Ups

The assistant includes a lightweight mechanism for resolving references in follow-up questions.

For example:

```text
User: What dataset was used in this research?

Assistant: The dataset was sourced from the UCI Machine Learning Repository.

User: How many instances does it contain?
```

The second question can be resolved to:

```text
How many instances does the dataset contain?
```

This resolved question is then routed to the document retriever.

---

## 🛡️ Grounding and Validation

The project includes several safeguards designed to reduce hallucinations.

### Evidence-based answering

Document questions are answered using retrieved passages rather than relying solely on the model's general knowledge.

### Source attribution

Retrieved passages include document and page metadata.

### Confidence levels

Answers are classified as:

- **High**
- **Medium**
- **Low**

### Unit validation

The project includes validation for questions involving values such as:

- dollars
- training cost
- time
- dataset size
- performance metrics

This helps avoid interpreting an unrelated number as the requested value.

### Research-specific distinctions

The final prompting logic explicitly warns the model not to confuse:

- dataset features
- CNN-extracted features
- model parameters
- performance metrics
- training time
- testing time

---

## 🧰 Tech Stack

| Technology | Purpose |
|---|---|
| Python | Core implementation |
| LangChain | LLM/RAG orchestration |
| Hugging Face Transformers | Local LLM inference |
| Qwen2.5-1.5B-Instruct | Final application LLM |
| Sentence Transformers | Text embeddings |
| FAISS | Vector similarity search |
| PyPDF | PDF document loading |
| NumExpr | Calculator tool |
| Pydantic | Structured response schema |
| Streamlit | Web interface |
| PyTorch | Model execution |

---

## 📦 Installation

### Option 1 — Google Colab

The notebook is designed to work well in Google Colab.

1. Open `Agentic_Research_Assistant.ipynb` in Google Colab.
2. Run the setup cells.
3. Install the required packages.
4. Upload a research-paper PDF when prompted.
5. Run the notebook cells in order.
6. The notebook generates:

```text
backend.py
app.py
requirements.txt
```

7. Start Streamlit:

```bash
streamlit run app.py
```

The notebook also contains optional Cloudflare tunnel commands for exposing the local Streamlit application through a temporary public URL.

---

### Option 2 — Local Environment

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it.

**Windows:**

```bash
.venv\Scripts\activate
```

**Linux/macOS:**

```bash
source .venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Then run:

```bash
streamlit run app.py
```

> Note: `app.py` and `backend.py` are generated by the notebook in the current project workflow. If they are not present in your repository yet, run the relevant notebook cells first.

---

## 📁 Suggested Repository Structure

After exporting the generated application files, a clean GitHub repository can look like:

```text
Agentic-Research-Assistant/
│
├── Agentic_Research_Assistant.ipynb
├── app.py
├── backend.py
├── requirements.txt
├── README.md
└── .gitignore
```

If you do not want generated files committed, you can keep the repository notebook-focused:

```text
Agentic-Research-Assistant/
│
├── Agentic_Research_Assistant.ipynb
├── README.md
└── .gitignore
```

---

## 🚀 Example Questions

After uploading a research paper, try questions such as:

```text
What dataset was used in this research?

How many instances are in the dataset?

How many features are used in the dataset?

What methodology was used?

What method was proposed?

What was the accuracy of the proposed method?

What optimizer was used?

What is 125 multiplied by 24?

What is machine learning?

How many instances does it contain?
```

---

## 🧪 Evaluation

The notebook contains evaluation examples for questions involving:

- dataset source
- number of instances
- number of features
- model accuracy
- dataset provenance
- methodology
- optimizer
- author information
- training cost
- mathematical calculations
- general knowledge questions

The notebook also experiments with different chunk sizes and retrieval settings to improve document retrieval.

---

## ⚙️ Model Configuration

The final application uses:

### LLM

```text
Qwen/Qwen2.5-1.5B-Instruct
```

The model is loaded through Hugging Face Transformers and uses:

- `float16` when CUDA is available
- `float32` otherwise
- automatic device placement

### Embedding Model

```text
sentence-transformers/all-MiniLM-L6-v2
```

### Retriever

```text
FAISS
```

The final backend retrieves up to **4 relevant chunks** for a query.

---
