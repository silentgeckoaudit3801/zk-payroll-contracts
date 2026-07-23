# Contract error cases

This reference groups common ZK Payroll contract failures by feature area so SDK, dashboard, and backend contributors can map low-level Soroban errors to clear recovery messages. It uses the names currently present in the contracts and distinguishes retryable operator problems from non-retryable authorization, validation, and state-transition failures.

## Retryability model

| Retryability | Meaning | Client behavior |
| --- | --- | --- |
| Retryable after state change | The same call can succeed after required on-chain or operator state changes. | Refresh contract state, guide the operator through the missing step, then retry. |
| Retryable with corrected input | The submitted payload is malformed, stale, duplicated, or inconsistent. | Keep the user on the flow, preserve safe form values, and ask for corrected input or a regenerated proof/payload. |
| Non-retryable for this caller | The caller lacks authorization or is not the expected role holder. | Stop automatic retries and surface the required role or signer. |
| Non-retryable terminal state | The target record is closed, already used, expired, or otherwise terminal. | Stop retries and direct the user to create a new payroll run, proof, view key, or period. |

## Company setup and employee onboarding

| Contract area | Error text or typed error | Likely cause | Retryability | Suggested client recovery |
| --- | --- | --- | --- | --- |
| `payroll_registry` | `Company already registered` | The admin address already has a company record. | Retryable with corrected input | Refresh company state and route the admin to manage the existing company instead of registering again. |
| `payroll_registry` | `Company not found` | A company id does not exist for an employee, admin rotation, or treasury rotation call. | Retryable with corrected input | Re-query companies and prevent actions against stale ids. |
| `payroll_registry` | `Employee not found` | The employee record was never added or was removed before update/status changes. | Retryable after state change | Refresh the employee list and offer onboarding before update/status actions. |
| `payroll_registry` | `Unauthorized: caller is not the company admin` / `Unauthorized` | The signer is not the registered company admin for the target company. | Non-retryable for this caller | Ask the admin signer to reconnect; do not retry with the same signer. |
| `salary_commitment` | `Already initialized` / `Not initialized` | The commitment contract was initialized twice or used before initialization. | Retryable after state change | For initialization, block duplicate setup; for missing setup, route operators to deployment/init checks. |
| `salary_commitment` | `Reference ID must be 1-256 characters` | A payroll reference id is empty or too long. | Retryable with corrected input | Validate client-side before submitting and keep the user in the edit flow. |
| `salary_commitment` | `Reference ID already assigned to another employee` | A reference id is reused for a different employee. | Retryable with corrected input | Ask for a new reference id or show the existing employee mapping if available. |

## Payroll execution and treasury checks

| Contract area | Error text or typed error | Likely cause | Retryability | Suggested client recovery |
| --- | --- | --- | --- | --- |
| `payroll` | `Deposit amount must be positive` / `Amount must be positive` | A deposit, authorization, or draft amount is zero or negative. | Retryable with corrected input | Validate amounts before signing and preserve the draft for correction. |
| `payroll` | `Unauthorized` / `Unauthorized: caller is not treasury owner` | The signer is neither the contract admin nor the configured treasury owner for the action. | Non-retryable for this caller | Stop retries and explain which role must sign. |
| `payroll` | `Draft already committed` | The draft hash for the payroll run already exists. | Non-retryable terminal state | Show the existing draft and require a new nonce/run for different data. |
| `payroll` | `A pending emergency request already exists` | A treasury emergency withdrawal request is already awaiting the second authorization. | Retryable after state change | Show the pending request and let the operator approve or cancel it. |
| `payroll` | `No pending emergency request to cancel` | A cancel call was submitted after the request was already cleared or never existed. | Retryable after state refresh | Refresh emergency state and disable the cancel action when none is pending. |
| `payroll` | `Array length mismatch` | Employees and amounts arrays do not have the same length. | Retryable with corrected input | Rebuild the batch payload and verify row counts before signing. |
| `payroll` | `Batch too large` | The employee batch exceeds `MAX_BATCH`. | Retryable with corrected input | Split the payroll run into smaller batches before resubmitting. |
| `payroll` | `Duplicate run nonce: this payroll batch has already been submitted` | A payroll run nonce is reused. | Non-retryable terminal state | Generate a fresh run nonce or show the existing submitted run. |
| `payroll` | `Draft hash not pre-committed: call commit_draft first` | Execution was attempted before the draft commitment exists. | Retryable after state change | Guide the operator through draft commit before execution. |
| `payroll` | `Expected spend mismatch: authorised ... but batch totals ...` | The authorized total does not match employee amount totals. | Retryable with corrected input | Recalculate totals and require a new authorization. |
| `payment_executor` | `Amount must be non-negative` | A payment amount is below zero. | Retryable with corrected input | Validate amounts before proof and payment submission. |
| `payment_executor` | `PeriodAlreadyExists` | A payroll period already exists for the requested company/period id. | Non-retryable terminal state | Use the existing period or create a new period id. |
| `payment_executor` | `PeriodNotFound` | Payment or close-period call references an unknown period. | Retryable after state refresh | Refresh period state and prevent action against stale ids. |
| `payment_executor` | `PeriodClosed` | Payment is attempted after period closure. | Non-retryable terminal state | Create a new period; do not retry payment in the closed period. |
| `payment_executor` | `Payroll is paused` | The pause manager reports the system as paused. | Retryable after state change | Disable submit actions and show the pause status until an operator unpauses. |

## Proof verification and commitment nullifiers

| Contract area | Error text or typed error | Likely cause | Retryability | Suggested client recovery |
| --- | --- | --- | --- | --- |
| `payment_executor` | `ProofExpired` | Proof age exceeds `MAX_PROOF_AGE_SECONDS`. | Non-retryable terminal state | Generate a fresh proof before retrying payment. |
| `payment_executor` | `ProofAlreadyUsed` | The proof nullifier has already been recorded. | Non-retryable terminal state | Show the prior payment/run and prevent duplicate settlement. |
| `payment_executor` | `AlreadyPaid` | Payment record already exists for employee and period. | Non-retryable terminal state | Refresh payment history and show the settled record. |
| `payment_executor` | `Invalid payment proof` | The proof verifier rejected the submitted proof/public inputs. | Retryable with corrected input | Regenerate proof inputs from current commitments and payroll period data. |
| `salary_commitment` | `Nullifier already used` | A commitment nullifier was recorded previously. | Non-retryable terminal state | Stop retries and surface duplicate-proof guidance. |
| `proof_verifier` | `Verifier already initialized` | Verification key setup was attempted more than once. | Non-retryable terminal state | Show current verifier config; require upgrade/migration flow for changes. |
| `proof_verifier` | `Already initialized` / `Not initialized` | Admin or verification key setup is duplicated or missing. | Retryable after state change | Route deployers through verifier initialization checks. |

## Audit access and view keys

| Contract area | Error text or typed error | Likely cause | Retryability | Suggested client recovery |
| --- | --- | --- | --- | --- |
| `audit_module` | `KeyNotFound` | No view key is stored for the auditor. | Retryable after state change | Ask an admin to grant audit access before retrying. |
| `audit_module` | `WrongAuditor` | Supplied key material or record does not belong to the caller. | Non-retryable for this caller | Ask the correct auditor to sign or request a new key grant. |
| `audit_module` | `KeyExpired` | Ledger sequence is beyond `expiration_ledger`. | Non-retryable terminal state | Request a new view key with a fresh expiration. |
| `audit_module` | `NotKeyGranter` | A revoke/update action is attempted by an admin that did not grant the key. | Non-retryable for this caller | Route the action to the granting admin. |
| `audit_module` | `InsufficientScope` | Aggregate-only scope is used for a commitment-level operation. | Retryable after state change | Request a broader audit scope or switch to aggregate-only UI flows. |
| `audit_module` | `CommitmentMismatch` | Supplied salary/blinding data does not match the stored commitment. | Retryable with corrected input | Recollect disclosure inputs and warn that the current evidence does not match. |
| `audit_module` | `InvalidViewKey` | Supplied view-key bytes differ from the stored key. | Non-retryable for this credential | Ask the auditor to use the correct current key or request re-grant. |

## Pause manager and operational state

| Contract area | Error text or typed error | Likely cause | Retryability | Suggested client recovery |
| --- | --- | --- | --- | --- |
| `pause_manager` | `Already initialized` | Pause manager operator was already configured. | Non-retryable terminal state | Show current operator and require a supported rotation flow for changes. |
| `pause_manager` | authorization failure from `operator.require_auth()` | Pause/unpause/set-operator was signed by a non-operator. | Non-retryable for this caller | Stop retries and prompt the configured operator signer. |
| `pause_manager` | `is_paused` returns `false` before initialization | Pause manager was not initialized or not configured. | Retryable after state change | Treat missing pause state as unpaused but surface deployment diagnostics to operators. |

## Related SDK and dashboard handling

- Use SDK error documentation in `zk-payroll-sdk/docs/ERRORS.md` when mapping these contract-level failures to SDK error codes.
- Use `zk-payroll-sdk/docs/TROUBLESHOOTING.md` for operator-facing recovery language.
- Dashboard flows should keep terminal failures visible in history instead of retrying silently.
- For retryable input failures, preserve the user's draft values and highlight the exact field or batch row that needs correction.
- For authorization failures, do not retry automatically; reconnect or switch to the required signer first.