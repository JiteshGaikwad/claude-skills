# Amazon Connect Generative AI Features

## Overview

Amazon Connect integrates generative AI across multiple touchpoints in the contact center workflow -- from post-contact analysis to manager-level insights. These features are distinct from the agentic self-service and agent-assist capabilities covered in [connect-ai-agents.md](./connect-ai-agents.md) and focus on supervisory, quality, and operational use cases.

---

## Post-Contact Summaries

### How It Works

After a voice contact ends, Amazon Connect automatically generates a structured, concise summary of the conversation using generative AI. The summary is displayed on the Contact Control Panel (CCP) or custom agent desktop.

### Key Characteristics

- **Voice contacts only** -- summaries are generated from the Contact Lens conversation transcript
- **Displayed on CCP** -- appears in the contact details panel after the call ends, during After Contact Work (ACW)
- **Structured format** -- the summary follows a consistent structure:
  - **Issue** -- what the customer contacted about
  - **Outcome** -- how the issue was resolved (or not)
  - **Action items** -- any follow-up actions required
- **Concise** -- typically 3-5 sentences, designed to be scannable
- **Available via API** -- accessible through the Contact Lens `GetContactSummary` or contact search results

### Use Cases

- **ACW reduction** -- agents spend less time writing post-call notes; the summary captures the essentials automatically
- **Supervisor review** -- managers can quickly review what happened on a call without listening to the recording
- **Handoff context** -- when a contact is transferred or a follow-up is needed, the summary provides immediate context
- **Compliance documentation** -- structured record of what was discussed and agreed upon

### Configuration

Post-contact summaries are enabled at the instance level and require Contact Lens to be active on the contact flow:

1. Enable Contact Lens on the contact flow (Set recording and analytics behavior block)
2. Enable post-contact summary in the Connect admin console under Analytics and optimization
3. Summaries are generated automatically for all Contact Lens-enabled voice contacts

---

## Generative AI-Powered Evaluation Recommendations

### How It Works

When managers evaluate agent performance using evaluation forms, generative AI can automatically populate form fields with recommended scores and justifications based on the contact transcript and recording.

### Key Characteristics

- **Auto-populate evaluation forms** -- AI analyzes the transcript and suggests scores for each evaluation criterion
- **Justification text** -- each recommended score includes a text explanation referencing specific parts of the conversation
- **Manager review required** -- recommendations are suggestions, not final scores; the manager reviews and can override
- **Criteria-aware** -- the AI understands the evaluation form structure and maps conversation evidence to specific criteria

### Supported Evaluation Criteria

The AI can provide recommendations for common evaluation dimensions:

| Criterion | What AI Evaluates |
|-----------|-------------------|
| Greeting/Opening | Did the agent greet the customer professionally? |
| Active listening | Did the agent acknowledge and paraphrase customer concerns? |
| Problem resolution | Was the customer's issue resolved during the contact? |
| Hold/transfer handling | Were holds and transfers handled according to procedure? |
| Compliance | Did the agent follow required scripts and disclosures? |
| Closing | Did the agent summarize, confirm next steps, and close professionally? |
| Empathy | Did the agent demonstrate empathy and emotional awareness? |
| Knowledge | Did the agent demonstrate product/process knowledge? |

### Workflow

1. Manager opens a completed contact for evaluation
2. Selects an evaluation form
3. Clicks "Get AI Recommendations" (or recommendations auto-populate if configured)
4. AI fills in recommended scores and justification text for each applicable criterion
5. Manager reviews, adjusts scores as needed, adds manual notes
6. Manager submits the final evaluation

### Benefits

- **Consistency** -- reduces evaluator bias by providing an objective AI baseline
- **Speed** -- evaluations that took 15-20 minutes can be completed in 5 minutes
- **Coverage** -- enables evaluating a larger percentage of contacts since each evaluation takes less time
- **Calibration** -- AI recommendations serve as a calibration tool across evaluation teams

---

## AI-Powered Manager Assistance (Preview)

### How It Works

Managers can ask natural language questions about their contact center performance and receive AI-generated answers with data, diagnosis, and recommendations.

### Capabilities

- **150+ metrics accessible** -- the AI can query and reason across the full range of Connect metrics (service level, handle time, abandonment rate, agent performance, queue performance, etc.)
- **Natural language queries** -- managers ask questions in plain English instead of building reports
- **Diagnosis** -- the AI identifies root causes of performance issues (e.g., "Service level dropped because handle time increased on the Billing queue due to a system outage")
- **Recommendations** -- suggests specific recovery actions (e.g., "Consider adding 3 agents to the Billing queue for the next 2 hours" or "Enable callback on the Support queue to reduce abandonment")

### Example Queries

| Manager Question | AI Response Type |
|-----------------|------------------|
| "Why did our service level drop yesterday?" | Root cause analysis with contributing factors |
| "Which agents are struggling this week?" | Performance ranking with specific areas for improvement |
| "How is the Billing queue performing today?" | Real-time metrics snapshot with trend comparison |
| "What should I do about the high abandonment rate?" | Actionable recommendations with expected impact |
| "Compare this week's performance to last week" | Week-over-week delta analysis across key metrics |
| "Which queue needs the most attention right now?" | Priority ranking based on current performance gaps |

### Current Status

This feature is in **preview**. It is available in select regions and may have limitations:

- Response accuracy depends on the breadth of metrics data available
- Complex multi-factor questions may require follow-up prompts
- Recommendations are suggestions and should be validated by the manager

---

## Coaching Workflows

### How It Works

Managers create structured coaching plans from evaluation scorecards, and agents receive, acknowledge, and track these plans within the Connect agent workspace.

### Workflow

```
Evaluation Completed
      |
      v
Manager Reviews Scorecard
      |
      v
Manager Creates Coaching Plan
  - Selects areas for improvement
  - Adds specific guidance/instructions
  - Sets target date
  - Optionally links to training resources
      |
      v
Agent Receives Coaching Plan
  - Notification in agent workspace
  - Reviews the plan details
      |
      v
Agent Acknowledges Plan
  - Confirms they've read and understood
  - Can add notes/questions
      |
      v
Progress Tracking
  - Subsequent evaluations show trend
  - Manager monitors improvement
  - Plan can be marked complete
```

### Manager Actions

- **Create coaching plan** -- initiated from a completed evaluation scorecard
- **Select improvement areas** -- choose specific evaluation criteria that need improvement
- **Add guidance** -- write specific instructions, best practices, or reference materials
- **Set targets** -- define the expected score improvement and timeline
- **Track progress** -- view agent's subsequent evaluations to measure improvement
- **Close plan** -- mark the coaching plan as complete when targets are met

### Agent Actions

- **View coaching plans** -- see all active coaching plans in the agent workspace
- **Acknowledge plan** -- confirm receipt and understanding
- **Add notes** -- write responses, ask questions, or document self-reflection
- **Track own progress** -- view their evaluation trend over time relative to coaching targets

### Integration Points

- **Evaluation forms** -- coaching plans are linked to specific evaluations and criteria
- **Contact Lens** -- AI-identified patterns can trigger coaching recommendations
- **Performance dashboards** -- coaching plan status and progress are visible in supervisor dashboards
- **Notifications** -- agents receive notifications when new coaching plans are created or updated
