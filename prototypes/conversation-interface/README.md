# PROTOTYPE / THROWAWAY

This is planning code for one question: what interface should Voidstation alpha use for starting, resuming, and holding Conversations on desktop and mobile?

It is for one engineer. Starting a Conversation is the priority. Nothing here is production-ready.

## Variants

- **A Prompt-first rail** puts an empty composer in the main area and keeps recent Conversations in a rail.
- **B Split ledger** keeps a Conversation index, message record, and context area visible on desktop.
- **C Command canvas** gives the composer the centre and holds the active Conversation as a compact trace.

Use `?variant=A`, `?variant=B`, or `?variant=C`. The default is A.

## Accents

Accent is independent from layout. Use `?accent=amber`, `?accent=cyan`, or `?accent=lime`. The default is amber.

## Run

From the repository root:

```sh
python3 -m http.server 4173
```

Open [http://localhost:4173/prototypes/conversation-interface/?variant=A&accent=amber](http://localhost:4173/prototypes/conversation-interface/?variant=A&accent=amber).

## Interaction checklist

- Start a new Conversation from the visible composer.
- Select a recent Conversation to resume it.
- Open the mobile Conversation list with the menu control.
- Send a prompt and wait for the simulated assistant response.
- Change variants with the floating controls or left and right arrow keys outside editable controls.
- Change the accent in prototype chrome.
- Reload the page and confirm the URL preserves variant and accent.

State is in memory only. Reloading clears new Conversations, messages, and drafts. Variant and accent come from the URL only.
