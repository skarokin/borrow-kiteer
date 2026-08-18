# borrow-kiteer

SDK for CRUD on per-user credentials for agent tools.

`kiteer` supports OAuth and direct pasting of credentials, keyed by `user_id` sourced from your identity provider in a backend you already own. Tools get a helper method that attaches the credential at request time, and if nothing is stored yet, you get a URL to send back to the user. Your OAuth callbacks or credential-pasting endpoint get an easy, secure way to update a user's credentials for use in `kiteer`.

`kiteer` assumes the following:
1. You host the OAuth/paste HTTP callbacks (same process as the agent is fine).
2. The agent already knows a trusted user_id (your IdP/session, not the LLM).
3. Agent and callbacks share one credential store.
4. Only those processes can read/write that store; treat rows as secrets.

## Install

```bash
pip install kiteer
```

## Quickstart

**NOTE:** Auth & credential store setup should live in the same module. The agent must be able to access this module.

### Use in Agents

All you need is to register a `kiteer.Auth()` object and use it in your tools.

```python
import kiteer
from kiteer.stores import JSONStore # or your own custom implementation
from my_auth_setup_module import github, openai

# store = kiteer.from_store(dynamodb_store(table="my_table_name", region="us-west-2"))
auth = kiteer.Auth(
    JSONStore(path="/credentials.json"),
    providers=[github, openai]
)

def my_tool():
    # user_id is from YOUR trusted identity provider, ensure it's passed in securely via context variables, etc.
    cred = store.get_credential(user_id="alice", provider="github")
    if cred.auth_invalid:
      # you control how to handle invalid auth. typically, this would surface as an interrupt
      return result.auth_link

    # ... use credential ...
    api_client = some_api_client.get(cred.token)
```

### Auth Setup

There are two supported auth mechanisms: (1) OAuth (2) pasting credentials.

You simply need to use `kiteer.oauth()` or `kiteer.paste()` to set up a credential provider.

#### Registering Providers

```python
# module my_auth_setup_module
import kiteer

github = kiteer.oauth(
    name="github",
    authorize_url="https://github.com/login/oauth/authorize",
    token_url="https://github.com/login/oauth/access_token",
    client_id=...,
    client_secret=...,
    scopes=["repo"],
    redirect_uri="https://auth.example.com/oauth/callback",
)

corp_resource = kiteer.paste(
    name="corp_resource",
    callback_url="https://auth.example.com/paste",
)
```

#### Auth Callbacks

`kiteer` expects you to host the callback HTTP endpoints. The agent and those endpoints share **one** credential store.

When `get_credential` has nothing usable, `auth_link` is a URL with `state=<ticket>`, where ticket is a random string. `kiteer` writes a row in the store with the ticket as the key:

```
ticket -> { user_id, provider, expiry, pkce_verifier }
```

The callback reads `state` from the query string, loads and deletes that row, and then saves the credential.

##### OAuth Callbacks

Point the provider's `redirect_uri` (and the GitHub OAuth app) at this endpoint.

```python
@app.get("/oauth/callback")
def oauth_callback(code: str, state: str):
    auth.complete_oauth(code=code, state=state)
```

`complete_oauth` consumes the ticket, exchanges `code` for tokens, and uses `set_credential` to set the token under the `user_id`.

##### Credential Paste Callbacks

`auth_link` for a paste provider is *your* paste URL plus the same `state` ticket, e.g. `https://auth.example.com/paste?state=...`. Set that public URL on the provider (`callback_url=...`).

```python
@app.post("/paste")
def paste_callback(state: str, secret: str):
    auth.complete_paste(state=state, secret=secret)
```

This does not take `user_id` from the client, it is all provided from state. To reiterate, *only the server hosting your agent needs to resolve user identity!*

### Credential Store Setup

Your store is persistence only. It does **not** implement `complete_oauth` or `complete_paste` — those are methods on `kiteer.Auth`. `Auth` talks to the provider, then calls your store:

```text
auth.complete_oauth(code, state)
  -> store.consume_state(state)      # you implement this
  -> exchange code at token_url
  -> store.set_credential(...)       # you implement this
```

The agent uses `get_credential` and `put_state` (when minting `auth_link`). The auth server only calls `auth.complete_*`; that is what invokes `consume_state` / `set_credential`.

```python
from kiteer.stores import BaseStore
from kiteer.types import Credential, StateTicket

class DynamoDBStore(BaseStore):
    def get_credential(self, user_id: str, provider: str) -> Credential | None:
        pass

    def set_credential(self, user_id: str, provider: str, credential: Credential) -> None:
        pass

    def delete_credential(self, user_id: str, provider: str) -> None:
        # not used by default; useful if you expose a disconnect endpoint
        pass

    def put_state(self, state: str, ticket: StateTicket) -> None:
        # insert a mapping of { ticket -> state }
        # agent uses this when minting auth_link
        pass

    def consume_state(self, state: str) -> StateTicket | None:
        # 1. get the ticket from state.
        # 2. delete the ticket from store.
        # 3. return the ticket; None if missing or expired.
        pass
```