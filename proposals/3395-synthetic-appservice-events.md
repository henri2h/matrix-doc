# MSC3395: Synthetic Appservice Events

Most services today have the concept of a webhook, which is used to inform an external service about
the state of another service. For instance when a new user is registered on a platform, it is useful
to alert a bot to send a welcome message.

In Matrix we already have the concept of services which complement the homeserver, known as Application
Services. However, they can only be informed of user actions that have taken place in rooms (with
the notable exception of presence / device updates via EDUs). This proposal aims to extend the existing
AS spec to include "synthetic" events where the homeserver can inform the appservice when a variety of
things have happened on the homeserver.

For clarity, this MSC introduces a modest set events that could be extended by further MSCs:

- User registration
- User login
- User logout
- User deactivation


## Proposal

An appservice must subscribe to the changes that it wishes to listen for. This can be done by setting a new
key in the appservice registration file: `m.synthetic_events`:

```yaml
m.synthetic_events:
    events:
        - "m.user.registration"
        - "m.user.login"
        - "m.user.logout"
        - "m.user.deactivated"
```

Then, when the homeserver wishes to inform a appservice of an event it would send the event over the appservice `/transaction`
API. The homeserver is NOT expected to retry these events if the appservice is down, 

```
PUT /_matrix/app/v1/transactions/{txnId}
```

```json5
{
    "m.synthetic_events": [{
        "type": "m.user.deactivated",
        "content": {
            "user_id": "@alice:example.com"
        },
        "ts": 1432735824653,
    }],
    "events": [/* ... */]
}
```

For each of the event types given above, there is a definition:

### `m.user.registration`

This should be sent when a new user registers on the homeserver.

```json5
{

        "type": "m.user.registration",
        "content": {
            "user_id": "@alice:example.com"
        }
}
```

### `m.user.login`

This should be sent when an existing user logs in on the homeserver.

```json5
{

        "type": "m.user.login",
        "content": {
            "user_id": "@alice:example.com",
            "device_id": "ABCDEF"
        }
}
```

### `m.user.logout`

This should be sent when an existing user logs out of their session.

```json5
{

        "type": "m.user.logout",
        "content": {
            "user_id": "@alice:example.com",
            "soft_logout": true,
            "device_id": "ABCDEF"
        }
}
```

### `m.user.deactivated`

This should be sent when an existing user deactivates their account.

```json5
{

        "type": "m.user.logout",
        "content": {
            "user_id": "@alice:example.com",
        }
}
```

## Potential issues


Appservices can now request permissions to reveal quite intimate details about each user, which means that homeserver
administrators will need to be more careful when adding new appservice registrations. However, it has always been the
case that appservices should be considered powerful tools that need review before being connected.

This proposal would not work over a federated context, as federated homeservers are not aware of foregin appservices.
That being said, the events suggested in this proposal are sensitive and are expected to only be shared with immediate
appservices connected to the homeserver. It would be possible for an appservice to "proxy" these events to a seperate room
if federation of the information is desired, though.


## Alternatives

One alternative would be to have homeserver implementations make an "events" room, which sends these events
as normal room events rather than synthetic events. Appservices would be joined and left to this room and would 
recieve events organically. However there are some problems with this approach:

- It increases the burden on the homeserver implementor to manage these rooms, including membership of 
  each appservice.
- It would require each of these events to be stored in the homeserver database, which would increase storage 
  costs over time and would require homeservers to store lots of activity in the database.
- While varying by implementation, it would be slower to store an event in a room and send it to an AS, than
  just to send it.
- Each event type would need it's own room to allow fine grained control over which events the appservice would
  recieve, at least until a form of per-message ACLs lands in the spec. 

For these reasons, the suggested proposal was chosen.

## Security considerations

These events will be sent down the same channel as other AS events, and so the security footprint
is not impacted.

## Unstable prefix

All keys mentioned in this document beginning with `m.` will use `uk.half-shot.mscXXXX.` as the prefix.