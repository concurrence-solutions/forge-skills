# Read-backs and pronunciation

Companion to `forge-tune-voice`. How an agent reads values back to a caller, and how to nudge a hard word toward a good pronunciation. Most of this is text you author, so it applies on `gpt_realtime`, where the model speaks plain text. Check every change on real calls, because a text simulation cannot show how a word sounds.

## Read-back formats

A speech-to-speech model can blur or merge digits when it reads them back, and a format you leave unstated changes from call to call. State the format in the agent's instructions, in words.

| Value | Format |
|---|---|
| Codes and ids | One character at a time with a clear separator. The caller confirms. After a correction, read the full corrected value. |
| Phone numbers | In groups, or digit by digit. Pick one. |
| Dates | A plain spoken form such as "August twelfth". |

- Pick one format per value type, and write it once: in the agent version's `background`, or in the `action_guidelines` of the state that reads the value.
- Describe the format in words. Do not write SSML or spell tags on `gpt_realtime`, because its output is plain text.
- Whether to read a value back at all is a product and compliance choice.
- These formats are vendor guidance. None is measured on the Platform, so listen to a call before you trust one.
- Keep this rule apart from the respelling rule below. A respelling is said as one smooth word, and a code is read one character at a time.

A sample block for the instructions:

```text
## Reading values back
Read a code or an id one character at a time with a short pause between characters, then ask the caller to confirm it. After a correction, read the whole corrected value.
Read a phone number in groups: the area code, then three digits, then four.
Say a date in plain words, such as "August twelfth".
```

For a code the caller reads aloud, see "Codes the caller reads in groups" in `handoffs-and-confirmations.md`.

## Pronunciation levers on gpt_realtime

The model speaks from the text it is given, so a pronunciation hint is text. There are two levers, and neither is a guarantee.

1. A short respelling list in the agent's instructions, under its own heading. Add a `## Pronunciation` heading after a blank line in the agent version's `background`.
2. A spoken form in a tool result. Return the spoken form beside the written name, and add one rule to the agent's instructions to say the spoken form. The rule belongs in the instructions, because the model treats tool results as information, not as instructions.

Each of these saves a new version or a new endpoint, and with an unpinned `release` a new version is live at once. Pin first (see `voice-verification.md`).

```bash
forge platform agent create-version <agent-id> --file agent-version.json --env test-org
forge platform context-graph create-version <graph-id> --file graph-version.json --env test-org
forge platform integration endpoint-create <integration-id> --file endpoint.json --env test-org
```

- Lever 1 goes in the agent version. For lever 2, the spoken form comes from the endpoint's `response_template` and the rule goes in the `action_guidelines` of the state that runs the tool.
- `response_template` is a Jinja template, and it needs a JSON or form-encoded response. A template that reads a key the response lacks fails the read, so fall back to the written name when the spoken form is missing.
- Either lever can help a little and unevenly, and a respelling can make a name worse. Treat any gain as direction, not proof.
- `realtime.transcription.prompt` and keyterms do not change how a word is spoken. The prompt guides the written transcript only, and keyterms feed the platform-managed recognizer.

For lever 2, the tool result carries the pair and the graph carries the one rule:

```json
{"name": "Acmevex", "spoken_name": "AK-mee-vex"}
```

```text
When a tool result has a spoken_name, say that spoken form for the name, as one smooth word. Never spell it out and never say the hyphens.
```

## Respelling notation

Write a respelling as plain syllables joined by hyphens, with the stressed syllable in capitals. For a made-up product called Acmevex:

```text
## Pronunciation
Say each respelling below as one smooth word. Never spell it out and never say the hyphens.
- Acmevex: AK-mee-vex
```

- Use plain syllable respellings, not IPA symbols, on `gpt_realtime`.
- Listen for the "th" sound, and for a capitalized syllable that is also a letter name. If either is said wrong, respell it.

## Add one term at a time

- Add a term only after you hear the agent say it wrong on a real voice call.
- Keep the list short. Add one term, then listen again.
- Keep an entry only if it is right in every call you listen to and no worse than the same call without it. "Every call" is a practical bar, not a measured threshold.
- Check each respelling against the product maker's guide or a standard dictionary's pronunciation guide before you test it.
- Accept a change by ear.

## What the transcript holds

The agent's recorded transcript holds the respelling it was told to say, not the written name. Anything that reads the agent's words, such as a summary, a search or a check on what the agent said, can see the respelled text. Keep the written name in tool arguments and tool results, and base logic on those instead of on the agent's spoken words.

```bash
forge platform conversation get <conversation-id> --include-tool-calls --json --env test-org
```

In the output, a turn's `text` shows the respelling and `tool_calls[].input` shows the written name.

## Pronunciation dictionary on the platform-managed runtime

On the platform-managed runtime, the workspace voice setting `pronunciation_dict_id` links a pronunciation dictionary. Check `session_provider` first (see `voice-settings.md`).

- It applies to every message a Cartesia voice speaks, greetings and fillers included.
- It has no effect on `gpt_realtime` calls, and none on a call an ElevenLabs voice speaks.
- With no `tts_provider` named, storing the id makes the workspace's calls speak Cartesia. Name `tts_provider` (`cartesia` or `elevenlabs`) when you intend another provider.
- The Platform stores the id. It does not create dictionaries.

```bash
forge platform voice-settings get --json --env test-org
forge platform voice-settings update --body '{"pronunciation_dict_id": "<dictionary-id>"}' --env test-org
```

This is a workspace setting, so it reaches every service in the workspace. Read the current values first, and listen to a call from each runtime you use.
