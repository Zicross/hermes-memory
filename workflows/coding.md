# Coding Workflow

## Current (June 2026)

### For ConstiuINT and future projects:

1. **Planning**
   - Specs in `planning/specs/`
   - Plans in `planning/plans/`
   - Handoffs in `planning/handoffs/`

2. **Execution**
   - User in Opus dispatches Gemma 4 for implementation
   - Use TDD for behavior changes
   - Tests: Vitest (unit), Playwright (e2e)

3. **Verification**
   - Run `npm run lint`, `npm run typecheck`, `npm run test`
   - Browser check for UI flows

4. **Commit**
   - One logical change per commit
   - User commits/pushes from local environment

## For Hermes (Strategic Only)

- Maintain specs, plans, handoffs
- Cross-project coordination
- Weekly briefs
- Identify missing links

## Anti-patterns to Avoid
- Building from chat memory instead of repo artifacts
- Letting a single agent both implement and review
- Shipping UI claims that exceed verified product behavior