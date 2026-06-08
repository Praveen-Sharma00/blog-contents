# Two Generals, One Impossible Promise

## The Bug at the Heart of the Internet

> Why "exactly-once delivery" is a lie — and why every distributed system you've ever used is built on a problem that *provably* has no solution.

*~7 min read · for engineers who like their riddles with a side of existential dread.*

---

**TL;DR** — Two parties communicating over an unreliable channel can **never** be 100% certain they agree on anything. It's not unsolved; it's *proven impossible*. This single fact is why TCP is "reliable" only in quotes, why your payment service can double-charge a card, and why "exactly-once delivery" is marketing, not engineering. Here's the story, the one-paragraph proof, and how senior engineers build working systems on top of an impossibility.

---

## 1. The Setup

Picture two generals.

Their armies are camped on two hills, and between them — in the valley below — sits the enemy city. General A is on the east hill. General B is on the west. To win, they must attack **at exactly the same time**:

- ✅ Both attack at dawn → they win.
- 💀 Only one attacks → that army is slaughtered and the war is lost.

There's just one problem. The only way to communicate is to send a messenger *through the valley* — straight through enemy territory. Any messenger might be captured. The message might never arrive. **And the sender has no way of knowing whether it got through.**

---

## 2. The Trap

General A decides: **"We attack at dawn."** He writes it down, hands it to a messenger, and sends him into the valley.

Now pause. Put yourself in A's boots. **Would you attack?**

> No. You have no idea if the message arrived. The messenger could be in an enemy prison right now. Attack alone and you die. You need General B to confirm.

So B receives it and sends an acknowledgment: **"Confirmed. Dawn."**

Now put yourself in B's boots. *He* just sent the confirmation. **Would *he* attack?**

> No. B has no idea if his confirmation got through. If A never receives it, A won't attack — and B attacking alone means B dies. So B needs A to confirm the confirmation.

And A sends *"I got your confirmation."* — but now A is back in the exact trap B was just in. Did **that** message arrive?

You see where this is going.

---

## 3. The Infinite Handshake

Every message needs an acknowledgment. But the acknowledgment is *itself* a message through the same unreliable valley — so it needs its own acknowledgment. Forever.

```
A → B:  "Attack at dawn."
B → A:  "Confirmed."
A → B:  "Got your confirmation."
B → A:  "Got your 'got your confirmation.'"
A → B:  "Got your 'got your got your...'"
        ...
```

No matter how many messages get through, the **last** one is always unconfirmed — and whoever sent it can't be sure it's safe to attack. A thousand round trips, a million: you haven't solved anything. You've just **moved the uncertainty one messenger further down the line.**

> **The Two Generals Problem.** There is no protocol — none, not now, not ever — that lets the generals guarantee a simultaneous attack over an unreliable channel. It was the *first* computer-communication problem ever proven unsolvable.

This isn't "we haven't found a solution yet." It's "we have **proven** no solution can exist."

---

## 4. The Proof (shorter than you'd think)

No PhD required. It's a proof by contradiction, and it's beautiful.

**Step 1 — Assume a solution exists.** Then among all working protocols, one uses the **fewest possible messages**. Call it the *shortest winning protocol*. Say it sends 100 messages — whatever the minimum is.

**Step 2 — Look at the very last message.** Two things must be true about it:

| Fact | Why |
|------|-----|
| **It must matter** | If both generals attacked correctly *without* it, we could delete it — giving a shorter winning protocol. But this was already the shortest. Contradiction. So the last message is **essential**. |
| **Its delivery isn't guaranteed** | The channel is unreliable. That messenger might be captured, and the sender can never know. |

**Step 3 — Collide the two facts.** The last message is *essential* — someone needs it to commit. But it *might vanish*. So when it's the one that gets captured:

- The general waiting on it never gets it → **he won't attack.**
- The sender has no idea it was lost → from his side, everything looks fine.

The two generals are now in **different states**. One attacks, one doesn't. The protocol fails.

**Conclusion.** Our assumption was wrong. No shortest winning protocol can exist — so **no winning protocol exists at all.** ∎

> Every message you add just becomes the *new* "last message," inheriting the exact same fatal flaw. There is no escape.

---

## 5. "Cool Riddle. So What?"

Here's why this should keep an engineer up at night:

> **You've been fighting the Two Generals Problem your entire career — you just called it by other names.**

Swap the metaphor for the machinery:

| Fairy tale | Reality |
|------------|---------|
| Two generals | Two computers |
| Messenger through the valley | Packet over the network |
| The valley | The internet |
| "Did my messenger arrive?" | **Every API call you've ever made** |

### 5a. TCP's "reliable connection" is a comfortable illusion

TCP is sold as a "reliable, connection-oriented protocol." But the network beneath it is just as treacherous as that valley — packets drop, duplicate, and arrive out of order. TCP's famous **three-way handshake** (`SYN` → `SYN-ACK` → `ACK`) *looks* like it solves coordination… but it's just the Two Generals handshake, **stopped early**.

TCP doesn't *defeat* the problem — it *pragmatically gives up* on perfection. It halts the infinite acknowledgment chain at "good enough" and accepts a tiny, non-zero chance of disagreement. That's not a bug. It's the **only thing any protocol can do.**

### 5b. The bug you've actually shipped: the double charge

This one will feel personal. Your app calls a payment API:

```
Your server → Payment API:  "Charge this card $50."
Payment API:                 (charges the card successfully ✅)
Payment API → Your server:   "Success!"   ← this response is LOST ❌
```

The charge **went through**. But the confirmation got eaten by the network. Your server is now General A, staring at the empty valley: *did it work or not?* Both choices are wrong:

- 🔁 **Retry the charge** → you might bill the customer **twice**.
- 🛑 **Don't retry** → you might hand over the product **for free**.

There is no way, from your side, to know which happened. This is the Two Generals Problem wearing a business suit — and it's why **"exactly-once delivery"** is one of the great myths of distributed systems. Anyone who claims their system guarantees it is either wrong, or is quietly doing the trick in the next section.

---

## 6. How Engineers Live With an Unsolvable Problem

We can't *solve* it. So we stopped trying to win the war and started managing the risk. Three moves you'll recognize:

### ① Idempotency keys — make duplicates *harmless*

Instead of *preventing* duplicate messages, make them not matter. Attach a unique ID — an **idempotency key** — to each request. The server remembers it; if the same key shows up twice, it returns the original result instead of acting again. *(This is exactly how Stripe's API works — and now you know why it has to.)* We didn't beat the problem; we made being wrong **cost nothing**.

### ② Pick your poison: at-least-once vs. at-most-once

Since exactly-once is impossible, every message queue (Kafka, SQS, RabbitMQ) forces a choice:

| Guarantee | Never loses? | Never duplicates? | You handle… |
|-----------|:---:|:---:|-------------|
| **At-most-once** | ❌ | ✅ | possible *data loss* |
| **At-least-once** | ✅ | ❌ | *duplicates* (with idempotency ①) |
| ~~Exactly-once~~ | — | — | **doesn't exist** |

There is no third row. The Two Generals Problem is the reason the menu has only two items.

### ③ Trade certainty for "very, very likely"

TCP retransmits a *few* times and calls it reliable. We don't need mathematical certainty — we need the probability of failure small enough that it almost never bites. We swap the impossible dream of *certainty* for the achievable goal of **overwhelming likelihood**.

---

## 7. The Real Lesson

The Two Generals Problem is a rite of passage because it forces a humbling realization:

> **Some problems don't have solutions. Maturity as an engineer is knowing the difference between a hard problem and an impossible one.**

- A **junior** engineer, told to "make sure this is delivered exactly once," grinds for a week trying to build it.
- A **senior** engineer recognizes the shape of the valley, says *"that's the Two Generals Problem — it can't be done,"* and designs a system where **duplicate delivery simply doesn't matter.**

The two generals could never *guarantee* they'd attack together. But a clever general doesn't need a guarantee — he builds a plan that **wins even when a messenger or two never makes it home.**

That's the whole job, really. Not eliminating uncertainty — the math forbids it — but **building things that work anyway.**

---

### Go deeper

If this scratched an itch, the rabbit hole continues:

- **FLP Impossibility Result** — you can't guarantee consensus in an asynchronous network if even *one* node can fail.
- **CAP Theorem** — consistency, availability, partition-tolerance: pick two.

The Two Generals Problem is where that entire story begins.

---

*Enjoyed this? Drop a 👏 and follow for more "wait… *what?*" computer-science deep dives.*
