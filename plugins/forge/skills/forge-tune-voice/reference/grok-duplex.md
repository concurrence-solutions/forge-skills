# Grok Duplex: the safety section and warm transfer

Companion to `forge-tune-voice`. `grok_duplex` is a voice family where xAI's Grok Voice listens, reasons and speaks for the whole call. A service on it keeps its family settings in `voice_config.grok_duplex`, read and written with `forge platform service voice-config` like every other block (see `voice-settings.md`). This page covers the two settings that change what the agent is told about safety and transfers, and the one pair of them that needs a check.

## The settings that matter on this family

- `grok_duplex.model`, `voice`, `speed`, `reasoning_effort`, `keyterms`, `language_hint` and `replace` work as they do on `grok_voice`. Pick the model and voice from `forge platform voice-families list` and `voice-families voices grok_duplex`.
- `grok_duplex.reply_language` sets the language the agent replies in. `follow_caller` (the default) replies in the caller's language when the service declares it, else the agent's default spoken language. `default_only` always replies in the agent's default spoken language. It is set when the call opens and does not change during the call.
- `grok_duplex.safety_block` replaces the agent's `## Safety` section (below).
- `forwarding.warm_transfer` chooses a warm or cold transfer, as on other families (see `handoffs-and-confirmations.md`).

A `--body` write that names `grok_duplex` replaces the whole object, so copy it from the saved file and send every key back.

## What `safety_block` does

- **Unset** (the default): the agent gets the platform's own `## Safety` section, unchanged.
- **Set**: the string replaces the whole section, as written. The platform adds, strips and rewords nothing in it, so start it with its own `## Safety` heading.
- Up to 4,000 characters, and not blank.
- It is not checked against any number list. You are responsible for every phone number it contains.
- The separate crisis block (the region's crisis resource) is not part of it. That block keeps the platform's text whatever you set.

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

`service-voice.json` can hold real forwarding numbers and the service's own safety text. Keep it out of notes, commits and pull requests.

## Caveats

- Among the Grok families, warm transfer is a `grok_duplex` capability. `grok_voice` refuses `forwarding.warm_transfer: true`. Read `capabilities.warm_transfer` for each family in `forge platform voice-families list --json` before you turn it on, and if your workspace's listing shows it false for `grok_duplex`, the save is refused.
- Dropping the platform's "say nothing yourself when connecting" and "never say a number you were not given" rules removes those protections, warm transfer or not.
- Any change to safety wording, yours or an adapted example, needs your own clinical review before it reaches callers. Try it on a test service first (see `voice-verification.md`).
