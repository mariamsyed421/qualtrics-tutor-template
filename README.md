# Qualtrics Tutor Template

A drop-in JavaScript engine for building interactive practice question interfaces inside Qualtrics surveys. Researchers paste one script into the survey header and get a fully styled tutoring interface — no coding required beyond swapping question IDs.

---

## What this does

- Renders **multiple choice**, **short answer**, and **code writing** questions inside any Qualtrics Text/Graphic question
- Shows **correct/incorrect feedback** after each answer, with optional explanatory text
- Displays an **objective signpost** above each question card (e.g. "Unit 1")
- Tracks a **question counter** ("Question 1", "Question 2") across multiple blocks
- Logs per-question correctness to Qualtrics **Embedded Data** for export

---

## Files

| File | What it is |
|---|---|
| `engine.html` | The tutor engine — paste contents into your survey header |

---

## Quick start

### Step 1 — Add your questions to the header

Open your survey in Qualtrics → **Look & Feel → General → Header → `<>` source view**

Paste the full contents of `engine.html`. At the bottom of the file, replace the sample `questionBank` array with your own questions (see [Generating your question bank](#generating-your-question-bank-with-an-llm) below).

### Step 2 — Set up Survey Flow

In **Survey Flow**, add an **Embedded Data** block above all your question blocks with one field:

```
questionIndex    (leave value blank)
```

Add one field per question for correctness logging:

```
q_001_correct
q_002_correct
... etc.
```

### Step 3 — Add one block per question

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

**Only change `id: "q_001"`** to match the question you want this block to show. Everything else is copy-paste identical across all blocks.

### Step 4 — Survey Flow layout

Your Survey Flow should look like this, top to bottom:

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

Each question in the bank is a JSON object. Required fields vary by question type.

### All question types

| Field | Value |
|---|---|
| `id` | Unique string, e.g. `"q_001"` |
| `objective` | Topic name shown above the card, e.g. `"unit_1"` |
| `type` | `"mcq"`, `"short_answer"`, or `"code"` |
| `prompt` | The question text |
| `feedbackType` | `"correct_answer"`, `"static"`, or `"none"` |

### If type is `"mcq"`

| Field | Value |
|---|---|
| `choices` | Array of 2–4 answer strings |
| `answer` | Must exactly match one entry in `choices` |

### If type is `"short_answer"` or `"code"`

| Field | Value |
|---|---|
| `answerKey` | The correct answer (exact match) |

### If feedbackType is `"static"`

| Field | Value |
|---|---|
| `staticExplanation` | Full explanation shown after answering |

### Example question bank

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

Instead of writing JSON by hand, paste the prompt below into Claude or ChatGPT with your questions at the bottom. It will output the exact code block to paste into your header.

---

### Ingestion Prompt

> Copy everything in the box below and paste it into Claude or ChatGPT. Add your questions at the bottom where it says `[PASTE YOUR QUESTIONS HERE]`.

```
Convert the following questions into a JSON array for a Qualtrics tutoring system.

RULES

Every question must have these fields:
- id — unique string, format: "q_001", "q_002", etc.
- objective — topic name, lowercase with underscores (e.g. "unit_1", "cell_biology")
- type — exactly one of: "mcq", "short_answer", "code"
- prompt — the full question text
- feedbackType — exactly one of: "correct_answer", "static", "none"

If type is "mcq":
- choices — array of 2 to 4 answer strings
- answer — must exactly match one entry in choices

If type is "short_answer" or "code":
- answerKey — the correct answer or a sample correct response

If feedbackType is "static":
- staticExplanation — a full explanation of why the correct answer is right

If feedbackType is "correct_answer" or "none":
- no additional fields needed

OUTPUT FORMAT

Output ONLY the window.Tutor.init({ questionBank: [...] }) call, starting with
window.Tutor.init({ and ending with }); — no explanation, no markdown fences, no preamble.

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
    }
  ]
});

MY QUESTIONS

[PASTE YOUR QUESTIONS HERE]
```

---

## Known limitations

- **Grading is exact match only** for short answer and code questions. LLM-based grading is a planned future addition.
- **Question counter** shows "Question N" without a total (e.g. "Question 1") because Qualtrics does not expose block count to JavaScript.
- **Question bank lives in the header**, so updating questions requires re-pasting the header.