# memos/

Inter-position memos for the roledef-spec org, per the memodef substrate and the OAGP per-recipient-working-repo convention. This folder is the **roledef-strategist seat's correspondence**; memos are addressed to positions, not incumbents, so they are the seat's institutional history and survive across sessions and incumbents.

Each memo is a `memodef:Memo` envelope (`.openthing`). A memo MAY carry its body inline (`body` field) or in a sibling body-ref file (`<same-stem>.body.md`); when a body-ref exists it travels with its envelope.

## Folder convention (inbox / read)

```
memos/
├── README.md   ← you are here
├── inbox/      ← active: incoming memos not yet processed
└── read/       ← archive: processed incoming memos + local copies of sent memos
```

This mirrors the canonical openbraid shape (`memos/inbox/` + `memos/read/`).

- **`inbox/`** — incoming memos addressed to this seat that have **not** yet been acted on. A fresh session reads `inbox/` first and **flags every `action_required: true` item**. When `inbox/` is empty there is no outstanding correspondence for the seat.
- **`read/`** — memos this seat is **done with**: processed incoming memos (action satisfied or informational/FYI), plus local copies of memos this seat has **sent** (whose canonical home is the recipient's own repo; the copy here is for audit co-location).

## Lifecycle (mark-as-read discipline)

1. A new incoming memo lands in `inbox/`.
2. A session processes it — takes the directed action, files any resulting proposal/decision, and (for `action_required` memos) closes the loop with a reply memo to the sender.
3. Once processed, the memo **moves to `read/`** (`git mv`, preserving history). `inbox/` then reflects only genuinely-open correspondence.
4. A memo this seat sends is filed in the recipient's repo; a local copy MAY be archived in `read/` for the audit trail.

## Filename convention

```
YYYY-MM-DD-HHMM--<from-position>--<to-position>--<short-subject>.openthing
```

Body-ref sibling (optional): same stem with `.body.md`.

## Why moved, not deleted

Processed memos are archived to `read/`, never deleted — they are the institutional record. "Decisions made and lost are decisions remade." A future session reconstructs the seat's full history by reading `read/` newest-to-oldest.

---

*Memos are addressed to seats, not people. The substrate is the memory.*
