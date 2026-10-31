# Voice settings and turn-taking

Companion to `forge-tune-voice`. These are the fields of a service's `voice_config` (the voice block), read and written with `forge platform service voice-config`.

## Read and write

```bash
forge platform service voice-config <service-id> --get --json --env test-org > service-voice.before.json
```

Without `--json`, `--get` prints a heading line before the JSON, so save with `--json` whenever you plan to send the file back. Each command runs exactly one of `--preset`, `--body`, `--realtime-voice` and `--get`.

This skill needs forge 0.1.82 or newer. Run `forge --version` before any `--preset` or `--body` write, and update first if it is older.

- `--preset` and `--body` read the service, write the keys you name over its current voice block, and send the whole block back. Keys you do not name keep their value.
- A nested object you name, such as `realtime` or `forwarding`, is replaced whole. Copy the object from the saved file, change the one key, and send every key back.
- A key set to `null` returns to its default. `held_turn_policy`, `filler_style` and `backchannel_delay_ms` take a value instead.
- The write is a read followed by a write, so avoid editing the same service from two places at once.

To change one key inside a nested object without retyping the rest, build the body from the saved file. This sample sets the `server_vad` silence period inside `realtime.turn_detection`, and keeps the other `turn_detection` keys and every other `realtime` key. Replace `500` with the value you are testing:

```bash
jq -c '{realtime: (.realtime | .turn_detection = ((.turn_detection // {"type": "server_vad"}) | .silence_duration_ms = 500))}' service-voice.before.json > body.json
forge platform service voice-config <service-id> --body "$(cat body.json)" --env test-org
```

The sample suits a saved `server_vad` block, or a service with no `turn_detection` saved. A `semantic_vad` block takes only `type` and `eagerness`, so to switch the type write the whole `turn_detection` object.

## Restore a saved copy

There is no dry run, and calls already running keep the settings they started with. To restore, send the saved block back and compare:

```bash
forge platform service voice-config <service-id> --body "$(cat service-voice.before.json)" --env test-org
forge platform service voice-config <service-id> --get --json --env test-org > service-voice.after.json
diff service-voice.before.json service-voice.after.json
```

- A `--body` write never removes a key. If the diff shows a key the saved copy lacks, such as a key the API added after you saved the copy, send that key as `null` and diff again.
- For `held_turn_policy`, `filler_style` and `backchannel_delay_ms`, send the default instead of `null`: `always`, `backchannel` and `400`.
- A service that has never had voice settings prints `null`, and a body must be an object. Note the keys you change, and undo each one the same way.

## Which runtime reads what

Check `session_provider` before you tune. It resolves from the service, then the agent version's `voice_config`, then `amigo`. A setting the serving runtime does not read is saved and ignored. The exception is `realtime`, which the API refuses with a 422 unless `session_provider` is `gpt_realtime`.

| Runtime | Settings it reads |
|---|---|
| `gpt_realtime` (speech to speech) | `realtime.*`, `held_turn_policy` |
| The platform-managed runtime (speech recognition, reasoning and speech output as separate steps) | `barge_in_cooldown_s`, the filler fields, `max_response_sentences`, `max_response_words`, `eager_eot_threshold`, `eot_timeout_ms`, `post_eot_pause_ms`, `transition_deadline_ms`, `empathy_hold_ms`, a state's `turn_policy` |
| Both | `barge_in_min_speech_s`, `forwarding`, `progress_vocabulary`, `progress_interval_ms` |

- `session_provider: amigo` names the platform-managed runtime.
- `max_response_sentences` and `max_response_words` default to null (unlimited) and cap streamed text approximately.

## Turn timing on the platform-managed runtime

- `eot_timeout_ms` is a silence timeout. When the caller is silent for that long, the call forces an end of turn.
- `eager_eot_threshold` does not change when the agent replies, because the agent replies at the final end of turn.
- `eager_eot_threshold` and `eot_timeout_ms` apply only on recognizers that accept them.
- `post_eot_pause_ms` (default 200) is a short pause after the caller stops speaking.
- `transition_deadline_ms` (default 800) is the longest silence after the caller's end of turn before a filler plays.
- `empathy_hold_ms` (default 500) is silence held on purpose for an empathy moment.

## Presets and voices

```bash
forge platform service voice-config <service-id> --preset gpt_realtime --env test-org
forge platform service voice-config <service-id> --preset balanced --env test-org
forge platform service voice-config <service-id> --realtime-voice marin --env test-org
```

- `gpt_realtime` sets only `session_provider: gpt_realtime` and keeps everything else. A service on `gpt_realtime` needs a context graph version that is exactly one action state, and the save returns a 422 for any other graph. Count the action states in the graph before you switch.
- `ultra_low_latency`, `balanced` and `quality` move the service to the platform-managed runtime and clear `realtime` and `realtime_voice`, because the API accepts both only under `gpt_realtime`. `balanced` also sets `filler_style`. `quality` sets `filler_style` and `max_buffer_delay_ms`. `ultra_low_latency` sets those two plus `filler_vocabulary`, `backchannel_delay_ms` and `max_response_sentences` (to 1). A preset can discard realtime tuning, so save the voice block first.
- `--realtime-voice` changes only the voice. It needs a service already on `gpt_realtime`, sets `realtime.voice`, and drops the deprecated `realtime_voice` shortcut. Accepted voices are alloy, ash, ballad, cedar, coral, echo, marin, sage, shimmer and verse.

## Barge-in on gpt_realtime

`barge_in_min_speech_s` (default 0.5 seconds) is a delay, not a filter. The agent keeps talking until the realtime session has reported the caller's speech for that long.

- A higher value makes the agent talk over an interrupting caller for about that much longer. Callers hear it as talking over them.
- A lower value yields sooner and lets shorter sounds cut the agent off.
- The cut counts any speech the realtime session detects. It does not wait for recognized words, and `barge_in_cooldown_s` does not apply.
- The minimum counts speech length only. A value high enough to let a short "uh huh" pass lets a short "stop" of the same length pass too.
- With `server_vad`, a sound counts for its own length plus the silence period that follows it.
- Speech over the opening notice and greeting does not cut them off.

Choose the value by listening to test calls with callers who interrupt and callers who murmur, and confirm it on a phone call, because a desk test can differ.

## Barge-in on the platform-managed runtime

`barge_in_min_speech_s` applies only to speech in progress. A finished utterance with recognized words cuts at once. In-progress speech also needs recognized words and has to clear the `barge_in_cooldown_s` gap (default 1.5 seconds). A state's `turn_policy.barge_in_enabled` (default true) can make one state uninterruptible, so check it when a single state ignores interruptions.

## Turn detection on gpt_realtime

Unset, `realtime.turn_detection` is `server_vad`. It detects speech by volume and ends the caller's turn after `silence_duration_ms` of silence (200 to 3000, 500 when unset). Its other fields are `threshold`, `prefix_padding_ms` and `idle_timeout_ms`.

`semantic_vad` decides from the words when the caller is done. Its `eagerness` (`auto`, `low`, `medium`, `high`) sets how long it may wait when unsure.

- Eagerness changes when a turn counts as finished. The barge-in minimum sets how long caller speech must last before it cuts the agent. Confirm on your own calls before you tune eagerness against talk-over.
- One setting applies for the whole call. A state's `turn_policy` is read only on the platform-managed runtime.
- A pause longer than the silence period ends the turn. A longer period waits out longer pauses and delays every reply by the same amount. For a caller who reads a code in groups, see "Codes the caller reads in groups" in `handoffs-and-confirmations.md`.

## Held turns

On `gpt_realtime`, `held_turn_policy` sets what the call does with a caller turn that commits while the agent's own notice and greeting play.

- `always` (default) answers it once the opening ends.
- `judged` answers it only if it asks for something the opening did not cover. A greeting, an acknowledgement or an echo of the opening goes unanswered. A turn it cannot judge is answered.
- `never` does not answer it. A real request spoken over the greeting waits until the caller speaks again.

Every turn is recorded and checked for a medical emergency, whatever the setting. `always` is the default. Test `judged` or `never` on a copy of the service and listen to calls before you change a live service. The setting applies to a call the agent opens itself. A call that waits for the caller to speak first answers the first turn.

## Noise reduction and transcripts

- `realtime.noise_reduction` takes `near_field`, `far_field` or `off`. It filters noise in the caller's audio, and `far_field` suits a microphone farther from the speaker. Unset uses the Platform default.
- `realtime.transcription.prompt` (1 to 1024 characters) guides the written caller transcript only. The speech model hears the audio itself, so the prompt does not change what the agent understands or says, and it does not stop the agent mishearing or answering early. Judge it in the recorded transcript.
- Keyterms, set on a service or in the workspace voice settings, bias the platform-managed speech recognizer. They do not change how a word is spoken.
- Send the complete `realtime` object when you set any of these.

Pronunciation settings and hints are in `read-backs-and-pronunciation.md`.
