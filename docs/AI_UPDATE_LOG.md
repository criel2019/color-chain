# AI Update Log (Client)

This file is a human/AI handoff log for the `color-chain` client repository.

## Update Rules (For Future AI / Humans)

1. Append a new entry at the top under `## Entries`.
2. Use a simple update number (`Update #1`, `Update #2`, ...) for this document.
3. Record:
   - update number
   - date
   - commit message summary
   - optional runtime/protocol version note (if relevant)
   - compatibility notes with server repo
   - next tasks
4. Keep entries concise and factual.
5. Do not delete old entries; supersede them with a newer entry.

## Cross-Repo References

- Client repo: `https://github.com/criel2019/color-chain`
- Server repo: `https://github.com/criel2019/color-chain-match-server`
- Client runtime version source: `index.html` (`window.ColorChain.version`)
- Protocol reference: server repo `docs/PROTOCOL.md`

## Entries

### Update #1 | 2026-02-24

- Summary:
- Added localization foundation (`en`, `ko`, `ja`, `es`) and runtime UI text switching.
- Added multiplayer-ready client hooks/events, incoming attack queue, and seedable gameplay RNG for deterministic match starts.
- Added 1v1 matchmaking UI and WebSocket client integration (queue, match, attack relay, result handling).
- Added GitHub Pages deployment workflow for client-only hosting.
- Added external WebSocket endpoint support via query string (`?ws=wss://...`) for hosted matchmaking servers.
- Server runtime was separated into its own repository (`color-chain-match-server`); client no longer includes Node server files.

- Commit intent (this commit):
- Preserve current client state plus handoff/version logging process for future AI collaboration.

- Commit message summary:
- Split client/server responsibilities, add matchmaking UI + external server support, and add AI handoff update log.

- Optional runtime note:
- `window.ColorChain.version` is currently `0.3.0`.

- Compatibility notes:
- Expects server message protocol documented in `color-chain-match-server/docs/PROTOCOL.md`.
- Current versus flow assumes relay-based server (`match_found`, `opponent_attack`, `match_result`, etc.).

- Next recommended client tasks:
- Add rematch/ready UX and reconnect flows.
- Localize matchmaking panel labels/buttons (`net.*`) for non-English locales.
- Split `index.html` into `core/ui/net/i18n` files for maintainability.
- Add protocol version display/validation during `hello`/`welcome`.
