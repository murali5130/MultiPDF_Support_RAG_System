# MultiPDF Support RAG System

A Streamlit app that lets you upload multiple PDFs and chat with them using **Retrieval-Augmented Generation (RAG)**. Answers are grounded in your documents, cite the source file, and pass through **three layers of guardrails** before they reach the user.

## Features

- **Multi-PDF chat:** upload several PDFs at once and ask questions across all of them
- **Source attribution:** every retrieved chunk carries its filename, and the answer mentions which document it came from
- **Semantic search:** documents are split into chunks, embedded, and indexed in FAISS for fast retrieval
- **Three-layer guardrails:**
  1. **Input guardrail:** classifies the question as `SAFE` or `THREAT` (blocks attempts to change the assistant's behavior, reveal its setup, or ask things unrelated to the documents)
  2. **Retrieval guardrail:** checks that the retrieved context is relevant to the question, to reduce hallucinations
  3. **Output guardrail (DLP):** scans the drafted answer for sensitive PII (exact salaries, SSNs, home addresses, personal phone numbers) and suppresses the response if it finds any
- **Grounded answers:** if the answer is not in the documents, the model says so instead of guessing

## How It Works

```
PDFs -> Extract text -> Split into chunks -> Embed -> FAISS index
                                                          |
User question -> [Layer 1: input guardrail] -> Similarity search (top 3 chunks)
                                                          |
                                         [Layer 2: retrieval guardrail]
                                                          |
                                      LLM answer (with source filenames)
                                                          |
                                         [Layer 3: output guardrail / DLP]
                                                          |
                                                     Final answer
```

1. **Process PDFs:** text is extracted with `pypdf`, split with `RecursiveCharacterTextSplitter` (chunk size 1000, overlap 200), embedded, and saved as a local FAISS index (`faiss_index/`).
2. **Ask a question:** the question is checked by the input guardrail, then the top 3 most similar chunks are retrieved.
3. **Validate and answer:** the retrieved context is checked for relevance, the LLM drafts an answer from it, and the draft is checked for PII leaks before it is shown.

## Tech Stack

| Component | Technology |
|---|---|
| UI | Streamlit |
| PDF parsing | pypdf |
| Text splitting | LangChain `RecursiveCharacterTextSplitter` |
| Embeddings | Hugging Face `sentence-transformers/all-MiniLM-L6-v2` |
| Vector store | FAISS |
| LLM | Groq (`openai/gpt-oss-120b`) via `langchain-groq` |
| Config | python-dotenv |

## Project Structure

```
MultiPDF_Support_RAG_System/
├── app.py                              # Streamlit app: indexing, retrieval, guardrails, chat UI
├── requirements.txt                    # Python dependencies
├── sample_office_lease_agreement.pdf   # Sample document for testing
├── sample_service_agreement.pdf        # Sample document for testing
└── .gitignore
```

`faiss_index/` is created automatically when you click **Process PDFs**.

## Getting Started

### Prerequisites

- Python 3.10 or higher
- A [Groq API key](https://console.groq.com/)

### Installation

```bash
git clone https://github.com/murali5130/MultiPDF_Support_RAG_System.git
cd MultiPDF_Support_RAG_System

python -m venv venv
# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

pip install -r requirements.txt
```

### Configuration

Create a `.env` file in the project root:

```
GROQ_API=your_groq_api_key_here
```

Never commit your `.env` file. Make sure it is listed in `.gitignore`.

### Run

```bash
streamlit run app.py
```

## Usage

1. Open the sidebar and upload one or more PDF files (you can try the two sample PDFs in this repo).
2. Click **Process PDFs** and wait for the success message.
3. Ask questions in the chat box, for example:
   - "What is the lease term in the office lease agreement?"
   - "What are the payment terms in the service agreement?"
   - "Which document mentions termination conditions?"

## Limitations

- **Text-based PDFs only:** scanned or image-only PDFs have no extractable text and will not be indexed. Use OCR first.
- **One index at a time:** clicking **Process PDFs** rebuilds `faiss_index/`, replacing the previously indexed documents.
- **No conversation memory:** each question is answered independently; earlier chat turns are not used as context.
- **LLM-based guardrails:** the guardrails are model-judged, so they can occasionally misclassify. The output guardrail may over-block answers that contain legitimate figures or addresses from contracts.
- **FAISS index loading:** the app loads the local index with `allow_dangerous_deserialization=True`. Only load an index that this app created yourself.

## Future Improvements

- Add OCR support for scanned PDFs
- Persist and manage multiple document collections
- Add conversation history to retrieval
- Show page numbers alongside source filenames

## Author

**Murali Krishna**
GitHub: [murali5130](https://github.com/murali5130)
