# N8N Workflow Examples

This directory contains example N8N workflows that demonstrate common use cases with Evolution API.

## Available Workflows

### 1. Welcome Message on Group Join

**File:** `welcome-message-on-group-join.json`

**Description:** Automatically sends a random welcome message when a new user joins a specific WhatsApp group, including the current participant count.

**Features:**
- Listens for `group-participants.update` events
- Filters by specific instance, group ID, and "add" action
- Fetches current group participant count
- Sends one of 20 random welcome messages in Portuguese
- Displays participant count (e.g., "You are participant 42/100")

**How it works:**
1. **Webhook Node** - Receives webhook from Evolution API
2. **Edit Fields** - Extracts event data (event type, instance, group ID, action, participants)
3. **user_joined_group (IF)** - Checks if:
   - Event is `group-participants.update`
   - Instance is `+11967308190`
   - Group ID is `120363405226577317@g.us`
   - Action is `add`
4. **Find Group Participants** - Gets current participant list
5. **Send Text Message** - Sends random welcome message with participant count

**To use this workflow:**

1. Import into N8N:
   - Go to N8N (http://localhost:5678)
   - Click "Add workflow" → "Import from File"
   - Select `welcome-message-on-group-join.json`

2. Update the workflow:
   - **Webhook path:** Generate a new webhook ID or keep existing
   - **Evolution credentials:** Configure your Evolution API credentials
   - **Instance name:** Change `+11967308190` to your WhatsApp instance
   - **Group ID:** Change `120363405226577317@g.us` to your target group ID
   - **Welcome messages:** Customize the messages array (optional)

3. Activate the workflow:
   - Click the toggle in the top-right to activate
   - Copy the **Production URL** (not test URL!)

4. Configure Evolution webhook:
   ```bash
   curl -X POST http://localhost:8080/webhook/set/YOUR-INSTANCE-NAME \
     -H "apikey: YOUR_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "url": "http://n8n-core:5678/webhook/YOUR-WEBHOOK-ID",
       "webhook_by_events": false,
       "events": [
         "MESSAGES_UPSERT",
         "MESSAGES_UPDATE",
         "GROUP_PARTICIPANTS_UPDATE",
         "CONNECTION_UPDATE"
       ]
     }'
   ```

**Getting your Group ID:**

To find your WhatsApp group ID, send a message in the group and check the Evolution API logs:

```bash
docker compose logs evolution-api-core | grep "group"
```

Or use the Evolution API to list groups:

```bash
curl -X GET http://localhost:8080/group/fetchAllGroups/YOUR-INSTANCE-NAME \
  -H "apikey: YOUR_API_KEY"
```

**Example Webhook Payload:**

```json
{
  "event": "group-participants.update",
  "instance": "+11967308190",
  "data": {
    "id": "120363405226577317@g.us",
    "author": null,
    "participants": [
      {
        "id": "13026753294427@lid",
        "phoneNumber": "554499424823@s.whatsapp.net",
        "admin": null
      }
    ],
    "action": "add"
  },
  "destination": "http://n8n-core:5678/webhook/651c501f-a6bf-4a10-be2b-7af13cf37d39",
  "date_time": "2025-11-02T00:30:25.622Z",
  "sender": "5511967308190@s.whatsapp.net",
  "server_url": "http://localhost:8080",
  "apikey": "60D4A3985E55-446A-A3A4-5C224F32F1B7"
}
```

## Creating Your Own Workflows

### Required Credentials

Most workflows require Evolution API credentials. To set them up in N8N:

1. Go to **Settings** → **Credentials**
2. Click **Add Credential**
3. Search for "Evolution API"
4. Enter:
   - **Name:** Evolution account (or custom name)
   - **API URL:** `http://evolution-api-core:8080`
   - **API Key:** Your API key from `.env.evolution`

### Common Evolution API Operations

**Messages:**
- Send Text Message
- Send Media (Image, Video, Audio, Document)
- Send Location
- Send Contact

**Groups:**
- List Groups
- Create Group
- Update Group Settings
- Add/Remove Participants
- Promote/Demote Admin
- Leave Group

**Instance:**
- Get Connection State
- Disconnect Instance
- Restart Instance

**Contacts:**
- List Contacts
- Get Contact Info
- Check if Number is on WhatsApp

### Webhook Events

Configure which events trigger your workflow:

- `MESSAGES_UPSERT` - New messages received
- `MESSAGES_UPDATE` - Message status updates (sent, delivered, read)
- `GROUP_PARTICIPANTS_UPDATE` - Group member changes
- `GROUPS_UPSERT` - Group information updates
- `CONNECTION_UPDATE` - Instance connection status changes
- `QRCODE_UPDATED` - QR code updates
- `CALL` - Incoming calls

## Tips & Best Practices

### 1. Use Docker Network Names

Always use container names when services communicate:
- Evolution from N8N: `http://evolution-api-core:8080`
- N8N from Evolution: `http://n8n-core:5678`

### 2. Activate Workflows

Remember to **activate** workflows (toggle in top-right). Test URLs only work during manual testing!

### 3. Filter Events

Use IF nodes to filter events before processing:
```javascript
{{ $json.body.event === "MESSAGES_UPSERT" }}
{{ $json.body.data.key.fromMe === false }} // Not sent by you
{{ $json.body.data.messageType === "conversation" }} // Text only
```

### 4. Error Handling

Add error handling nodes to catch failures:
- **Stop on Error:** Disable in node settings for graceful degradation
- **Error Trigger:** Create a separate workflow to handle errors
- **Retry:** Configure retry attempts in HTTP Request nodes

### 5. Rate Limiting

WhatsApp has rate limits. Avoid:
- Sending too many messages in short time
- Bulk messaging without delays
- Automated responses to every message

Add delays using **Wait** node or **Schedule Trigger** for bulk operations.

### 6. Testing

Use pinned data to test workflows without sending real messages:
- Click node → **Pin data**
- Add sample webhook payload
- Test workflow logic without live data

## Contributing

Have a useful workflow to share? Feel free to add it to this directory with:
- Descriptive filename (kebab-case)
- Documentation in this README
- Example webhook payload (if applicable)

## Support

- [N8N Documentation](https://docs.n8n.io/)
- [Evolution API Documentation](https://doc.evolution-api.com/)
- [N8N Community Forum](https://community.n8n.io/)
