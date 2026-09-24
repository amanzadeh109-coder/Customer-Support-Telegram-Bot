# Customer Support Ticket Bot

An n8n workflow that turns Telegram messages into support tickets. A local Gemma 4 model classifies each message as **billing**, **technical**, or **general**; n8n records the ticket in Google Sheets and sends a category-specific reply. Billing questions about USD/EUR exchange rates also use a live public API. Unrecognized input and model failures follow a fallback route.

## Workflow

```text
Telegram Trigger → Ollama HTTP Request → Parse/classify response (Code)
                                         ├→ Google Sheets: Append Row
                                         └→ Switch by AI category
                                            ├→ billing → Frankfurter API → Telegram billing reply
                                            ├→ technical → Telegram technical reply
                                            ├→ general → Telegram general reply
                                            └→ fallback → Telegram fallback reply
```

The Switch uses the category returned by Gemma 4. The Code node validates that category and supplies `fallback` when the response is missing, invalid, or unsupported. The Google Sheets branch records the ticket regardless of which reply branch runs.

## Stack

- n8n in Docker, with a persistent Docker volume
- Telegram Bot API for incoming messages and replies
- Ollama running locally with `gemma4:e2b`
- Google Sheets for ticket logging
- [Frankfurter](https://frankfurter.dev/) for the public USD/EUR reference rate
- Cloudflare Tunnel for an HTTPS address reachable by Telegram

## Requirements

- Docker and a running n8n instance
- Ollama and the `gemma4:e2b` model available to n8n
- A Telegram bot created through BotFather
- A Google Cloud project with Google Sheets API enabled, an OAuth web client, and an account permitted to authorize the app
- A Google Sheet whose first row contains `timestamp`, `user`, `category`, and `original_message`
- A public HTTPS URL for n8n's Telegram webhook; this project used a temporary Cloudflare Tunnel

## Set up

1. Import the exported workflow JSON into n8n.
2. Connect your Telegram bot credentials to the Telegram Trigger and all Telegram reply nodes.
3. Make Ollama reachable from the n8n container. In this Docker Desktop setup, the classification HTTP Request calls `http://host.docker.internal:11434/api/chat`. Change the hostname if your setup differs. Make sure the specified model is installed.
4. Connect the Google Sheets OAuth2 credential to **Append row in sheet**. In Google Cloud, add the **exact OAuth Redirect URL displayed in the n8n credential** to the OAuth web client's authorized redirect URIs. Select your own spreadsheet and sheet tab in the node.
5. Ensure n8n advertises the correct public HTTPS URL to Telegram. This setup configured `WEBHOOK_URL` and `N8N_EDITOR_BASE_URL` for the tunnel. Check the redirect URL n8n displays before authorizing Google OAuth; if it changes, update the Google Cloud OAuth client accordingly.
6. Review the Ollama classification prompt, the Switch rules, and the Telegram reply text, then publish the workflow.
7. Send a new message to your Telegram bot and check that it replies and appends a row in Google Sheets.

**Keep credentials private.** Do not commit Telegram tokens, OAuth client secrets, or other credentials. Inspect the exported workflow JSON for pasted secrets before pushing it to GitHub. n8n credential references in an export must be reconnected in another n8n instance.

## Example behavior

| Telegram message | AI category | Expected result |
| --- | --- | --- |
| “My card was charged twice. I need a refund.” | Billing | Billing acknowledgment and a logged ticket |
| “What is the USD to EUR exchange rate for my payment?” | Billing | Billing acknowledgment plus the API's reference rate and date |
| “I can't log in to my account.” | Technical | Technical support reply and a logged ticket |
| A general support question | General | General support reply and a logged ticket |
| Unsupported or unrecognized input | Fallback | Fallback reply and a logged ticket |

The public exchange-rate request runs on the billing branch. The reply includes its rate only for exchange-related questions when the API response includes a rate. The API node is configured to continue on error, so a billing acknowledgment can still be sent if that request fails.

## Testing

The workflow was tested with billing, technical, and general messages; an exchange-rate question returned a dated USD/EUR rate. Tickets appeared in the Google Sheet. A photo without a text caption exercised the fallback route. After publishing, a fresh Telegram message received a reply without pressing **Execute workflow** in the editor.

For a submission or demo, include the workflow JSON and screenshots of the workflow, representative Telegram replies, and the matching Google Sheets rows. Hide account details and tokens in screenshots.

## Deployment notes

- Ollama, Docker, n8n, and the tunnel must be running for this local deployment to process new Telegram messages.
- A temporary `trycloudflare.com` address can change when the tunnel restarts. If it does, update n8n's public URL and any OAuth redirect URI that uses that address, then recheck the Telegram trigger.
- If the Google OAuth app is in testing mode, confirm that the signing-in account is an authorized test user. Authorization may need renewal according to Google's testing rules.
- The exchange rate is a reference rate, not a promise of the final rate applied to a particular payment.
