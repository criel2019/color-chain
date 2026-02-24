# Color Chain

Standalone fork extracted from `partydeck` (`Temps/color-chain.html`).

## Local Play

Open `index.html` in a browser.

## 1v1 Matchmaking

This client supports 1v1 versus matchmaking through an external WebSocket match server.

The server is managed in a separate repository (`color-chain-match-server`) so client and server can evolve independently.

- Server repo: `https://github.com/criel2019/color-chain-match-server`

### Play 1v1

1. Open `index.html` in two browsers/devices.
2. Run or deploy the separate match server.
3. In each client, connect to the same server URL.
4. Enter a player name and press `FIND MATCH`.
5. When matched, the game starts automatically in `VERSUS` mode.

## Notes

- Multiplayer client uses a relay-based match protocol (attack packets + result relay).
- See `docs/FOUNDATION_MULTIPLAYER_I18N.md` for the expansion plan.
- See `docs/AI_UPDATE_LOG.md` for AI/handoff update history.

## GitHub Pages (Client Only)

GitHub Pages can host the game client (`index.html`) but cannot run the Node/WebSocket matchmaking server.

- Added workflow: `.github/workflows/pages.yml`
- After enabling Pages in repo settings, pushes to `main` will deploy the client
- Connect the hosted page to an external WebSocket server

Example:

- `https://<user>.github.io/<repo>/?ws=wss://your-match-server.example.com`
