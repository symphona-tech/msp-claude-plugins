# ConnectWise PSA Webhook Configuration

## Webhook Callback

ConnectWise can POST to your endpoint on entity changes:

```json
{
  "Action": "updated",
  "ID": 54321,
  "Type": "ticket",
  "MemberID": 123,
  "Callback": {
    "ID": 54321,
    "Type": "ticket"
  }
}
```

## Registering Callbacks

**No tool registers a callback, and this is not a request to make from here.**
Callback registration is a PSA administration action; the shape below documents
what such a registration looks like so a callback payload received elsewhere can
be understood, not so that a client issues it.

```json
{
  "url": "https://your-server.com/webhook",
  "objectId": 0,
  "type": "ticket",
  "level": "owner",
  "description": "Ticket updates webhook"
}
```
