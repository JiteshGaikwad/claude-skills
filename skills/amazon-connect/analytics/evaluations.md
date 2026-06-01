# Evaluations and Quality Management

Amazon Connect provides a comprehensive evaluation framework for assessing agent and self-service interaction performance, automating quality reviews, and driving coaching workflows.

---

## Evaluation Forms

Evaluation forms define the criteria used to assess interactions. They consist of sections, questions, and scoring rules. You can create different forms for each business unit, queue, or for evaluating agent vs. self-service (bot/AI agent) interactions.

### Form Structure

```
Evaluation Form
  |-- Section 1: Greeting & Opening
  |     |-- Question 1.1: Did the agent greet the customer? (Single select)
  |     |-- Question 1.2: Did the agent verify identity? (Single select)
  |
  |-- Section 2: Data Collection
  |     |-- Question 2.1: Were all required fields collected? (Numeric 1-5)
  |     |-- Question 2.2: Was the account number confirmed? (Single select)
  |
  |-- Section 3: Script Adherence
  |     |-- Question 3.1: Did the agent follow the disclosure script? (Single select, critical)
  |     |-- Question 3.2: Was the hold procedure followed? (Single select)
  |
  |-- Section 4: Closing
        |-- Question 4.1: Did the agent summarize next steps? (Single select)
        |-- Question 4.2: Did the agent ask if there was anything else? (Single select)
```

### Question Types

| Type | Description |
|---|---|
| **Single selection** | Choose one option from a predefined list (e.g., Yes/No, Good/Fair/Poor). Can include scoring. |
| **Multiple selection** | Choose multiple answers from a list (e.g., products discussed, non-compliant behaviors). |
| **Text field** | Free-form text response from the evaluator. |
| **Number** | Numeric rating on a defined scale with configurable ranges (e.g., 1-5, 1-10). |
| **Date** | Date picker answer. |

### Conditional Questions

Questions can be conditionally enabled or disabled based on answers to other questions:

- A parent question must be **Single selection** or **Multiple selection** and cannot be optional.
- Configure one or more answer values that trigger the conditional question.
- Conditionally enabled questions are disabled by default; conditionally disabled questions are enabled by default.
- If Gen AI automation is enabled on a conditional question, it counts towards the Gen AI usage limit regardless of whether it was triggered.

### Question Instructions

The **Instructions to evaluators** field provides guidance for both human evaluators and generative AI. Clear, specific instructions improve both human consistency and AI accuracy.

### Scoring

#### Enabling Scoring

1. On the **Scoring** tab, check **Enable scoring**.
2. This enables score assignment on Single select answers and range-based scoring on Number questions.

#### Score Assignment

- **Single select**: Assign a score to each answer option.
- **Number**: Define ranges with scores (e.g., 0 interruptions = 10 points, 1-4 = 5 points, 5-10 = 1 point).
- **Automatic fail**: Any answer can be configured as **0 (Automatic fail)**. This can apply to the section, subsection, or entire form. The evaluation receives a zero score for the affected scope.

#### Weight Distribution

Weights determine each section/question's contribution to the overall score.

- **Weight by section**: Evenly distribute question weights within each section. Set section-level weights.
- **Weight by question**: Set individual question weights directly.
- When you change one weight, others auto-adjust so the total remains 100%.
- **Exclude optional questions from scoring**: Assigns all optional questions a weight of zero and redistributes among remaining questions.
- The overall evaluation score is a weighted average expressed as a percentage (0-100%).

### Form Lifecycle

| State | Description |
|---|---|
| **Draft** | Form is being created or edited (inactive, locked except while being worked on). Use **Save draft** to preserve work, or **Save and validate** to validate without activating. |
| **Active** | Published and available for evaluations. Only one version is active; completed evaluations retain the version they used. |
| **Locked** | A version that has been activated/published. It stays locked even after deactivation and becomes a **historical** version; you can activate a historical version to save it as a new version. (There is no "Inactive" state.) |

Activating a form makes it available to evaluators. The previous version is no longer selectable for new evaluations, but historical evaluations retain their form version.

---

## Manual Evaluations

Quality analysts manually evaluate contacts by:

1. Searching for a contact via Contact Search (or receiving a shared URL/task).
2. On the **Contact details** page, choosing **Evaluations**.
3. Selecting an evaluation form from the dropdown and clicking **Start evaluation**.
4. Reviewing the recording/transcript alongside the form.
5. Answering each question. Use section arrows to collapse/expand long forms.
6. Adding notes (per-question or overall).
7. Choosing **Save** to keep as Draft, or **Submit** to complete.

### Evaluation States

| State | Description |
|---|---|
| **Draft** | Evaluation started but not submitted. Can be edited, returned to later, or deleted. |
| **Submitted** | Evaluation completed and submitted. Can be edited only with Edit permission. Can be resubmitted. |

Optional questions can be skipped or marked as **Not applicable**. A confirmation warning appears before submitting with skipped optional questions.

---

## Automated Evaluations

Automated evaluations use generative AI (powered by **Amazon Bedrock**) and Contact Lens analytics to assess conversations at scale. They work for **voice and chat** contacts analyzed by conversational analytics (manual evaluations cover all types: voice/chat/email/task).

> **Gen-AI limits/availability:** fully-automated gen-AI answers and **Ask AI** recommendations are each capped at **10 questions per contact** (this cap does **not** apply to category- or metric-based automation). Gen-AI-filled evaluations are **not available** in **Africa (Cape Town), Asia Pacific (Mumbai), Asia Pacific (Seoul), or AWS GovCloud (US-West)**. Form languages: English, Spanish, Portuguese, French, German, Italian, Chinese, Japanese, Korean — cross-language evaluation is supported (e.g. fill an English form from a Spanish transcript); justifications default to English.

### How It Works

1. Define an evaluation form with questions suitable for automation.
2. Configure automation on each question (see automation methods below).
3. Enable **Enable fully automated submission of evaluations** toggle.
4. Activate the form.
5. Create a Contact Lens rule that triggers automated evaluation for matching contacts.
6. Contact Lens analyzes each contact, the AI evaluates against the form questions, and completed evaluations are stored with scores and AI-generated justifications.

### Three Automation Methods

#### 1. Contact Lens Categories (Single select and Multiple select)

- Map category presence/absence to answer values.
- Example: If category `ProperGreeting` is present, answer is "Yes".
- For optional questions: first check applicability via a category, then evaluate.
- Multiple selection: all conditions execute sequentially; multiple categories can each select an answer.
- Requires pre-configured Contact Lens rules that categorize contacts.

#### 2. Generative AI (Single select and Text field)

- AI analyzes the transcript using the question title and instructions.
- Provides an answer with justification.
- Clear, complete sentences in question titles and specific evaluation criteria in instructions improve accuracy.
- Cannot currently automate evaluations of self-service (bot/AI agent) interactions.

#### 3. Metrics (Number questions)

- Automatically fill numeric answers using Contact Lens metrics (sentiment score, non-talk time percentage, interruption count) or contact metrics (longest hold duration, number of holds, agent interaction duration).

### Capabilities

- **Scale** -- Evaluate 100% of contacts automatically (vs. typical 1-3% manual sample).
- **Consistency** -- Every contact assessed against the same criteria, eliminating evaluator bias.
- **Speed** -- Evaluations available shortly after contact completion.
- **Coverage** -- Identify issues across the entire contact volume.

### Limitations

- Not all question types are automatable. Free-form judgment questions may require manual review.
- Accuracy depends on Contact Lens transcription quality and the clarity of evaluation criteria.
- Automated evaluations should be periodically validated against manual evaluations for calibration.
- Gen AI automation on conditional questions counts toward usage limits even if not triggered.

---

## Generative AI Recommendations (Ask AI)

When performing manual evaluations, generative AI can pre-populate answers:

1. Open a contact for evaluation.
2. Select an evaluation form.
3. Click **Ask AI** / **Get AI recommendations**.
4. Review each recommendation with its confidence level and justification.
5. Accept, modify, or override as needed.
6. Submit the evaluation.

This accelerates manual evaluation while maintaining human oversight. Requires the **Evaluation forms - ask AI assistant** permission.

---

## Screen Recording

Screen recording captures the agent's desktop (all open apps, up to 3 monitors) during voice/chat/task contacts, synced with the audio and transcript on the Contact details page. It requires a **client application** installed on the agent device, instance Data-storage config, and a flow block.

**Full reference — client app install, system/network requirements, enablement, MP4/5fps specs, EventBridge status, and the `Screen recording - Access` / `Screen recording - Enable download button` permissions — is in [screen-recording.md](screen-recording.md).**

---

## Coaching Workflows

Coaching connects evaluation results to agent development.

### Creating a Coaching Plan

1. From a completed evaluation, select **Create coaching plan**.
2. Include specific interaction examples (the evaluated contact).
3. Define coaching objectives and focus areas based on the evaluation scores.
4. Add notes and guidance for the agent.
5. Assign a coach and participant(s).

### Agent Experience

1. Agent receives a notification about the coaching plan.
2. Agent reviews the evaluation, feedback, and interaction examples.
3. Agent can listen to the recording and read the transcript of the example interaction.
4. Agent acknowledges the coaching plan.
5. Agent can add their own notes and comments.

### Manager Tracking

- Track coaching plan status (pending, acknowledged, completed).
- View coaching history per agent.
- Correlate coaching with subsequent evaluation score improvements.

---

## Calibration Sessions

Calibration drives consistency/accuracy in how managers evaluate agent performance. It is gated by the **Evaluation forms - manage calibration sessions** permission ("create and manage calibration sessions"). *(AWS has no dedicated calibration documentation page — only the permission is documented; the operational mechanics below are the generally-understood usage, not verbatim from the admin guide.)*

- Admins create calibration sessions to compare how different evaluators score the same contact.

---

## Contact Sampling

Managers can randomly sample agents' contacts for evaluation:

- Select agents (e.g., all agents in a hierarchy).
- Specify sample size (e.g., 5 random contacts per agent).
- Define time range (e.g., last week).
- Requires **Sample contacts** permission.

---

## Rules Integration

Two rule types trigger automated evaluations (action: **Submit automated evaluation** → select a form). Rules apply only to **new** contacts — they cannot be run against past/stored conversations.

1. **Conversational analytics rule** (default) — event source **"A Contact Lens post-call analysis is available"** or **"...post-chat analysis is available"**. Identify contacts with conditions like Agents, Agent hierarchy, AI agent, Queues, Initiation method.
2. **Evaluation forms rule** — event source **"A Contact Lens evaluation result is available"**; condition is a specific answer or score on **another** evaluation form — enabling **chained** evaluations.

Notes: an automated evaluation **cannot override a manually submitted** one (it fails and logs to CloudWatch); automated results are marked **"submitted by Contact Lens automation"**; submitting multiple forms per contact requires multiple rules.

---

## Permissions

### Evaluation Form Permissions

| Permission | Description |
|---|---|
| **Evaluation forms - manage form definitions - Create** | Create new evaluation forms. |
| **Evaluation forms - manage form definitions - View** | View evaluation forms. |
| **Evaluation forms - manage form definitions - Edit** | Edit evaluation forms. |
| **Evaluation forms - manage form definitions - Delete** | Delete draft evaluation forms. |

### Evaluation Execution Permissions

| Permission | Description |
|---|---|
| **Evaluation forms - perform contact evaluations - View** | View submitted evaluations on accessible contacts (subject to tag-based access control). |
| **Evaluation forms - perform contact evaluations - Create** | Create new evaluations, view and edit draft evaluations. Also enables search evaluations by form, score, date, evaluator, status. |
| **Evaluation forms - perform contact evaluations - Edit** | Edit submitted evaluations. |
| **Evaluation forms - perform contact evaluations - Delete** | Delete draft and submitted evaluations. |
| **Evaluation forms - view my received evaluations** | Agents can search for and view completed evaluations they received (not drafts/under review/calibrations). |
| **Evaluation forms - ask AI assistant** | Access the Ask AI button for Gen AI recommendations during evaluations. |
| **Evaluation forms - manage calibration sessions** | Create and manage calibration sessions. |
| **Sample contacts** | Randomly sample agents' contacts for evaluation. |

### Coaching Permissions

| Permission | Description |
|---|---|
| **Coaching - my coaching sessions - View** | View sessions where you are coach or participant. |
| **Coaching - my coaching sessions - Create** | Create sessions with yourself as coach. |
| **Coaching - my coaching sessions - Edit** | Edit sessions where you are coach. |
| **Coaching - my coaching sessions - Delete** | Delete sessions where you are coach. |
| **Coaching - manage coaching sessions - View** | View any coaching session (admin/QA manager). |
| **Coaching - manage coaching sessions - Create** | Create sessions and assign any user as coach. |
| **Coaching - manage coaching sessions - Edit** | Edit any coaching session. |
| **Coaching - manage coaching sessions - Delete** | Delete any coaching session. |

### Additional Required Permissions

| Permission | Description |
|---|---|
| **Rules** | Create, view, edit, delete rules for contact categorization and automated evaluation triggers. |
| **Screen recordings - View** | View screen recordings on contact details page. |
| **Recorded conversations - Listen** | Listen to call recordings during evaluations. |

---

## APIs

### Form Management

| API | Description |
|---|---|
| `CreateEvaluationForm` | Create a new evaluation form in Draft state. |
| `UpdateEvaluationForm` | Update a draft evaluation form. |
| `ActivateEvaluationForm` | Activate a draft form, making it available for evaluations. |
| `DeactivateEvaluationForm` | Deactivate an active form. |
| `DescribeEvaluationForm` | Get details of an evaluation form. |
| `ListEvaluationForms` | List all evaluation forms in the instance. |
| `ListEvaluationFormVersions` | List versions of a specific evaluation form. |
| `DeleteEvaluationForm` | Delete a draft evaluation form. |

### Evaluation Operations

| API | Description |
|---|---|
| `StartContactEvaluation` | Start an evaluation for a specific contact using a specific form. Creates a Draft evaluation. |
| `UpdateContactEvaluation` | Update answers in a draft evaluation. |
| `SubmitContactEvaluation` | Submit a completed evaluation. |
| `DescribeContactEvaluation` | Get details of a specific evaluation. |
| `ListContactEvaluations` | List evaluations for a specific contact. |
| `SearchContactEvaluations` | Search evaluations across contacts with filters. |
| `DeleteContactEvaluation` | Delete a draft evaluation. |

### Search Filters

`SearchContactEvaluations` supports filtering by:

| Filter | Description |
|---|---|
| **EvaluationFormId** | Specific evaluation form. |
| **AgentId** | Specific agent. |
| **ScoreRange** | Minimum and/or maximum evaluation score. |
| **DateRange** | Evaluation submission date range. |
| **Status** | Draft or Submitted. |
| **AutomaticFail** | Whether the evaluation resulted in an automatic fail. |

---

## Data Model

### Evaluation Record

| Field | Type | Description |
|---|---|---|
| `EvaluationId` | String | Unique evaluation identifier. |
| `ContactId` | String | The evaluated contact. |
| `InstanceId` | String | Connect instance. |
| `EvaluationFormId` | String | The form used. |
| `EvaluationFormVersion` | Integer | Version of the form used. |
| `EvaluatorArn` | String | ARN of the evaluator (user or SYSTEM for automated). |
| `Score` | Object | Overall score percentage and per-section scores. |
| `Answers` | Map | Map of question ID to answer value, score, and notes. |
| `Notes` | Object | Overall evaluation notes. |
| `Status` | String | DRAFT or SUBMITTED. |
| `AutomaticFail` | Boolean | Whether any critical section triggered an automatic fail. |
| `CreatedTime` | ISO 8601 | When the evaluation was created. |
| `SubmittedTime` | ISO 8601 | When the evaluation was submitted. |
| `EvaluationSource` | String | Manual, assisted automation, or fully automatic. |
| `EvaluationType` | String | Standard or calibration. |
| `Resubmitted` | Boolean | Whether the evaluation was resubmitted. |
| `AcknowledgementStatus` | String | ACKNOWLEDGED or UNACKNOWLEDGED. |
| `EvaluatedParticipantRole` | String | Role of evaluated participant (agent, bot, AI agent). |

---

## Best Practices

1. **Start with 3-5 sections** -- Keep forms focused. Too many questions lead to evaluator fatigue and inconsistency.
2. **Use critical sections sparingly** -- Reserve automatic fail for true compliance requirements, not quality preferences.
3. **Write clear instructions** -- Specific, complete-sentence instructions improve both human and AI evaluation accuracy.
4. **Calibrate regularly** -- Have multiple evaluators assess the same contacts and compare scores to ensure consistency.
5. **Combine automated + manual** -- Use automated evaluations for coverage and manual evaluations for nuanced assessment.
6. **Close the loop** -- Always create coaching plans for low scores. Evaluations without follow-up do not improve performance.
7. **Version forms carefully** -- When updating criteria, create a new version rather than modifying the active form to preserve historical comparability.
