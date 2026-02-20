# Devin Reminders Action

A reusable GitHub Action for scheduling, listing, and firing reminders for [Devin.ai](https://devin.ai) sessions. Uses GitHub Actions artifacts for storage and the Devin API to deliver reminders.

## Features

- Schedule reminders for existing Devin sessions (`put`)
- List all pending reminders and filter due items (`list`)
- Fire due reminders and clean up automatically (`cron`)
- Optional Slack notifications (opt-in via `slack-channel` input)
- Timezone-aware display for notification messages

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `action` | Action to perform: `put`, `list`, or `cron` | Yes | |
| `remind-at` | ISO 8601 timestamp with timezone offset for when the reminder fires. Must be in the future and no more than 3 days ahead. Required for `put`. | No | |
| `reminder-message` | Message to deliver when the reminder fires. Required for `put`. | No | |
| `agent-session-url` | Devin session URL to ping when the reminder fires. Required for `put`. | No | |
| `human-user` | Person identifier (email, GitHub handle, or Slack user ID) to CC on notifications. | No | |
| `devin-token`| Devin API token. | Yes | |
| `slack-token` | Slack bot token. Only needed if `slack-channel` is set. | No | |
| `slack-channel` | Slack channel name for notifications. Leave empty to skip Slack. | No | |
| `reminder-timezone` | Timezone for displaying times in notifications. Accepts IANA names (e.g. `America/Los_Angeles`) or UTC offsets. Does not affect parsing of `remind-at`. | No | `UTC` |

## Outputs

| Name | Description |
|------|-------------|
| `reminders-json` | JSON array of all current reminders |
| `due-json` | JSON array of reminders that are currently due |
| `due-count` | Number of reminders currently due |
| `due-guids` | Newline-delimited list of GUIDs for due reminders |
| `total-count` | Total number of reminders in the list |
| `item-guid` | GUID of the newly added reminder (only for `put`) |
| `popped-count` | Number of reminders removed after cron firing |

## Usage

### Schedule a Reminder

```yaml
- uses: aaronsteers/devin-reminders-action@v1
  with:
    action: put
    remind-at: "2026-02-20T17:00:00-08:00"
    reminder-message: "Check on the deployment status"
    agent-session-url: "https://app.devin.ai/sessions/abc123"
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    slack-token: ${{ secrets.SLACK_BOT_TOKEN }}
    slack-channel: devin-reminders
    reminder-timezone: America/Los_Angeles
```

### List Reminders

```yaml
- uses: aaronsteers/devin-reminders-action@v1
  id: reminders
  with:
    action: list
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}

- run: echo "Due: ${{ steps.reminders.outputs.due-count }} / Total: ${{ steps.reminders.outputs.total-count }}"
```

### Full Workflow (Cron + Manual)

```yaml
name: Devin Reminders

on:
  schedule:
    - cron: "*/30 * * * *"
  workflow_dispatch:
    inputs:
      action:
        description: "Action: put, list, or cron"
        required: true
        type: choice
        options: [put, list, cron]
      reminder_message:
        description: "Reminder message (required for put)"
        type: string
      remind_at:
        description: "ISO 8601 timestamp with offset (required for put)"
        type: string
      agent_session_url:
        description: "Devin session URL (required for put)"
        type: string

permissions:
  actions: write

jobs:
  reminders:
    runs-on: ubuntu-latest
    steps:
      - uses: aaronsteers/devin-reminders-action@v1
        with:
          action: ${{ github.event_name == 'schedule' && 'cron' || inputs.action }}
          remind-at: ${{ inputs.remind_at }}
          reminder-message: ${{ inputs.reminder_message }}
          agent-session-url: ${{ inputs.agent_session_url }}
          devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
          slack-token: ${{ secrets.SLACK_BOT_TOKEN }}
          slack-channel: devin-reminders
          reminder-timezone: America/Los_Angeles
```

## How It Works

1. **`put`** schedules a reminder by appending it to a JSON artifact
2. **`list`** reads the artifact, filters due reminders, and outputs counts and JSON
3. **`cron`** reads the artifact, fires each due reminder via the Devin API, pops successful ones, and uploads the updated artifact

## Storage Model

Reminders are stored as a JSON array in a GitHub Actions artifact (`devin-reminders-list`). The artifact is persisted across workflow runs using `dawidd6/action-download-artifact@v14` with `search_artifacts: true`, and updated via `actions/upload-artifact@v4` with `overwrite: true`. Old artifacts expire via the 4-day retention policy.

## License

This project is licensed under the terms of the [MIT License](LICENSE).
