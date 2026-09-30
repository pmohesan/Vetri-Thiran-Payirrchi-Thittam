# ComicCraft – AI Comic Story Creator

Turn a story idea into a 5-panel illustrated comic (with narration, dialogue and a
downloadable PDF).

| Stage | Tool |
|---|---|
| Panel outline (JSON) | Gemini Flash Lite (`gemini-3.5-flash-lite`) |
| Narration + dialogue | Gemini Flash Lite (`gemini-3.5-flash-lite`) |
| Illustrations | Hosted Stable Diffusion via Hugging Face Inference Providers |
| Web app / API | FastAPI + Jinja2 |
| PDF export | fpdf2 |

## Project structure

```
comiccraft/
├── app/
│   ├── main.py            # FastAPI app, static mount
│   ├── routes.py          # pages + JSON API + pipeline
│   ├── config.py          # paths, env vars, model names
│   ├── gemini_client.py   # shared Gemini helper
│   ├── gemini_flash.py    # generate_outline()
│   ├── gemini_pro.py      # generate_story()
│   ├── image_generator.py # generate_image()
│   ├── layout_builder.py  # build_comic_layout()
│   └── exporters.py       # save_pdf()
├── templates/             # index.html, comic_preview.html, export_success.html
├── static/                # panels/ (images), exports/ (PDFs), fonts/, images/
├── tests/test_app.py      # offline tests
├── .vscode/               # debug + pytest settings
├── .env.example           # copy to .env
├── requirements.txt
└── requirements-dev.txt
```

## 1. Setup in VS Code

**Prerequisites:** Python 3.10–3.12, VS Code with the *Python* extension, a free Gemini
API key ([aistudio.google.com/apikey](https://aistudio.google.com/apikey)).
About 6 GB of free disk space for PyTorch + the Stable Diffusion weights.

1. **File → Open Folder…** and choose the `comiccraft` folder.
2. Open the terminal (**Ctrl+\`**) and create a virtual environment:

   ```bash
   # Windows (PowerShell)
   python -m venv .venv
   .venv\Scripts\Activate.ps1

   # macOS / Linux
   python3 -m venv .venv
   source .venv/bin/activate
   ```
   (If PowerShell blocks activation, run `Set-ExecutionPolicy -Scope Process RemoteSigned` first.)
3. **Ctrl+Shift+P → Python: Select Interpreter →** pick the `.venv` one.
4. Install dependencies:

   ```bash
   pip install -r requirements-dev.txt
   ```
   *NVIDIA GPU?* Install the CUDA build of PyTorch first (see pytorch.org → "Get Started"),
   then run the command above.
5. Create your settings file:

   ```bash
   # Windows: copy .env.example .env      macOS/Linux: cp .env.example .env
   ```
   Open `.env` and set `GEMINI_API_KEY=...` and `HF_API_KEY=...`.

## 2. Run

```bash
uvicorn app.main:app --reload
```
or press **F5** in VS Code (uses `.vscode/launch.json`).

* App: <http://127.0.0.1:8000>
* Interactive API docs: <http://127.0.0.1:8000/docs>

> **Hosted images:** the default backend sends each panel prompt to Hugging Face Inference
> Providers. Provider credits or billing may apply. To use a local Stable Diffusion model
> instead, set `IMAGE_GENERATION_PROVIDER=local` in `.env`; the first run downloads model
> weights and CPU generation can be slow.

### Fast test mode (no GPU, no download)
Set `USE_PLACEHOLDER_IMAGES=true` in `.env` and restart. Gemini still writes the real
story; images are simple labelled placeholders. Set it back to `false` for real art.

## 3. Test

**Automated (no API key/GPU needed):**
```bash
pytest -v
```

**Manual:**
1. Open <http://127.0.0.1:8000/health> → `"gemini_key_configured": true`.
2. Open <http://127.0.0.1:8000/test-image> → returns a JSON path; view it at `/<path>`.
3. On the home page submit the form → preview with 5 panels appears.
4. Click **Download Your Comic as PDF** → the PDF downloads and you land on the success page.
5. Scenario 2: regenerate with tone **Funny** and style **Comic Book** and compare.
6. API: in `/docs` open `POST /generate-comic/json` → *Try it out* with

   ```json
   {"prompt": "A brave fox exploring an enchanted forest", "character_name": "Ember",
    "setting": "forest", "tone": "funny", "style": "comic book"}
   ```

## Configuration (`.env`)

| Variable | Default | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | – | **Required** |
| `HF_API_KEY` | – | Required for hosted image generation; Inference Providers permission |
| `IMAGE_GENERATION_PROVIDER` | `hf-inference` | `hf-inference` (hosted) or `local` |
| `HF_IMAGE_MODEL` | `stabilityai/stable-diffusion-3-medium-diffusers` | Hosted image model |
| `GEMINI_FLASH_MODEL` | `gemini-3.5-flash-lite` | Outline model |
| `GEMINI_PRO_MODEL` | `gemini-3.5-flash-lite` | Story model |
| `SD_MODEL_ID` | `stable-diffusion-v1-5/stable-diffusion-v1-5` | Optional local image model |
| `SD_STEPS` | `25` | Diffusion steps (lower = faster) |
| `IMAGE_SIZE` | `512` | Image width/height |
| `USE_PLACEHOLDER_IMAGES` | `false` | Skip Stable Diffusion |

## Troubleshooting

* **`GEMINI_API_KEY is not set`** – `.env` must be in the project root (next to `requirements.txt`); restart the server after editing it.
* **`404 model not found`** – set `GEMINI_FLASH_MODEL` and `GEMINI_PRO_MODEL` to a model available to your key, such as `gemini-3.5-flash-lite`.
* **Hosted image generation returns 401/403** – create a fine-grained Hugging Face token with Inference Providers permission and set `HF_API_KEY` in `.env`.
* **Local image generation runs out of memory** – switch to the hosted backend with `IMAGE_GENERATION_PROVIDER=hf-inference`, or use placeholder mode.
* **Black/"Blocked by safety checker" image** – the Stable Diffusion safety filter flagged the prompt; rephrase the story.
* **Very slow** – expected on CPU; use `SD_STEPS=15` or the placeholder mode.
* **`ModuleNotFoundError: app`** – start uvicorn from the project root (the folder containing `app/`).
