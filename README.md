# Codeforces Auto-Submit via CPH-Submit

A lightweight helper to submit solutions directly from Vim or shell using the CPH-Submit browser extension—**no extra files in this repo**, just copy the code blocks below into your local setup.

---

## 🚀 Quick Start

1. **Install Prerequisites**
   - **Browser Extension**: Install [CPH-Submit](https://github.com/agrawal-d/cph-submit) on Chrome or Firefox.
   - **Node.js** (v14+) and **Python 3** on your machine.
2. **Create a local helper directory** (optional):
   ```bash
   mkdir -p ~/.cp-helper
   ```
3. **Copy the three code blocks below** into files under `~/.cp-helper`:
   - **companion.js** (one-shot HTTP server)
   - **post.js** (payload builder)
   - **bashrc_snippet.sh** (your shell function)

   You can do this by opening each file and pasting the corresponding code block.

4. **Make scripts executable**:
   ```bash
   chmod +x ~/.cp-helper/companion.js ~/.cp-helper/post.js
   ```

5. **Source the shell snippet**. Add to the end of your `~/.bashrc` or `~/.zshrc`:
   ```bash
   # Include Codeforces auto-submit helper (all code in README)
   source ~/.cp-helper/bashrc_snippet.sh
   ```
   Then reload your shell:
   ```bash
   source ~/.bashrc  # or source ~/.zshrc
   ```

6. **Add Vim mapping**. In your `~/.vimrc` include:
   ```vim
   " Submit current file to Codeforces
     command! CFSubmit write | execute '!bash -ic "codeforces submit ' . shellescape(expand('%:p')) . '"'
   nnoremap <F5> :CFSubmit<CR>
   nnoremap <leader>cs :CFSubmit<CR>
   ```
   Then reload Vim:
   ```vim
   :source ~/.vimrc
   ```

## 🔧 Code Blocks

### 1) `companion.js`

> A one-shot HTTP server on port 27121. It queues one payload, waits for the extension poll, then clears and exits.

```js
#!/usr/bin/env node
// companion.js — one-shot CPH helper
const http = require('http');
const PORT = 27121;
let saved = { empty: true };

const server = http.createServer((req, res) => {
  let body = '';
  req.on('data', d => body += d);
  req.on('end', () => {
    // Queue POSTed payload
    if (req.method === 'POST' && body) {
      try { saved = JSON.parse(body); console.log('🗳 Queued:', saved.problemName); }
      catch (e) { console.error('⚠ JSON error:', e); }
    }
    // Respond
    res.writeHead(200, {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    });
    res.end(JSON.stringify(saved));

    // On extension poll, clear & exit
    if (req.headers['cph-submit'] === 'true') {
      saved = { empty: true };
      console.log('✅ Picked up — shutting down.');
      server.close(() => process.exit(0));
    }
  });
});

server.listen(PORT, () =>
  console.log(`🚀 Companion listening on http://localhost:${PORT}`)
);
```

Save this as `~/.cp-helper/companion.js`.

---

### 2) `post.js`

> Reads your source file, builds JSON payload, and POSTs to the companion server.

```js
#!/usr/bin/env node
// post.js — build and POST payload
const fs = require('fs');
const { spawnSync } = require('child_process');
const [,, file, url, problemName, langId] = process.argv;

// Read source
let source;
try { source = fs.readFileSync(file, 'utf8'); }
catch (e) { console.error('✖ Cannot read file:', file); process.exit(1); }

// Build payload
const payload = { empty: false, url, problemName, sourceCode: source, languageId: Number(langId) };

// POST via curl
spawnSync('curl', [
  '-s', '-X', 'POST',
  '-H', 'Content-Type: application/json',
  '--data-binary', JSON.stringify(payload),
  'http://localhost:27121'
]);

console.log(`📝 Posted ${file} → ${problemName}`);
```

Save this as `~/.cp-helper/post.js`.

---

### 3) `bashrc_snippet.sh`

> Defines the `codeforces` function that starts companion, invokes `post.js`, waits, and cleans up.

```bash
#!/usr/bin/env bash
# bashrc_snippet.sh — source this in your ~/.bashrc

codeforces() {
  if [[ "$1" != "submit" ]]; then
    command codeforces "$@"; return;
  fi
  local file="$2"
  [[ -f "$file" ]] || { echo "Usage: codeforces submit <file>" >&2; return 1; }

  # Parse contest & index
  local fname="$(basename "$file")"
  local base="${fname%.*}"
  local code="${base%%_*}"
  local contest="${code//[^0-9]/}"
  local idx="${code//[^A-Za-z]/}"; idx="${idx^^}"

  # Build URL & problemName
  local url="https://codeforces.com/contest/${contest}/problem/${idx}"
  local problemName="${contest}${idx}"

  # Map extension to programTypeId
  local ext="${file##*.}" langId
  case "$ext" in
    cpp)  langId=54 ;;  # GNU G++17
    py)   langId=31 ;;  # Python 3
    java) langId=60 ;;  # Java 11
    *)    echo "✖ Unsupported ext: .$ext" >&2; return 1;;
  esac

  # 1) Start companion
  ~/.cp-helper/companion.js &>/dev/null &
  local CPID=$!; sleep 0.2

  # 2) Post payload
  node ~/.cp-helper/post.js \
    "$file" \
    "$url" \
    "$problemName" \
    "$langId"

  # 3) Wait for exit
  local waited=0
  while kill -0 $CPID 2>/dev/null && (( waited < 100 )); do
    sleep 0.1; (( waited++ ));
  done
  if kill -0 $CPID 2>/dev/null; then
    kill $CPID; echo "⚠ Companion killed PID $CPID"
  else
    echo "🚀 Companion exited"
  fi
}
```

Save this as `~/.cp-helper/bashrc_snippet.sh`, then **source** it in your shell:

```bash
source ~/.cp-helper/bashrc_snippet.sh
```

---

## 📝 Troubleshooting

- **command not found**: ensure `~/.cp-helper` is in your PATH or use full paths.
- **Extension not polling**: verify CPH-Submit is enabled and allowed on `localhost:27121`.
- **Wrong problem index**: filenames must be `<contest><index>_<desc>.ext`, e.g. `1613c_Poisoned_Dagger.cpp`.

---

## 📄 License

This file is licensed under the Apache License. See the [LICENSE](LICENSE) file for details.

