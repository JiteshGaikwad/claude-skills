# Evaluations and Quality Management

Amazon Connect provides a comprehensive evaluation framework for assessing agent performance, automating quality reviews, and driving coaching workflows.

---

## Evaluation Forms

Evaluation forms define the criteria used to assess agent interactions. They consist of sections, questions, and scoring rules.

### Form Structure

```
Evaluation Form
  |-- Section 1: Greeting & Opening
  |     |-- Question 1.1: Did the agent greet the customer? (Yes/No)
  |     |-- Question 1.2: Did the agent verify identity? (Yes/No)
  |
  |-- Section 2: Data Collection
  |     |-- Question 2.1: Were all required fields collected? (1-5 scale)
  |     |-- Question 2.2: Was the account number confirmed? (Yes/No)
  |
  |-- Section 3: Script Adherence
  |     |-- Question 3.1: Did the agent follow the disclosure script? (Yes/No, critical)
  |     |-- Question 3.2: Was the hold procedure followed? (Yes/No)
  |
  |-- Section 4: Closing
        |-- Question 4.1: Did the agent summarize next steps? (Yes/No)
        |-- Question 4.2: Did the agent ask if there was anything else? (Yes/No)
```

### Question Types

| Type | Description |
|---|---|
| **Single select** | Choose one option from a predefined list. Can include scoring. |
| **Text** | Free-form text response from the evaluator. |
| **Numeric** | Numeric rating on a defined scale (e.g., 1-5, 1-10). |

### Scoring

- Each question can have a **weight** that determines its contribution to the overall score.
- Sections can be weighted relative to each other.
- The overall evaluation score is a weighted average expressed as a percentage (0-100%).

### Automatic Fail (Critical Sections)

- Sections or individual questions can be marked as **critical**.
- If a critical question receives a failing answer, the entire evaluation automatically fails regardless of the overall score.
- Common use: compliance questions (disclosure read, identity verified, prohibited phrases not used).

### Form Lifecycle

| State | Description |
|---|---|
| **Draft** | Form is being created or edited. Cannot be used for evaluations. |
| **Active** | Form is active and can be used for evaluations. Only one version of a form can be active. |
| **Inactive** | Form has been deactivated. Existing evaluations using this form are preserved but no new evaluations can be created with it. |

---

## Manual Evaluations

Quality analysts manually evaluate contacts by:

1. Searching for a contact via Contact Search.
2. Selecting an evaluation form.
3. Reviewing the recording/transcript.
4. Answering each question in the form.
5. Adding notes (per-question or overall).
6. Submitting the evaluation.

### Evaluation States

| State | Description |
|---|---|
| **Draft** | Evaluation started but not submitted. Can be edited. |
| **Submitted** | Evaluation completed and submitted. Read-only. |

---

## Automated Evaluations

Automated evaluations use generative AI and Contact Lens analytics to assess all (or a configured subset of) conversations at scale, without manual reviewer effort.

### How It Works

1. Define an evaluation form with questions suitable for automation.
2. Associate the form with automated evaluation rules.
3. Contact Lens analyzes each contact.
4. The AI evaluates the contact against the form questions using transcript, sentiment, categories, and other analytics.
5. The completed evaluation is stored with scores and AI-generated justifications.

### Capabilities

- **Scale** — Evaluate 100% of contacts automatically (vs. the typical 1-3% manual sample).
- **Consistency** — Every contact is assessed against the same criteria, eliminating evaluator bias.
- **Speed** — Evaluations available shortly after contact completion.
- **Coverage** — Identify issues across the entire contact volume, not just sampled contacts.

### Limitations

- Not all question types are automatable. Free-form judgment questions may require manual review.
- Accuracy depends on Contact Lens transcription quality and the clarity of evaluation criteria.
- Automated evaluations should be periodically validated against manual evaluations for calibration.

---

## Generative AI Recommendations

When performing manual evaluations, generative AI can pre-populate answers:

- The AI analyzes the contact transcript and analytics.
- For each question, it suggests an answer with a confidence level and justification.
- The evaluator can accept, modify, or override the recommendation.
- This accelerates manual evaluation while maintaining human oversight.

### Workflow

1. Open a contact for evaluation.
2. Select an evaluation form.
3. Click **Get AI recommendations**.
4. Review each recommendation with its justification.
5. Accept or override as needed.
6. Submit the evaluation.

---

## Screen Recording

Screen recording captures the agent's desktop during voice interactions, enabling reviewers to see exactly what the agent did during the contact.

### What It Captures

- Agent's CCP (Contact Control Panel) actions.
- Applications the agent accessed.
- Data entry and navigation.
- Screen content visible to the agent.

### Configuration

- Enabled per contact flow using the `Set recording and analytics behavior` block.
- Requires the Amazon Connect agent workspace or a supported CCP.
- Recordings are stored in the configured S3 bucket.

### Viewing

- Screen recordings are available on the Contact details page.
- Playback is synchronized with the audio recording and transcript.
- Reviewers can see agent actions alongside what was being said.

### Use Cases

- Verify data entry accuracy.
- Identify workflow inefficiencies.
- Detect unauthorized application usage.
- Training material creation.

---

## Coaching Workflows

Coaching connects evaluation results to agent development.

### Creating a Coaching Plan

1. From a completed evaluation, select **Create coaching plan**.
2. Include specific interaction examples (the evaluated contact).
3. Define coaching objectives and focus areas based on the evaluation scores.
4. Add notes and guidance for the agent.
5. Assign the coaching plan to the agent.

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
| `SearchContactEvaluations` | Search evaluations across contacts with filters (form, agent, score range, date range). |
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

---

## Best Practices

1. **Start with 3-5 sections** — Keep forms focused. Too many questions lead to evaluator fatigue and inconsistency.
2. **Use critical sections sparingly** — Reserve automatic fail for true compliance requirements, not quality preferences.
3. **Calibrate regularly** — Have multiple evaluators assess the same contacts and compare scores to ensure consistency.
4. **Combine automated + manual** — Use automated evaluations for coverage and manual evaluations for nuanced assessment.
5. **Close the loop** — Always create coaching plans for low scores. Evaluations without follow-up do not improve performance.
6. **Version forms carefully** — When updating criteria, create a new version rather than modifying the active form to preserve historical comparability.
