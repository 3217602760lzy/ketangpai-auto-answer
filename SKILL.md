---
name: ketangpai-auto-answer
description: Automate answering quiz questions on ketangpai.com from an existing logged-in Chrome session. Use when Codex needs to extract all questions first, derive an answer key, compare current selections against target answers, click only the mismatched radio or checkbox options, wait for save confirmation toasts, re-check answers, and submit the quiz safely.
---

# Ketangpai Auto Answer

## Overview

This skill automates answering quizzes on ketangpai.com using Chrome control. The core workflow:
1. **Extract** all questions by navigating through them
2. **Analyze** each question to determine the correct answer
3. **Answer** by comparing current state vs target, only toggling mismatches
4. **Submit** with confirmation dialog handling

## Key Technical Notes

### Ketangpai DOM Quirks

- **Radio buttons** are custom `<div role="radio">` elements, NOT native `<input type="radio">`. Playwright's `.check()` does not work. Use `tab.dom_cua.click({ node_id })` instead.
- **Checkbox text** includes a special icon character `` before the option letter (e.g., `" A. 使用价值"`). When parsing [checked] state from DOM snapshots, match against the letter + period pattern (e.g., `A.`), NOT against the accessible name directly.
- **Save confirmation**: After each answer, the page shows a "作答保存成功" toast. Wait 2-3 seconds before the next interaction.
- **Page navigation**: Uses "上一题" / "下一题" text buttons.

## Workflow

### Phase 1: Setup

```js
const { setupBrowserRuntime } = await import("<plugin-root>/scripts/browser-client.mjs");
await setupBrowserRuntime({ globals: globalThis });
const chrome = await agent.browsers.get("extension");
await chrome.nameSession("🔎 自动答题");
const tab = await chrome.tabs.new();
await tab.goto("<quiz-url>");
await tab.playwright.waitForLoadState({ state: "load" });
await tab.playwright.waitForTimeout(2000);
```

### Phase 2: Extract All Questions

Navigate through all questions and record question text, type (单选题/多选题/判断题), and options:

```js
const allQuestions = [];
function extractQ(snap) {
  const lines = snap.split('\n');
  const r = { num: 0, type: "", question: "", options: [] };
  for (const l of lines) {
    const nm = l.match(/paragraph: "(\d+)"/);
    if (nm) r.num = parseInt(nm[1]);
    const tm = l.match(/heading "(.+)" \[level=2\]/);
    if (tm && tm[1].includes('题')) r.type = tm[1];
    const qm = l.match(/heading "(.+)" \[level=3\]/);
    if (qm) r.question = qm[1];
    if (l.includes('radio "') || l.includes('checkbox "')) {
      const m = l.match(/checkbox "(.+?)"/) || l.match(/radio "(.+?)"/);
      if (m && m[1].trim()) {
        const opt = m[1].replace(/^[]+\s*/, '');
        if (!r.options.includes(opt)) r.options.push(opt);
      }
    }
  }
  return r;
}

// First question
allQuestions.push(extractQ(await tab.playwright.domSnapshot()));

// Navigate through remaining questions
for (let i = 2; i <= totalQ; i++) {
  await tab.playwright.getByText("下一题").click();
  await tab.playwright.waitForTimeout(2000);
  allQuestions.push(extractQ(await tab.playwright.domSnapshot()));
}
```

### Phase 3: Determine Answers

Analyze each question to determine the correct answer. For single-choice, record the expected option index (0=A, 1=B, 2=C, 3=D). For multi-select, record which indices should be checked. For judgment (判断题), 0=对, 1=错.

Store in answer key objects:
```js
// Single choice & judgment answers: option letter → index {0,1,2,3}
var singleKey = { 1:3, 2:1, ... };

// Multi-select: array of indices to check
var multiKey = { 9:[0,1,2,3], 10:[0,1,2], ... };
```

### Phase 4: Navigate Back to Q1

```js
for (let i = 0; i < totalQ - 1; i++) {
  await tab.playwright.getByText("上一题").click();
  await tab.playwright.waitForTimeout(200);
}
```

### Phase 5: Answer Each Question

#### Single-choice / Judgment (radio buttons)

```js
async function ansRadio(correctIdx) {
  var d = await tab.dom_cua.get_visible_dom();
  var ids = [...d.matchAll(/node_id=(\d+)[^<]*?role[=:]/g)].map(m => m[1]);
  // Check what's currently selected via DOM snapshot
  var snap = await tab.playwright.domSnapshot();
  var checked = snap.split('\n').filter(l => l.includes('radio "') && l.includes('[checked]'));
  var currentIdx = -1;
  if (checked.length > 0) {
    var txt = checked[0];
    var m = txt.match(/radio "([A-D])/);
    if (m) currentIdx = m[1].charCodeAt(0) - 65;
    else if (txt.includes('radio "对" [checked]')) currentIdx = 0;
    else if (txt.includes('radio "错" [checked]')) currentIdx = 1;
  }
  // Only click if different (avoids unnecessary toggling)
  if (currentIdx !== correctIdx && ids.length > correctIdx) {
    await tab.dom_cua.click({ node_id: ids[correctIdx] });
    await tab.playwright.waitForTimeout(2000);
  }
}
```

#### Multi-select (checkboxes) — CRITICAL: compare, don't blindly toggle

```js
async function ansCheck(indices) {
  var d = await tab.dom_cua.get_visible_dom();
  var ids = [...d.matchAll(/node_id=(\d+)[^<]*?role[=:]/g)].map(m => m[1]);
  var snap = await tab.playwright.domSnapshot();
  // Find which checkboxes are currently [checked] by matching the letter pattern
  var checkedLines = snap.split('\n').filter(l => l.includes('checkbox "') && l.includes('[checked]'));
  
  for (var i = 0; i < Math.min(ids.length, 5); i++) {
    // Parse current state: check if this option letter appears in [checked] lines
    var letter = String.fromCharCode(65 + i); // A, B, C, D...
    var isChecked = checkedLines.some(function(l) {
      return l.includes(letter + '.') && l.includes('[checked]');
    });
    var wantChecked = indices.indexOf(i) >= 0;
    
    // Only toggle if current state differs from desired state
    if (isChecked !== wantChecked) {
      await tab.dom_cua.click({ node_id: ids[i] });
      await tab.playwright.waitForTimeout(400);
    }
  }
  await tab.playwright.waitForTimeout(2000);
}
```

#### Navigating to next question

```js
var nextBtn = tab.playwright.getByText("下一题");
var hasNext = await nextBtn.count(); // Check if button exists
if (hasNext > 0) {
  await nextBtn.click();
  await tab.playwright.waitForTimeout(3000);
}
```

**Important**: For the last question, there is no "下一题" button — it may show "最后一题" (Last Question) instead.

### Phase 6: Submit

```js
// Click submit
await tab.playwright.getByText("提交").click();
await tab.playwright.waitForTimeout(2000);

// Handle confirmation dialog
var dialog = await tab.getJsDialog();
if (dialog) {
  // JavaScript confirm dialog
  await dialog.accept();
} else {
  // HTML modal with 确定 button
  var confirmBtn = tab.playwright.getByText("确定", { exact: true });
  if (await confirmBtn.count() > 0) {
    await confirmBtn.click();
  }
}
await tab.playwright.waitForTimeout(3000);
```

### Phase 7: Cleanup

```js
await chrome.tabs.finalize({ keep: [{ tab: tab, status: "deliverable" }] });
```

## Common Pitfalls

### ❌ Toggling off already-checked answers
When using `dom_cua.click()` on a checkbox that is already checked, it UNCHECKS it. Always check current state first and compare.

### ❌ "对" text matching too many elements
`tab.playwright.getByText("对")` matches dozens of elements containing "对立", "相对", etc. For judgment questions, use `dom_cua` with node IDs instead.

### ❌ JSON.stringify affecting regex
`JSON.stringify(await tab.dom_cua.get_visible_dom())` escapes `"` to `\"`, breaking regex patterns. Work with the raw string from `get_visible_dom()` directly.

### ❌ Variable redeclaration in Node REPL
Top-level `const` / `var` / `function` declarations persist across `js` calls. Use unique names or reset the kernel with `js_reset` when conflicts arise.
