# quizzy2.0: fixes

A browser quiz app: you paste a quiz written by an AI chat in a fixed text format (questions, an ANSWERS section, an optional DETAILED EXPLANATIONS section) and the page turns it into an interactive quiz.

## Logic & data

### A question that could not be read shifts every later answer (High)

**Why it's a problem.** A question whose options did not match the expected `a)` form was silently skipped, and the next questions were renumbered. The answer key still used the numbers the AI wrote, so from that point on every question was graded against the answer of the next one, and the final score was wrong with no error.

**AI-written code**

```js
if (line.match(/^Question\s+\d+/i)) {
    currentQuestion = {
        questionNumber: questions.length + 1,   // renumbered: ignores the number in the text
        questionText: '',
        options: []
    };
}
...
answers[questionNum - 1] = answerLetter;        // the key uses the AI's own numbers
...
this.correctAnswers = this.answers;             // matched to questions by position
```

**Fixed code**

```diff
-            if (line.match(/^Question\s+\d+/i)) {
+            const header = line.match(/^Question\s+(\d+)/i);
+            if (header) {
 ...
-                    questionNumber: questions.length + 1,
+                    questionNumber: parseInt(header[1], 10),
```

```diff
-            this.correctAnswers = this.answers; // Answers are now stored as letters directly
+            // Look answers and explanations up by each question's own number
+            this.correctAnswers = this.questions.map(q => this.answers[q.questionNumber - 1] ?? null);
+            this.explanations = this.questions.map(q => explanations[q.questionNumber - 1] ?? '');
```

```diff
-        document.getElementById('question-number').textContent = `${question.questionNumber}.`;
+        document.getElementById('question-number').textContent = `${this.currentQuestionIndex + 1}.`;
```

**Status.** Fixed (untested)

**Lesson.** Match data by an explicit key (the question number), not by position in two separate lists.

### The correct answer is shown, and then the answer can still be changed (High)

**Why it's a problem.** Picking an option immediately highlighted the correct one, but the user could then click the correct option and the score counted the last click. Any quiz could end with a perfect score, which makes the score meaningless.

**AI-written code**

```js
selectOption(letter, optionElement) {
    const wasAlreadyAnswered = this.selectedAnswers[this.currentQuestionIndex] !== null; // computed, never used
    document.querySelectorAll('.option').forEach(opt => opt.classList.remove('selected', 'correct', 'incorrect'));
    optionElement.classList.add('selected');
    this.selectedAnswers[this.currentQuestionIndex] = letter;   // overwritten on every click
    this.showInstantFeedback(letter);
}
```

**Fixed code**

```diff
-        // Check if this question was already answered
+        // The first choice counts: once the correct answer has been shown, it cannot be changed
         const wasAlreadyAnswered = this.selectedAnswers[this.currentQuestionIndex] !== null;
+        if (wasAlreadyAnswered) {
+            return;
+        }
```

```diff
         this.hideFeedback();
+
+        // An answered question shows its result again (the answer is locked)
+        const answered = this.selectedAnswers[this.currentQuestionIndex];
+        if (answered !== null) {
+            this.showInstantFeedback(answered);
+        }
```

**Status.** Fixed (untested)

**Lesson.** If a value is computed to guard a rule, make sure the code actually uses it.

## Reliability & quality

### The quiz crashes when a question has no usable answer (High)

**Why it's a problem.** If the ANSWERS section was missing a line, or used a format the parser did not accept (for example `1: b) Paris`), the correct letter was `undefined`. On a wrong pick the code called `correctLetter.toUpperCase()`, which threw, so no feedback was shown and the question stopped responding. Nothing warned at load time that the answers did not match the questions.

**AI-written code**

```js
const correctLetter = this.correctAnswers[this.currentQuestionIndex];   // may be undefined
...
correctAnswerText.textContent = `Correct answer: ${correctLetter.toUpperCase()}`;   // throws
```

**Fixed code**

```diff
+            // Every question needs a valid answer whose option exists, or grading breaks
+            const missing = this.questions
+                .filter((q, i) => !q.options.some(o => o.letter === this.correctAnswers[i]))
+                .map(q => q.questionNumber);
+            if (missing.length > 0) {
+                alert(`No valid answer found for question ${missing.join(', ')}. Please check the ANSWERS section (format "1: b").`);
+                return;
+            }
```

```diff
-                correctAnswerText.textContent = `Correct answer: ${correctLetter.toUpperCase()}`;
+                correctAnswerText.textContent = correctLetter ? `Correct answer: ${correctLetter.toUpperCase()}` : 'No answer given for this question';
```

**Status.** Fixed (untested)

**Lesson.** Check parsed data once, up front, and tell the user exactly what is missing.

### The parser accepts only one exact format (Moderate)

**Why it's a problem.** Section headers had to start a line (`**ANSWERS**`, which chat assistants often write, was not found), options had to be exactly `a)`, and any line after the options (such as an option that wraps onto a second line) was added to the question text. Small differences in AI output gave broken quizzes instead of a clear message.

**AI-written code**

```js
const answersMatch = text.match(/\nANSWERS\s*:?\s*\n/i);
...
if (line.match(/^[a-d]\)/i)) { /* option */ }
...
const match = line.match(/^(\d+)[:.]\s*([a-d])$/i);          // answers: "1: b" only
const match = line.match(/^Q(\d+):\s*(.+)$/i);               // explanations: "Q1: ..." only
```

**Fixed code**

```diff
+    // Removes Markdown that AI chats often add (**bold**, __bold__, # headings, > quotes)
+    cleanLine(line) {
+        return line.replace(/\*\*|__/g, '').replace(/^[#>\s]+/, '').trim();
+    }
```

```diff
-        const answersMatch = text.match(/\nANSWERS\s*:?\s*\n/i);
+        const answersMatch = text.match(/\n[ \t#*_]*ANSWERS[ \t*_]*:?[ \t*_]*\r?\n/i);
```

```diff
-                if (line.match(/^[a-d]\)/i)) {
+                // Options: a) text, a. text, (a) text, A) text
+                const option = line.match(/^\(?([a-d])[).]\s*(.*)$/i);
+                if (option) {
 ...
+                } else if (currentQuestion.options.length > 0) {
+                    // A line after the options continues the last option (wrapped text)
+                    currentQuestion.options[currentQuestion.options.length - 1].text += ' ' + line;
```

```diff
-            const match = line.match(/^(\d+)[:.]\s*([a-d])$/i);
+            // "1: b", also "1. B", "Q1: b", "1: b) Paris", "1: (b)"
+            const match = line.match(/^Q?(\d+)\s*[:.)]\s*\(?([a-d])\b/i);
```

**Status.** Fixed (untested)

**Lesson.** When the input comes from another program's free text, be tolerant in what you read and strict in what you check afterwards.

### Leftover console logging and unused functions (Moderate)

**Why it's a problem.** Every parse printed the questions, answers and explanations to the browser console. `extractCorrectAnswer` and a `copyPrompt` wrapper were never called, and a variable was computed and never used. Dead code makes the real behaviour harder to follow.

**AI-written code**

```js
console.log('Starting quiz parsing...');
console.log('Final parsing results:', { questionsCount, answers, explanations });
console.log('Parsed answers:', this.answers);

extractCorrectAnswer(answerLine) { /* never called */ }

// Legacy copy function (keeping for backward compatibility)
function copyPrompt() { copyGeneratedPrompt(); }                // never called
```

**Fixed code**

```diff
-        console.log('Starting quiz parsing...');
-        console.log('Section splits:', { ... });
-        console.log('Final parsing results:', { ... });
-            console.log('Parsed questions:', this.questions.length);
-            console.log('Parsed answers:', this.answers);
-    extractCorrectAnswer(answerLine) { ... }
-// Legacy copy function (keeping for backward compatibility)
-function copyPrompt() { copyGeneratedPrompt(); }
```

**Status.** Fixed (untested)

**Lesson.** Remove console logging and unused code before it becomes the thing people read first.

## UI / UX

### The "perfect score" glow stays on after the next quiz (Moderate)

**Why it's a problem.** The perfect-score styling was added to the score card on a perfect result and never removed. After one perfect run, every later result (a retake or a new quiz) still glowed as if it were perfect.

**AI-written code**

```js
if (isPerfectScore) {
    resultsTitle.textContent = 'Perfect Score';
    document.querySelector('.score-card').classList.add('perfect-score');   // never removed
}
```

**Fixed code**

```diff
         const isPerfectScore = correctCount === this.questions.length;
+        // clear the glow left over from an earlier perfect run
+        document.querySelector('.score-card').classList.remove('perfect-score');
```

**Status.** Fixed (untested)

**Lesson.** Whatever a state adds to the page, the next state must take away.
