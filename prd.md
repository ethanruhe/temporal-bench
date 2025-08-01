This report outlines a pragmatic strategy for creating an automated benchmark to evaluate how effectively foundational LLMs can integrate Temporal into existing codebases.

---

## **Benchmark Strategy and File Structure**

The core strategy is to create an automated, repeatable benchmark that uses a "test case" driven approach. Each test case consists of a "before" code sample, a "golden" solution (after), and descriptive metadata. The benchmark script will iterate through these cases, prompt various LLMs to perform the refactoring, and use a powerful "grader" LLM to score the results based on semantic correctness.

This approach provides a quantitative, automated assessment of model capabilities without brittle, exact-match string comparisons.

### **File Structure**

The project will be organized in a clear, scalable directory structure and assumes a `~/.env` file:

temporal-bench/  
├── main.py                 \# Main benchmark runner script  
├── results/                \# Output directory for reports and raw model outputs  
│   └── report.md           \# Final summary report  
├── prompts/  
│   ├── refactor\_prompt.txt \# Master prompt for the code transformation task  
│   └── grading\_prompt.txt  \# Master prompt for the evaluation task  
└── test\_cases/  
    ├── go/  
    │   ├── 01\_hello\_world/  
    │   │   ├── before.go  
    │   │   ├── after.go  
    │   │   └── metadata.json  
    │   └── ...  
    ├── python/  
    │   ├── 01\_define\_workflow/  
    │   │   ├── before.py  
    │   │   ├── after.py  
    │   │   └── metadata.json  
    │   └── ...  
    └── (directories for java, php, typescript, dotnet, ruby)

* **main.py**: The orchestrator that loads tests, calls models, and generates the report.  
* **test\_cases/**: Contains subdirectories for each language. Each test case within a language folder includes:  
  * before.{ext}: The starting code.  
  * after.{ext}: The target "golden" implementation.  
  * metadata.json: A file containing a natural language description of the task, e.g., {"description": "Refactor the function to be a Temporal Activity with a custom retry policy."}.

---

## **Scoring Methodology**

To avoid the fragility of diff or exact string matching, we will employ an **LLM-as-a-Judge** pattern for scoring. A high-capability model (e.g., Claude 3 Opus or GPT-4o) will act as the "grader."

The grader will be given the original code (before), the golden solution (after), and the candidate model's output. It will be instructed to return a JSON object with a score and a justification.

### **Scoring Rubric**

* **2 (Full Credit):** The generated code is **semantically equivalent** to the golden solution. It correctly implements the required Temporal concepts. Minor differences in style, comments, or variable names are ignored.  
* **1 (Partial Credit):** The code attempts to use the Temporal SDK but has significant errors, such as incorrect API usage, wrong signatures, or logical incompleteness.  
* 0 **(No Credit):** The code is completely incorrect, hallucinates APIs, is not valid for the language, or fails to use Temporal at all.

### **Score Aggregation**

* Language Score: For each model, scores for a specific language are summed and normalized to a 0-100 scale.  
  Language Score=NumCaseslang​×2∑Scoreslang​​×100  
* Aggregate Score: The total score across all languages for a model, also normalized to 100\.  
  Aggregate Score=NumCasestotal​×2∑Scorestotal​​×100

---

## **Pseudocode for Scripts**

The benchmark will be driven by a main script and several helper functions.

### **main.py (Orchestrator)**

\`\`\`  
function main():  
    // 1\. Setup  
    API\_KEYS \= load\_api\_keys\_from\_env()  
    MODELS\_TO\_TEST \= \['gpt-4o-mini', 'claude-3-haiku', 'gemini-1.5-flash'\]  
    GRADER\_MODEL \= 'claude-3-opus'  
      
    // 2\. Load all test cases from the \`test\_cases\` directory  
    test\_cases \= discover\_test\_cases('test\_cases/')  
    results \= {}

    // 3\. Iterate through each test case and model  
    for case in test\_cases:  
        before\_code \= read\_file(case.before\_path)  
        after\_code \= read\_file(case.after\_path)  
        task\_desc \= read\_json(case.metadata\_path).description

        for model in MODELS\_TO\_TEST:  
            // 3a. Prompt the model to refactor the code  
            refactor\_prompt \= build\_refactor\_prompt(case.language, task\_desc, before\_code)  
            generated\_code \= call\_llm(model, refactor\_prompt)

            // 3b. Grade the generated code using the grader model  
            grading\_prompt \= build\_grading\_prompt(before\_code, after\_code, generated\_code)  
            grade\_json \= call\_llm(GRADER\_MODEL, grading\_prompt, response\_format='json')  
              
            // 3c. Store the score, rationale, and raw output  
            store\_result(results, case.name, model, grade\_json, generated\_code)

    // 4\. Calculate final scores and generate report  
    final\_scores \= calculate\_final\_scores(results)  
    generate\_markdown\_report(final\_scores, results)  
    print("Benchmark complete. View report at results/report.md")  
\`\`\`

### **Helper Functions**

* **load\_api\_keys\_from\_env()**: Reads .env and returns a dictionary mapping model families or specific models to their API keys.  
* **discover\_test\_cases()**: Traverses the test\_cases directory and returns a list of objects, each representing a test case with its file paths and language.  
* **call\_llm()**: A wrapper that takes a model name and prompt, selects the correct client library (OpenAI, Anthropic, Google), and returns the model's response. It handles authentication and error logging.  
* **calculate\_final\_scores()**: Aggregates the raw scores into the per-language and overall scores for the final report.  
* **generate\_markdown\_report()**: Formats the final and raw results into a human-readable markdown file.

---

## **Prompt Engineering**

Prompts will be stored in template files and populated by the script to ensure consistency.

### **Refactoring Prompt (prompts/refactor\_prompt.txt)**

This prompt is for the models being tested.

You are a principal software engineer specializing in Temporal.io. Your task is to refactor the provided {{language}} code to use the Temporal SDK.

\*\*Task:\*\* {{task\_description}}

Modify the "BEFORE" code to implement the task using correct Temporal concepts. Output only the final, complete code block. Do not include any explanations.

\---  
\*\*BEFORE CODE ({{language}}):\*\*  
\`\`\`{{language}}  
{{code}}

---

**REFACTORED CODE ({{language}}):**

\#\#\# Grading Prompt (\`prompts/grading\_prompt.txt\`)

This prompt is for the high-capability "grader" model.

\`\`\`text  
You are a principal engineer at Temporal. Evaluate an AI's attempt to refactor code to use Temporal based on semantic equivalence, not exact text match.

Compare the \`CANDIDATE\` code to the \`GOLDEN AFTER\` code.

\*\*Scoring Rubric:\*\*  
\- \*\*2 (Full Credit):\*\* Semantically equivalent and functionally correct.  
\- \*\*1 (Partial Credit):\*\* Attempts to use Temporal but has significant errors.  
\- \*\*0 (No Credit):\*\* Completely incorrect or fails to use Temporal.

Respond with a single JSON object with keys \`score\` (integer) and \`rationale\` (string).

\---  
\*\*\`BEFORE\` CODE:\*\*

{{before\_code}}

\---  
\*\*\`GOLDEN AFTER\` CODE (Ground Truth):\*\*

{{after\_code}}

\---  
\*\*\`CANDIDATE\` CODE (AI Generated):\*\*

{{candidate\_code}}

\---  
\*\*YOUR JSON EVALUATION:\*\*

---

## **Future Extensibility**

This benchmark is designed to be easily extended.

### **Adding New Test Cases**

To add a new test, an engineer simply creates a new folder in the appropriate language directory (e.g., test\_cases/typescript/04\_signals\_and\_queries/) and adds the before.ts, after.ts, and metadata.json files. The benchmark script will automatically discover and include it in the next run.

### **Future Refactoring**

As the benchmark matures, it can be enhanced:

1. **Objective Graders:** A pre-check step could be added to run a **linter or compiler** against the generated code. If it fails to compile or has linting errors, it could automatically receive a score of 0, providing a deterministic layer of validation before the more subjective LLM grading.  
2. **Code Generation Tasks:** The benchmark can be extended to test code generation from scratch. The metadata.json could include a "type": "generate" field. For these tasks, no before file would be used, and the prompt would be adjusted to ask for code generation based solely on the task description.  
3. **AST Analysis:** For ultimate rigor, the LLM grader could be supplemented or replaced with a system that parses both the golden and candidate code into **Abstract Syntax Trees (ASTs)**. Comparing the structure of these trees would provide a highly accurate, deterministic measure of semantic equivalence, though this is a significant engineering investment.