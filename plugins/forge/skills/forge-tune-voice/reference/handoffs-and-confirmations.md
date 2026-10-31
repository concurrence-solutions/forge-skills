# Handoffs, codes and spoken confirmations

Companion to `forge-tune-voice`. Three things callers notice quickly: whether they can reach a person, whether a code they read aloud is taken whole, and whether what the agent says it did matches what its tools did.

## Reaching a person: `voice_config.forwarding`

`forwarding` is opt-in. When it is null, the agent has no `forward_call` tool, so the agent cannot transfer a caller who asks for a person.

| Field | What it does |
|---|---|
| `forward_to` | The single number a transfer dials. Null on a dynamic service. |
| `warm_transfer` | Chooses a warm or cold transfer. Set it explicitly. |
| `approved_destinations` | On a dynamic service, the numbers the agent may choose from, each with a label. |
| `fallback_to` | The line the call is handed to when the platform cannot run the agent. |
| `hours`, `after_hours_destination` | Send transfers outside the staffed hours to one line. |
| `test_destinations` | The numbers a test call can transfer to. A call from a workspace test caller number reaches only these, unless the caller is also listed in `verification_callers`, which reach the live lines. |

A `--body` write replaces the whole `forwarding` object, so send every key you want to keep:

```bash
forge platform service voice-config <service-id> --env test-org --body '{
  "forwarding": {
    "forward_to": null,
    "warm_transfer": true,
    "approved_destinations": {"555-010-1234": "Scheduling desk"},
    "fallback_to": "555-010-1234"
  }
}'
```

Write real numbers in E.164 format. Each destination must also be on the line's allowed forwarding list.

### The fallback line

Set `fallback_to` on any service that should reach a person when the platform cannot run the agent.

- The call is handed to this line as a cold transfer, and it wins over a single `forward_to` number there.
- `forward_call` does not use it, and the agent does not see it unless you also list the number in `approved_destinations`.
- Put the number on the voice use case's allowed forwarding list before you save it. If the number is refused or does not answer, the call ends.
- Without `fallback_to`, a caller on a dynamic service has no line to be handed to when the platform cannot run the agent.

### After hours

On a dynamic service (`forward_to` null with `approved_destinations`), set `hours` and `after_hours_destination` together. Outside the staffed hours every `forward_call` dials that one line, whichever approved line the agent picked.

- `after_hours_destination` is one of the approved destinations.
- `hours` holds a `timezone` (an IANA name), `windows`, and optionally `closed_dates` and `closed_on_us_federal_holidays`. Each window has a `day` (`mon` to `sun`), an `open` time and a `close` time in 24-hour `HH:MM` form. A window runs from `open` up to, not including, `close`, and a weekday with no window is closed all day. A closed date, and a federal holiday when `closed_on_us_federal_holidays` is true, close the whole day.
- The clock is read when `forward_call` runs, in the time zone of the hours.
- Write one closed-office line and do not promise a department line outside the hours. Confirm the behavior on a test call.
- The `--body` rule above applies. Send every key you keep in `forwarding`, `fallback_to` included.

```json
"forwarding": {
  "forward_to": null,
  "warm_transfer": true,
  "approved_destinations": {"555-010-1234": "Scheduling desk"},
  "hours": {
    "timezone": "America/New_York",
    "windows": [{"day": "mon", "open": "09:00", "close": "17:00"}],
    "closed_dates": ["2026-12-25"],
    "closed_on_us_federal_holidays": true
  },
  "after_hours_destination": "555-010-1234"
}
```

## The announcement and the transfer that does not connect

- On `gpt_realtime`, the transfer dials after its announcement has finished playing. Caller speech over the announcement is read first: a request to stop cancels the transfer, an acknowledgement such as "okay, thanks" lets it go ahead, and a sound shorter than the service's `barge_in_min_speech_s` does neither. Keep the line that introduces `forward_call` to one short sentence, and test transfers on a quiet line.
- A transfer that goes through ends the agent's part of the call. A warm transfer that reaches voicemail, a fax, no answer or a failed dial, or whose answerer hangs up before the hand-off, puts the caller back with the agent. So does a cold transfer the platform cannot hand off, for example to a destination missing from the line's forwarding list. The agent is told to say so and keep helping. Write one authored line for that case and listen for it on a test call.
- State the transfer sentence in one place. A sentence quoted in several places gives the agent several versions to choose from.
- Do not write a rule that keeps the agent silent after the transfer tool whatever the result says. When the transfer fails, the caller is left in silence.

## Codes the caller reads in groups

A caller who reads a one-time code in groups, with pauses between them, can be answered after the first group. Neither `server_vad` nor `semantic_vad` reliably keeps the groups in one turn, and a silence shorter than the caller's pause splits them. A longer silence period merges the groups only if it exceeds the pause, and it delays every reply by as much. So settle it in the tool and the graph.

- Take the code through a tool argument. The model builds tool arguments from the audio it heard, and the written transcript can split or drop digits.
- Let the tool decide whether the code is complete. It accepts the digits heard so far and returns an explicit status for an incomplete code, with how many digits it holds. The Platform defines no code statuses, so the names are yours.
- Put the "all N digits" check in the tool's `description`. Return the incomplete status before the tool calls any provider that limits check attempts.
- Give the graph one rule for the incomplete status, and no other rule about incomplete codes. The rule keeps every digit heard across turns and never asks the caller to start over. Choose how the agent waits: say nothing and wait for more digits, or say how many digits it has and ask only for the rest.
- Do not also write a prompt rule that counts digits or asks for the whole code again. Two rules that answer an incomplete code in opposite ways make the agent loop, re-ask digits and talk over the caller.

The one-rule wording is a design recommendation, not a measured result. Test it with scripted calls at several pause lengths. A simulation mocks the tool, so check the contract with the endpoint test and a real conversation. Judge a code capture on the tool input in the call record (see `voice-verification.md`).

```bash
forge platform integration test <integration-id> <endpoint-name> --body '{"params":{"code":"123"}}' --json --env test-org
```

## Spoken confirmations: declare what an endpoint does

The Platform runs reply checks that compare what the agent says it did with what its tools did. A check can miss or over-match a phrasing, so test your wording on calls. An endpoint that declares what it does gives a done claim something to rest on.

Declare `declarations.writes` on every integration endpoint whose result the agent confirms to the caller.

| `kind` | Use it for |
|---|---|
| `booking`, `cancellation`, `reschedule` | Appointment changes |
| `message` | A message recorded for your team, such as a ticket, a handoff or a callback request |
| `other` | Any other change, such as sending a code or a form to the caller |

The API reference describes the other kinds, `none` and `varies`. Give each write a `success` rule that reads the vendor's own answer: `{"reported_by": "http_status"}`, `{"reported_by": "field", "pointer": "/status", "equals": "sent"}` or `{"reported_by": "graphql"}`. For an endpoint that sends a code, add this to the endpoint JSON:

```json
"declarations": {
  "writes": {
    "kind": "other",
    "success": {"reported_by": "field", "pointer": "/status", "equals": "sent"}
  }
}
```

```bash
forge platform integration endpoint-create <integration-id> --file endpoint.json --env test-org
```

- With no declaration, or a write declared `other`, a claim of a booking, cancellation or message is cut on voice and replaced by a short correction. A claim of another action, such as a code sent, can be checked against the writes that succeeded on the call.
- On text and platform-managed voice, `message` also makes the endpoint the tool for taking a message when a caller asks for a person and the conversation cannot transfer. Do not use it for a code send.
- `endpoint-create` writes a new endpoint. To change the declarations on an existing endpoint, use the web console or the API.

Declare `declarations.reads` as `{"patient_in": "none"}` on every endpoint and data query that returns only practice data, such as hours, locations, open slots or articles. With no read declaration an endpoint counts as returning a patient record, so the record-details check can treat a public answer from it as record details. An endpoint that returns a patient's record declares `patient_in` as `params` or as `response`, and those count as a record:

```json
{"patient_in": "params", "params": ["patient_id"]}
{"patient_in": "response", "records": "/patients", "patient": "/patient_id"}
```

With `params`, list the request parameters that name the patient, each a property of the endpoint's `input_schema`. With `response`, `records` is a JSON Pointer to the records in the response and `patient` is a JSON Pointer to the patient's identifier in each record. For a workspace data query, put the same `declarations` in the body of `forge platform data-query create` or `update`.

Declare `declarations.writes.kind` as `none` on a POST endpoint that only reads, so it retries like a read after a timeout. This changes retries only. Approval policy and role grants treat POST, PUT, PATCH and DELETE endpoints as writes whatever the declaration says.

Check an endpoint before a call depends on it:

```bash
forge platform integration test <integration-id> <endpoint-name> --body '{"params":{}}' --json --env test-org
```

The result reports whether the call wrote (`wrote`) and its `outcome`. The test makes the real call, so use a test recipient.
