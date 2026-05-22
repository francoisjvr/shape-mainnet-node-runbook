# Appendix C: Log Patterns

## Healthy geth patterns

Look for lines that indicate:
- imported new potential chain segment
- chain head advancing
- normal execution progress

## Healthy op-node patterns

Look for lines that indicate:
- successfully processed payload
- unsafe payload insertion continues
- derivation keeps moving

## Concerning patterns

- `no space left on device`
- `exec: "geth": executable file not found in $PATH`
- repeated engine auth failures
- repeated upstream L1 request failures
- repeated forkchoice updates with no net forward movement for long periods

## Interpretation rule

A single scary-looking log line is less important than the pattern over time.
