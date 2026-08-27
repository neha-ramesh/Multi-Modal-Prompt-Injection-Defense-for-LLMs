# 🛡️ Multi-Modal Prompt Injection Defense for LLMs

A multi-modal security pipeline designed to detect and block potentially malicious prompts before they are forwarded to a Large Language Model (LLM).

The system combines a **rule-based heuristic filter** with a **fine-tuned DistilBERT classifier**, providing two levels of analysis before a safe prompt is forwarded to **Google Gemini**.

The system supports:

* 📝 Text inputs
* 🖼️ Image inputs through OCR
* 🎙️ Audio inputs through speech transcription

---

## 🏗️ System Architecture

```text
                    User Input
                        │
          ┌─────────────┼─────────────┐
          │             │             │
         Text          Image         Audio
          │             │             │
          │        Tesseract OCR    Whisper
          │             │             │
          └─────────────┼─────────────┘
                        │
                        ▼
                 Extracted Text
                        │
                        ▼
                  Emoji Removal
                        │
                        ▼
             ┌─────────────────────┐
             │  Layer 1: Heuristic │
             │       Filter        │
             └──────────┬──────────┘
                        │
                 ┌──────┴──────┐
                 │             │
              LOW RISK     MEDIUM/HIGH
                 │             │
                 ▼             ▼
             DistilBERT       BLOCK
                 │
          ┌──────┴──────┐
          │             │
        SAFE          UNSAFE
          │             │
          ▼             ▼
       Gemini          BLOCK
          │
          ▼
     Final Response
```

The architecture follows a cascaded defense approach: inexpensive heuristic checks are performed first, and only inputs that pass the first layer are processed by the semantic classifier.

---

# 🔐 Layer 1 — Heuristic Filter

The first layer uses predefined **regular-expression patterns** to identify suspicious inputs.

The filter checks three main categories:

### Injection Markers

Examples include patterns involving:

* `### ... ###`
* `--- ... ---`
* `[INST] ... [/INST]`
* `<| ... |>`
* Prompt/instruction blocks

### Obfuscation Patterns

The filter checks for patterns such as:

* Base64-like strings
* Unicode escape sequences
* HTML character encoding
* Percent-encoded characters
* Non-ASCII character patterns

### Suspicious Keywords and Phrases

The filter also checks for patterns associated with potentially risky requests, including:

* Financial information
* Bank accounts
* Credit cards
* OTPs
* URLs
* Urgent requests
* Account verification
* Prize/reward claims
* Payment-related requests

Each matched pattern adds `1.0` to the heuristic score.

---

## 📊 Risk Scoring

The heuristic score determines whether the input is allowed to proceed.

| Risk Score | Risk Level | Action             |
| ---------: | ---------- | ------------------ |
|        `0` | **Low**    | Allowed to Layer 2 |
|        `1` | **Medium** | Blocked            |
|      `≥ 2` | **High**   | Blocked            |

The implementation assigns the risk level based on the accumulated pattern-match score. Only inputs classified as **Low risk** proceed to the DistilBERT layer.

---

# 🤖 Layer 2 — DistilBERT Semantic Classifier

Inputs that pass the heuristic layer are analyzed using a **fine-tuned DistilBERT sequence classification model**.

The classifier performs binary classification:

```text
0 → Safe
1 → Unsafe
```

The input is tokenized with:

* Maximum sequence length: `256`
* Truncation enabled
* Padding enabled

The model returns:

* Predicted class
* Confidence score

The report describes the DistilBERT model as the semantic layer for identifying more context-dependent attacks that may not be captured by simple pattern matching.

---

# ✨ Gemini Integration

If a prompt passes the heuristic layer and DistilBERT classifies it as **Safe**, it is forwarded to **Gemini 2.5 Flash** for response generation.

```text
Heuristic → Safe
      ↓
DistilBERT → Safe
      ↓
Gemini 2.5 Flash
```

If either security layer identifies the input as unsafe, the request is **not forwarded to Gemini**.

The Gemini API key is loaded through an environment variable:

```text
GOOGLE_API_KEY
```

Do not upload the actual `.env` file or API key to GitHub.

---

# 🖼️ Image Input

Images are processed using **Pytesseract**, a Python wrapper for Tesseract OCR.

```text
Image
  ↓
Tesseract OCR
  ↓
Extracted Text
  ↓
Heuristic Filter
  ↓
DistilBERT
```

The extracted text is processed through the same text-based security pipeline rather than being sent directly to the downstream LLM.

---

# 🎙️ Audio Input

Audio files are processed using **OpenAI Whisper** for automatic speech recognition.

```text
Audio
  ↓
Whisper
  ↓
Transcribed Text
  ↓
Heuristic Filter
  ↓
DistilBERT
```

The resulting transcript is analyzed by the same two-layer text security pipeline.

---

# 🌐 FastAPI

The complete security pipeline is exposed through a **FastAPI** endpoint:

```text
POST /ingest
```

The endpoint accepts:

* `text`
* `image`
* `audio`
* `metadata`

Each request receives a unique request ID, and processing time is recorded for different stages of the pipeline.

---

# 📊 Model Performance

The fine-tuned DistilBERT classifier achieved:

### Validation Accuracy

> **97.90%**

### Classification Performance

| Class            | Precision | Recall | F1-Score |
| ---------------- | --------: | -----: | -------: |
| Safe             |      0.98 |   0.98 |     0.98 |
| Unsafe           |      0.98 |   0.98 |     0.98 |
| Weighted Average |      0.98 |   0.98 |     0.98 |

The evaluation was performed on **124,508 validation samples**.

### Confusion Matrix

|                   | Predicted Safe | Predicted Unsafe |
| ----------------- | -------------: | ---------------: |
| **Actual Safe**   |         61,113 |            1,372 |
| **Actual Unsafe** |          1,239 |           60,784 |

The resulting false-negative count for unsafe prompts was **1,239**.

---

# ⚡ Performance

The report evaluates the system in terms of model efficiency and processing latency.

| Component           | Reported Time |
| ------------------- | ------------: |
| Heuristic Filter    |  ≈ `0.0002 s` |
| DistilBERT          |    ≈ `0.23 s` |
| OCR                 |    ≈ `1.08 s` |
| Whisper ASR         |      ≈ `11 s` |
| Text/Image Pipeline |     < `1.3 s` |

Audio processing introduces additional latency because of the Whisper transcription stage.

---

# 📚 Training Data

The DistilBERT classifier was trained using data aggregated from **five sources**.

### Datasets

1. **WildJailbreak Dataset**

   * Vanilla and adversarial harmful prompts
   * Vanilla and adversarial benign prompts

2. **Safe/Unsafe Prompt Classification Dataset**

   * Over 230,000 prompts

3. **Prompt Injection in the Wild**

4. **GCG Suffix Attack Prompts**

5. **Rephrased Adversarial Prompts**

The datasets were standardized into a common binary classification format:

```text
0 → Safe
1 → Unsafe
```

After consolidation and duplicate removal, the resulting corpus contained **622,549 prompts**.

---

# 🔬 Model Comparison

Several approaches were evaluated before selecting the final DistilBERT classifier.

| Model                                       | Validation Accuracy |
| ------------------------------------------- | ------------------: |
| TF-IDF + SVM                                |              94.38% |
| TF-IDF + Linear SVC                         |              94.39% |
| TF-IDF + Logistic Regression                |              89.22% |
| DistilBERT Embeddings + Logistic Regression |              88.00% |
| DistilBERT Embeddings + Linear SVC          |              88.00% |
| **Fine-Tuned DistilBERT**                   |          **97.90%** |

The fine-tuned DistilBERT model was selected as the semantic classifier because it achieved the highest validation accuracy among the evaluated approaches.

---

# 🛠️ Technologies Used

* **Python**
* **FastAPI**
* **PyTorch**
* **Hugging Face Transformers**
* **DistilBERT**
* **OpenAI Whisper**
* **Tesseract OCR / Pytesseract**
* **Pillow**
* **Google Gemini**
* **python-dotenv**
* **Regular Expressions**

---

# 📁 Project Structure

```text
Prompt-Injection-Defense/
│
├── main.py
├── requirements.txt
├── .env.example
│
└── model/
    ├── config.json
    ├── tokenizer_config.json
    ├── special_tokens_map.json
    ├── vocab.txt
```

The `model/` directory contains the locally saved DistilBERT model and tokenizer files used by the application.

---

# ⚙️ Setup

### 1. Install Dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure Gemini

Create a `.env` file using `.env.example`:

```text
GOOGLE_API_KEY=your_api_key_here
```

Do **not** commit your actual API key.

### 3. Run the FastAPI Application

```bash
uvicorn main:app --reload
```

The API will expose the `/ingest` endpoint for processing text, image and audio inputs.

---

# 🎯 Project Objective

The project aims to provide a **multi-modal, layered defense mechanism against LLM prompt injection**.

The central approach is:

> **Fast heuristic screening → semantic DistilBERT classification → controlled Gemini access**

By converting image and audio inputs into text before applying the security layers, the same core detection mechanism can be applied across multiple input modalities.

The overall design follows a **defense-in-depth** approach, where multiple complementary checks are used rather than relying on a single detection mechanism.
