Root cause found and confirmed by reproduction. Nothing in the project was edited.

**The problem**

The live preflight in [run_gemini_live_test.py:108](scripts/run_gemini_live_test.py#L108) forces offline mode and then sets the Hugging Face cache location to a folder inside the project:

```python
os.environ["HF_HUB_OFFLINE"] = "1"
os.environ["TRANSFORMERS_OFFLINE"] = "1"
os.environ.setdefault("HF_HOME", str(config.BASE_DIR / ".venv" / "huggingface"))
```

On your personal laptop the embedding model was cached under `.venv\huggingface` inside the project, so this worked. On this laptop that folder does not exist. The `.venv` folder is gitignored, so pulling the code never brought the model cache with it. Your model cache here lives in the default location, `C:\Users\308231\.cache\huggingface`, which the preflight never looks at because it has already redirected the cache path.

Since offline mode is on and the redirected cache is empty, the model load fails. The preflight catches that exception, replaces it with the generic code `local_preflight_failed`, and [live.py:115](src/ui/live.py#L115) turns that into the message you saw. That is why the UI gives no hint about the real cause.

**Evidence**

- Reproducing the preflight's exact environment fails with: "couldn't connect to huggingface.co and couldn't find them in the cached files".
- The same retrieval with the default cache location succeeds: 5 chunks, retrieval not weak. So the Chroma index and incident data are fine. Only the model cache path is wrong.

**How to fix it without changing code**

The `setdefault` call only applies when `HF_HOME` is not already set. Pick one option:

1. **Point HF_HOME at your existing cache (quickest).** In the same PowerShell window, before launching Streamlit:

```powershell
$env:HF_HOME = "$env:USERPROFILE\.cache\huggingface"
.\.venv\Scripts\python.exe -m streamlit run app.py --server.address 127.0.0.1 --server.fileWatcherType none --browser.gatherUsageStats false
```

2. **Populate the project-local cache the code expects (permanent for this laptop).** Run once with internet access:

```powershell
$env:HF_HOME = "$PWD\.venv\huggingface"
.\.venv\Scripts\python.exe -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2'); print('CACHED')"
```

After that, launching Streamlit normally will work with no environment variable needed, matching how your personal laptop is set up.

**One side note.** Your uncommitted change in [app.py:9-10](app.py#L9-L10) adds a `load_dotenv()` call. It is not what caused this error, but it contradicts the documented design that the dashboard never loads `.env` and reads `GEMINI_API_KEY` only from the server process environment. The docs under `docs/` show the model cache path on the personal laptop was `C:\Users\bonny\OneDrive\Desktop\TELECOM\.venv\huggingface`, which confirms the cache was machine-local there too.
