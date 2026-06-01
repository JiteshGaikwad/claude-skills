# Queued Callback in Amazon Connect

Queued callback lets a customer keep their place in the queue without staying on the line during high wait times — an agent calls them back when it's their turn. There is **no dedicated "Create callback" block**: a callback is built from `Get customer input` → `Store customer input` → **`Set callback number`** → **`Transfer to queue`** (the **Transfer to callback queue** tab).

---

## How callbacks keep their place in queue

- The callback can stay in the **original inbound queue** or go to a **separate dedicated callback queue** (a dedicated queue gives clearer real-time reporting — you can see how many customers are waiting for callbacks).
- To preserve position when using a dedicated queue, set the callback queue at the **same priority** as the inbound queue in the routing profile. Connect evaluates routing profiles first; with equal priority, the **oldest call start time** is pushed first across all same-priority queues.
- Example: a call arrives at 10:00 and the customer requests a callback at 10:05 — Connect uses the **10:00** start time for ordering, not 10:05.

---

## Setup steps (overview)

1. **Set up a queue** specifically for callbacks (so real-time reports show how many customers await callbacks).
2. **Set up caller ID** — the name and phone number shown to customers when you call them back.
3. **Add the callback queue to a routing profile** so waiting contacts route to agents.
4. **Create the queued-callback flow** that offers the callback (see "Build the flow" below).
5. **Associate a phone number** with the inbound flow.
6. *(Optional)* **Callback creation flow** — runs when a callback is created; the contact is only enqueued if it has a `Transfer to queue` block. Use it to `Check contact attributes` (e.g. drop duplicates / resolved issues) before queuing, or to set a customer queue flow via `Set customer queue flow`.
7. *(Optional)* **Customer queue flow for callback** — runs if you choose a `Set customer queue flow` block for the **Set creation flow** option; can transfer the contact between queues. (You can also manually remove a callback with the `StopContact` API.)
8. *(Optional)* **Outbound whisper flow** — played to the customer after they pick up, before connecting to the agent (e.g. "Hello, this is your scheduled callback…").
9. *(Optional)* **Agent whisper flow** — played to the agent right after they accept, before joining the customer (e.g. "You're about to connect to John, who requested a refund…").
10. *(Optional)* **Caller ID** — what the customer sees when dialed; must be a number claimed in your instance. Reflected as the **system endpoint** in contact records. **Takes precedence** over the queue's outbound number.
11. **Dial mode** — choose **agent first** (default) or **customer first**.
    > **Customer first** is available **only when Next Generation Amazon Connect is enabled** for the instance. Disabling Next Gen after activating customer-first disables it. It is not available in the pay-per-feature pricing model.

---

## The routing process

1. The customer leaves their number; the contact is queued and routed to the next available agent.
2. After an agent **accepts the callback in the CCP**, Connect dials the customer.
   - If no agents are available, callbacks stay in queue up to **7 days** after creation, then Connect removes them automatically. (Remove manually with the `StopContact` API.)
3. If there's **no answer**, Connect retries up to the number of times you specify.
4. If the call goes to **voicemail**, it is considered **connected** (no further retries).
5. If the customer **calls again** while in the callback queue, it's treated as a **new call**. (To avoid duplicate callback requests, see the AWS Contact Center blog on preventing duplicate callbacks.)

---

## How queued callbacks affect queue limits

- Queued callbacks **count toward the queue size limit**. When the queue reaches its limit:
  - The next **callback** is routed to the **error branch**.
  - The next **incoming call** gets a **reorder tone** (fast busy — no transmission path available).
- Recommendation: set queued callbacks at **lower priority** than incoming calls, so agents only work callbacks when inbound volume is low.

---

## Build the flow

A basic queued-callback flow uses these blocks: **Get customer input**, **Store customer input**, **Set callback number**, **Play prompt**, **Transfer to queue**, **Disconnect/hang up**. You can build it as a Customer queue flow, Transfer to agent, or Transfer to queue flow type.

**Steps:**

1. **Routing → Contact flows**; select an existing flow or **Create flow**.
2. Add **`Get customer input`** and prompt for the choice, e.g. *"Press 1 to receive a callback. Press 2 to stay in queue."* Use **Add another condition** to add options **1** and **2**.
3. Add **`Store customer input`** and prompt for the callback number, e.g. *"Please enter your phone number."*
   - In **Customer input**, select **Phone number**, then:
     - **Local format** — customers call from the same country as your instance's AWS Region.
     - **International format / Enforce E.164** — customers call from other countries/regions.
4. Add **`Set callback number`**: set **Type = System**, and **Attribute = Store customer input** (this is the number Connect dials). *(You can also use System > Customer Number to call back the inbound number.)*
5. Add **`Transfer to queue`** and configure the **Transfer to callback queue** tab (see options below).

### Transfer to callback queue — options

| Option | Meaning | Example value |
|---|---|---|
| **Initial delay** | Time between the callback contact being initiated in the flow and the customer being put in queue for the next available agent. | `99` seconds |
| **Maximum number of retries** | Number of *retries* after the initial callback. `2` ⇒ up to **three** total attempts (initial + 2 retries). A retry happens only if it **rings with no answer**; voicemail counts as connected (no retry). **Double-check this value** — a high number (e.g. 20) means excessive work for agents and too many calls to the customer. | `2` |
| **Minimum time between attempts** | Wait time before retrying when the customer doesn't answer. | `10` minutes |

### Optional parameters (in Transfer to queue)

- **Set working queue** — transfer to a dedicated callback queue. If not set, Connect uses the queue set earlier in the flow.
- **Caller ID number to display** — what the customer sees on the callback.
- **Set creation flow** — control the experience of the callback contact (a new contact, separate from the inbound voice contact):
  - **Dial mode** — agent first (default) or customer first (Next Gen only; see `customer-first-cb`).
  - **Callback creation flow** — flow run when the callback contact is created. Requirements: must be the default **Contact flow (inbound)** type, and must include a **`Transfer to queue`** block to enqueue the contact. In it you can:
    - Use `Check contact attributes` (including customer profiles) to terminate duplicates or already-resolved issues.
    - Add `Set customer queue flow`; within that customer queue flow, combine the `Get metrics` block with `GetCurrentMetricData` to read queue wait time and, e.g., send an advance SMS telling the customer to expect a callback.

After building, configure the other branches and error handling. See the **Sample queue configurations flow** (new instances) or **Sample queued callback flow** (previous instances) for a complete example.

---

## Callbacks from chat, task, or email

The **Transfer to Callback** option in `Transfer to queue` also supports callbacks initiated from **chat, task, or email** contacts. For example, a customer reaching out after hours (via chat message or a webform that creates a task) can request a voice callback.

---

## Queued callbacks in real-time metrics

- A callback is initiated when the `Transfer to queue` block runs (creating the callback in a callback queue). After any **Initial delay**, it goes into the queue and shows in the **In queue** column on the Real-time metrics page until an agent is available. (Initial delay shifts the contact between the **Scheduled** and **In queue** metrics — see "How Initial delay affects Scheduled and In queue metrics".)
- When the callback connects to an agent, a **new contact record** is created; its **Initiation Timestamp** = when the callback was *initiated in the flow* (step 1), not when it connected.
- **Tip:** to see just how many customers are waiting for a callback, route callbacks to a **callback-only queue**. There is currently **no way to see the phone numbers** of contacts waiting for callbacks.

### Callback metrics

| Metric | Meaning |
|---|---|
| **Callback contacts** | Count of contacts initiated from a queued callback (how many customers opted for a callback). |
| **Callback contacts handled** | Callbacks initiated from a queued callback **and** handled by an agent (how many were answered). |
| **Callback attempts** (`SUM_RETRY_CALLBACK_ATTEMPTS`) | Contacts where a callback was attempted but the customer didn't pick up. |

---

## Learn more

- How **Initial delay** affects Scheduled vs. In-queue metrics (`scheduled-vs-inqueue`)
- Failed callback attempts (`failed-callback-attempt`)
- Real-time metrics example for a queued-callback flow (`queued-callback-example`)

(See also: `core/routing-and-queues.md` for queues/routing profiles, and `flows/blocks.md` for the `Set callback number` and `Transfer to queue` block references.)
