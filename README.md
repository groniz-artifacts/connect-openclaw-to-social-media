# How to Connect OpenClaw to Social Media

OpenClaw can connect to social media through the Groniz Skill, remote MCP server, or authenticated CLI. It can prepare posts from files and recurring operational inputs, while Groniz handles OAuth, per-platform formatting, scheduling, and delivery across 32+ networks.

Connecting the tools does not mean giving the agent blanket permission to publish. Let it prepare a draft, pause for human review, and discover the live destination schema. Require separate approval before any external write. This boundary matters even more for a long-running agent. A timer, watcher, or approved source can start a drafting run, but it cannot approve publication. Keep a clear audit trail, and leave the operator responsible for every approved destination.

## The connection and approval flow

```text
Approved source or event
  -> OpenClaw prepares a destination-specific draft
  -> human reviews facts, voice, and timing
  -> OpenClaw discovers the live Groniz integration
  -> human approves the exact write action
  -> Groniz schedules or publishes
  -> operator verifies the native result
```

OpenClaw's own chat or channel conversations are not the same thing as Groniz social publishing. The connection described here gives OpenClaw a controlled route to external destinations through Groniz.

## Prerequisites

You need:

- An OpenClaw workspace with permission to load the chosen integration method.
- A Groniz account and API key, or an authenticated Groniz CLI session.
- A social destination already connected to Groniz.
- An approved source such as a release note, incident update, or event brief.
- A named reviewer and publication approval rule.
- The intended time and timezone if the post will be scheduled.

Keep the integration and its publishing policy in the narrowest OpenClaw scope that fits the workflow. Only operators who prepare or approve social posts should have access.

## Choose one connection path

### Skill

Install the Groniz CLI skill:

```bash
npx skills add groniz/groniz-cli
```

The Skill teaches OpenClaw the Groniz commands and installs the CLI behind them. Authenticate the CLI before a publishing run.

### MCP

Add the remote Groniz server:

```bash
openclaw mcp add groniz \
  --url https://mcp.groniz.com/mcp/YOUR_API_KEY \
  --transport streamable-http
```

This key-in-URL form is recommended for OpenClaw because it avoids header-handling edge cases. Since the API key is part of the URL, configuration output, shell history, screenshots, and debug logs may expose it. Keep the real URL out of version control and redact it from shared diagnostics.

### Native CLI

The CLI path works when OpenClaw can invoke a locally authenticated binary:

```bash
curl -fsSL https://groniz.com/install.sh | sh
groniz auth:login
```

The login command uses a browser device flow and does not require an API key. Whichever path you select, test it with a read-only identity or integration-list action before enabling a publishing run.

## Why a review gate matters for an agent workflow

Recurring operational sources such as release records, community schedules, incident-resolution notes, and approved content queues give OpenClaw structured input to work from. Their publication context can still change. An internal incident note may contain private infrastructure details. A release record may describe something not enabled for every customer, or an event time may have changed.

A review catches those differences before delivery. It also keeps two decisions separate: "this draft is accurate" and "this account should publish it at this time." The [Codex social publishing guide](https://groniz.com/blog/automate-social-media-posting-with-codex) uses the same operating model with another agent. [Turning a GitHub release into an X thread with Codex](https://groniz.com/blog/github-release-to-x-thread-with-codex) follows one source-to-social workflow. [The Discord and Telegram product-update guide](https://groniz.com/blog/openclaw-product-updates-discord-telegram) covers a comparable OpenClaw community workflow.

## Prepare a bounded publishing source

Do not ask OpenClaw to infer an announcement from an entire workspace. Create an approved source record:

```yaml
id: update-2026-07-20
status: approved-for-drafting
audience: existing-users
summary: Export jobs now provide progress events.
evidence:
  - docs/export-progress.md
link: https://example.com/changelog/export-progress
available_at: 2026-07-20T16:00:00Z
exclude:
  - internal queue names
  - performance claims
```

The `available_at` field constrains the content. It does not instruct OpenClaw to publish automatically. The agent should check the field and still wait for explicit approval of the destination and delivery time.

## Reusable asset: OpenClaw publishing policy

Save a policy like this in the scope where OpenClaw loads workflow guidance:

```markdown
# Social publishing policy

1. Draft only from sources marked approved-for-drafting.
2. Cite the source path for every factual claim.
3. Produce a distinct draft for each destination.
4. Remove confidential identifiers and unsupported outcomes.
5. Stop after presenting the draft and review checklist.
6. Treat content approval and publication approval as separate states.
7. Before a write, run identity, integration-list, and live-settings checks.
8. Upload approved media first and use the returned Groniz `.path`.
9. Show account, destination, content, media, and time for final approval.
10. Record the returned post ID and verify the native result.
11. Treat an ambiguous timeout as unknown, not failed, until the native destination is checked.
```

The policy tells OpenClaw what it must show at each transition. A reviewer can check those states; an instruction to "be careful" gives them nothing concrete to verify.

## Create the channel-native draft

Use a prompt that names both the source and the destination:

```text
Read updates/update-2026-07-20.yaml and its evidence files. Draft one post
for [DESTINATION] and explain the reader value in the opening. Preserve the
availability date and all qualifiers. Return a claim-to-source list, proposed
media, and destination-specific review questions. Do not use Groniz write
tools. Stop for content approval.
```

If you need several destinations, run this step separately for each one. The Discord version may work best as a direct community announcement, while LinkedIn may call for a first-person professional interpretation. Using the same source does not mean reusing the same copy.

## Review content, account, and timing

Check the facts, links, names, dates, availability, and the difference between shipped and planned work. Remove internal identifiers and support-only instructions. Then read the message as a member of the destination community. It should explain why the update matters without overstating its impact.

Then verify the operational target. Name the exact Groniz integration, connected account, post type, media, time, and timezone. Record content approval first. Record publication approval only after the final request is visible.

## Discover and deliver through Groniz

For the Skill or CLI route, begin with:

```bash
groniz whoami
groniz integrations:list
groniz integrations:settings <integration-id>
```

The live settings response is the source of truth for required fields, current limits, supported post types, and available dynamic integration tools. Copied tutorials can go stale. Invoke only the dynamic tools listed for that integration because capabilities vary across destinations.

For MCP, have OpenClaw perform the equivalent read-only identity, integration-list, and schema-discovery steps with the tools exposed by the server. Follow their current descriptions and responses instead of assuming the CLI command names also apply to MCP.

Upload any approved media before scheduling:

```bash
groniz upload ./assets/update-diagram.png
```

Use the returned `.path` in the request. Ask OpenClaw to show the final request derived from the live schema, including the destination and time, and then pause for publication approval. Groniz can publish immediately or schedule the request. The operator decides which mode is authorized.

With MCP, use only the write tool exposed for the selected integration and populate it from the live schema. OpenClaw must show the fully resolved request and wait for approval before it calls that tool.

For one approved destination, the scheduling command has this shape:

```bash
groniz posts:create \
  -c "APPROVED_CONTENT_FOR_THIS_TARGET" \
  -m "<returned-groniz-media-.path>" \
  -s "2026-08-20T15:00:00Z" \
  -t schedule \
  -i "TARGET_INTEGRATION_ID" \
  --settings '<required-settings-json>'
```

Replace the settings placeholder with every value required by that integration's live response. The `-m` value must be the `.path` returned by the upload. For additional destinations, repeat discovery and build a separate command with that target's settings. OpenClaw must wait for approval of each resolved write command.

## Verify delivery and handle failures

Capture the returned Groniz post ID, status, scheduled or published time, and platform URL when available. Then inspect the native destination. Confirm the account, content, formatting, link, media, and timestamp.

If the action fails, refresh the live settings before editing the request. Check authorization, permissions, required fields, content length, post type, uploaded media path, and timezone. Before retrying, determine whether the platform accepted the original operation. A timeout does not prove that nothing was posted. Keep the draft unchanged when the fault is operational. If the correction changes the content, account, destination, media, or delivery time, require fresh approval.

Keep a small failure log:

```markdown
Source ID:
Integration ID:
Requested action/timezone:
Post ID/status:
Native post found: yes/no
Error category:
Correction:
Retry approved:
```

## Measure the next iteration

Use the destination's own analytics and community response. Available metrics differ by platform and account, so track only the native measures you can verify. Include questions, corrections, and support conversations caused by the update, and link each observation to the source ID and approved draft. This creates an auditable record. Groniz does not supply cross-platform audience intelligence or automatically choose the best time.

To add the publishing layer to your workspace, [create a Groniz API key for Connectors](https://groniz.com/console/connectors/api-keys).
