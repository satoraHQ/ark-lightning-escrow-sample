# Edge Cases

A list of things outside the happy path.

## Boltz fails to create entry Lightning invoice

- Retry creating the LN-Arkade swap (client-side).

## Entry Lightning invoice expiry

- Retry creating the LN-Arkade swap (client-side).

## Escrow server fails to complete the offchain transaction (to claim or refund)

- The gem submits the checkpoints fully signed, so Arkade finalizes within the `submit` call (one-shot submit). A spend is no longer left "registered but not finalized" because the arbiter stopped between two calls.
- What can still go wrong is the arbiter not learning the outcome: a timeout or crash after Arkade accepted the submit. An error from `spend_escrow_offchain` therefore does not mean the escrow is unspent.
- The gem records the spend in its store *before* submit, so the fix is to call `spend_escrow_offchain` again with the same trade ID, or `resume_pending_spend(trade_id)`. Arkade answers the re-submit with "duplicated offchain tx"; the gem then reads the co-signed ark tx and the server-signed checkpoints back from the indexer, verifies them, and finalizes (a no-op if Arkade already had).
- The one case it cannot finish alone is an indexer that does not serve the server's signatures: the record stays at `submitting` and a later retry tries again.

_`ark-escrow` now covers this._

## Escrow contract expires and becomes recoverable

- Instead of trying to release directly from recoverable state, first refresh the escrow VTXO back into the same escrow address via the arbiter backend.
- After the refreshed escrow VTXO is spendable again, run the normal offchain release/refund flow.
- It's just another mode of *spending* in Arkade.

_`ark-escrow` + `@satora/swap` now cover this._

## Boltz fails to pay exit Lightning invoice

- After the exit VHTLC has been funded (transaction from escrow to VHTLC), if Boltz encounters an error paying the Lightning invoice, the only way to continue is to refund the VHTLC (collaboratively with Boltz for convenience/speed or unilaterally after a timelock expires).
- Lendaswap *could* offer a feature to retry, by routing the refund into another Boltz VHTLC (a new Arkade-LN swap).
- Alternatively, refund could go into an Arkade wallet for the client. Retry would then be executed at that level instead.
- Since the Boltz VHTLC is just another type of VTXO, it can also expire and become recoverable. In such a scenario, refund would require refreshing/settling that VTXO independently on the client or with help from infrastructure that participates in that contract.

_Our libraries should help you with this. To be implemented._

## Unilateral exit in case Arkade is down or loss of confidence

Usually not exposed directly to the client (inconvenient, hard UX), but it should be made possible (for example through an escrow recovery tool).

_Our libraries should help you with this. To be implemented._

## Arbiter refuses to sign

- Alice and Bob can cooperate (outside of the platform).
- An escrow recovery tool (similar to the one used when Arkade disappears, but with offchain transactions as a default) can facilitate this.

_Our libraries should help you with this. To be implemented._
