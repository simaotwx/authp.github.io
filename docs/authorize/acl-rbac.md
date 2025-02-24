---
sidebar_position: 9
---

# Access Lists and Role-based Access Control (RBAC)

The `allow` and `deny` directives are the series of entries defining how to
authorize claims. In the above example, the plugin authorizes access for the holders of "roles"
claim where values are any of the following: "anonymous", "guest", "admin".

## Sources of Role Information

By default, the plugin finds role information in the following token fields:

* `roles`
* `role`
* `group`
* `groups`
* `app_metadata` - `authorization` - `roles`
* `realm_access` - `roles`

In the below example, the use has a single role, i.e. `anonymous`.

```json
{
  "exp": 1596031874,
  "sub": "jsmith",
  "name": "Smith, John",
  "email": "jsmith@gmail.com",
  "roles": [
    "anonymous"
  ],
  "origin": "localhost"
}
```

Additionally, the token validation component of the plugin recognized that roles
may be in other parts of a token, e.g. `app_metadata - authorization - roles`:

```json
{
  "app_metadata": {
    "authorization": {
      "roles": ["admin", "editor"]
    }
  }
}
```

Additionally, `realm_access` - `roles`:

```json
{
  "realm_access": {
    "roles": ["admin", "editor"]
  }
}
```

References:

* [Auth0 Docs - App Metadata](https://auth0.com/docs/users/concepts/overview-user-metadata)
* [Netlify - Role-based access control with JWT - External providers](https://docs.netlify.com/visitor-access/role-based-access-control/#external-providers)

## Anonymous Role

By default, if the plugin does not find role information in JWT token, then
automatically treats the token having the following two roles:

* `anonymous`
* `guest`

For example, it happens when:
* `roles` and `app_metadata` are not present in a token
* `app_metadata` does not contain `authorization`

## Granting Access with Access Lists

Access list rule consists of 3 sections:

* Comment
* Conditions
* Actions

The rule has the following syntax:

```
acl rule {
  comment
  conditions
  action
}
```

For example:

```
acl rule {
  comment Allow viewer and editor access, log, count, and stop processing
  match roles viewer editor
  allow stop counter log debug
}
```

### Comment

The comment section is a string to identify a rule.

The section is a single statement.

### Conditions

The conditions section consists of one or more statements matching the fields
of a token.

There are the types of conditions:

1. match the value of a particular token field, e.g. `roles`
2. match the HTTP method, e.g. GET, POST, etc.
3. match the HTTP URI path, e.g. `/api`

The condition syntax follows:

```
[exact|partial|prefix|suffix|regex|always] match <field> <value> ... <valueN>
[exact|partial|prefix|suffix|regex|always] match method <http_method_name>
[exact|partial|prefix|suffix|regex|always] match path <http_path_uri>
```

The special use case is the value of `any` with `always` keyword. If provided,
it matches any value in a token field. It is synonymous to the field being
present. For example, the following condition match when a token has `org`
field. The value of the field is not being checked

```
always match org any
```

The following conditions match when a token has `roles` field with the values
of either `viewer` or `editor` and has `org` field with the value of `nyc`.

```
match roles viewer editor
match org nyc
```

The following conditions match when a token has `roles` field with the values
of either `viewer` or `editor` and `org` field begins with `ny`.

```
match roles viewer editor
prefix match org ny
```

### Actions

The actions section is a single line instructing how to deal with a token
which matches the conditions.

The potential values for actions follow. Please note the first keyword
could be `allow` or `deny`.

```
allow
allow counter
allow counter log <error|warn|info|debug>
allow log <error|warn|info|debug>
allow log <error|warn|info|debug> tag <value>
allow stop
allow stop counter
allow stop counter log <error|warn|info|debug>
allow stop log <error|warn|info|debug>
allow any
allow any counter
allow any counter log <error|warn|info|debug>
allow any log <error|warn|info|debug>
allow any stop
allow any stop counter
allow any stop counter log <error|warn|info|debug>
allow any stop log <error|warn|info|debug>
```

By default the ACL rule hits are not being logged or counted.

The `log <error|warn|info|debug>` keyword enables the logging of rule hits.
If the log level is not being set, it defaults to `info`.

The `tag` keyword instructs the plugin to add a tag to the log output.

The `counter` keyword enables the counting of hits. The counters could be
exposed with prometheus exporter.

The `stop` keyword instructs the plugin to stop processing ACL rules after
the processing the one with the `stop` keyword.

The `any` keyword instructs the plugin to trigger actions when any of the
conditions match. By default, all the conditions must match to trigger
actions.

### ACL Shortcuts

Here are the patterns of one-liner allowed for use:

```
allow roles viewer editor with method get /internal/dashboard
allow roles viewer editor with method post
deny roles anonymous guest with method get /internal/dashboard
deny roles anonymous guest with method post
allow roles anonymous guest
allow audience https://localhost/ https://example.com/
```

### Primer

In this example, the user logging via Facebook Login would get role `user`
added to his/her roles. The `acl rule` directives specify matches and actions.

```
localhost, 127.0.0.1 {
  route /auth* {
    authp {
      backends {
        github_oauth2_backend {
          method oauth2
          realm github
          provider github
          client_id Iv1.foobar
          client_secret barfoo
          scopes user
        }
      }
      ui {
        links {
          "My Identity" "/auth/whoami" icon "las la-star"
          "My Settings" /auth/settings icon "las la-cog"
          "Guests" /guest/
          "Users" /app/
          "Administrators" /admin/
        }
      }
      transform user {
        exact match sub 123456789
        exact match origin facebook
        action add role user
      }
      enable source ip tracking
    }
  }

  route /prometheus* {
    authorize {
      primary yes
      allow roles authp/admin authp/user authp/guest
      allow roles admin user guest
      validate bearer header
      set auth url /auth
      inject headers with claims
    }
    respond * "prometheus" 200
  }

  route /guest* {
    authorize {
      acl rule {
        comment allow guests only
        match role guest
        allow stop log error
      }
      acl rule {
        comment default deny
        always match iss any
        deny log error
      }
    }
    respond * "my app - guests only" 200
  }

  route /app* {
    authorize {
      acl rule {
        match role user admin
        allow stop log error
      }
      acl rule {
        always match iss any
        deny log error
      }
    }
    respond * "my app - standard users and admins" 200
  }

  route /admin* {
    authorize {
      acl rule {
        match role admin
        allow stop log error
      }
    }
    respond * "my app - admins only" 200
  }

  route /version* {
    respond * "1.0.0" 200
  }

  route {
    # trace tag="default"
    redir https://{hostport}/auth/login 302
  }
}
```

The log messages would look like this:

```
ERROR   http.authentication.providers.authorize       acl rule hit    {"action": "deny", "tag": "rule1", "user": {"addr":"10.0.2.2","iss":"https://localhost:8443/auth/oauth2/facebook/authorization-code-callback","jti":"yrQcSolE6SZAPeY38szaNQbtUtfyrj0HmfEq8hvL","name":"Paul Greenberg","origin":"facebook","roles":["user","authp/guest"],"sub":"10158919854597422"}}
```

## Default Allow ACL

If `authorize` configuration contains the following directive, then the "catch-all"
action is `allow`.

```
authorize {
  acl default allow
}
```

## Forbidden Access

By default, `caddyauth.Authenticator` plugins should not set header or payload of the
response. However, caddy, by default, responds with 401 (instead of 403),
because `caddyauth.Authenticator` does not distinguish between authorization (403)
and authentication (401).

The plugin's default behaviour is responding with `403 Forbidden`.

However, one could use the `set forbidden url` Caddyfile directive to redirect
users to a custom 403 page.

```
authorize {
  set forbidden url /custom_403.html
}
```
