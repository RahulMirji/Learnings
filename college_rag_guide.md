# Building a Free College RAG Agent

Since you want to build a real-world RAG system for your college (like the **College AI Assistant** mentioned in your notes) using **100% free resources**, we will use a fully local, open-source tech stack.

Here is the architecture we will use:

*   **LLM (Text Generation):** [Ollama](https://ollama.com/) (Run Llama-3 or Phi-3 locally for free).
*   **Embeddings:** HuggingFace `all-MiniLM-L6-v2` via `sentence-transformers` (Runs locally, completely free).
*   **Vector Database:** [ChromaDB](https://www.trychroma.com/) (Open-source, runs locally on your machine).
*   **Framework:** [LangChain](https://www.langchain.com/) (Open-source Python framework to connect it all).
*   **Frontend:** [Streamlit](https://streamlit.io/) (Free, easy Python UI framework).

---

## Step 1: Install Required Software

1.  **Download Ollama:** Go to [Ollama.com](https://ollama.com/) and install it on your Mac.
2.  **Pull an LLM Model:** Open your terminal and run:
    ```bash
    ollama run llama3
    ```
    *(This downloads the open-source Llama 3 model by Meta. It will take a few minutes. Once it's downloaded and running, you can exit the chat by typing `/bye`)*
3.  **Install Python Libraries:** Open your terminal and run:
    ```bash
    pip install langchain langchain-community langchain-huggingface chromadb sentence-transformers streamlit pypdf
    ```

## Step 2: Organize Your Data

Create a folder for your project and a subfolder inside it called `data`. Place your college documents (like the Student Handbook, Event Circulars, or Rulebooks in `.pdf` format) inside this `data` folder.

## Step 3: Write the Backend (The RAG Pipeline)

Create a file named `rag_backend.py`. This script will read your PDFs, process them, and set up the LangChain pipeline to query Ollama.

```python
import os
from langchain_community.document_loaders import PyPDFDirectoryLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_community.llms import Ollama
from langchain_core.prompts import PromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# 1. Load Documents
print("Loading college documents...")
loader = PyPDFDirectoryLoader("./data")
docs = loader.load()

# 2. Chunk Text
print("Chunking text...")
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)

# 3. Create Embeddings & Store in Vector DB
print("Creating embeddings and vector store...")
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
vectorstore = Chroma.from_documents(documents=splits, embedding=embeddings, persist_directory="./chroma_db")
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# 4. Setup Local LLM (Ollama)
llm = Ollama(model="llama3")

# 5. Build Prompt Template
template = """Use the following pieces of context from the college handbook to answer the question at the end.
If you don't know the answer or if the context doesn't have the information, just say that you don't know, don't try to make up an answer.
Keep the answer concise and helpful for students.

Context: {context}

Question: {question}

Helpful Answer:"""
custom_prompt_template = PromptTemplate.from_template(template)

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

# 6. Chain it all together
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | custom_prompt_template
    | llm
    | StrOutputParser()
)

def ask_question(question_text):
    return rag_chain.invoke(question_text)

if __name__ == "__main__":
    # Test it out directly in terminal
    response = ask_question("What is the minimum attendance required?")
    print("\nAnswer:", response)
```

## Step 4: Write the Frontend (Streamlit)

Create a second file called `app.py`. This will give you a neat web interface.

```python
import streamlit as st
from rag_backend import ask_question

st.set_page_config(page_title="College AI Assistant", page_icon="🎓")

st.title("🎓 College AI Assistant")
st.write("Ask me anything about college rules, attendance, or exams!")

# Initialize chat history
if "messages" not in st.session_state:
    st.session_state.messages = []

# Display chat messages from history
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# React to user input
if prompt := st.chat_input("Ask a question about the college..."):
    # Display user message in chat message container
    st.chat_message("user").markdown(prompt)
    # Add user message to chat history
    st.session_state.messages.append({"role": "user", "content": prompt})

    # Generate response
    with st.spinner("Searching college documents..."):
        response = ask_question(prompt)
        
    # Display assistant response in chat message container
    with st.chat_message("assistant"):
        st.markdown(response)
        
    # Add assistant response to chat history
    st.session_state.messages.append({"role": "assistant", "content": response})
```

## Step 5: Run Your Application

In your terminal, start the app by running:
```bash
streamlit run app.py
```

This will automatically open your web browser to `http://localhost:8501`, and you will have a fully functioning, beautiful RAG AI Assistant specific to your college documents, **running completely on your local machine for absolutely $0.**
