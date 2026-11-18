# Friday

Backend-only personal AI assistant. See `PLAN.md` for the full plan and progress.

## Setup (Windows)

1. Install [Ollama](https://ollama.com) and pull the two models:
   ```
   ollama pull qwen3.5:4b
   ollama pull nomic-embed-text
   ```
   No GPU? Skip Ollama and use the NVIDIA API instead (`FRIDAY_PROVIDER=nvidia`, see `PLAN.md`).
2. Create the virtual environment and install dependencies:
   ```
   py -3.13 -m venv .venv
   .venv\Scripts\activate
   pip install -r requirements.txt
   ```
3. Copy `.env.example` to `.env` and fill in your values (`.env.example` arrives in step 0.2).
4. Check it imports:
   ```
   python -c "import friday"
   ```
