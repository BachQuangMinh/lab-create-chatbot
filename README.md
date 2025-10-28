# lab-create-chatbot

Simple Flask + Transformers web chat UI.

## Quick start

Prerequisites
- Python 3.10
- GPU recommended (model inference requires significant memory; see note below)
- Docker (optional)

Install and run locally
```sh
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
# Run the server (see note about host binding for Docker below)
python app.py
```

Open the UI in your browser:
- http://localhost:5000

Build & run with Docker
```sh
docker build -t lab-chatbot .
docker run -p 5000:5000 lab-chatbot
```

If you use Docker, ensure the Flask app listens on all interfaces (0.0.0.0). See the suggested change to [app.py](app.py) below.

## What each file does

- [app.py](app.py)  
  - Flask server and model code. Main symbols:
    - [`handle_prompt`](app.py) — POST handler for model requests.
    - [`model`](app.py) — the loaded Transformer model (facebook/blenderbot-400M-distill).
    - [`tokenizer.encode_plus`](app.py) — tokenization usage in the current code.
    - [`conversation_history`](app.py) — in-memory history list.
- [templates/index.html](templates/index.html)  
  - HTML page. Loads the frontend JS and CSS via Flask static routing.
- [static/script.js](static/script.js)  
  - Frontend logic:
    - [`sendMessage`](static/script.js) — collects user input and POSTs it as `{prompt: message}` to the server.
    - [`addMessage`](static/script.js) — renders messages in the chat.
  - Important: the fetch URL must point to `/chatbot` (see suggested change below).
- [static/css/style.css](static/css/style.css)  
  - Page styling for the chat UI.
- static assets: [static/Bot_logo.png](static/Bot_logo.png), [static/Error.png](static/Error.png), [static/favicon.ico](static/favicon.ico), [static/user.jpeg](static/user.jpeg)
- [requirements.txt](requirements.txt)  
  - Python dependencies.
- [Dockerfile](Dockerfile)  
  - Container recipe (exposes port 5000).
- [.gitignore](.gitignore), [.theia/settings.json](.theia/settings.json)

## How files interact (end-to-end)

1. Browser loads [templates/index.html](templates/index.html). That page includes:
   - CSS: [static/css/style.css](static/css/style.css)
   - JS: [static/script.js](static/script.js)

2. User types a message and submits:
   - [`sendMessage`](static/script.js) serializes the message as JSON `{prompt: message}` and makes a POST to `/chatbot`.

3. Server:
   - [`handle_prompt`](app.py) receives the POST, parses the JSON, constructs token inputs using [`tokenizer.encode_plus`](app.py), calls [`model`](app.py).generate(...), decodes tokens back to text and returns the generated text.

4. Frontend receives the text and [`addMessage`](static/script.js) displays it.

## Important notes & recommendations

- Current `static/script.js` contains a placeholder URL (`www.example.com/chatbot`). Change it to the relative endpoint `/chatbot` so it reaches the running Flask server. See suggested change below.
- The server currently uses in-memory [`conversation_history`](app.py). That grows unbounded and will break generation as sequences get long. Consider truncating history or using a rolling window.
- Model inference is resource heavy. Run on a machine with a GPU and sufficient VRAM. Running on CPU will be slow and may not fit memory.
- For multi-user production use, conversation history should be stored per-session or per-user (not the global `conversation_history` list).
- `tokenizer.encode_plus()` returns token IDs and masks and is useful when combining `history` and `input_text`. You can also use `tokenizer(text)` (the high-level API) in future refactors.

## Key ideas learned

- Use Flask to create a web application with a ChatGPT-like interface (see [app.py](app.py) and [templates/index.html](templates/index.html)).
- `tokenizer.encode()` vs `tokenizer.encode_plus()`:
  - `tokenizer.encode()` returns just token IDs.
  - `tokenizer.encode_plus()` returns extra fields like `attention_mask`, making it more suitable for feeding both history and input. In future you can use `tokenizer(text)` directly.
  - See usage in [app.py](app.py) (`tokenizer.encode_plus`).
- [`static/script.js`](static/script.js) sends the user's message as `{prompt: message}` and the server reads it in [`handle_prompt`](app.py) using Flask's request handling.
- The source must be run on a server with a GPU for reasonable performance and reliability when serving the model.

## Suggested code fixes

Change the POST URL in [static/script.js](static/script.js) from the placeholder to the app endpoint:
````js
// filepath: [script.js](http://_vscodecontentref_/0)
// ...existing code...
  async function makePostRequest(msg) {
-    const url = 'www.example.com/chatbot';  // Make a POST request to this url, which is to be replaced with the actual url of the model server
+    // Use the relative Flask endpoint so the browser reaches the running Flask app
+    const url = '/chatbot';
    const requestBody = {
      prompt: msg
    };
    // ...existing code...
  }
// ...existing code...
````
