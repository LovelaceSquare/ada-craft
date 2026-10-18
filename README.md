# Ada Craft

Patches for **WordCraft**, **GridCraft** and **DeckCraft**, the open-source office apps by the
[ArtCraft team](https://github.com/storytold), that let a local assistant such as
[Ada](https://github.com/LovelaceSquare) work side by side with you in the window you already
have open: the same document, spreadsheet or deck, with every change visible as it happens.

> **Not official.** This project is based on WordCraft, GridCraft and DeckCraft by the ArtCraft
> team. It is not made, endorsed or supported by them. The original apps are licensed
> MIT OR Apache-2.0; their `LICENSE-MIT`, `LICENSE-APACHE`, `NOTICE` and `ATTRIBUTION.md` files
> ship unchanged in every build, and the patches here are released under the same dual license.

## What the patches add

All three apps already have a local control channel (JSON lines on `127.0.0.1`) and an MCP
server. The patches make that channel usable for an assistant that joins a window you opened
yourself, and safe to leave on.

| | WordCraft | GridCraft | DeckCraft |
|---|:-:|:-:|:-:|
| Per-window key on every control request | already upstream | **added** | **added** |
| `--control auto`: each window picks a free port | **added** | **added** | **added** |
| "Let assistants connect to this window" setting (off by default) | **added** | **added** | **added** |
| "Assistant connected" indicator in the status bar | **added** | **added** | **added** |
| MCP `--automation-read-root` / `--automation-write-root` | already upstream | **added** | **added** |

- **Per-window key.** When a window opens its control channel it writes a random 256-bit key to
  `control-key.local-<port>` in the app's settings folder and removes it when the window
  closes. Every request must carry the key (or authenticate once per connection); a wrong or
  missing key closes the connection, and a connection that doesn't authenticate within 30 s is
  dropped. The app's own `mcp --connect` reads the key file and sends the key to loopback
  addresses only. Upstream GridCraft and DeckCraft 0.7.0 accepted unauthenticated commands from
  any local program; these patches close that.
- **`--control auto`** (also `<APP>_CONTROL_PORT=auto`): the window listens on a free port
  chosen by the system and names its key file after the real port, so listing the
  `control-key.local-*` files finds every open window.
- **Let assistants connect.** With this setting on (File › Options in WordCraft and GridCraft,
  Preferences in DeckCraft), every window starts as if launched with `--control auto`. It is
  off by default and is also available as a command (`ui.allowAssistants` in WordCraft and
  GridCraft, `app.allowAssistants` in DeckCraft).
- **Indicator.** While at least one authenticated connection is open, the status bar shows
  "Assistant connected"; `ui.inspect` reports the count as `controlClients`.
- **Automation roots for GridCraft and DeckCraft.** `gridcraft-cli mcp` and `deckcraft-cli mcp`
  take `--automation-read-root DIR` and `--automation-write-root DIR`, with WordCraft's
  semantics: every file the agent's tools open, insert, save, export or render to must lie inside
  the matching folder. Paths are made absolute with their links and junctions followed, `..`
  that leaves a root is refused, a root left out grants no access of its kind, and combining the
  roots with `--connect` is an error, since they can't confine a running app. Upstream DeckCraft
  added the roots after 0.7.0 on top of an engine module 0.7.0 doesn't have; the DeckCraft patch
  ports them to 0.7.0, with a small `io` module through which every named-file read and write
  passes.

User-interface text is in English only; the upstream translation catalogs are checked against
each other in their tests.

## Repository layout

```
upstream.json                pinned upstream tag and commit for each app
wordcraft/patches/*.patch    patches on top of WordCraft v0.7.0 (git format-patch)
gridcraft/patches/*.patch    patches on top of GridCraft v0.7.0
deckcraft/patches/*.patch    patches on top of DeckCraft v0.7.0
.github/workflows/build.yml  Windows build
```

No upstream source is copied here. A build clones each app, checks out the pinned commit,
verifies that the release tag still points to it, and applies the patches with `git am`.

## Builds

`build.yml` runs on GitHub Actions (manually, or when the patches change) on `windows-latest`:

1. Clone the upstream app at the pinned commit and apply the patches.
2. `cargo build --release --locked` for the app and its CLI.
3. Package a zip with both executables, the upstream license and notice files, and a
   `build-info.json` that records the upstream repository, tag, commit and the patches applied.
4. Upload the zip with its SHA-256 and a
   [build provenance attestation](https://docs.github.com/actions/security-for-github-actions/using-artifact-attestations),
   so anyone can check that a zip came from this repository and workflow:

   ```sh
   gh attestation verify <app>-ada-<version>-windows-x64.zip --owner LovelaceSquare
   ```

The workflow holds no secrets and signs nothing. Releases that Ada installs are signed outside
GitHub with the same key that signs Ada's own updates; Ada verifies that signature and asks
before installing or updating.

## Building locally

```sh
git clone https://github.com/storytold/wordcraft
cd wordcraft
git checkout <commit from upstream.json>
git am ../ada-craft/wordcraft/patches/*.patch
cargo build --release --locked -p wordcraft -p wordcraft-cli
```

The same steps apply to GridCraft (`-p gridcraft -p gridcraft-cli`) and DeckCraft
(`-p deckcraft -p deckcraft-cli`).

## Moving to a new upstream version

```sh
git checkout -b ada-X.Y.Z vX.Y.Z
git am -3 ../ada-craft/<app>/patches/*.patch   # resolve conflicts if any
cargo test --locked
git format-patch -o ../ada-craft/<app>/patches vX.Y.Z..HEAD
```

Then update the tag and commit in `upstream.json`. A new upstream release is only offered to
Ada users after it builds, its tests pass, and the owner of the release key signs it.

## Upstream

The changes are written in the upstream projects' style so they can be proposed there. If
upstream adopts them, this repository will point back to the official builds.

## License

The patches are licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or
[MIT license](LICENSE-MIT), at your option. WordCraft, GridCraft and DeckCraft are copyright
the ArtCraft team and their contributors, under the same terms.
