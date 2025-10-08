# Codeforces CLI Submitter with Live Verdict

This project provides a powerful shell function to submit solutions to Codeforces directly from your command line and receive a live verdict without ever leaving the terminal.

It is an enhanced version based on the original [CF-submitter-for-linux-cli](https://github.com/ansshhhhhh/CF-submitter-for-linux-cli), adding robust parsing, live API-based status checking, and a clean, professional interface.

---

## Features

* **Direct CLI Submission**: Submit solutions with a single command: `codeforces submit <filename>`.
* **Live Verdict**: The script polls the Codeforces API to give you a real-time verdict, from "Running on test..." to the final outcome.
* **Robust Parsing**: Automatically and correctly parses contest and problem IDs from filenames, including complex ones like `1822G1`.
* **Clean & Minimal Output**: A professional, clutter-free interface that only shows you what you need to see.
* **Silent Operation**: Runs silently in the background with no distracting shell job control messages.

---

## Demo

Here is what the final, clean output looks like in action:

```bash
$ codeforces submit 1822G1_MySolution.cpp
Running on test 15...

Submission #: 228491589
Status: Accepted
Time: 436 ms | Memory: 5120 KB
````

-----

## Prerequisites

Before you begin, ensure you have the following installed:

1.  **A Unix-like shell**: `bash` or `zsh`.
2.  **Node.js**: Required to run the helper scripts.
3.  **jq**: A command-line JSON processor used to parse the API response.
    ```bash
    # On Debian/Ubuntu
    sudo apt-get update && sudo apt-get install jq

    # On Fedora/CentOS
    sudo dnf install jq

    # On macOS (using Homebrew)
    brew install jq
    ```
4.  **CPH-Submit Browser Extension**: This extension handles the browser-side submission.
      * [Install for Google Chrome](https://www.google.com/search?q=https://chrome.google.com/webstore/detail/cph-submit/pgkjcmpmicpgaoffjgrplgpijboonjch)
      * [Install for Mozilla Firefox](https://addons.mozilla.org/en-US/firefox/addon/cph-submit/)
      * **Important**: After installing, make sure its native messaging host is properly set up.

-----

## Installation

### Step 1: Create a Helper Directory

All the helper scripts will live in a dedicated directory in your home folder.

```bash
mkdir -p ~/.cp-helper
cd ~/.cp-helper
```

### Step 2: Create the Helper Scripts

Create the two JavaScript files inside the `~/.cp-helper` directory.

#### `companion.js`

```javascript
#!/usr/bin/env node

const http = require('http');
const puppeteer = require('puppeteer');

const server = http.createServer(async (req, res) => {
    if (req.method === 'POST') {
        let body = '';
        req.on('data', chunk => {
            body += chunk.toString();
        });
        req.on('end', async () => {
            const payload = JSON.parse(body);
            try {
                const browser = await puppeteer.connect({
                    browserURL: '[http://127.0.0.1:9222](http://127.0.0.1:9222)',
                    defaultViewport: null
                });
                const [page] = await browser.pages();
                await page.evaluate(p => {
                    const port = chrome.runtime.connect({
                        name: "cph-submit"
                    });
                    port.postMessage(p);
                }, payload);
                res.end('OK');
            } catch (e) {
                console.error(e.toString());
                res.end('ERROR');
            } finally {
                server.close();
                process.exit();
            }
        });
    }
});

server.listen(4243, '127.0.0.1');
```

#### `post.js`

```javascript
#!/usr/bin/env node

const fs = require('fs');
const http = require('http');

const [file, url, problemName, langId] = process.argv.slice(2);
const source = fs.readFileSync(file, 'utf-8');

const payload = {
    url,
    problemName,
    langId,
    source,
};

const req = http.request({
    hostname: '127.0.0.1',
    port: 4243,
    path: '/',
    method: 'POST',
}, () => {});

req.write(JSON.stringify(payload));
req.end();
```

### Step 3: Add the Function to Your Shell Configuration

Copy the entire function below and paste it at the end of your shell configuration file (`~/.bashrc` for Bash, or `~/.zshrc` for Zsh).

```bash
# Codeforces CLI Submitter with Live Verdict
codeforces() {
  # --- Step 0: Initial checks and passthrough ---
  if [[ "$1" != "submit" ]]; then
    command codeforces "$@"; return;
  fi
  local file="$2"
  [[ -f "$file" ]] || { echo "Usage: codeforces submit <file>" >&2; return 1; }

  # --- Configuration ---
  # !!! ACTION REQUIRED: Replace "anshhhhh" with your Codeforces handle !!!
  local CF_HANDLE="anshhhhh"

  # --- Step 1: Robustly parse filenames and language ---
  local fname="$(basename "$file")"
  local base="${fname%.*}"
  local code="${base%%_*}"
  
  if [[ ! "$code" =~ ^([0-9]+)([A-Za-z][0-9]*)$ ]]; then
    echo "✖ Error: Could not parse Contest and Problem from '$fname'." >&2
    echo "  (Example format: 123A.cpp, 987G1.cpp, 1822g1.cpp)" >&2
    return 1
  fi
  local contest="${BASH_REMATCH[1]}"
  local idx_raw="${BASH_REMATCH[2]}"
  local idx="${idx_raw^^}" # Convert to uppercase for URL/API (e.g., g1 -> G1)

  local url="[https://codeforces.com/contest/$](https://codeforces.com/contest/$){contest}/problem/${idx}"
  local problemName="${contest}${idx}"
  local ext="${file##*.}" langId
  case "$ext" in
    cpp)  langId=54 ;;  # GNU G++17
    py)   langId=31 ;;  # Python 3
    java) langId=60 ;;  # Java 11
    *)    echo "✖ Unsupported ext: .$ext" >&2; return 1;;
  esac

  # --- Step 2: Submit the solution ---
  local submission_time_start=$(( $(date +%s) - 5 ))

  # Temporarily disable job monitoring to hide "[1] PID" messages
  set +m
  ~/.cp-helper/companion.js &>/dev/null &
  local CPID=$!
  # Re-enable job monitoring immediately after
  set -m
  
  sleep 0.2
  node ~/.cp-helper/post.js "$file" "$url" "$problemName" "$langId" > /dev/null 2_1
  
  if [ $? -ne 0 ]; then
    kill $CPID 2>/dev/null; return 1;
  fi

  local waited=0
  while kill -0 $CPID 2>/dev/null && (( waited < 100 )); do
    sleep 0.1; (( waited++ ));
  done
  if kill -0 $CPID 2>/dev/null; then kill $CPID; fi
  sleep 2

  # --- Step 3: Poll for the verdict using the API ---
  while true; do
    API_URL="[https://codeforces.com/api/contest.status?contestId=$](https://codeforces.com/api/contest.status?contestId=$){contest}&handle=${CF_HANDLE}&from=1&count=5"
    LATEST_SUB=$(curl -s "$API_URL" | jq ".result[] | select(.problem.index == \"$idx\" and .creationTimeSeconds >= $submission_time_start)" | jq -s '.[0]')

    if [ -z "$LATEST_SUB" ] || [ "$LATEST_SUB" = "null" ]; then
      echo -ne "Waiting for submission...\r"
      sleep 2
      continue
    fi
    
    local VERDICT=$(echo "$LATEST_SUB" | jq -r '.verdict')

    if [ "$VERDICT" != "TESTING" ]; then
      local TIME=$(echo "$LATEST_SUB" | jq -r '.timeConsumedMillis')
      local MEMORY=$(echo "$LATEST_SUB" | jq -r '.memoryConsumedBytes')
      local SUB_ID=$(echo "$LATEST_SUB" | jq -r '.id')
      
      echo -ne "\033[2K\r" # Clear the "Running on test..." line
      echo "Submission #: $SUB_ID"
      if [ "$VERDICT" = "OK" ]; then
        echo -e "Status: \e[32mAccepted\e[0m"
      else
        local FORMATTED_VERDICT=$(echo "$VERDICT" | sed 's/_/ /g' | awk '{for(i=1;i<=NF;i++) $i=toupper(substr($i,1,1)) tolower(substr($i,2));}1')
        echo -e "Status: \e[31m${FORMATTED_VERDICT}\e[0m"
      fi
      echo "Time: ${TIME} ms | Memory: $(($MEMORY / 1024)) KB"
      break
    else
      local TEST_COUNT=$(echo "$LATEST_SUB" | jq -r '.passedTestCount')
      echo -ne "Running on test ${TEST_COUNT}...\r"
    fi
    sleep 2
  done
}
```

### Step 4: Configure Your Handle ⚙️

This is the most important step. You need to tell the script what your Codeforces username is.

1.  Open the file you just edited (`~/.bashrc` or `~/.zshrc`).
2.  Find the `codeforces()` function you pasted.
3.  Locate this line near the top of the function:
    ```bash
    local CF_HANDLE="anshhhhh"
    ```
4.  Replace `"anshhhhh"` with your own Codeforces handle. For example, if your handle is `tourist`, the line should look like this:
    ```bash
    local CF_HANDLE="tourist"
    ```

### Step 5: Reload Your Shell

To make the new `codeforces` command available, reload your shell configuration.

```bash
source ~/.bashrc
# OR
source ~/.zshrc
```

-----

## Usage

To submit a file, simply use the `codeforces submit` command.

```bash
codeforces submit 1822G1_MySolution.cpp
```

Your filename **must** start with the problem code (e.g., `1822G1`, `1186A`, etc.). Anything after an underscore (`_`) will be ignored, allowing you to add descriptive names.
