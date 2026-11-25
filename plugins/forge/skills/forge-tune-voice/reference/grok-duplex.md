# Grok Duplex: what the family runs, its safety section and warm transfer

Companion to `forge-tune-voice`. `grok_duplex` is a voice family on xAI's Grok Voice where xAI's server takes every turn: it decides when the caller has finished and when the agent may be cut off. A service on it keeps its family settings in `voice_config.grok_duplex`, read and written with `forge platform service voice-config` like every other block (see `voice-settings.md`). This page needs forge 0.1.88 or newer, the release whose `voice-families list` shows each family's channels and emergency floor.

## Symptom to check

| What you see | Check first | Where to read more |
|---|---|---|
| A save is refused with `voice_family_graph_program` | The version set's agent runs a context graph | What the family runs |
| A save is refused with `voice_family_settings` on `declared_languages` | A language other than `en` or `es` | Languages |
| A save is refused with `voice_family_agent_version` | `session_provider` set on the agent version | What the family runs |
| A save is refused with `voice_family_residency` | The workspace has regional data residency | What the family runs |
| The agent replies in the wrong language | `grok_duplex.reply_language`, `declared_languages` | Languages |
| A barge-in, held-turn, filler or silence setting changes nothing | The key is one the family does not read | Settings the family does not read |
| A crisis line, 911 reply or emergency ending is never spoken | The key is one the family does not read | Settings the family does not read |
| A write ran without the caller agreeing | The prompt's confirmation rule | What the family runs |
| On a warm transfer, the person who answers hears nothing about the caller | `grok_duplex.safety_block` with `forwarding.warm_transfer` | The pairing rule, then Check a service for the pair |
| An outbound call or an operator takeover is refused | The family's capabilities | Not supported |

## Read the family and the service

Every command here only reads. Save what you read before any change (Step 2 in `SKILL.md`).

```bash
forge platform voice-families list --env test-org
forge platform voice-families list --json --env test-org > voice-families.json
jq '(.families? // .)[] | select(.id == "grok_duplex") | {availability, capabilities, models}' voice-families.json
forge platform service voice-config <service-id> --get --json --env test-org > service-voice.json
jq '{session_provider, grok_duplex, forwarding}' service-voice.json
forge platform service get <service-id> --json --env test-org > service.json
jq '{declared_languages, contact_lines, crisis_call_911_reply, medical_emergency_ending, emergency_floor}' service.json
```

- `capabilities` shows `served_languages`, `channels`, `floor_enforced`, `multi_state`, `warm_transfer`, `outbound_calls` and `operator_takeover`. Trust the listing over this page when they differ.
- `availability` says whether your workspace can use the family, and why not.
- These files can hold real forwarding numbers and the service's own safety text. Keep them out of notes, commits and pull requests.
- forge has no command for a service's transfer-outcome instructions (Transfers below). Read and change those in the web console or with the API.

## What the family runs

- **A direct program only.** The agent version runs its instructions and tools with no context graph. A save whose version set runs a context graph is refused with `voice_family_graph_program`, so a graph-based service stays on another family, such as `grok_voice`. With no graph there are no states to split and no dynamic behaviors.
- **Set on the service.** `session_provider: grok_duplex` on the service is the only switch: there is no feature flag or per-workspace allowlist. An agent version cannot select it (`voice_family_agent_version`). A workspace with regional data residency is refused (`voice_family_residency`).
- **Channels:** `app`, `phone`, `playground` and `test_call`. A direct program answers an inbound phone call on the `phone` channel. Outbound calls are refused.
- **No engine emergency floor.** Nothing in the platform detects an emergency on this family, plays an emergency script or cuts in. The crisis block and the `## Safety` section of the agent's instructions are the only emergency handling. Decide whether that is acceptable for a line before it takes phone traffic.
- **No write consent.** A write tool runs as soon as the model calls it, exactly like a read. The platform asks the caller nothing, so the agent's instructions must confirm a write with the caller before calling the tool. Check them before you grant a write tool.
- **No check on what the agent says.** The platform runs no reply check on this family. The spoken-confirmation checks in `handoffs-and-confirmations.md` do not cut its speech, and nothing checks the sentence it says to an operator. The agent's instructions are the only control on what it claims.
- **No failover.** If the xAI session fails, the call does not move to another provider.

## Languages

- **English and Spanish only.** A service on `grok_duplex` declares `en`, `es` or both in `declared_languages`. Any other language is refused with a 422 `voice_family_settings` on `declared_languages`.
- **`grok_duplex.reply_language`** picks the reply language:
  - `follow_caller` (the default): the caller's language when the service declares it, else the agent's default spoken language.
  - `default_only`: always the agent's default spoken language.
- **Set once.** The rule is written into the agent's instructions when the call opens and never changes during the call.
- **The platform does not detect the caller's language.** Only the model follows the caller. A caller who needs another language is offered the service's `forwarding.unsupported_language_destination` line when it has one.
- **Platform-played audio stays in the default language**: the opening notice, the closing clip of an `end_call` that said nothing, and the transfer line.
- **Leave `grok_duplex.language_hint` unset on a service that serves both languages.** A hint pins xAI's transcription to one language.

## Settings the family does not read

These save without an error and have no effect on a `grok_duplex` call:

- In `voice_config`: `silence_policy`, `barge_in_min_speech_s`, `barge_in_cooldown_s`, `held_turn_policy`, `filler_style`, `progress_interval_ms` and `progress_vocabulary`. xAI's server decides turns and barge-in.
- On the service: `contact_lines`, `crisis_call_911_reply`, `medical_emergency_ending` and `emergency_floor`. The platform lists each one a service sets as an ignored setting, and none of them is spoken. To change what the agent is told about safety, use `grok_duplex.safety_block`.

The family reads `forwarding`, `keyterms`, `declared_languages` and its own `grok_duplex` block. The block's `model`, `voice`, `speed`, `reasoning_effort`, `keyterms`, `language_hint` and `replace` work as they do on `grok_voice`. Pick the model and voice from `voice-families list` and `voice-families voices grok_duplex`. A `--body` write that names `grok_duplex` replaces the whole object, so copy it from the saved file and send every key back.

## Transfers

- **The transfer line.** When the agent calls `forward_call`, the platform has the call say a short, fixed transfer line in the session voice before it dials, in English or Spanish. The agent is told to say nothing itself as it connects a caller. Do not give the agent a sentence of its own to say as it connects the caller.
- **Cold or warm.** `forwarding.warm_transfer` chooses, as on other families (see `handoffs-and-confirmations.md`). A warm transfer runs on phone calls:
  1. The caller hears the transfer line and is held while the line rings. The model hears nothing.
  2. The receiver is screened by `receiver_classifier`: `twilio_amd` (the default) or `platform`, which needs `warm_transfer: true`.
  3. A person who answers is briefed by the agent in one sentence, then the caller is handed over and the agent leaves.
  4. A queue follows `warm_transfer_on_queue`: `hand_over` (the default) hands the caller to the queue with no briefing, `treat_as_miss` treats it as no answer.
  5. Voicemail, a busy line or no answer follow `warm_transfer_on_miss`: `hand_back` (the default) returns the caller to the agent, `offer_number` also lets the agent give the line's number, `cold_to_fallback` sends the caller cold to `fallback_to`, which it needs.
  6. An operator who hangs up before the hand-off returns the caller to the agent.
- `warm_transfer_answer_timeout_s` (10 to 55, default 30) is how long the line rings. A `forward_call` has been measured to get a spoken reply when it is answered 60 s late, and a held line has been measured to stay open for 150 s, so the ring window fits well inside both.
- **While the agent briefs, only call control's tools run** (`forward_call`, `end_call`, `stop_recording`). A service tool called then is refused and gets no reply of its own: the briefing is the one reply.
- **What the agent is told of each outcome** comes from five transfer-outcome instructions, set per service in the web console or with the API: `call_control.transfer_briefing_instruction`, `transfer_operator_left_instruction`, `transfer_miss_instruction`, `transfer_miss_number_instruction` and `transfer_queue_instruction`. They take effect on `grok_duplex`. An empty value adds nothing, and the outcome itself still reaches the agent.
- **A failed xAI session during a transfer** ends the call as failed, and the platform finishes or undoes the transfer.
- Off the phone (app, playground), `forward_call` reports that it cannot transfer, so test transfers on a phone call to a test line.

## What `safety_block` does

- **Unset** (the default): the agent gets the platform's own `## Safety` section, unchanged.
- **Set**: the string replaces the whole section, as written. The platform adds, strips and rewords nothing in it, so start it with its own `## Safety` heading.
- Up to 4,000 characters, and not blank.
- It is not checked against any number list. You are responsible for every phone number it contains.
- The separate crisis block (the region's crisis resource) is not part of it. That block keeps the platform's text whatever you set.
- It is sent as written on a Spanish call too, so a section meant for Spanish callers names what to say in Spanish itself.

The platform's section also carries two rules worth knowing before you replace it:

- The agent says nothing itself while it connects a caller, because the call says the transfer line for it.
- The agent never says a phone number that no tool result on the call and no instruction gave it.

A section that drops either rule removes that protection. Any wording change to safety text needs your own clinical review.

## The pairing rule: `safety_block` with a warm transfer

With `forwarding.warm_transfer: true` and no `safety_block`, the platform's section gets a warm connect rule: the agent tells the person who answers, in one sentence, what the caller needs.

**A service that sets `safety_block` gets no warm rule**, warm transfer or not. Your section is sent as written. If it does not say how to brief, the agent can stay silent when the operator answers, and the operator hears nothing about the caller.

So when a service sets both, its section must itself say, in substance:

1. Say nothing while the call connects the caller.
2. Only when `forward_call`'s result says the operator has answered, tell the operator in one sentence what the caller needs.
3. Then say nothing more, unless the call hands the caller back.

An example clause to adapt into your own section:

```text
When you connect the caller with someone, call forward_call and say nothing while the call connects them. Only when forward_call's result says the operator has answered, tell the operator in one sentence what the caller needs. Then say nothing more unless the call hands the caller back to you.
```

Keep step 3's "unless the call hands the caller back". A warm transfer that reaches voicemail, no answer or a hang-up returns the caller to the agent, and a rule that keeps the agent silent whatever happens leaves the caller in silence.

## Check a service for the pair

This check only reads. It never saves or changes a service. If the section is missing the briefing, propose the edit to the user and let them review and apply it.

1. Save the voice block and read the two settings:

   ```bash
   forge platform service voice-config <service-id> --get --json --env test-org > service-voice.json
   jq '{session_provider, safety_block_set: (.grok_duplex.safety_block != null), warm_transfer: (.forwarding.warm_transfer == true)}' service-voice.json
   ```

2. If `session_provider` is `grok_duplex` and both values are `true`, print the section and read it:

   ```bash
   jq -r '.grok_duplex.safety_block' service-voice.json
   ```

3. Look for the three parts of the pairing rule. A section that says only "say nothing when you connect the caller", with no briefing once the operator answers, is the gap.
4. Tell the user what is missing, quote the example clause above, and say that the whole `grok_duplex` object goes back in the `--body` write. Write only when the user asks, after they have reviewed the wording (Step 4 in `SKILL.md`).

## Not supported

- Context graph programs and multi-state flows (direct programs only).
- Dynamic behaviors.
- Operator takeover.
- Outbound calls.
- Languages other than English and Spanish.
- Failover to a second provider.

Among the Grok families, warm transfer is a `grok_duplex` capability: `grok_voice` refuses `forwarding.warm_transfer: true`. Read `capabilities` in `voice-families list --json` before you rely on any of this, because the listing is what the platform runs today.
