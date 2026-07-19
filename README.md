# Global AI Transcriber — Streamlit Community Cloud Deployment

## What was fixed in this copy

1. **Headless browser** — `browser/browser_manager.py` and `ui/browser_process.py` now
   launch Chromium with `headless=True` instead of `headless=False`. Cloud servers have
   no display, so this is required for it to run there at all.
2. **`app.py` now installs Chromium itself on first boot** — Streamlit Cloud has no
   shell post-install hook, so a `st.cache_resource`-wrapped function runs
   `playwright install chromium` once per container. This is what was missing last
   time, causing the `Executable doesn't exist` error.
3. **`app.py` bridges Streamlit Secrets into `os.environ`** — so your existing
   `os.getenv("OPENAI_API_KEY")`-style code keeps working when the key is set via
   Streamlit Cloud's Secrets panel instead of a committed `.env`.
4. **Linux-friendly `requirements.txt`** — `torch` and `pyannote.audio` removed (they
   aren't actually imported anywhere, only mentioned in a comment, and are a common
   source of install failures on Streamlit Cloud).
5. **`packages.txt` added** — the Linux system libraries Chromium needs to run headless.
6. **Speaker-labeling bug fixed** — `speech/transcript_reviewer.py` was computing the
   speaker-labeled text (`Speaker 1: ...`, `Speaker 2: ...`) but never writing it back
   to `transcript.text`, so it silently never appeared in the output even though the
   app reported "Speaker labels applied." Now fixed — the labeled text is saved.
   Note: speaker labeling only works with the **Faster-Whisper** provider (the default),
   since that's the only one that returns segment timestamps; the GPT-4o Transcribe
   (OpenAI) provider doesn't currently supply the data it needs.
7. **`.gitignore` added** — keeps `.env`, saved Intron login sessions, audio files, and
   logs out of GitHub.

## Before you deploy: your OpenAI key

Your `.env` file has a live OpenAI API key in plain text. If this key has been pushed
to GitHub or shared anywhere before, rotate it at platform.openai.com before deploying.
Never commit `.env` — put the key in Streamlit Cloud's Secrets panel instead (see below).

## Step-by-step deployment

### 1. Push this code to GitHub
```bash
cd GlobalAITranscriber
git init
git add .
git status   # confirm .env and browser/sessions/*.json are NOT listed
git commit -m "Prepare for Streamlit Cloud deployment (headless + fixes)"
git branch -M main
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

If you're pushing to an **existing repo** that previously had `.env` committed, GitHub's
push protection will keep blocking you even after you delete the file, because the key
is still in your commit history. In that case, either:
- Delete the GitHub repo and re-create it, then push this fresh `git init`, or
- Use `git rm --cached .env && git commit --amend` on your very first commit before
  it's ever pushed (doesn't help if it's already been pushed once).

### 2. Create the app on Streamlit Community Cloud
1. Go to [share.streamlit.io](https://share.streamlit.io) and sign in with GitHub.
2. **New app** → pick your repo → branch `main` → main file path `app.py`.
3. Before deploying, open **Advanced settings → Secrets** and add:
   ```toml
   OPENAI_API_KEY = "sk-..."
   ```
   plus any other keys your `.env` currently holds.
4. Click **Deploy**.

### 3. First boot is slower than usual
`playwright install chromium` runs automatically on first load — adds roughly 1–2
minutes once, then it's cached for the container's lifetime.

### 4. Re-authenticating with Intron
Your saved Intron session isn't committed (excluded for security), so the first run on
the cloud will need to log in again through whatever flow `auth_manager.py` uses.

## If it still fails after deploying

Check **Manage app → logs** for the exact error. If you see the same
`Executable doesn't exist` error again, it usually means the *old* code got pushed
instead of this fixed copy — check `ui/browser_process.py` line ~106 on GitHub directly
and confirm it says `headless=True`, and that `app.py` contains
`_ensure_playwright_browser_installed`.

## Reverting to local desktop use

To go back to a visible browser window locally, change `headless=True` back to
`headless=False` in both `browser/browser_manager.py` (~line 74) and
`ui/browser_process.py` (~line 106).
