---
name: forge-tune-voice
description: Tune how a phone agent behaves on live calls with the forge CLI by reading a service's voice settings, changing one layer at a time with a saved copy to roll back to, and checking each change on real calls. Use when a forge binary is on PATH, a service has voice settings, and the user says the agent interrupts, talks over or cuts off callers, answers before they finish, asks callers to repeat or restart a code, mispronounces a name, reads a value back badly, will not hand off to a person, or says it sent something and the reply is cut, or asks about voice settings, barge-in, turn detection or test calls. Not for building, validating, syncing or simulating an agent (use forge-build-agent, forge-validate, forge-sync, forge-simulate).
---

# Forge Tune Voice

Tune a phone agent without guessing. Read the live settings, change one layer, check the change on a real call, then keep it or restore the saved copy. Settings, agent text and tools each have their own levers, and a symptom usually has one right owner.

## When to use

Use this skill when the user says things like:

- "The agent talks over callers" or "it cuts callers off."
- "It answers before the caller finishes."
- "It asks the caller to repeat a code" or "it makes the caller start the code again."
- "It says a product name wrong" or "it reads numbers back badly."
- "It will not hand off to a person" or "the transfer never connects."
- "It says it sent something and the reply is cut."
- "Which voice settings should I change?" (barge-in, turn detection, presets, voices)

Observable preconditions: a `forge` binary on `PATH`, a profile in `.env.platform.<env>` that `forge auth status --platform` accepts, and a service with voice settings. This skill needs forge 0.1.82 or newer. Examples use `--env test-org` as the profile name, here and in `reference/`, with placeholder ids.

### When NOT to use - route to a sibling instead

- Deciding where a behavior belongs before you build (settings, graph, tool or function) -> `/forge-agent-design`.
- Building the agent, graph, tools or service -> `/forge-build-agent`.
- Regression runs, parity gates, promote and rollback -> `/forge-simulate`.
- Checking local entity JSON -> `/forge-validate`. Reading or pushing entity JSON -> `/forge-sync`.

## The rule

**One layer per round.** Work from the layer that is cheapest to undo to the hardest, and verify each change before the next. Keep the scripted calls, and the counts you take from them, the same every round.

| Layer | Save first | Undo |
|---|---|---|
| Service voice settings | `voice-config --get --json` to a file | Send the saved block back with `--body` and diff (Step 6) |
| Agent or graph text | `version-set get release --json` to a file, after you pin `release` (Step 3) | Move the pin back to the numbers you wrote down with `version-set upsert`. A saved copy with null pins is no undo, because the new version was already live |
| Tool or endpoint contract | The endpoint JSON you created it from, or a copy of its definition from the web console or the API | Put the saved values back in the web console or with the API's endpoint update. `endpoint-create` refuses a name that already exists. Every service that uses the endpoint shares it, and it sits outside a version set |
| Graph structure (split or merge states) | The same pins | The same pin move. `gpt_realtime` needs exactly one action state, so it takes no split states |

- Keep a saved copy before every change. A restore is one command, and a diff confirms it.
- Pin what you test. An empty pin on `release` serves the latest version, so a save in the middle of a round changes what you are testing.
- Judge sound on real audio. Text simulations do not show talk-over, turn timing, digit pacing or pronunciation.
- Give each behavior one owner. Two instructions that govern the same moment give opposite orders, and the agent has to pick one. To audit a state for conflicts, follow "Collision audit: one owner per behavior" in `reference/agent-context-graph-authoring.md` of `/forge-build-agent`.

## Workflow

Read first. A `voice-config` write is **live for the next call and has no dry run**, so run it only when the user asks. Version-set changes are dry runs until you add `--apply`.

### 1. Preflight

```bash
forge --version
forge auth status --platform --env test-org
```

Stop if the version is older than 0.1.82, and update forge before you write anything.

### 2. Read the live settings and save copies

```bash
forge platform service voice-config <service-id> --get --json --env test-org > service-voice.before.json
forge platform version-set get <service-id> release --json --env test-org > release.before.json
```

Open `service-voice.before.json` and read `session_provider`. If it is null, the agent version's `voice_config.session_provider` applies, and with neither set the calls run on `amigo`, the platform-managed runtime. A finished call reports the runtime it ran on in `session_provider` on `run get`. Each runtime reads different settings, and a setting the serving runtime does not read is saved and ignored (see `reference/voice-settings.md`).

### 3. Pin what you test

If `release.before.json` shows a null `agent_version_number` or `context_graph_version_number` (printed as `latest` without `--json`), the live service serves the latest version of that component. Pin both numbers. On a first pass, pin the newest version of each, which keeps the live behavior the same. This is a dry run until you add `--apply`:

```bash
forge platform version-set upsert <service-id> release --copy-from release --agent-version <agent-version> --context-graph-version <graph-version> --env test-org
```

Take the numbers from `forge platform agent list-versions <agent-id>` and `forge platform context-graph list-versions <graph-id>`, and write them down before each promote. What a version-set write replaces, and how behavior pins and the `release-previous` backup work, is in `/forge-build-agent`, Step 7A. To try a draft without touching callers, use a separate test service (see `reference/voice-verification.md`).

### 4. Pick the lever and change one thing

Find the symptom in the table below, then change one field. A `--body` write keeps the keys it does not name. A nested object it does name, such as `realtime` or `forwarding`, is replaced whole, so send every key you want to keep inside it.

```bash
forge platform service voice-config <service-id> --body '{"barge_in_min_speech_s": 0.8}' --env test-org
```

### 5. Check on real audio and read the call record

forge does not place voice calls. Ask the user to place the same scripted calls after each change, from a headset or a handset held to the ear. They can use the Playground voice test in the web console or a phone call from a workspace test caller number (see `reference/voice-verification.md`). Then read what happened:

```bash
forge platform run list --kind conversation --service-id <service-id> --env test-org
forge platform run get <run-id> --json --env test-org
forge platform conversation get <conversation-id> --include-tool-calls --json --env test-org
```

Count the same behaviors every round (talk-over, early answers, code restarts, early forwards) and compare rounds at a sample size that fits (see `reference/voice-verification.md`).

### 6. Keep it or restore the copy

```bash
forge platform service voice-config <service-id> --body "$(cat service-voice.before.json)" --env test-org
forge platform service voice-config <service-id> --get --json --env test-org > service-voice.after.json
diff service-voice.before.json service-voice.after.json
```

A `--body` write never removes a key. If the saved copy is `null` because the service had no voice settings, send each key you changed as `null` instead, and do the same for any key the diff shows that the saved copy lacks. For `held_turn_policy`, `filler_style` and `backchannel_delay_ms`, send the default instead of `null`: `always`, `backchannel` and `400`. Then diff again. To move a pin back, upsert `release` to the numbers you wrote down.

## Symptom to lever

| What you hear | Check first | Lever and where to read more |
|---|---|---|
| The agent keeps talking after the caller starts to speak | `session_provider`, `barge_in_min_speech_s` | On `gpt_realtime`, lower `barge_in_min_speech_s` (for example from 0.5 to 0.3). The minimum is a delay before the agent yields, so a larger value reads as talking over the caller, and a smaller one lets shorter sounds cut the agent off. On the platform-managed runtime check `barge_in_cooldown_s` and the state's `turn_policy`. `voice-settings.md`, Barge-in |
| The agent answers before the caller finishes | `realtime.turn_detection` | On `gpt_realtime`, raise `silence_duration_ms` with `server_vad` (for example from 500 to 700) or lower `eagerness` with `semantic_vad`. On the platform-managed runtime, `eager_eot_threshold` does not change when the agent replies, and the other end-of-turn fields are in `voice-settings.md`. `voice-settings.md`, Turn detection |
| A hello over the greeting gets a second reply | `held_turn_policy` | The default is `always`. Read the modes in `voice-settings.md`, Held turns, and test another mode on a copy of the service first |
| The agent answers after the first group of a code, or asks the caller to start over | The tool input in the call record | The code tool decides whether the code is complete, and the graph holds one rule for an incomplete status. `handoffs-and-confirmations.md`, Codes the caller reads in groups |
| Digits blur or merge when the agent reads a value back | The format rule in the agent text | State the format in words. `read-backs-and-pronunciation.md`, Read-back formats |
| A name is said wrong | `session_provider` | On `gpt_realtime`, a respelling in the agent text or a spoken form in a tool result. On the platform-managed runtime, `pronunciation_dict_id`. Accept any change by ear. `read-backs-and-pronunciation.md` |
| It says it sent or booked something and the reply is cut | The endpoint's `declarations.writes` | Declare the kind and a success rule. `handoffs-and-confirmations.md` |
| A reply with opening hours or open slots is cut for an unverified caller | The endpoint's `declarations.reads` | Declare `patient_in` as `none` on an endpoint that returns only practice data. `handoffs-and-confirmations.md`, Spoken confirmations |
| It will not hand off, or the transfer never connects | `voice_config.forwarding` | Set the block, `fallback_to` and a short announcement. `handoffs-and-confirmations.md` |
| After closing time, a transfer rings a line nobody answers | `voice_config.forwarding.hours` | Set `hours` and `after_hours_destination` together on a dynamic service. `handoffs-and-confirmations.md`, After hours |
| Test calls cut themselves off or answer themselves | How you place the call | Ask the user to use a headset or a handset. `voice-verification.md` |
| Results change between rounds | The pins on `release` | Pin both numbers. `voice-verification.md` |

The files in `reference/` hold the depth: `voice-settings.md` (settings, runtimes, turn-taking), `read-backs-and-pronunciation.md` (read-back formats, pronunciation levers), `handoffs-and-confirmations.md` (forwarding, transfers, codes, declared writes and reads) and `voice-verification.md` (pins, test paths, the call record, counts, sample size).

## Safety

- Read first. `--get`, `version-set get`, `run list`, `run get` and `conversation get` change nothing.
- A `voice-config` write is live for the next call and has no dry run. Save the copy first, change one field, and write only when the user asks. Calls already running keep the settings they started with.
- `version-set upsert`, `promote` and `rollback` are dry runs until you add `--apply`. Add it only when the user asks.
- `agent create-version`, `context-graph create-version`, `integration endpoint-create` and `service create` write at once, with no dry run. Run them only when the user asks. `service delete` asks for a confirmation and removes the service.
- `workspace test-callers-update` replaces the stored list, and `voice-settings update` changes a workspace setting. Read the current values first.
- `integration test` makes the real call to the vendor. Use a test recipient and test records.
- Synthetic data only. Use placeholders (`Acme Corp`, `test-org`, `user@example.com`, `555-010-1234`, `00000000-0000-0000-0000-000000000000`). Tool output and transcripts can hold caller data, and a saved voice block can hold real forwarding numbers, so keep those files out of notes, commits and pull requests.
