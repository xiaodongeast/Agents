# Prompt Optimization Workflow with Human-in-the-Loop

## Overview

This workflow uses a **meta-learning approach** where one AI agent (the "Teacher") optimizes prompts for another AI agent (the "Student"), guided by human feedback in an iterative loop.

---

## Components

### 1. Teacher Agent (Prompt Optimizer)
- **Role**: Analyzes gaps between actual vs. desired outputs
- **Output**: Optimized prompts + rationale for changes
- **Memory**: Maintains conversation history across iterations

### 2. Student Agent (Execution Agent)
- **Role**: Executes tasks using the current optimized prompt
- **Lifecycle**: Recreated fresh each iteration with new prompt
- **Memory**: No history retention (tests prompt in isolation)

### 3. Human Expert
- **Role**: Provides test cases and preferred outcomes
- **Purpose**: Guides optimization direction through feedback

---

## Workflow Diagram

```
┌──────────────────────────────────────┐
│  Initialize Teacher Agent            │
│  Set initial task + example          │
└─────────────┬────────────────────────┘
              │
              ▼
┌──────────────────────────────────────┐
│  Teacher generates initial prompt    │
└─────────────┬────────────────────────┘
              │
              ▼
        ┌─────────────┐
        │ MAIN LOOP   │ ◄───────────┐
        └─────┬───────┘             │
              │                     │
              ▼                     │
┌──────────────────────────────────┐│
│ Create Student with new prompt   ││
└─────────────┬────────────────────┘│
              │                     │
              ▼                     │
┌──────────────────────────────────┐│
│ Human: Provide test task         ││
└─────────────┬────────────────────┘│
              │                     │
              ▼                     │
┌──────────────────────────────────┐│
│ Student: Execute task            ││
└─────────────┬────────────────────┘│
              │                     │
              ▼                     │
┌──────────────────────────────────┐│
│ Human: Provide preferred outcome ││
└─────────────┬────────────────────┘│
              │                     │
              ▼                     │
┌──────────────────────────────────┐│
│ Teacher: Analyze & optimize      ││
└─────────────┬────────────────────┘│
              │                     │
              ▼                     │
         Satisfied? ────No──────────┘
              │
              Yes
              ▼
           [Exit]
```

---

## Pseudo Code

```python
# ============================================
# SETUP PHASE
# ============================================

# Create teacher agent (persistent across iterations)
teacher_agent = create_agent(
    name="PromptOptimizationAgent",
    instructions="Expert in prompt engineering...",
    response_format=PromptOptimizationResult
)

# Initial task with example
task = """
Your task description here with examples.
Example: Input: [sample input]
         Expected Output: [sample output]
"""

# Get initial optimized prompt
conversation_thread = None
response = await teacher_agent.get_response(task, conversation_thread)
optimized_prompt = parse_json(response.content)
conversation_thread = response.thread  # Save for continuity


# ============================================
# OPTIMIZATION LOOP
# ============================================

while True:

    # --- Display teacher's current suggestions ---
    print("Teacher's Suggestion:")
    print(f"  Optimized Prompt: {optimized_prompt['optimized_prompt']}")
    print(f"  Rationale: {optimized_prompt['rationale']}")
    print(f"  Problems Fixed: {optimized_prompt['problem_identified']}")


    # --- Create student agent with current prompt ---
    student_agent = create_agent(
        name="ExecutionAgent",
        instructions=optimized_prompt['optimized_prompt'],  # Use teacher's prompt
        # Note: Created fresh each time, no memory
    )


    # --- Get test case from human ---
    user_task = input("User: Enter your test task (or 'exit'): ")

    if user_task == 'exit':
        break


    # --- Student executes the task ---
    student_output = await student_agent.get_response(user_task, thread=None)
    print("Student Output:", student_output.content)


    # --- Get human feedback ---
    preferred_outcome = input("Your preferred outcome: ")


    # --- Teacher analyzes and refines ---
    feedback = f"""
    Analyze the extraction results using your optimized prompt:

    **Input text**: {user_task}
    **Your prompt outcome**: {student_output.content}
    **User preferred outcome**: {preferred_outcome}

    Compare and provide your analysis.
    """

    response = await teacher_agent.get_response(
        feedback,
        conversation_thread  # Continue the conversation!
    )

    # Update for next iteration
    optimized_prompt = parse_json(response.content)
    conversation_thread = response.thread  # Maintain context
```

---

## Key Concepts

### 🔄 Thread Continuity
```python
# Teacher remembers everything:
conversation_thread = response.thread  # Saved after each teacher interaction

# Student gets fresh start:
student_output = await student_agent.get_response(task, thread=None)
```

### 🎯 Dynamic Agent Creation
```python
# Student recreated each loop with NEW prompt
student_agent = create_agent(
    instructions=optimized_prompt['optimized_prompt']  # Latest version
)
```

### 📊 Feedback Loop
```
Example → Teacher Optimizes → Student Tests →
Human Validates → Teacher Refines → Repeat
```

---

## Example Session

### Example 1: Text Summarization Task
```
Iteration 1:
───────────
Teacher: "Optimized Prompt: Summarize the text in 2-3 sentences..."
         "Rationale: Added conciseness and length constraints..."

User: [Provides long article about AI trends]
Student: "AI is growing. Many applications exist. Future looks bright."
User Preferred: "AI adoption accelerated in 2024 across healthcare and finance sectors.
                 Key innovations include multimodal models and improved reasoning.
                 Challenges remain in regulation and ethical deployment."

Teacher: "Problem: Missing domain specificity and concrete details..."

Iteration 2:
───────────
Teacher: "Optimized Prompt: Extract key facts with domain context and specific examples..."
         "Rationale: Added instructions for identifying concrete details and trends..."

User: [Provides another article]
Student: "Quantum computing advances in 2024 focused on error correction and scalability.
          IBM and Google achieved 1000+ qubit systems with improved stability.
          Commercial applications emerging in drug discovery and cryptography."
User Preferred: "Perfect!"

→ Exit
```

### Example 2: Data Extraction Task
```
Iteration 1:
───────────
Teacher: "Optimized Prompt: Extract structured data from unstructured text..."

User: "John Smith, age 45, visited on 03/15/2024 for annual checkup. BP: 120/80."
Student: "Name: John Smith, Age: 45"
User Preferred: "Name: John Smith | Age: 45 | Date: 03/15/2024 | Visit Type: Annual Checkup | BP: 120/80"

Teacher: "Problem: Missing fields and structured format..."

Iteration 2:
───────────
Teacher: "Optimized Prompt: Extract ALL mentioned fields using pipe-separated format..."

User: [Provides another patient note]
Student: "Name: Jane Doe | Age: 32 | Date: 04/20/2024 | Visit Type: Follow-up | BP: 118/75"
User Preferred: "Excellent!"

→ Exit
```

---

## Important Notes

| Aspect | Teacher Agent | Student Agent |
|--------|--------------|---------------|
| **Lifespan** | Entire session | Single iteration |
| **Memory** | Full conversation history | None (thread=None) |
| **Purpose** | Learn to write better prompts | Test current prompt |
| **Updates** | Instructions never change | Instructions updated each loop |



## Code Structure

```
debug_outcome.py
├── Teacher Agent Setup
│   ├── PromptOptimizationResult (response schema)
│   └── prompt_agent (instance)
│
├── Student Agent (created in loop)
│   └── execution_agent (recreated each iteration)
│
└── __main__
    ├── Initialize teacher with task
    ├── Loop:
    │   ├── Show teacher's suggestions
    │   ├── Create new student
    │   ├── Get user task
    │   ├── Student executes
    │   ├── Get preferred outcome
    │   └── Teacher analyzes & optimizes
    └── Exit when satisfied
```

---

## Why This Works

1. **Separation of Concerns**: One agent optimizes, one executes
2. **Iterative Refinement**: Each cycle incorporates new learnings
3. **Human Expertise**: Ground truth from domain experts
4. **Context Accumulation**: Teacher learns patterns across examples
5. **Isolated Testing**: Student has no bias from previous attempts

