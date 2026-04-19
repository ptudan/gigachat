# gigachat (static Nostr channel chat)

`gigachat` is a no-backend, static HTML app for Nostr channel chat.

pages: https://ptudan.github.io/gigachat/

- Uses NIP-28 style channel events:
  - `kind:40` for topic/channel creation
  - `kind:42` for channel messages
- Uses a browser-local temporary keypair (`localStorage`) for anonymous posting.
- Supports:
  - topic create/join
  - chat stream in each topic
  - nested thread overlay (max depth 4)