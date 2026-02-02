# Error Conventions (DomainError)

## Rules
- Application layer throws DomainError ONLY.
- Application never throws HttpException.
- Adapters:
  - Expected business error -> map to DomainError.
  - Unexpected error -> throw raw (500).

## Error code naming
- UPPER_SNAKE_CASE
- Prefixed by feature

Examples:
- BALANCE_NOT_FOUND
- SUPPORT_TICKET_CLOSED
- FILE_TYPE_NOT_ALLOWED
