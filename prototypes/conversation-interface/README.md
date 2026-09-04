# PROTOTYPE / THROWAWAY

This is a HITL comparison for one engineer. It asks one question: which mobile interface gets a User into a Voidstation Conversation with the least friction? It is not production work.

A user-provided T3.chat screenshot informed the structure, not a pixel copy or a brand treatment. Voidstation keeps its own product language and avoids calling Conversations anything else.

## Desktop DNA

Desktop uses a narrow near-black Conversation index, a dark-plum message canvas, a compact User message at right, plain Assistant copy on the canvas, small copy actions, and a bottom composer. The sidebar has search, recent Conversations, New Conversation, and the local User. Settings stays at the upper right.

## Mobile variants

- **A Docked composer** keeps a persistent two-row composer with a visible Message label. Its second row holds model, Web/Search, Skill, and Attach controls.
- **B Compact composer** keeps typing in one row. The plus button opens a separate action tray above it.
- **C Composer sheet** leaves the most room for messages. Its collapsed prompt opens a focused, closable message sheet in one tap.

Use `?variant=A`, `?variant=B`, or `?variant=C`. The default is `A`.

## Palettes

Palette choice is independent from the variant and lives only in the URL.

- Mint `#06d6a0`
- Deep teal `#14b8a6`
- Seafoam `#5eead4`
- Cyan `#22d3ee`
- Amber `#f2b84b`
- Coral `#ff6b6b`

For example: `?variant=C&accent=coral`. The default is Mint.

## Run

From the repository root, run this command and keep that terminal open:

```sh
python3 -m http.server 4173 --bind 0.0.0.0
```

On Windows, open [http://localhost:4173/prototypes/conversation-interface/](http://localhost:4173/prototypes/conversation-interface/).

If Windows cannot reach the WSL server, get the WSL address:

```sh
hostname -I | awk '{print $1}'
```

Then open `http://<that-address>:4173/prototypes/conversation-interface/` from Windows.

## Interaction checklist

- Start a New Conversation and confirm the composer receives focus.
- Select a recent Conversation from the desktop index or mobile drawer.
- Search recent Conversations.
- Send a message, keep typing while the simulated Assistant response waits, then navigate away before it arrives.
- Copy a message and check that only that control changes briefly.
- Open and close the drawer with its button, backdrop, and Escape.
- Open the palette from the bottom prototype switcher and choose each swatch.
- Change variants with the switcher or left and right arrow keys when an editable control is not focused.
- Reload and confirm the URL keeps the variant and palette.

State is in memory only. Reloading clears new Conversations, messages, drafts, selected model, toggles, and attached filename.
