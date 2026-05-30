# High-Stakes Actions

Use this reference when reviewing flows where the app should not predict success locally before server confirmation.

This reference is about confirmation, trust, and preventing misleading UI in financial, legal, destructive, irreversible, security-sensitive, medical, identity-related, or server-authoritative actions.

For reversible optimistic UI, read `references/optimistic-updates.md`. For loading indicators and state transitions, read `references/loading-states.md`. For perceived-performance validation, read `references/validation-and-testing.md`.

## Core Idea

High-stakes actions should not appear complete until the authoritative system confirms the result.

In low-risk flows, optimistic UI can improve perceived performance by showing the expected local result immediately. In high-stakes flows, premature success can mislead the user, create trust problems, or cause serious product, legal, financial, security, or support issues.

The goal is not to make the action feel instant. The goal is to make the state honest, clear, and trustworthy.

## What the Agent Can and Cannot Decide

The agent can inspect and improve:

* UI that shows success before server confirmation
* missing progress state during submission or authorization
* missing duplicate-submission protection
* missing failure state
* unclear final confirmation
* destructive actions without confirmation
* local state that can diverge from server-authoritative state
* flows that use optimistic UI where rollback would be unsafe
* flows that need idempotency or request tracking
* copy that implies completion too early

The agent cannot decide alone:

* whether a product action is legally high-stakes
* whether a financial or medical action has compliance requirements
* whether a business domain permits local prediction
* whether a specific delay or confirmation step is required by regulation
* whether server-side semantics make an action reversible

When risk is unclear, recommend confirming the behavior with product, design, backend, legal, security, or compliance owners.

## High-Stakes Candidates

Treat these as high-stakes by default:

* money transfer
* payment authorization
* purchase confirmation
* account deletion
* irreversible content deletion
* legal consent
* identity verification
* medical submission
* security setting change
* password, email, phone, or recovery-method change
* permission or access-level change
* booking scarce inventory
* server-calculated eligibility
* server-calculated pricing
* trading, investment, or loan actions
* actions that affect another person or organization

Do not show a final success state for these actions until the authoritative response arrives.

## Main Anti-Pattern: Optimistic Final Success

Risky:

```swift
@MainActor
func confirmTransfer() {
    state = .confirmed(
        message: "Transfer complete"
    )

    Task {
        try await transferService.submitTransfer()
    }
}
```

This tells the user the transfer is complete before the server confirms it.

Prefer explicit progress and final confirmation:

```swift
@MainActor
func confirmTransfer() async {
    state = .submitting
    isSubmitDisabled = true

    do {
        let receipt = try await transferService.submitTransfer()
        state = .confirmed(receipt)
    } catch {
        state = .failed(error)
        isSubmitDisabled = false
    }
}
```

The UI responds immediately by entering a submitting state, but final success appears only after confirmation.

## State Model

High-stakes flows need explicit states.

Risky:

```swift
enum TransferState {
    case idle
    case done
}
```

This cannot distinguish waiting, confirmed success, failure, or retry.

Prefer:

```swift
enum TransferState {
    case idle
    case reviewing(TransferDraft)
    case submitting
    case confirmed(TransferReceipt)
    case failed(TransferError)
}
```

For flows with server-side authorization, use names that reflect the actual backend state:

```swift
enum PaymentState {
    case idle
    case awaitingCard
    case authorizing
    case authorized(PaymentReceipt)
    case declined(PaymentDeclineReason)
    case failed(Error)
}
```

Avoid state names that imply certainty before confirmation.

## Confirmation Before Starting

For destructive, irreversible, or significant actions, the user may need an explicit confirmation step before the request starts.

Example:

```swift
enum AccountDeletionState {
    case idle
    case confirming
    case deleting
    case deleted
    case failed(Error)
}
```

Flow:

```swift
@MainActor
func requestAccountDeletion() {
    state = .confirming
}

@MainActor
func confirmAccountDeletion() async {
    state = .deleting

    do {
        try await accountService.deleteAccount()
        state = .deleted
    } catch {
        state = .failed(error)
    }
}
```

The first step asks for intent. The second step performs the irreversible operation.

Do not use confirmation prompts for every small action. Use them when the consequence is meaningful, destructive, difficult to undo, or trust-sensitive.

## Progress During Submission

High-stakes actions should acknowledge that work is happening.

Use progress states like:

* submitting
* authorizing
* verifying
* processing
* waiting for confirmation
* deleting
* saving security change

Avoid copy that implies final success too early.

Risky copy:

```text
Transfer complete...
```

while the request is still running.

Prefer:

```text
Submitting transfer...
Waiting for bank confirmation...
```

For unknown duration, use an indeterminate progress indicator. For known multi-step work, show real step progress when available. Do not fake precise progress if the app does not know how much work remains.

## Duplicate Submission Protection

High-stakes flows should prevent accidental duplicate requests.

Risky:

```swift
Button("Confirm") {
    Task {
        await model.confirmTransfer()
    }
}
```

If the button remains enabled, the user may submit the same operation multiple times.

Prefer:

```swift
Button("Confirm") {
    Task {
        await model.confirmTransfer()
    }
}
.disabled(model.isSubmitting)
```

Model side:

```swift
@MainActor
func confirmTransfer() async {
    guard !isSubmitting else { return }

    isSubmitting = true
    state = .submitting

    defer {
        isSubmitting = false
    }

    do {
        let receipt = try await transferService.submitTransfer()
        state = .confirmed(receipt)
    } catch {
        state = .failed(error)
    }
}
```

For financial, purchase, booking, or destructive operations, client-side disabling is not enough by itself. If duplicate submission would be harmful, suggest backend idempotency or request identifiers when applicable.

The agent can suggest idempotency, but cannot implement server guarantees from the client alone.

## Server-Authoritative Result

High-stakes flows should prefer the server response as the canonical result.

Risky:

```swift
let localReceipt = TransferReceipt(
    amount: draft.amount,
    recipient: draft.recipient,
    status: .completed
)

state = .confirmed(localReceipt)
```

Prefer:

```swift
let receipt = try await transferService.submitTransfer(draft)
state = .confirmed(receipt)
```

The server may calculate fees, reject the action, return a pending status, require additional verification, or provide a canonical identifier.

Do not invent final local receipts for server-authoritative actions.

## Pending Is Not Success

Some high-stakes actions may be accepted but not completed.

Examples:

* payment authorized but not captured
* transfer submitted but pending review
* booking requested but awaiting provider confirmation
* identity verification submitted but under review
* account change requested but awaiting email confirmation

Represent pending separately:

```swift
enum BookingState {
    case idle
    case submitting
    case pendingProviderConfirmation(BookingRequestID)
    case confirmed(BookingConfirmation)
    case declined(BookingDeclineReason)
    case failed(Error)
}
```

Do not collapse pending into success unless the product explicitly defines pending as a successful final state and communicates it clearly.

## Failure Recovery

High-stakes failure states should be clear, specific, and safe.

A failure state should explain:

* whether the request was rejected
* whether the final status is unknown
* whether the user can retry safely
* whether retry could duplicate the operation
* whether the user should check transaction history, activity, or status details
* whether support or manual review may be needed

Avoid generic copy:

```swift
state = .failed(error)
errorPresenter.show("Something went wrong")
```

Prefer domain-specific recovery that reflects the real backend result:

```swift
state = .failed(
    TransferErrorPresentation(
        title: "Transfer could not be completed",
        message: "Please review the details and try again.",
        canRetry: true
    )
)
```

Use this only when the backend clearly rejected the operation or confirmed that it was not completed.

Do not encourage retry when the operation may already have been submitted unless the backend is idempotent or the app can verify the result safely.

## Unknown Outcome

High-stakes flows need a plan for unknown outcomes.

An unknown outcome can happen when:

* the request times out
* the app loses connectivity after submission
* the server accepts the request but the response is lost
* the app is terminated during submission
* the authorization provider does not return a clear result

Represent unknown separately when needed:

```swift
enum SubmissionState {
    case idle
    case submitting
    case confirmed(Receipt)
    case failed(Error)
    case unknown(SubmissionID?)
}
```

For unknown outcomes, prefer:

* fetch status from server
* show “checking status”
* prevent immediate duplicate submission
* provide a safe support or activity-link path
* use idempotency or request identifiers where available

Avoid:

* showing success without confirmation
* showing failure when the operation may have succeeded
* automatically retrying an operation that may already be submitted

## Destructive Actions

Destructive actions need extra care because rollback may be impossible or costly.

Review:

* Is the action clearly labeled?
* Is the destructive consequence clear?
* Is there a confirmation step?
* Is there an undo path?
* Is undo real or just visual?
* Is server confirmation required before removing the item permanently from UI?
* What happens if deletion fails?
* What happens if the app closes during deletion?

For low-risk reversible deletion, optimistic removal with undo may be acceptable.

For irreversible deletion, prefer an explicit confirmation step before starting the operation, and show final success only after the server confirms.

Risky:

```swift
@MainActor
func deleteWorkspace(id: Workspace.ID) {
    workspaces.remove(id)

    Task {
        try await workspaceService.delete(id)
    }
}
```

This removes the workspace locally before the authoritative deletion result is known. If the request fails, times out, or returns an unknown status, the UI has already presented the destructive action as complete.

Prefer separating intent, confirmation, progress, and final result:

```swift
enum WorkspaceDeletionState {
    case idle
    case confirming(id: Workspace.ID)
    case deleting(id: Workspace.ID)
    case deleted(id: Workspace.ID)
    case failed(id: Workspace.ID, Error)
}
```

```swift
@MainActor
func requestWorkspaceDeletion(id: Workspace.ID) {
    state = .confirming(id: id)
}
```

The UI can present a confirmation prompt such as:

```text
Are you sure you want to delete this workspace?
This action cannot be undone.
```

Only after the user confirms should the app start the destructive operation:

```swift
@MainActor
func confirmWorkspaceDeletion(id: Workspace.ID) async {
    state = .deleting(id: id)

    do {
        try await workspaceService.delete(id)
        workspaces.remove(id)
        state = .deleted(id: id)
    } catch {
        state = .failed(id: id, error)
    }
}
```

If deletion fails, keep the item visible or restore it from a known local snapshot. Do not present the item as permanently deleted until the server confirms the result.

For destructive actions, the confirmation copy should be specific:

* name what will be deleted
* explain whether the action is reversible
* avoid vague prompts like “Are you sure?” without context
* use destructive styling for the final confirmation action when the platform supports it
* prevent repeated submission while deletion is in progress

## Security-Sensitive Changes

Security-sensitive changes should not appear complete until confirmed.

Examples:

* changing password
* changing email
* changing phone number
* enabling two-factor authentication
* disabling two-factor authentication
* changing recovery method
* changing account permissions
* revoking device access
* changing privacy settings

Risky:

```swift
@MainActor
func changeEmail(to email: String) {
    profile.email = email

    Task {
        try await accountService.changeEmail(to: email)
    }
}
```

Prefer a pending or verification state:

```swift
enum EmailChangeState {
    case idle
    case submitting
    case verificationRequired(maskedEmail: String)
    case confirmed(String)
    case failed(Error)
}
```

The displayed account email should reflect the server-confirmed state unless the product explicitly supports pending email changes.

## Trustworthy Copy

Copy should match the real state of the operation.

Avoid premature certainty:

* “Done” before confirmation
* “Paid” before authorization
* “Deleted” while deletion is pending
* “Verified” before verification completes
* “Your changes are saved” before the server accepts them

Prefer accurate status:

* “Submitting…”
* “Authorizing payment…”
* “Waiting for confirmation…”
* “Deleting…”
* “Verification required”
* “Request submitted”
* “Pending review”
* “Confirmed”

For unknown results:

* “We could not confirm the status.”
* “Checking status…”
* “Do not retry until the status is checked.”

Do not use scary copy unless the risk is real. High-stakes copy should be clear, not dramatic.

## Deliberate Delays

Do not add artificial delays as a default way to make high-stakes operations feel trustworthy.

A high-stakes flow should feel trustworthy because it shows accurate states, confirmation, progress, and recovery.

A short deliberate delay may be a product decision in narrow cases, but the agent should not introduce it unless the product requirement is explicit.

Prefer:

* real progress
* clear confirmation state
* accurate pending state
* final success after server response
* safe failure recovery

over artificial waiting.

## Agent Review Questions

When reviewing a high-stakes action, ask:

1. What is the authoritative source of truth?
2. Can the local app safely predict success?
3. Is the action reversible?
4. What happens if the request fails?
5. What happens if the outcome is unknown?
6. Can repeated taps submit duplicate operations?
7. Does the UI disable or guard repeated submission?
8. Is final success shown only after confirmation?
9. Does the copy distinguish submitting, pending, confirmed, failed, and unknown?
10. Is retry safe?
11. Is rollback real or only visual?
12. Does the operation need a confirmation step before starting?
13. Does the backend need idempotency or request tracking?
14. Is product, legal, security, or compliance review needed?

## Implementation Checklist

When proposing changes for high-stakes actions, check:

* [ ] Is optimistic final success avoided?
* [ ] Is there an explicit submitting or authorizing state?
* [ ] Is final success shown only after authoritative confirmation?
* [ ] Is pending represented separately from confirmed success?
* [ ] Is unknown outcome represented when possible?
* [ ] Are duplicate submissions prevented in the UI?
* [ ] Is backend idempotency or request tracking suggested when duplicate submission would be harmful?
* [ ] Is destructive intent confirmed before the operation starts?
* [ ] Is failure recovery clear and safe?
* [ ] Is retry behavior safe?
* [ ] Is copy accurate and not prematurely certain?
* [ ] Is stale local state avoided for server-authoritative data?
* [ ] Is security-sensitive state confirmed before display changes?
* [ ] Are compliance/product/security unknowns called out instead of guessed?
* [ ] Is validation based on success, failure, timeout, duplicate tap, and unknown-outcome scenarios?

## Validation

Validate high-stakes flows with scenarios that cover success and uncertainty.

Recommended validation:

* normal success
* backend rejection
* network failure before submission
* network failure after submission
* timeout with unknown result
* repeated taps
* app backgrounding during submission
* app termination during submission
* retry after failure
* retry after unknown outcome
* server returns pending instead of confirmed
* server returns canonical result different from local draft
* destructive action cancellation
* accessibility review for confirmation and error states

Do not claim the flow is safe unless failure, duplicate, and unknown-outcome paths are tested or explicitly handled.

Correct phrasing:

* “This avoids showing final success before confirmation.”
* “This makes the pending state explicit.”
* “Validate duplicate taps, timeout, and unknown-result behavior.”

Avoid:

* “This makes the operation faster.”
* “This guarantees the operation is safe.”
* “The user can just retry if something fails.”
* “Optimistic rollback is enough for this financial/destructive/security action.”

For broader device, runtime, and production validation, read `references/validation-and-testing.md`.

## Review Output Guidance

When using this reference, explain:

```markdown
## Finding

The flow predicts success locally before the authoritative system confirms the result.

## User impact

The user may believe a financial, destructive, legal, security-sensitive, or otherwise high-stakes action completed when it is still pending, failed, or unknown.

## Recommended change

Use explicit confirmation, submitting, pending, confirmed, failed, and unknown states. Show final success only after the server-authoritative result arrives. Prevent duplicate submission and define safe failure recovery.

## Safety checks

Call out whether the action is reversible, whether retry is safe, whether idempotency is needed, and whether product/legal/security/compliance review is required.

## Validation

Test success, rejection, timeout, unknown outcome, repeated taps, retry, and app interruption.
```
