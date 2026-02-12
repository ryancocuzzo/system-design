# Generate Exercise

1. Find next number in `exercises/`
2. Create `exercises/NNN/problem.md`:
   - Context (2-3 sentences)
   - Functional requirements
   - Non-functional requirements (quantified: QPS, latency, storage, availability)
   - Constraints
3. Create empty `exercises/NNN/design.md`
4. Add a row to the Problem Index table in `README.md`: `| Full | NNN | [Problem Title] |`. Insert in numerical order among Full rows.
5. Display problem inline

## Problem Requirements

- Genuine tradeoffs (no single obvious answer)
- Scale quantified with numbers
- Designable in 30-40 minutes

## Categories (rotate)

- Data systems (storage, caching, search)
- Real-time systems (messaging, notifications)
- Transactional systems (payments, inventory)
- Platform services (rate limiting, auth)
- Content systems (file storage, feeds)

Do not include solutions, architecture hints, or tradeoffs in the problem.
