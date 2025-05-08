![Cursor Logo](https://github.com/user-attachments/assets/40469c95-e6d7-4ace-bf85-d7d88ce72c02)

# Cursor Chat History Vectorizer & Dockerized Search API

Vectorize your Cursor chat history and serve it via a simple search API.

This project provides tools to:
1.  Extract chat history from local Cursor IDE data (`state.vscdb` files within workspace storage).
2.  Generate text embeddings for user prompts using a local Ollama instance (`nomic-embed-text`).
3.  Store the extracted prompts and their embeddings in a LanceDB vector database.
4.  Include a Dockerized **FastAPI application (referred to as an "MCP server" in this context)** to search this LanceDB database via a simple API endpoint.

## ✨ Project Goal

The primary goal is to make your Cursor chat history searchable and usable for Retrieval Augmented Generation (RAG) or other LLM-based analysis by:

*   Converting user prompts into vector embeddings stored efficiently in LanceDB.
*   Providing a simple and accessible API server to perform vector similarity searches against your vectorized history.

## 🚀 Features

*   **Data Extraction:** Scans specified Cursor workspace storage paths for `state.vscdb` SQLite files.
*   **Prompt Extraction:** Extracts user prompts from the `aiService.prompts` key within the database files.
*   **Embedding Generation:** Uses a locally running Ollama instance to generate embeddings for extracted prompts.
    *   **Embedding Model:** `nomic-embed-text:latest` (default dimension 768).
*   **Vector Database Storage:** Stores original text, source file, role, and vector embeddings in a LanceDB database.
    *   **LanceDB URI:** `./cursor_chat_history.lancedb` (for the extractor) / `/data/cursor_chat_history.lancedb` (inside Docker container)
    *   **Table Name:** `chat_history`
*   **Dockerized Search API:** Includes a `Dockerfile` to build a container for the FastAPI search server.
*   **FastAPI Server (`main.py`):** Acts as the "MCP server" for handling search requests.
*   **API Endpoints:**
    *   `/search_chat_history` (POST): Performs vector similarity search.
    *   `/health` (GET): Checks server status and connections (Ollama, LanceDB).

## 📋 Requirements

**For Running the Extraction Script (`cursor_history_extractor.py`):**

*   **Python 3.7+**
*   **Ollama:** Ensure Ollama is installed and running on your local machine. Pull the `nomic-embed-text` model:
    ```bash
    ollama pull nomic-embed-text:latest
    ```
*   **Python Packages:** Install required packages:
    ```bash
    pip install ollama lancedb pyarrow pandas python-dotenv
    ```
*   **File Access:** Read access to your Cursor workspace storage directory (default: `C:\Users\<name>\AppData\Roaming\Cursor\User\workspaceStorage`).

**For Running the Search API (`main.py`) via Docker:**

*   **Docker Desktop** (Windows/Mac) or **Docker Engine** (Linux).
*   An **accessible Ollama instance** from the Docker container's network.
*   The **LanceDB database directory** (`./cursor_chat_history.lancedb`) already created by the extraction script.

## ⚙️ Setup & Configuration

The process involves two main steps:

1.  **Run the extraction script** to create or update the LanceDB database on your host machine.
2.  **Build and run the Docker container** for the search API, mounting the database created in Step 1.

### Step 1: Extract & Create Database (Host Machine)

1.  **Clone/Download the Project:**
    ```bash
    git clone https://github.com/markelaugust74/Cursor-history-API.git
    cd Cursor-history-API
    ```
2.  **Install Python dependencies** for the extractor:
    ```bash
    pip install -r requirements.txt
    ```
3.  **Verify Paths (if necessary):**
    *   Update the `WORKSPACE_STORAGE_PATH` variable in `cursor_history_extractor.py` if your Cursor data is not in the default location.
    *   Ensure you have write permissions in the directory where you run the script, as `./cursor_chat_history.lancedb` will be created here.
4.  **Ensure Ollama is Running:** Start your Ollama server and confirm `nomic-embed-text:latest` is available (`ollama list`).
5.  **Execute the extraction script:**
    ```bash
    python cursor_history_extractor.py
    ```
    This script will print progress and, if successful, create the `./cursor_chat_history.lancedb` directory containing your vectorized history.

### Step 2: Build & Run API Docker Container

1.  **Navigate** to the project directory containing the `Dockerfile`, `main.py`, and the `./cursor_chat_history.lancedb` directory created in Step 1.
2.  **Build the Docker image:**
    ```bash
    docker build -t cursor-chat-search-api .
    ```
3.  **Run the Docker container:**
    ```bash
    docker run -p 8001:8001 \
        -v /path/to/your/cursor_chat_history.lancedb:/data/cursor_chat_history.lancedb \
        -e OLLAMA_HOST="http://host.docker.internal:11434" \
        cursor-chat-search-api
    ```
    *   `-p 8001:8001`: Maps port 8001 on your host machine to port 8001 inside the container (where the FastAPI app runs).
    *   `-v /path/to/your/cursor_chat_history.lancedb:/data/cursor_chat_history.lancedb`: **This is CRUCIAL.** Replace `/path/to/your/cursor_chat_history.lancedb` with the **absolute path** on your host machine to the `cursor_chat_history.lancedb` directory created by the extraction script. This mounts your host database into the container at `/data/cursor_chat_history.lancedb`, the location expected by `main.py`. (Use forward slashes for paths even on Windows in Docker commands, or ensure proper escaping/configuration).
    *   `-e OLLAMA_HOST="..."`: Sets the `OLLAMA_HOST` environment variable inside the container. `http://host.docker.internal:11434` is common for Docker Desktop to reach the host. For Linux, you might need a different approach (e.g., host network mode, or using the host's IP accessible from the container).
4.  The FastAPI application (your "MCP server") should now be running and accessible via `http://localhost:8001`.

## ▶️ How to Run

The overall workflow is:

1.  Execute `python cursor_history_extractor.py` periodically on your host machine to create/update `./cursor_chat_history.lancedb`.
2.  Run the `docker run` command from the project root (where the `.lancedb` directory exists) to start the API server. This server will access the LanceDB database via the volume mount.

## 📁 Output

*   `./cursor_chat_history.lancedb`: A directory created by the extraction script containing the LanceDB vector database. Its schema includes `vector` (float list), `text` (string), `source_db` (string), and `role` (string).
*   A running API server inside the Docker container, accessible externally via the mapped port (default 8001), providing the defined API endpoints.

## 🔌 API Usage

Once the Docker container is running and the API is accessible (e.g., at `http://localhost:8001`), you can interact with it.

### Health Check (`GET /health`)

Checks the server's status and its connections to Ollama and LanceDB.

```bash
curl http://localhost:8001/health
