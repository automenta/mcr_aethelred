# Model Context Reasoner (MCR)

**Vision:** To create an adaptive neuro-symbolic reasoning system that intelligently selects, executes, and optimizes its own reasoning strategies to achieve maximum accuracy, reliability, and versatility. "Aethelred" means "well-advised" or "wise counsel," reflecting the system's ability to reason about its own reasoning process.

---

### **1. Core Architecture**

The system will be redesigned around a central **MCR Orchestrator**. This orchestrator manages a flexible workflow based on the user's request and a chosen "power level." It replaces a fixed pipeline with a dynamic, stateful process.


**Key Components:**

*   **API Gateway / User Interface:** The entry point for all user interactions (CLI, TUI, Web UI, API calls).
*   **MCR Orchestrator:** The heart of the system. It receives requests, consults the `Strategy Selector`, and directs the `Execution Engine` to run the chosen workflow.
*   **Strategy Selector:** A new, intelligent component responsible for choosing the optimal workflow strategy based on the request, context, and historical performance data.
*   **Execution Engine:** Executes the individual stages of a workflow, calling upon LLM providers, the Reasoner, and other programmatic tools.
*   **Knowledge Subsystem:**
    *   **Session Manager:** Manages the state of individual reasoning sessions (facts, history). Persisted via SQLite or Redis.
    *   **Ontology Manager:** Manages a library of persistent, global ontologies.
*   **Performance & Learning Subsystem:** A background system that logs all outcomes, analyzes performance, and provides data to the `Strategy Selector` and `Tuning Mode`.

---

### **2. The Abstract Workflow Metamodel**

This is the core abstraction that allows for flexible and evolving strategies. Any reasoning task is defined as a **Workflow**, which is a directed graph composed of `Stages` and `Decision Points`.

**Metamodel Primitives:**

*   **Artifact:** A piece of data produced or consumed by a stage (e.g., `NL_Text`, `SIR_JSON`, `Prolog_Clause`, `QueryResult`, `NL_Explanation`).
*   **Action:** An operation performed within a stage.
    *   `LLM_Generate`: Call an LLM with a specific prompt template.
    *   `Programmatic_Transform`: Deterministically convert one artifact to another (e.g., SIR_JSON -> Prolog_Clause).
    *   `Reasoner_Execute`: Run a query against the knowledge base.
    *   `Semantic_Compare`: Compare two artifacts for meaning (e.g., using embeddings or an LLM call).
*   **Stage:** A step in the workflow that executes an `Action` to transform input `Artifacts` into an output `Artifact`.
*   **Decision Point:** A node in the workflow that routes to different `Stages` based on the content of an `Artifact` (e.g., an LLM-based critique).
*   **Workflow:** A specific, concrete implementation of the metamodel, defining a sequence of stages and decision points to accomplish a task (e.g., `Assert`, `Query`).

**Example Workflow Implementations:**

1.  **`Direct-S1` Strategy (Basic Power Level):**
    *   `Stage 1`:
        *   **Input:** `NL_Text`
        *   **Action:** `LLM_Generate` (using `NL_TO_PROLOG_DIRECT` prompt)
        *   **Output:** `Prolog_Clause`

2.  **`Verified-SIR-R1` Strategy (Robust Power Level):**
    *   `Stage 1: Translate`
        *   **Input:** `NL_Text`
        *   **Action:** `LLM_Generate` (using `NL_TO_SIR` prompt)
        *   **Output:** `SIR_JSON`
    *   `Stage 2: Verify`
        *   **Input:** `SIR_JSON`
        *   **Action:** `LLM_Generate` (using `SIR_TO_NL` prompt)
        *   **Output:** `Generated_NL_Text`
    *   `Stage 3: Critique (Decision Point)`
        *   **Input:** `NL_Text`, `Generated_NL_Text`
        *   **Action:** `Semantic_Compare`
        *   **Output:** `Critique_Result` (Pass/Fail)
        *   **Routing:** If `Pass`, go to Stage 4. If `Fail`, go to Stage 5.
    *   `Stage 4: Synthesize (Success Path)`
        *   **Input:** `SIR_JSON`
        *   **Action:** `Programmatic_Transform` (JSON to Prolog)
        *   **Output:** `Prolog_Clause`
    *   `Stage 5: Clarify (Failure Path)`
        *   **Input:** `NL_Text`, `Generated_NL_Text`
        *   **Action:** `LLM_Generate` (using `GENERATE_CLARIFICATION_QUESTION` prompt)
        *   **Output:** `NL_Explanation` (to be shown to the user)

This metamodel is powerful enough to define any strategy, including hypothetical reasoning (which would add a `KB_Sandbox` action) or multi-hop explanations.

---

### **3. Learning to Choose and Evolve Strategies**

#### **Strategy Selector**

The `Strategy Selector` is the key to adaptive behavior.

*   **Inputs:**
    *   **Request Features:** The user's input text, length, complexity (e.g., number of clauses).
    *   **Context Features:** Current KB schema, session history, loaded ontologies.
    *   **Performance Data:** Historical success rates of different strategies for similar inputs.
    *   **User Preference:** The selected "Power Level" (`Fast`, `Balanced`, `Robust`).

*   **Mechanism:**
    *   **Level 1 (Heuristic):** Starts with a simple rule-based system. E.g., `IF input_length < 10 words AND all_predicates_in_schema THEN strategy = 'Direct-S1'`.
    *   **Level 2 (Learning):** The `Performance & Learning Subsystem` logs the outcome of every workflow. The selector uses this data to build a simple predictive model (e.g., a lightweight decision tree or a Bayesian model) that maps input features to the strategy with the highest probability of success.

#### **Training & Evolution Modes**

The system will support three operational modes to facilitate learning:

1.  **Online Mode (Standard Operation):**
    *   The system uses the `Strategy Selector` to perform tasks.
    *   **Implicit Learning:** Every success, failure, and user correction is logged by the `Performance & Learning Subsystem`. This data continuously refines the performance model used by the selector.
    *   **Prompt A/B Testing:** The system can be configured to occasionally use a variant prompt template (e.g., `NL_TO_SIR_v2`) and log its performance, allowing for gradual, automated prompt optimization.

2.  **Evaluation Mode (Offline Batch Processing):**
    *   **Purpose:** To rigorously benchmark strategies and prompts against a "golden dataset" of problems and expected outcomes.
    *   **Process:** The operator provides a dataset. The system runs *every* available strategy against each problem and generates a detailed report comparing accuracy, cost (token usage), and latency.
    *   **Outcome:** A performance matrix that clearly shows which strategies and prompts are best for which types of problems, providing crucial data for the `Strategy Selector`.

3.  **Tuning Mode (LLM-Assisted Evolution):**
    *   **Purpose:** To use the LLM to help the system evolve its own prompts.
    *   **Process:** An operator (or an automated agent) identifies a poorly performing prompt based on data from the other modes. The system then uses a "meta-prompt" to ask an LLM to fix it.
    *   **Meta-Prompt Example:**
        > "The following prompt template for 'NL_TO_QUERY' has a 40% failure rate when dealing with questions about inheritance. Here are 5 examples of user questions and the incorrect Prolog it generated. Please analyze the failures and provide a revised, more robust prompt template that would handle these cases correctly."
    *   **Outcome:** A new candidate prompt (`NL_TO_QUERY_v3`) is generated, which can then be benchmarked in `Evaluation Mode` before being deployed in `Online Mode`.

---

### **4. Supported Capabilities & Power Levels**

All existing and discussed features are supported through this new architecture.

**User-Selectable Power Levels:**

*   **`Fast`:** Prioritizes speed and low cost. Primarily uses the `Direct-S1` strategy. Good for simple, trusted inputs.
*   **`Balanced`:** The default mode. Uses the `Strategy Selector` to make an intelligent choice between simple and complex strategies.
*   **`Robust`:** Prioritizes accuracy above all. Always uses the most rigorous available strategy, such as `Verified-SIR-R1`, and will actively ask for clarification when ambiguous.

| Feature                 | Implementation in MCR v12.0                                                                                                                                                             |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Fact/Rule Assertion** | Handled by an `Assert` workflow. The specific stages depend on the chosen Power Level.                                                                                                    |
| **Querying**            | Handled by a `Query` workflow. The `Robust` version would include multi-hop explanation generation.                                                                                     |
| **Session Management**  | Core function of the `Knowledge Subsystem`, now with persistence.                                                                                                                       |
| **Ontology Operations** |
| `get/add/remove`        | Core functions of the `Ontology Manager`.                                                                                                                                               |
| `recommendations`       | A new feature. When a user asserts a fact, the `Orchestrator` can trigger a semantic search against ontology descriptions to suggest loading relevant ones.                               |
| **Hypothetical Mode**   | Implemented as a special `HypotheticalQuery` workflow that uses a `KB_Sandbox` action to create a temporary, isolated copy of the knowledge base for the query.                          |
| **Explainability**      | A core part of the `Robust` query workflow. The reasoner's proof trace is captured as an `Artifact` and fed to an `LLM_Generate` stage to produce the final `NL_Explanation`.             |
| **CLI / TUI / Demos**   | All these frontends will now interact with the `MCR Orchestrator`'s API, which abstracts away the complexity of the underlying strategy. They can pass a `powerLevel` parameter. |

This design creates a powerful, adaptive, and self-improving system. It formalizes the reasoning process into a flexible metamodel, enabling it to not only perform complex neuro-symbolic tasks but also to learn and optimize how it does so over time.



