# PROTOTYPE / THROWAWAY

This is a HITL comparison for one engineer. It asks one question: which mobile interface gets a User into a Voidstation Conversation with the least friction? It is not production work.

A user-provided T3.chat reference informed the structure, not a pixel copy or a brand treatment. Voidstation keeps its own product language and avoids calling Conversations anything else. A second smartphone reference informed the mobile axis.

## Desktop DNA

Desktop uses a narrow near-black Conversation index, a dark-plum message canvas, a compact User message at right, plain Assistant copy on the canvas, icon-only copy actions, and a bottom composer. The sidebar has Conversation search, recent Conversations, New Conversation, and the local User. The prototype palette stays at the upper right.

## Mobile variants

Variant A is the current preference, pending final revision approval. B and C remain available for comparison.

- **A Docked composer** uses two small floating clusters at the top instead of a title bar. The left cluster opens Conversations, searches Conversations, or starts a new Conversation. The right control opens the prototype palette. Its compact bottom composer has a plus action menu for Model, reasoning effort, and Skills.
- **B Compact composer** keeps the shared one-row composer and action menu in a conventional mobile layout.
- **C Composer sheet** leaves more room for messages. Its collapsed prompt opens a focused, closable message sheet with the shared controls.

Use `?variant=A`, `?variant=B`, or `?variant=C`. The default is `A`.

Agent search is always available to Voidstation, so the composer has no Web or Search capability toggle. Conversation search remains in navigation.

The action menu keeps Model choices as prototype data: Auto, Kimi K2, and OpenAI-compatible. Reasoning effort is in-memory only and offers Instant, Low, Medium, and High. Skills opens a native dialog with the only decided alpha candidate, the built-in `proof` Skill. It starts enabled and can be disabled. Runtime details are still being decided.

Attachment is a paperclip-only control inside the `+` action menu. It has an accessible name, and a selected filename appears in the composer status text.

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
- Search recent Conversations from navigation.
- Open the Variant A action menu. Change Model or reasoning effort, open Skills, then enable or disable `proof`.
- Attach a file with the paperclip and check that its name appears in composer status text.
- Send a message, keep typing while the simulated Assistant response waits, then navigate away before it arrives.
- Copy a message and check that its icon and accessible status change briefly.
- Open and close the drawer, Skills dialog, and palette with their controls, backdrops, and Escape.
- Open the palette from Variant A's top-right cluster or the bottom prototype switcher and choose each swatch.
- Change variants with the switcher or left and right arrow keys when an editable control is not focused.
- Reload and confirm the URL keeps the variant and palette.

State is in memory only. Reloading clears new Conversations, messages, drafts, selected model, reasoning effort, Skill state, and attached filename.
