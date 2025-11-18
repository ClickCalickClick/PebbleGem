Based on the feasibility assessment of the `claude-for-pebble` repository and the capabilities of the Gemini API, here is the step-by-step process to create **PebbleGem**.

This project will be **serverless**. The heavy lifting (API communication, Markdown stripping, and text chunking) will happen on the phone via **PebbleKit JS**, ensuring the Watch only receives clean, displayable text.

### Prerequisites
*   **Pebble SDK:** You will need the Pebble SDK (likely via the `pebble-dev/docker-sdk`) installed.
*   **Gemini API Key:** A valid API Key from Google AI Studio.

---

### Step 1: Project Setup & Renaming
First, we clone the base structure and detach it from the original identity.

1.  **Clone the Repository:**
    ```bash
    git clone https://github.com/breitburg/claude-for-pebble.git pebble-gem
    cd pebble-gem
    ```
2.  **Update `package.json` / `appinfo.json`:**
    *   Open `package.json` (or `appinfo.json` if present).
    *   Change `"uuid"` to a new random UUID (to prevent overwriting the original app).
    *   Change `"longName"` to "PebbleGem".
    *   Change `"companyName"` to your name.

---

### Step 2: Configure Settings (Clay)
The app uses "Clay" for the settings page (where users enter their API key). We need to rebrand this.

1.  **Locate the Config:** Look in `src/pkjs/config.js` or `src/pkjs/index.js` for the Clay configuration.
2.  **Update Fields:**
    *   Find the input field labeled "Anthropic API Key".
    *   Change the `label` to "Gemini API Key".
    *   Change the `id` or `messageKey` to `geminiApiKey` (optional, but helps clarity).
    *   *Note: If you change the key name here, you must update the C code to read the new key, or simply keep the internal key ID as `API_KEY` to save effort.*

---

### Step 3: The Core Logic (PebbleKit JS)
This is the most critical step. We are replacing the Anthropic API call with Google Gemini.

**File:** `src/pkjs/index.js`

You will replace the existing `XMLHttpRequest` logic. The Gemini API allows API keys in the URL query parameters, which is very stable for PebbleKit JS.

**The New Logic Implementation:**

```javascript
// Helper: Strip Markdown (Gemini sends **bold**, Pebble can't render it)
function cleanGeminiText(text) {
  return text
    .replace(/\*\*(.*?)\*\*/g, '$1') // Remove bold
    .replace(/\*(.*?)\*/g, '$1')     // Remove italics
    .replace(/```[\s\S]*?```/g, '[Code Block]') // Simplify code blocks
    .replace(/`/g, '')                // Remove inline code ticks
    .trim();
}

// Main Function: Call Gemini
function fetchGeminiResponse(prompt, apiKey) {
  var url = 'https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=' + apiKey;

  var xhr = new XMLHttpRequest();
  xhr.open('POST', url);
  xhr.setRequestHeader('Content-Type', 'application/json');

  xhr.onload = function() {
    if (xhr.status === 200) {
      try {
        var response = JSON.parse(xhr.responseText);
        
        // 1. Parse Gemini's deep JSON structure
        var rawText = response.candidates[0].content.parts[0].text;
        
        // 2. Clean Markdown for the Watch
        var cleanText = cleanGeminiText(rawText);

        // 3. Send to Watch (Using existing repo's chunking logic)
        // The repo likely has a function called sendChunkedMessage or similar.
        // We pass the cleanText to that function.
        sendToWatch(cleanText); 

      } catch (e) {
        console.log('Error parsing Gemini response: ' + e);
        sendToWatch("Error: Invalid JSON from Gemini.");
      }
    } else {
      console.log('Gemini API Error: ' + xhr.responseText);
      sendToWatch("Error: API returned " + xhr.status);
    }
  };

  var data = JSON.stringify({
    "contents": [{
      "parts": [{
        "text": prompt
      }]
    }]
  });

  xhr.send(data);
}
```

**Integration Note:**
You will need to find the event listener `Pebble.addEventListener('appmessage', ...)` in the original file and replace the call to the Anthropic function with `fetchGeminiResponse(dict.PROMPT, apiKey);`.

---

### Step 4: Buffer Management (Crucial)
Pebble apps crash if you send too much data at once. The `claude-for-pebble` repo handles this, but Gemini is more verbose.

1.  **Verify Chunk Size:** Ensure the `sendToWatch` function (or equivalent in the repo) splits strings into chunks smaller than 2000 bytes. A safe limit for Pebble is often **512 bytes per chunk** to prevent buffer overflows.
2.  **Modifying the Sender:**
    If the repo doesn't have robust chunking, implement a loop in JS:
    ```javascript
    function sendToWatch(text) {
      var chunkSize = 500; // Safe size
      var parts = text.match(new RegExp('.{1,' + chunkSize + '}', 'g'));
      
      // Logic to send parts[0], wait for ACK, then send parts[1]...
      // The original repo likely contains a "Queue" system. Reuse it.
    }
    ```

---

### Step 5: Update the C Code (UI)
The C code runs on the watch. We want to change the branding.

1.  **Search and Replace:**
    *   Search `src/c/` for string literals like "Claude", "Anthropic".
    *   Replace them with "Gemini".
2.  **Splash Screen:**
    *   If there is a loading screen text like "Waiting for Claude...", change it to "Asking Gemini...".
3.  **Memory Safety:**
    *   The C code relies on receiving AppMessages. Since we handled the Markdown stripping on the phone (JS), the C code receives plain text, which requires no extra processing power on the watch.

---

### Step 6: Build and Deploy
1.  **Build:**
    ```bash
    pebble build
    ```
2.  **Install:**
    Enable Developer Mode on your phone's Pebble app, connect to the developer IP, and install.
    ```bash
    pebble install --phone IP_ADDRESS_OF_PHONE
    ```
3.  **Configure:**
    Open the settings page on the Pebble app (it will load your new Clay config), paste your Gemini API Key, and save.

### Summary of Architecture Change

| Feature | Original (Claude) | New (PebbleGem) |
| :--- | :--- | :--- |
| **Endpoint** | `api.anthropic.com` | `generativelanguage.googleapis.com` |
| **Auth** | `x-api-key` header | `?key=` URL Query Param |
| **Payload** | `{"messages": [...]}` | `{"contents": [{"parts": [...]}]}` |
| **Parsing** | Markdown supported (maybe) | **Must strip Markdown in JS** |
| **Server** | None (Direct) | None (Direct) |