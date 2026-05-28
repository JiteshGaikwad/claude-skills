# Workforce Management (WFM)

Amazon Connect Workforce Management provides ML-powered forecasting, capacity planning, scheduling, and adherence monitoring. It is a native capability within Amazon Connect -- no third-party WFM software required.

---

## Forecasting

Forecasting predicts future contact volume and average handle time using historical data from the Connect instance.

### How It Works

1. **Data ingestion** -- WFM reads historical contact records (volume, handle time, abandonment) from the Connect instance. Minimum 8 weeks of data recommended; more data improves accuracy.
2. **ML model training** -- Amazon Connect trains ML models on the historical data, automatically detecting patterns: day-of-week, time-of-day, seasonality, holidays, and trends.
3. **Forecast generation** -- the system generates forecasts for configurable periods (daily, weekly, up to 18 months out).
4. **Auto-update** -- forecasts are automatically refreshed daily as new data arrives.

### Forecast Outputs

| Output | Description |
|---|---|
| **Contact volume** | Predicted number of inbound contacts per interval (15-min, 30-min, or hourly). |
| **Average handle time (AHT)** | Predicted average handle time per interval. |
| **Confidence intervals** | Upper and lower bounds showing forecast uncertainty. |

### Visualization

Forecasts are displayed as graphs in the WFM console:

- Line charts showing predicted volume over time.
- Overlays of actual vs. predicted for past periods.
- Adjustable date range and granularity.

### Manual Overrides

Forecasters can manually adjust forecasts for known events:

- Marketing campaigns expected to spike volume.
- Planned outages or service disruptions.
- Holidays or special events not captured in historical data.

Overrides are applied as percentage adjustments or absolute volume changes to specific date ranges.

### Forecast Groups

Organize queues into forecast groups for aggregate forecasting:

- Group queues that share similar traffic patterns or are staffed by the same agent pool.
- Forecasts are generated per forecast group.
- Each queue can belong to only one forecast group.

---

## Capacity Planning

Capacity planning estimates long-term staffing requirements based on forecasts and service level goals.

### Planning Horizon

- Plan up to **18 months** into the future.
- Granularity: weekly or monthly.

### Inputs

| Input | Description |
|---|---|
| **Forecast** | Contact volume and AHT predictions from the forecasting module. |
| **Service level goal** | Target percentage of contacts answered within a threshold (e.g., 80% in 30 seconds). |
| **Shrinkage** | Percentage of scheduled time agents are unavailable (breaks, training, meetings, absenteeism). Typically 25-35%. |
| **Occupancy target** | Maximum agent utilization to prevent burnout. |

### Scenario-Based Optimization

Create multiple capacity plans with different assumptions:

- **Base scenario** -- standard forecast, current service level goals.
- **Growth scenario** -- forecast adjusted +20% for business growth.
- **Austerity scenario** -- higher service level threshold, lower headcount.
- **Seasonal scenario** -- holiday-adjusted forecast with temporary staff.

Compare scenarios side-by-side to make hiring and budget decisions.

### Output

- **FTE requirements** -- full-time equivalent headcount needed per week/month.
- **Hiring timeline** -- when to start hiring and training to meet future demand.
- **Cost estimates** -- based on configured agent cost rates.
- **Service level impact** -- projected service level at different staffing levels.

---

## Scheduling

Scheduling generates optimized agent schedules that balance service level targets, agent preferences, and business rules.

### Schedule Generation

1. **Select forecast group** -- choose which forecast group to schedule for.
2. **Define scheduling horizon** -- typically 1-4 weeks out.
3. **Configure optimization target**:
   - **Service Level** -- optimize to meet a percentage of contacts answered within threshold, per channel.
   - **Average Speed of Answer (ASA)** -- optimize to meet a target ASA, per channel.
4. **Run the optimizer** -- the system generates a schedule that minimizes staffing cost while meeting the target.

### Shift Profiles

Shift profiles are weekly schedule templates that define the allowable working patterns:

| Parameter | Description |
|---|---|
| **Shift start window** | Earliest and latest allowed start time (e.g., 7:00 AM - 9:00 AM). |
| **Shift duration** | Minimum and maximum shift length (e.g., 8-10 hours). |
| **Days on/off** | Number of working days per week and consecutive day-off requirements. |
| **Break rules** | Number, duration, and timing constraints for breaks (e.g., 15 min break after 2 hours, 30 min lunch between hours 4-5). |
| **Overtime** | Whether overtime is allowed, max overtime hours per week. |

Multiple shift profiles can exist for different agent groups (full-time, part-time, flexible).

### Staffing Groups

Staffing groups link agents to forecast groups:

- A staffing group contains a set of agents who can handle contacts for a specific forecast group.
- Agents can belong to multiple staffing groups (cross-trained agents).
- Each staffing group references a shift profile that governs scheduling rules.

### HR and Business Rules Compliance

The scheduler respects:

- **Minimum rest between shifts** -- configurable gap (e.g., 11 hours between shifts).
- **Maximum consecutive working days** -- prevent scheduling more than N days in a row.
- **Skill-based constraints** -- ensure agents with required skills are scheduled for specialized queues.
- **Time-off requests** -- approved time-off is blocked from scheduling.
- **Contractual hours** -- respect minimum and maximum weekly/monthly hours per agent.

### Schedule Publishing

After generation, schedules go through a review and publish workflow:

1. Scheduler reviews the generated schedule in the WFM console.
2. Scheduler can make manual adjustments (swap shifts, adjust break times).
3. Scheduler publishes the schedule.
4. Published schedules appear in the agent workspace (agents view their upcoming schedule).

---

## Schedule Adherence

Schedule adherence monitors whether agents follow their published schedules in real time.

### Adherence Metrics

| Metric | Formula |
|---|---|
| **Adherence percentage** | (Time in adherent status / Total scheduled time) x 100 |
| **Conformance** | (Total time worked / Total scheduled time) x 100 |
| **Non-adherent time** | Total duration agent was in a non-adherent status during scheduled activity |
| **Out of adherence events** | Count of times agent deviated from schedule |

### How It Works

1. The system compares the agent's actual status (from the CCP) to their scheduled activity for each time interval.
2. Each scheduled activity maps to one or more acceptable agent statuses:
   - "Scheduled: Productive" maps to Available, On Contact, ACW.
   - "Scheduled: Break" maps to Break (custom status).
   - "Scheduled: Training" maps to Training (custom status).
3. If the agent's actual status does not match an acceptable status for the current scheduled activity, they are flagged as non-adherent.

### Real-Time Dashboard

Supervisors view adherence in real time:

- Agent list showing current status, scheduled activity, and adherence state (adherent / non-adherent).
- Color-coded indicators (green = adherent, red = non-adherent).
- Drill-down to individual agent timeline showing scheduled vs. actual throughout the day.
- Alerts when agents are non-adherent beyond a configurable threshold (e.g., non-adherent for more than 5 minutes).

### Historical Adherence Reports

- Daily, weekly, and monthly adherence reports.
- Filter by team, staffing group, or individual agent.
- Trend analysis to identify chronic adherence issues.

---

## Productivity Metrics

Beyond adherence, WFM tracks:

| Metric | Description |
|---|---|
| **Agent occupancy** | Percentage of available time spent handling contacts. |
| **Idle time** | Time in Available status with no contact. |
| **Productive time** | Time handling contacts (talk + hold + ACW). |
| **Shrinkage** | Time in non-productive statuses (breaks, training, meetings, offline). |
| **Schedule efficiency** | Ratio of required staff to scheduled staff. |

---

## Personas and Permissions

WFM functionality is role-gated via security profiles:

### Administrator

- Full access to all WFM features.
- Configures forecast groups, staffing groups, shift profiles.
- Manages WFM settings and permissions.

### Forecaster

- Creates, views, and edits forecasts.
- Applies manual overrides.
- Publishes forecasts for scheduling.

### Scheduler

- Creates and publishes schedules.
- Manages time-off requests.
- Makes manual schedule adjustments.
- Views adherence data.

### Capacity Planner

- Creates and manages capacity plans.
- Runs scenario comparisons.
- Views long-term FTE projections.

### Agent

- Views their own published schedule in the agent workspace.
- Sees upcoming shifts, breaks, and activities.
- Submits time-off requests.
- Cannot view other agents' schedules or WFM configuration.

---

## Integration with Agent Workspace

Agents see their WFM schedule directly in the agent workspace:

- **Schedule view** -- calendar showing upcoming shifts with start/end times, break windows, and scheduled activities.
- **Today's timeline** -- visual bar showing the day's schedule with current position highlighted.
- **Next activity** -- notification of the next scheduled activity (e.g., "Break in 15 minutes").
- **Time-off requests** -- submit and track requests from the workspace.

The schedule view is read-only for agents. Only schedulers and administrators can modify schedules.

### Time Off Management
- **Allowances**: Set annual/monthly allowances per agent or group
- **Accrual**: Hours accrued per pay period, configurable rules
- **Carryover**: Max carryover hours, expiration dates
- **Request/Approval**: Agent submits → manager reviews → auto-updates schedule
- **CSV Upload**: Bulk import time-off balances

### Overtime Management
- **Rules**: Max hours per day/week, approval required
- **Voluntary vs Mandatory**: Agents can opt-in or be assigned
- **Scheduling**: System suggests agents based on availability and skills
- **Cost Tracking**: Overtime hours tracked in reports

### Surge Management
- Detect surges via real-time metrics (queue size, wait time spikes)
- Dynamic staffing: extend shifts, cancel breaks, activate on-call agents
- Voluntary overtime offers to available agents
- Queue overflow to backup queues

### Multi-Skill Scheduling
- Agents scheduled across multiple queues/skills
- Skill priority weighting in schedule optimization
- Migration from single-skill: gradual rollout recommended
- Impact on forecast accuracy during transition period
