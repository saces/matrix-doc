# MSC1957: Integration manager discovery

*Note*: This is a required component of [MSC1956 - Integrations API](https://github.com/matrix-org/matrix-doc/pull/1956)

Users should have the freedom to choose which integration manager they want to use in their client, while
also accepting suggestions from their homeserver and client. Clients need to know where to find the different
integration managers and how to contact them.


## Proposal

A single logged in user may be influenced by zero or more integration managers at any given time. Managers
are sourced from the client's own configuration, homeserver discovery information, and the user's personal
account data in the form of widgets. Clients should support users using more than one integration manager
at a given time, although the rules for how this can be handled are defined later in this proposal.

#### Client-configured integration managers

This is left as an implementation detail. In the case of Riot, this is likely to be part of the existing
`config.json` options, although liekly modified to support multiple managers instead of one.

#### Homeserver-configured integration managers

The integration managers suggested by a homeserver are done through the existing
[.well-known](https://matrix.org/docs/spec/client_server/r0.4.0.html#get-well-known-matrix-client) discovery
mechanism. The added optional fields, which should not affect a client's ability to log a user in, are:
```json
{
    "m.integrations": {
        "managers": [
            {
                "api_url": "https://integrations.example.org",
                "ui_url": "https://integrations.example.org/ui"
            },
            {
                "api_url": "https://bots.example.org"
            }
        ]
    }
}
```

As shown, the homeserver is able to suggest multiple integration managers through this method. Each manager
must have an `api_url` which must be an `http` or `https` URL. The `ui_url` is optional and if not provided
is the same as the `api_url`. Like the `api_url`, the `ui_url` must be `http` or `https` if supplied.

The `ui_url` is ultimately treated the same as a widget, except that the custom `data` is not present and
must not be templated here. Variables like `$matrix_display_name` are able to function, however. Integration
managers should never use the `$matrix_user_id` as authoritative and instead seek other ways to determine the
user ID. This is covered by other proposals.

The `api_url` is the URL clients will use when *not* embedding the integration manager, and instead showing
its own purpose-built interface.

#### User-configured integration managers

Users can specify integration managers in the form of account widgets. The `type` is to be `m.integration_manager`
and the content would look something similar to:
```json
{
    "url": "https://integrations.example.org/ui?displayName=$matrix_display_name",
    "data": {
        "api_url": "https://integrations.example.org"
    }
}
```

The `api_url` in the `data` object is required and has the same meaning as the homeserver-defined `api_url`.
The `url` of the widget is analogous to the `ui_url` from the homeserver configuration above, however normal
widget rules apply here.

The user is able to have multiple integration managers through use of multiple widgets.

#### Display order of integration managers

Clients which have support for integration managers should display at least 1 manager, but may decide
to display multiple via something like tabs. Clients must prefer to display the user's configured
integration managers over any defaults, and if only displaying one manager must pick the first
manager after sorting the `state_key`s in lexicographical order. Clients may additionally display
default managers if they so wish, and should preserve the order defined in the various defaults.
If the user has no configured integration managers, the client must prefer to display one or more
of the managers suggested by the homeserver over the managers recommended by the client.

The client may optionally support a way to entirely disable integration manager support, even if the
user and homeserver have managers defined.

The rationale for having the client prefer to use the user's integration managers first is so that
the user can tailor their experience within Matrix if desired. Similarly, a homeserver may wish to
subject all of their users to the same common integration manager as would be common in some organizations.
The client's own preference is a last ditch effort to have an integration manager available to the
user so they don't get left out.

#### Displaying integration managers

Clients simply open the `ui_url` (or equivalent) in an `iframe` or similar. In the current ecosystem,
integration managers would receive a `scalar_token` to idenitify the user - this is no longer the case
and instead integration managers must seek other avenues for determining the user ID. Other proposals
cover how to do this in the context of the integrations API.

Integration managers shown in this way must be treated like widgets, regardless of source. In practice
this means exposing the Widget API to the manager and applying necessary scoping to keep the manager
as an account widget rather than a room widget.


## Tradeoffs

We could limit the user (and by extension, the homeserver and client) to exactly 1 integration manager
and not worry about tabs or other concepts, however this restricts a user's access to integrations.
In a scenario where the user wants to use widgets from Service A and bots from Service B, they'd
end up switching accounts or clients to gain access to either service, or potentially just give up
and walk away from the problem. Instead of having the user switch between clients, we might as well
support this use case, even if it is moderately rare.

We could also define the integration managers in a custom account data event rather than defining them
as a widget. Doing so just adds clutter to the account data and risks duplicating code in clients as
using widgets gets us URL templating for free (see the section later on in this proposal about account
widgets for more information).


## Future extensions

Some things which may be desirable in the future are:
* Avatars for the different managers
* Human-readable names for the different managers
* Supporting `ui_url`s targeting specific clients for a more consistent design


## Security considerations

When displaying integration managers, clients should not trust that the input is sanitary. Integration
managers may only be at HTTP(S) endpoints and may still have malicious intent. Ensure any sandboxing
on the manager is appropriate such that it can communicate with the client, but cannot perform unauthorized
actions.