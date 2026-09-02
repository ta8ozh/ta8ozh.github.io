---
title: RedactFlow
permalink: /tools/redactflow.html
---

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>RedactFlow</title>
  <meta name="description" content="RedactFlow is a simple browser-based tool for sanitizing logs and text by replacing sensitive information with reusable local replacement rules.">
  <meta name="keywords" content="log sanitizer, text replacer, redact logs, sensitive data remover, text replacement tool, log cleaner, privacy tool">

  <style>
    :root {
      color-scheme: dark;
      --bg: #0b1220;
      --panel: #111827;
      --border: #273449;
      --text: #e5e7eb;
      --muted: #94a3b8;
      --primary: #3b82f6;
      --primary-hover: #2563eb;
      --danger: #ef4444;
    }

    * { box-sizing: border-box; }

    body {
      margin: 0;
      min-height: 100vh;
      background:
        radial-gradient(circle at top left, rgba(59,130,246,.10), transparent 30%),
        var(--bg);
      color: var(--text);
      font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont,
                   "Segoe UI", sans-serif;
    }

    .container {
      width: min(1100px, calc(100% - 32px));
      margin: 40px auto;
    }

    .brand {
      display: flex;
      align-items: center;
      gap: 12px;
      margin-bottom: 8px;
    }

    .logo {
      width: 42px;
      height: 42px;
      border-radius: 12px;
      display: grid;
      place-items: center;
      background: linear-gradient(135deg, #2563eb, #60a5fa);
      color: #fff;
      font-weight: 900;
      font-size: 20px;
      box-shadow: 0 10px 30px rgba(37,99,235,.25);
    }

    h1 {
      margin: 0;
      font-size: clamp(26px, 4vw, 38px);
      letter-spacing: -0.02em;
    }

    .subtitle {
      margin: 0 0 24px 54px;
      color: var(--muted);
      line-height: 1.5;
    }

    .card {
      background: rgba(17, 24, 39, .94);
      border: 1px solid var(--border);
      border-radius: 16px;
      padding: 18px;
      box-shadow: 0 16px 40px rgba(0,0,0,.22);
      margin-bottom: 18px;
    }

    textarea,
    input {
      width: 100%;
      border: 1px solid var(--border);
      background: #0d1525;
      color: var(--text);
      border-radius: 10px;
      outline: none;
      transition: border-color .15s, box-shadow .15s;
    }

    textarea:focus,
    input:focus {
      border-color: var(--primary);
      box-shadow: 0 0 0 3px rgba(59,130,246,.15);
    }

    textarea {
      min-height: 360px;
      resize: vertical;
      padding: 16px;
      font: 14px/1.55 "SFMono-Regular", Consolas, "Liberation Mono", monospace;
      tab-size: 4;
    }

    input {
      padding: 11px 12px;
      font-size: 14px;
    }

    .section-title {
      font-size: 16px;
      font-weight: 700;
      margin-bottom: 14px;
    }

    .rules {
      display: grid;
      gap: 10px;
    }

    .rule-row {
      display: grid;
      grid-template-columns: minmax(0,1fr) auto minmax(0,1fr) auto;
      gap: 10px;
      align-items: center;
    }

    .arrow {
      color: var(--muted);
      font-size: 18px;
    }

    button {
      border: 0;
      border-radius: 10px;
      cursor: pointer;
      padding: 11px 15px;
      font-weight: 700;
      font-size: 14px;
      transition: transform .05s ease, background .15s ease;
    }

    button:active { transform: translateY(1px); }

    .primary {
      background: var(--primary);
      color: white;
    }

    .primary:hover { background: var(--primary-hover); }

    .secondary {
      background: #243047;
      color: var(--text);
    }

    .secondary:hover { background: #2e3c56; }

    .danger {
      background: rgba(239,68,68,.14);
      color: #fecaca;
      padding: 10px 12px;
    }

    .danger:hover {
      background: rgba(239,68,68,.24);
    }

    .actions,
    .text-actions {
      display: flex;
      flex-wrap: wrap;
      gap: 10px;
      margin-top: 14px;
    }

    .status {
      min-height: 22px;
      margin-top: 12px;
      color: var(--muted);
      font-size: 13px;
    }

    .status.success {
      color: #86efac;
    }

    .empty {
      padding: 14px;
      color: var(--muted);
      border: 1px dashed var(--border);
      border-radius: 10px;
      text-align: center;
    }

    .footer-note {
      margin-top: 18px;
      color: var(--muted);
      font-size: 12px;
      line-height: 1.5;
    }

    @media (max-width: 720px) {
      .subtitle { margin-left: 0; }

      .rule-row {
        grid-template-columns: 1fr;
      }

      .arrow {
        display: none;
      }

      .danger {
        width: 100%;
      }
    }
  </style>
</head>
<body>
  <main class="container">
    <div class="brand">
      <div class="logo">R</div>
      <h1>RedactFlow</h1>
    </div>

    <p class="subtitle">
      Clean logs and text instantly using reusable replacement rules stored in your browser.
    </p>

    <section class="card">
      <div class="section-title">Text</div>

      <textarea
        id="textInput"
        placeholder="Paste your log, text, or other content here..."
      ></textarea>

      <div class="text-actions">
        <button class="primary" id="replaceBtn">Apply All Replacements</button>
        <button class="secondary" id="copyBtn">Copy Text</button>
        <button class="secondary" id="clearTextBtn">Clear Text</button>
      </div>

      <div id="status" class="status"></div>
    </section>

    <section class="card">
      <div class="section-title">Replacement Rules</div>

      <div id="rules" class="rules"></div>

      <div class="actions">
        <button class="secondary" id="addRuleBtn">+ Add Rule</button>
        <button class="danger" id="clearRulesBtn">Delete All Rules</button>
      </div>

      <div class="footer-note">
        Replacement rules are stored locally in your browser using <code>localStorage</code>.
        Your pasted text is not uploaded anywhere.
      </div>
    </section>
  </main>

  <script>
    const STORAGE_KEY = "redactFlowRulesV1";

    const textInput = document.getElementById("textInput");
    const rulesContainer = document.getElementById("rules");
    const addRuleBtn = document.getElementById("addRuleBtn");
    const replaceBtn = document.getElementById("replaceBtn");
    const copyBtn = document.getElementById("copyBtn");
    const clearTextBtn = document.getElementById("clearTextBtn");
    const clearRulesBtn = document.getElementById("clearRulesBtn");
    const statusEl = document.getElementById("status");

    let rules = loadRules();

    function loadRules() {
      try {
        const saved = localStorage.getItem(STORAGE_KEY);

        if (!saved) {
          return [{ find: "", replace: "" }];
        }

        const parsed = JSON.parse(saved);

        return Array.isArray(parsed) && parsed.length
          ? parsed
          : [{ find: "", replace: "" }];
      } catch {
        return [{ find: "", replace: "" }];
      }
    }

    function saveRules() {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(rules));
    }

    function renderRules() {
      rulesContainer.innerHTML = "";

      if (!rules.length) {
        const empty = document.createElement("div");
        empty.className = "empty";
        empty.textContent = "No replacement rules yet.";
        rulesContainer.appendChild(empty);
        return;
      }

      rules.forEach((rule, index) => {
        const row = document.createElement("div");
        row.className = "rule-row";

        const findInput = document.createElement("input");
        findInput.type = "text";
        findInput.placeholder = "Text to find";
        findInput.value = rule.find ?? "";
        findInput.autocomplete = "off";
        findInput.spellcheck = false;

        findInput.addEventListener("input", () => {
          rules[index].find = findInput.value;
          saveRules();
        });

        const arrow = document.createElement("div");
        arrow.className = "arrow";
        arrow.textContent = "→";

        const replaceInput = document.createElement("input");
        replaceInput.type = "text";
        replaceInput.placeholder = "Replace with";
        replaceInput.value = rule.replace ?? "";
        replaceInput.autocomplete = "off";
        replaceInput.spellcheck = false;

        replaceInput.addEventListener("input", () => {
          rules[index].replace = replaceInput.value;
          saveRules();
        });

        const deleteBtn = document.createElement("button");
        deleteBtn.className = "danger";
        deleteBtn.textContent = "Delete";

        deleteBtn.addEventListener("click", () => {
          rules.splice(index, 1);
          saveRules();
          renderRules();
        });

        row.append(findInput, arrow, replaceInput, deleteBtn);
        rulesContainer.appendChild(row);
      });
    }

    function showStatus(message, success = false) {
      statusEl.textContent = message;
      statusEl.className = success ? "status success" : "status";

      clearTimeout(showStatus.timer);

      showStatus.timer = setTimeout(() => {
        statusEl.textContent = "";
        statusEl.className = "status";
      }, 3500);
    }

    addRuleBtn.addEventListener("click", () => {
      rules.push({ find: "", replace: "" });
      saveRules();
      renderRules();

      requestAnimationFrame(() => {
        const inputs = rulesContainer.querySelectorAll("input");

        if (inputs.length >= 2) {
          inputs[inputs.length - 2].focus();
        }
      });
    });

    replaceBtn.addEventListener("click", () => {
      let text = textInput.value;

      if (!text) {
        showStatus("Paste some text first.");
        return;
      }

      const activeRules = rules.filter(rule => rule.find !== "");

      if (!activeRules.length) {
        showStatus("There are no active replacement rules.");
        return;
      }

      let totalReplacements = 0;

      activeRules.forEach(rule => {
        const parts = text.split(rule.find);
        const count = parts.length - 1;

        if (count > 0) {
          totalReplacements += count;
          text = parts.join(rule.replace);
        }
      });

      textInput.value = text;

      if (totalReplacements > 0) {
        showStatus(`${totalReplacements} replacement${totalReplacements === 1 ? "" : "s"} applied.`, true);
      } else {
        showStatus("No matching text was found.");
      }
    });

    copyBtn.addEventListener("click", async () => {
      if (!textInput.value) {
        showStatus("There is no text to copy.");
        return;
      }

      try {
        await navigator.clipboard.writeText(textInput.value);
        showStatus("Text copied to clipboard.", true);
      } catch {
        textInput.select();
        document.execCommand("copy");
        showStatus("Text copied to clipboard.", true);
      }
    });

    clearTextBtn.addEventListener("click", () => {
      textInput.value = "";
      showStatus("Text cleared.");
    });

    clearRulesBtn.addEventListener("click", () => {
      if (!confirm("Delete all saved replacement rules?")) {
        return;
      }

      rules = [{ find: "", replace: "" }];
      saveRules();
      renderRules();
      showStatus("All replacement rules have been deleted.");
    });

    renderRules();
  </script>
</body>
</html>
