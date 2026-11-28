> Greenfield standard: chosen before implementation code existed; revisit once the repository has real code.

# System tests (Capybara + Selenium)

## Scope
- Use ActionDispatch::SystemTestCase with Capybara and headless Selenium Chrome, under test/system. CI runs bin/rails test:system and keeps failure screenshots.
- Cover the critical journeys only: sign-up, sign-in, sign-out, password reset, and each core create/edit flow once it exists. Add or update a system test whenever one of these journeys changes.
- Keep validation permutations and authorization edge cases in Minitest controller tests; system tests check that the pieces work together in a browser.

## Writing tests
- Find elements the way a user does: fill_in by label, click_button / click_link by visible text, within landmarks or headings. Avoid CSS/XPath chains and do not add data-testid attributes; if an element cannot be found by its label or role, fix the markup.
- Never use sleep. Rely on Capybara's waiting matchers (assert_text, assert_selector, assert_no_selector, assert_current_path) to synchronise with Turbo Frame and Stream updates and Stimulus changes.
- Cover at least one invalid submission per form-based journey and assert the inline error appears next to the field.
- Use fixtures for data, sign in through a small helper that drives the real sign-in form or a test-only session helper, and keep each test independent.

## Privacy and session checks
- After sign-out, assert that revisiting a protected page redirects to sign-in and that going back does not show the previous user's data.
- For at least one core resource, assert a second user cannot open the first user's record URL.

## Accessibility and mobile
- Exercise critical journeys with the keyboard where practical (tab to controls, submit with Enter) and check that focus lands sensibly after Turbo updates.
- Run the critical journeys at a mobile viewport (for example 375x812) as well as desktop, so navigation and primary actions remain reachable.
- These checks support WCAG 2.2 AA basics; they do not prove full conformance.

## Leave out
- Playwright, Cypress or Cucumber; exhaustive browser/device matrices; screenshot-only assertions; real external services (stub them deterministically).
