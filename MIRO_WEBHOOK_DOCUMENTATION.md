# Miro Webhook Documentation

This documentation outlines how to configure and receive [Miro](https://mirobiometrics.com/) webhook notifications for real-time enrollment, recognition, and deletion events.

When a webhook URL is configured on an Identity Instance, Miro will send an asynchronous HTTP POST request to that URL each time an enrollment, recognition, or deletion event occurs. This allows your backend to react to biometric events in near real time without.

## Assumptions

* This documentation assumes you have set up a Miro Identity Instance via the [Miro Admin Dashboard](https://admin.mirobiometrics.com/).

* You have an HTTPS endpoint ready to receive webhook events.

## Configuring a Webhook URL

### Step 1: Open Instance Settings

1. Sign in to the [Miro Admin Dashboard](https://admin.mirobiometrics.com/).
2. Navigate to your Identity Instance.
3. Click the **Edit** icon to open the instance settings dialog.

### Step 2: Set the Webhook URL

Enter your endpoint URL in the **Webhook URL** field.

**Requirements:**
| Requirement | Description |
|-------------|-------------|
| Protocol | Must be `https://` |
| Max length | 2048 characters |
| Accessibility | Must be publicly reachable (no localhost, private IPs, or link-local addresses) |

### Step 3: Save

Click **Save**. The dashboard will display a green checkmark next to the URL once it is accepted.

### Clearing a Webhook

To stop receiving notifications, clear the Webhook URL field and save. A confirmation dialog will appear before the URL is removed.


## Webhook Payload

When an event occurs, Miro sends a `POST` request to your configured URL with a JSON body.

**Headers:**
```
Content-Type: application/json
```

**Body:**
```json
{
  "action": "ENROLL",
  "timestamp": "2026-03-04T12:00:00.000Z",
  "instanceId": "i.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "profileId": "p.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "customerId": "user-123",
  "requestId": "r.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `action` | string | The event type: `ENROLL`, `RECOGNIZE`, `DELETE`, or `ADMIN_DELETE` |
| `timestamp` | string | ISO 8601 timestamp of the event |
| `instanceId` | string | The Identity Instance that processed the request |
| `profileId` | string | The profile involved in the event |
| `customerId` | string \| null | The customer provided ID associated with the profile, if one was provided during enrollment |
| `requestId` | string | Unique identifier for the API request that triggered the event |


## Behavior Notes

* Your endpoint must accept `POST` requests and return a response within **5 seconds**. Miro does not retry failed deliveries.
* Webhooks are delivered on a **best-effort** basis. Failed deliveries (network errors, timeouts) are **not retried**.
* Events are sent in batches as they occur. Delivery order is not guaranteed.
* Webhook delivery does not affect the outcome of the originating API request; an enrollment or recognition will succeed regardless of whether the webhook delivery succeeds.
* Webhooks are usually delivered within a few seconds of an event, though an SLA is not guaranteed.
