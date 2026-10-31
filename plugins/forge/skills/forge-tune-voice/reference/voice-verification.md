# Verify voice changes

Companion to `forge-tune-voice`. How to pin what you test, where a draft runs, which test path shows what, how to read the call record, and how to count behaviors across rounds.

## Pin what you test

Pin `release` to the versions you test before you change anything, and change the pins only on purpose. An empty pin serves the latest version, and `forge platform push --apply` posts a new version of each agent and context graph it pushes, whether or not the content changed. With an unpinned `release`, every push changes the live service. The rule, the commands and what a version-set write replaces are in `/forge-build-agent`, Step 7A.

A call or text thread already running keeps the configuration it loaded. A pin does not freeze voice settings, tools, integrations or data. To confirm what a finished call ran with, read `manifest` (the agent and graph versions, the version set and the models) and `session_provider` in `forge platform conversation get <conversation-id> --json` or `forge platform run get <run-id> --json`. `manifest` can be null on older records.

## Where a draft runs

Calls, text conversations, the Playground and simulations load the service's `release` set. A named set such as `candidate` is a saved selection to compare with `release` and promote. The `--branch` option on simulation commands labels the run and does not choose a version set.

To run the same scripted checks against a draft, give a separate test service the draft versions as its `release`, and promote on the live service after the checks pass. The commands are in `/forge-simulate`, step 6. For voice, make the test service match the live one first, or the two sets of calls differ in the settings you are tuning:

```bash
jq -c 'del(.forwarding)' service-voice.before.json > test-voice.json
forge platform service voice-config <test-service-id> --body "$(cat test-voice.json)" --env test-org
```

- The copy leaves out `forwarding`, so a test call never dials a real number. Add a `forwarding` block with test lines when you want to test transfers on the test service.
- Skip the copy when `service-voice.before.json` reads `null`.
- The test service also needs the live model preferences, which step 6 of `/forge-simulate` copies. Compare rounds only when both services match.
- Keep the test service for the next round. `forge platform service delete` removes it after a confirmation prompt, so run it only when the user asks.

Read the resolved configuration of a named set with the harness context:

```bash
forge platform agent-run harness-context --service-id <service-id> --version-set candidate --json --env test-org
```

Bridge simulations mock every tool. A test conversation runs tools live, so use test records.

## Which test path shows what

forge does not place a voice call. Text checks run through `forge platform conversation` and the simulation commands. Voice checks run through the paths below.

| Path | Audio | Tools |
|---|---|---|
| `forge platform sim bridge` and `forge platform simulation ...` | Text only | Simulation mode |
| `forge platform conversation create` and `send-message` | Text only | Live |
| The Playground voice test in the web console | Real audio | Live |
| A phone call from a workspace test caller number | Real audio | `dry_run` results |

- Simulations send synthetic caller text through the agent and leave out voice-only markup. They check behavior and tool coverage. They do not show pronunciation, digit pacing, talk-over or turn timing.
- A phone call from a workspace test caller number is left out of billing, metrics and analytics, and its tools return `dry_run` results. To check what the agent says from a tool result, use the Playground, because a `dry_run` result carries no real output.
- The API also has a test-call WebSocket. A workspace can hold up to 5 test calls at once.
- Registering test numbers needs workspace admin or owner access. `test-callers-update` replaces the stored list, so read it first and send every number you want to keep. Write real numbers in E.164 format.

```bash
forge platform workspace test-callers-get --env test-org
forge platform workspace test-callers-update --numbers "555-010-1234" --env test-org
```

## Place test calls from a headset

A speaker can feed the agent's voice back into the microphone. On a speakerphone or a laptop speaker the agent can then cut itself off or answer itself. Ask the user to place human test calls with a headset, or a handset held to the ear. Judge barge-in and turn-detection settings on real calls.

## Read the call record

Judge a round from the call record, not from impressions or the caller transcript.

```bash
forge platform run list --kind conversation --service-id <service-id> --env test-org
forge platform run get <run-id> --json --env test-org
forge platform conversation get <conversation-id> --include-tool-calls --json --env test-org
```

`run list` finds the call (its Type column shows the channel), and `run get` gives its conversation id. `conversation get` returns each turn with `speech_start_ms`, `speech_end_ms` and `interrupted`, and each tool call with `tool_name`, `input`, `succeeded` and `duration_ms`. `succeeded` is true, false, or null while the call is pending. Without `--json`, the tool call table labels outcomes ok, failed and pending.

- Judge a code capture on the tool input. The caller transcript is a separate, asynchronous transcription that can split or drop digits.
- Score a read-back only from agent lines where `interrupted` is not true, because an interrupted line can include words the caller never heard.
- Times are positions on the call recording and can run slightly ahead of what the caller hears. Listen to a sample to confirm.
- Tool output is opt-in because it can hold sensitive data. Use synthetic calls, and keep transcripts and tool inputs out of notes, commits and pull requests.

## Count behaviors, not impressions

Score each round on the tool call statuses (ok, failed, pending) and a short list of counts per call. Define each count once and keep the definition unchanged every round. The word and second limits belong to your test harness, not to the Platform, so choose yours.

| Count | What it measures |
|---|---|
| Talk-over | A caller line starts while an agent line is still playing after the opening. Split it into agent yielded and agent kept talking. |
| Early answer | An agent line starts soon after a very short caller line, and the caller continues before it ends. |
| Code restart | The caller re-reads digits already given, or the tool received a partial code. |
| Forwarded early | `completion_reason` is `forwarded` early in the call (for example, within half a minute) without a caller request. |

Read them from `conversation get --json`: each turn's `role`, `speech_start_ms`, `speech_end_ms`, `interrupted` and `text`, each tool call's `input`, and the call's `completion_reason` and `duration_seconds`. `completion_reason` is a string, and `forwarded` is one of its values. Recorded agent times lead playback slightly, so listen to a sample of calls to confirm the counts.

## How many calls

`/forge-simulate` asks for at least 10 simulations per topic as the go or no-go gate, and about 30 or more to characterize a behavior or compare two versions. The same sizes apply to calls. Five calls per arm show direction only.

| Calls per arm | Gap in calls that reaches p below 0.05 |
|---|---|
| 5 | 4 or more. 4 of 5 against 1 of 5 gives p of about 0.2. |
| 10 | 5 to 6 |
| 30 | 6 to 9 (20 to 30 points) |

Use five calls to find a direction, then decide on more. These figures come from Fisher's exact test, which assumes independent calls. Scripted calls from one synthetic voice are not independent draws of real callers, so treat the gaps as minimums. Run the same scripted calls for every layer, record the counts each round, and use synthetic callers and placeholder numbers only.
