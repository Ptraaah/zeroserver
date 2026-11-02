# Security notes and threat model

This document is a working security review of ZeroServer. It records what the current
design can and cannot promise, and what still has to be confirmed by reading
`index.html` line by line.

Scope of this pass: `README.md` and the repository layout. `index.html` was not part of
the sample that was reviewed, so every item below marked **[verify]** is a question to
answer against the source, not a confirmed defect. Items without that marker follow
directly from the documented design.

## 1. What the SHA-256 badge actually proves

The README says the receiver hashes every chunk and that "a green badge means the copy
is identical to the original". Two different properties are being conflated here:

- **Integrity** - the bytes that arrived are the bytes that were hashed.
- **Authenticity** - the hash being compared against really came from the sender.

Hashing every chunk on arrival proves integrity. It proves authenticity only if the
expected hash reached the receiver by a path an attacker cannot also rewrite. If the
sender's per-file digest travels inside the same WebRTC data channel as the file data,
then anyone who can alter the data stream can alter the digest with it, and the badge
still turns green. In that case the check is a corruption detector (bit rot, truncated
chunks, a buggy resume path) and not a tamper detector. The README wording should say
so, and the UI should not imply more.

**[verify]** Where does the expected hash come from - a message on the same
`RTCDataChannel`, or something the receiver derives independently? Is there a per-chunk
MAC, or only a whole-file digest computed by the sender?

Practical fixes, cheapest first:

1. Word the badge accurately: "no corruption detected" rather than "identical to the
   original", unless the hash has an independent path.
2. Put the digest in the link fragment (`#...`) next to the peer ID. The fragment is
   never sent to a server, so the receiver compares against a value that arrived over a
   different channel than the file bytes, and a relay or signalling party cannot
   rewrite both consistently.
3. Keep the resume path's re-read-and-rehash behaviour; it is the right shape.

## 2. The trust boundary is the signalling exchange, not the badge

File data is DTLS-encrypted peer to peer, which does defend against a network attacker
between the two browsers. But WebRTC authenticates peers by DTLS fingerprints exchanged
during signalling. Anyone who controls or can tamper with the signalling exchange - the
public PeerJS broker, or a network attacker who can rewrite it - can substitute
fingerprints and sit in the middle. The DTLS layer will be perfectly happy, and no
end-to-end hash check inside the channel will notice.

So the honest statement is: "no server ever holds your file", not "no server can
influence your transfer". Both can be true at once.

**[verify]** Is the broker used only for `offer`/`answer`/ICE, with no application data
relayed through PeerJS? Is the DTLS fingerprint ever pinned or displayed out of band?
Is the TURN credential worker scoped so it hands out short-lived credentials only?

Related: the page loads PeerJS and qrcode-generator from a CDN. Without Subresource
Integrity and a Content-Security-Policy, a compromised or malicious CDN serves code
that can read every file the user picks. That is the largest single supply-chain risk
in a "one file, no build step" design.

**[verify]** Do the `<script>` tags carry `integrity` and `crossorigin`? Is there a
`Content-Security-Policy` meta tag restricting `script-src` to the two CDNs?

## 3. Joining a room: guessing and brute force

- A 4-digit PIN is 10,000 possibilities. That is only meaningful with a hard attempt
  limit, an enforced delay, and a constant-time comparison. Without a lockout it is a
  speed bump measured in seconds.
- **[verify]** Is the PIN checked before any file metadata is sent, or after the
  receiver has already seen the file list? Is the attempt counter on the sender side
  (the only side that can be trusted to enforce it), and does a failed attempt tear the
  channel down rather than just returning `false`?
- **[verify]** How much entropy does the peer ID carry, and is it regenerated per
  session? A short or sequential ID is guessable, and the broker will happily connect a
  stranger who guesses it.
- One-time links and the 30-minute expiry must be enforced by the sender tab flipping
  state, since there is no server to enforce anything. Confirm a completed transfer
  actually kills the peer, and that a reconnecting receiver cannot resume a link that
  was declared dead.

## 4. A hostile peer

Receiver attacking the sender:

- Declaring interest and then never reading, stalling the sender's flow control
  indefinitely - needs an idle timeout on the sender side.
- Requesting repeated re-sends of chunks already delivered, to burn the sender's CPU
  and bandwidth.
- Fast reconnect cycling to multiply TURN relay allocation and exhaust the free quota,
  which degrades the service for everyone.

Sender attacking the receiver:

- Announcing a small file list and then streaming more bytes than declared - the
  receiver must bound total bytes and per-file bytes by what was announced and abort.
- Filenames that are absolute paths or contain `..`. The File System Access API makes
  the user pick a destination, which mitigates this, but OPFS paths and displayed names
  still need sanitising.
- Enormous declared sizes to trigger the "refuse up front" free-space check and then
  sending something different, or tiny declared sizes to bypass the check and blow
  memory in the in-memory fallback.
- Never sending the final chunk, so the receiver keeps partial data and an open hash
  context forever.

## 5. Recommendations, ordered by value per line of code

1. Add `integrity` + `crossorigin` to both CDN script tags, and a CSP meta tag. Highest
   value, smallest diff.
2. Make the SHA-256 badge's wording match what it actually proves; move the expected
   digest into the link fragment if it is currently sent in-band.
3. Add a hard attempt limit and channel teardown for PIN failures.
4. Add an idle timeout and a declared-size budget on the receiving side.
5. Sanitise announced filenames before display and before any OPFS write.

Nothing here requires a backend, a build step or a dependency change.
