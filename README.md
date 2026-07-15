# Qualtrics Tutor Template

A drop-in JavaScript engine for building interactive practice question interfaces inside Qualtrics surveys. Researchers paste one script into the survey header and get a fully styled tutoring interface with no coding required beyond swapping question IDs.

---

## What this does

- Renders **multiple choice**, **short answer**, and **code writing** questions inside any Qualtrics Text/Graphic question
- Shows **correct/incorrect feedback** after each answer — simple, static explanatory, or AI-generated
- Displays an **objective signpost** above each question card (e.g. "Unit 1")
- Tracks a **question counter** ("Question 1", "Question 2") across blocks
- Logs per-question correctness to Qualtrics **Embedded Data** for export

---

## Files

| File | What it is |
|---|---|
| `engine.html` | The tutor engine — paste contents into your survey header |

---

## Quick start

### Step 1 — Deploy the grading proxy (required for LLM feedback only)

If you want AI-generated feedback (`feedbackType: "llm"`), you need to deploy a small proxy that holds your Gemini API key securely. If you only use `"correct_answer"` or `"static"` feedback, skip this step.

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/YOUR_GITHUB_USERNAME/tutor-proxy&env=GEMINI_API_KEY&envDescription=Your%20Google%20Gemini%20API%20key&envLink=https://aistudio.google.com/app/apikey&project-name=tutor-proxy&repository-name=tutor-proxy)

1. Click the button above
2. Log in to Vercel (free account works)
3. Enter your Gemini API key when prompted — get one free at [aistudio.google.com](https://aistudio.google.com/app/apikey)
4. Click Deploy
5. Copy your deployment URL (e.g. `https://tutor-proxy-yourname.vercel.app`)
6. In `engine.html`, find `callLLMGrader` and replace the proxy URL with yours

---

### Step 2 — Add your questions to the header

Open your survey in Qualtrics → **Look & Feel → General → Header → `<>` source view**

Paste the full contents of `engine.html`. At the bottom, replace the sample `questionBank` array with your own questions. See [Generating your question bank](#generating-your-question-bank-with-an-llm) below, or write JSON directly using the schema in [Question bank format](#question-bank-format).

### Step 3 — Set up Survey Flow

In **Survey Flow**, add an **Embedded Data** block above all your question blocks:

```
questionIndex    (leave value blank)
```

Add one field per question for correctness logging:

```
q_001_correct
q_002_correct
... etc.
```

### Step 4 — Add one block per question

For each question, add a **Text/Graphic** question to your survey. In that question's JavaScript editor, paste:

```javascript
Qualtrics.SurveyEngine.addOnload(function() {
  this.hideNextButton();
  var qi = parseInt(Qualtrics.SurveyEngine.getJSEmbeddedData("questionIndex") || "0") + 1;
  var tutor = Tutor.renderQuestion({
    id: "q_001",              // ← CHANGE: match the question id in your bank
    container: this.questionContainer,
    onComplete: function(result) {
      Qualtrics.SurveyEngine.setJSEmbeddedData("questionIndex", String(qi));
      Qualtrics.SurveyEngine.setJSEmbeddedData(result.id + "_correct", result.correct ? "1" : "0");
    }
  });
  tutor.setupDisplay(qi);
  document.querySelector(".tutor-next-btn").onclick = function() {
    this.clickNextButton();
  }.bind(this);
});
```

**Only change `id: "q_001"`** per block. Everything else is identical across all blocks.

### Step 5 — Survey Flow layout

```
Embedded Data
  questionIndex =
  q_001_correct =
  q_002_correct =
  ...

Block: Question 1    ← Text/Graphic question with snippet (id: "q_001")
Block: Question 2    ← Text/Graphic question with snippet (id: "q_002")
...
```

---

## Question bank format

### Fields required for all question types

| Field | Value |
|---|---|
| `id` | Unique string, e.g. `"q_001"` |
| `objective` | Topic name shown above the card, e.g. `"unit_1"` |
| `type` | `"mcq"`, `"short_answer"`, or `"code"` |
| `prompt` | The question text |
| `feedbackType` | `"correct_answer"`, `"static"`, `"llm"`, or `"none"` |

### If type is `"mcq"`

| Field | Value |
|---|---|
| `choices` | Array of 2–4 answer strings |
| `answer` | Must exactly match one entry in `choices` |

### If type is `"short_answer"` or `"code"`

| Field | Value |
|---|---|
| `answerKey` | The correct answer (used for exact match grading) |

### If feedbackType is `"static"`

| Field | Value |
|---|---|
| `staticExplanation` | Full explanation shown after answering |

### If feedbackType is `"llm"` (requires proxy)

| Field | Value |
|---|---|
| `fullCredit` | Rubric describing what earns full credit |
| `partialCredit` | Rubric describing what earns partial credit |
| `sampleResponse` | A correct sample answer shown after feedback |

---

## Feedback types

| feedbackType | What it shows | Requires proxy? |
|---|---|---|
| `"correct_answer"` | Correct / Not quite + the right answer | No |
| `"static"` | Correct / Not quite + a pre-written explanation | No |
| `"llm"` | Full Credit / Partial Credit / Needs More Work + AI feedback + sample response | Yes |
| `"none"` | Nothing shown | No |

---

## Example question bank

```javascript
window.Tutor.init({
  questionBank: [
    {
      "id": "q_001",
      "objective": "unit_1",
      "type": "mcq",
      "prompt": "In a __ participants design, participants are randomly assigned to different groups.",
      "choices": ["within", "repeated", "independent", "between"],
      "answer": "between",
      "feedbackType": "correct_answer"
    },
    {
      "id": "q_002",
      "objective": "unit_1",
      "type": "short_answer",
      "prompt": "What is the major disadvantage of a within-participants design?",
      "answerKey": "carryover effects",
      "feedbackType": "static",
      "staticExplanation": "Within-participants designs are susceptible to order and carryover effects because participants experience all conditions sequentially."
    },
    {
      "id": "q_003",
      "objective": "unit_1",
      "type": "short_answer",
      "prompt": "Explain the third variable problem and give an example.",
      "feedbackType": "llm",
      "fullCredit": "Student correctly defines the third variable problem as a correlation between two variables that exists only because both are caused by a third variable, and provides a valid example.",
      "partialCredit": "Student shows partial understanding but explanation is incomplete or example is missing.",
      "sampleResponse": "The third variable problem occurs when two variables appear correlated but both are actually caused by a third variable. For example, ice cream sales and crime rates are correlated, but both are caused by hot weather, not each other."
    },
    {
      "id": "q_004",
      "objective": "unit_1",
      "type": "code",
      "prompt": "Write a line of Python code that assigns the value 5 to a variable called x.",
      "answerKey": "x = 5",
      "feedbackType": "correct_answer"
    }
  ]
});
```

---

## Generating your question bank with an LLM

Instead of writing JSON by hand, paste the prompt below into Claude, ChatGPT, or Gemini with your questions at the bottom. It will output the exact code block to paste into your header.

---

### Ingestion Prompt

> Copy everything in the box below and paste it into an LLM. Add your questions at the bottom where it says `[PASTE YOUR QUESTIONS HERE]`.

```
Convert the following questions into a JSON array for a Qualtrics tutoring system.

RULES

Every question must have these fields:
- id — unique string, format: "q_001", "q_002", etc.
- objective — topic name, lowercase with underscores (e.g. "unit_1", "cell_biology")
- type — exactly one of: "mcq", "short_answer", "code"
- prompt — the full question text
- feedbackType — exactly one of: "correct_answer", "static", "llm", "none"

If type is "mcq":
- choices — array of 2 to 4 answer strings
- answer — must exactly match one entry in choices

If type is "short_answer" or "code":
- answerKey — the correct answer (used for exact match grading)

If feedbackType is "static":
- staticExplanation — a full explanation shown after answering

If feedbackType is "llm":
- fullCredit — rubric describing what earns full credit
- partialCredit — rubric describing what earns partial credit
- sampleResponse — a correct sample answer shown after AI feedback

If feedbackType is "correct_answer" or "none":
- no additional fields needed

OUTPUT FORMAT

Output ONLY the window.Tutor.init({ questionBank: [...] }) call, starting with
window.Tutor.init({ and ending with }); — no explanation, no markdown fences, no preamble.
Do not use apostrophes or smart quotes inside any string values.

EXAMPLE OUTPUT

window.Tutor.init({
  questionBank: [
    {
      "id": "q_001",
      "objective": "unit_1",
      "type": "mcq",
      "prompt": "In a __ participants design, participants are randomly assigned to different groups.",
      "choices": ["within", "repeated", "independent", "between"],
      "answer": "between",
      "feedbackType": "correct_answer"
    },
    {
      "id": "q_002",
      "objective": "unit_1",
      "type": "short_answer",
      "prompt": "What is the major disadvantage of a within-participants design?",
      "answerKey": "carryover effects",
      "feedbackType": "static",
      "staticExplanation": "Within-participants designs are susceptible to order and carryover effects because participants experience all conditions sequentially."
    },
    {
      "id": "q_003",
      "objective": "unit_1",
      "type": "short_answer",
      "prompt": "Explain the third variable problem and give an example.",
      "feedbackType": "llm",
      "fullCredit": "Student correctly defines the third variable problem and provides a valid example.",
      "partialCredit": "Student shows partial understanding but explanation is incomplete or example is missing.",
      "sampleResponse": "The third variable problem occurs when two variables appear correlated but both are caused by a third variable."
    }
  ]
});

MY QUESTIONS

[PASTE YOUR QUESTIONS HERE]
```

---

## Known limitations

- **Short answer and code grading is exact match** unless `feedbackType: "llm"` is used.
- **Question bank lives in the header**, so updating questions requires re-pasting the header.
- **Question counter shows "Question N"** without a total because Qualtrics does not expose block count to JavaScript.
- **LLM feedback requires the proxy** — the Gemini API key cannot be stored safely in Qualtrics itself.