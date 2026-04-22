# Building a Free Mobile RAG App for HKBK College

## The Goal
You want to build a mobile application where HKBK college students can chat and ask questions about the college, getting accurate answers based on your actual college documents (Rulebooks, Circulars, Timetables). The entire stack—from the database to the AI models—should be **100% free**.

## Free Tech Stack Architecture

To make this work as a mobile app without paying for servers, you should split the system into a **Mobile Frontend** and a **Cloud Backend**.

### 1. Frontend: Mobile App
*   **Framework:** **Flutter**
*   **Cost:** **$0** (Open-source, allows you to build for Android and iOS simultaneously).

### 2. Backend API (The Bridge)
*   **Framework:** **Python (FastAPI)**
*   **Hosting:** **Render** (Web Service Free Tier) or **HuggingFace Spaces** (Free Docker env).
*   **Cost:** **$0**

### 3. Vector Database (The Memory)
This is where you store your college PDFs after converting them into searchable numbers (embeddings).
*   **Tool:** **Qdrant Cloud** (Offers a generous perpetual free tier of 1GB—enough for thousands of PDFs) OR **Pinecone** (1 Free Starter Index).
*   **Cost:** **$0**

### 4. Embedding Model (Text to Numbers)
*   **Tool:** **HuggingFace `all-MiniLM-L6-v2`** (Runs locally inside your free backend) or **Google Gemini Embeddings** (Free tier).
*   **Cost:** **$0**

### 5. LLM (The Brain)
*   **Tool:** **Google Gemini 1.5 Flash API** (Google AI Studio has a very generous free tier of 15 Requests Per Minute) or **Groq API** (Lightning-fast free tier for Llama 3 models).
*   **Cost:** **$0**

---

## Step-by-Step Implementation Roadmap

### Phase 1: Data Preparation & Ingestion
Instead of processing PDFs inside the mobile app (which is slow and hard), you do it once on your personal computer.

1.  Gather all HKBK PDFs (syllabus, attendance rules, event details).
2.  Write a simple Python script using **LangChain** to read the text.
3.  Split the text into chunks (e.g., 1000 characters per chunk).
4.  Convert these chunks into embeddings.
5.  Upload these vectors over the internet to your **free Qdrant Cloud** or **Pinecone** database. 
*Now, your college knowledge is permanently accessible online.*

### Phase 2: Build the RAG Backend API
Your mobile app needs a public web address to send questions to. We build this with Python.

1.  Create a **FastAPI** app (`main.py`).
2.  Set up an endpoint: `POST /ask`
3.  Inside the `/ask` route, the backend performs the RAG magic:
    *   Takes the student's question.
    *   Converts the question to an embedding.
    *   Searches the **Qdrant/Pinecone** database for the top 3 matching chunks of college info.
    *   Takes those college chunks + the student's question and sends them to the **Gemini API** or **Groq API**.
    *   Returns the AI's final answer as a JSON response.

### Phase 3: Deploy the API for Free
1.  Push your FastAPI backend code to **GitHub**.
2.  Create an account on [Render.com](https://render.com) or HuggingFace.
3.  Connect your GitHub and deploy it on the **Free Tier**. 
4.  You will get a public URL like `https://hkbk-rag-api.onrender.com/ask`.

### Phase 4: Build the Flutter Mobile App
Now that the brain is online, build the Face!

1.  Create a Flutter project for your mobile app.
2.  Build a **Chat UI**. You can build it yourself with a `ListView.builder` or use the `flutter_chat_ui` package.
3.  Use the `http` package in Flutter to send requests to your backend:
    *   **App Sends:** `{"question": "What is the attendance required for final exams?"}`
    *   **Backend Replies:** `{"answer": "According to the HKBK handbook, you need a minimum of 75% attendance..."}`
4.  Display this reply as a chat bubble from the "HKBK Assistant".

---

## Why this specific architecture?
*   **Real Mobile App capability:** The heavy lifting is done on the cloud, so the app runs smoothly on any cheap phone.
*   **Zero Infrastructure Costs:** Qdrant Cloud, Render, and Gemini API combined give you an enterprise-grade stack for exactly $0.
*   **Easy to Update:** If a new college rule comes out, you just upload the new PDF to your Python script and update the database. You don't have to update the mobile app in the Play Store!
