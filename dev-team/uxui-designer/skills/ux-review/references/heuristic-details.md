# Heuristic Evaluation — Expanded Reference

## Nielsen's 10 Heuristics: Detailed Checkpoints

### 1. Visibility of System Status

The system should keep users informed about what is going on through appropriate feedback within reasonable time.

**Checkpoints:**
- Loading indicators appear within 300ms for async operations
- Progress bars show determinate progress for operations >3 seconds
- Current navigation location is highlighted in nav elements
- Form submission state is immediately visible (button disabled, spinner shown)
- File upload shows progress percentage and estimated time
- System errors are surfaced immediately, not silently swallowed
- Real-time data shows last-updated timestamp
- Background processes show status in a persistent location

**Common Violations:**
- Silent API failures with no user feedback
- "Submitting..." state with no timeout or error recovery
- Navigation that doesn't indicate the current page
- Bulk operations with no progress indicator

### 2. Match Between System and Real World

The system should speak the user's language, with words, phrases, and concepts familiar to the user.

**Checkpoints:**
- Button labels use verbs the user understands ("Save draft" not "Persist entity")
- Error messages explain the problem in plain language, not error codes
- Icons match conventional meanings (trash = delete, pencil = edit)
- Workflow order matches the user's mental model of the task
- Dates use the user's locale format
- Units match user expectations (currency, measurement, timezone)
- Technical identifiers are hidden or supplementary, not primary
- Metaphors are consistent and culturally appropriate

**Common Violations:**
- Database field names exposed as form labels ("created_at")
- HTTP status codes shown to end users ("Error 422")
- Developer terminology in user-facing text ("null", "undefined", "instance")
- Workflow that follows data model structure rather than user task flow

### 3. User Control and Freedom

Users often perform actions by mistake and need a clearly marked "emergency exit."

**Checkpoints:**
- Undo available for destructive actions (delete, archive, send)
- Cancel/close button visible on all modals and overlays
- Browser back button works as expected (no broken history)
- Multi-step forms allow returning to previous steps
- Drafted content is auto-saved
- Bulk actions can be reversed
- Confirmation dialogs for irreversible actions clearly state consequences
- Escape key closes overlays and cancels operations

**Common Violations:**
- Delete with no undo or confirmation
- Modal with no close button (only submit)
- Single-page app that breaks browser back navigation
- Multi-step form that loses data on back navigation

### 4. Consistency and Standards

Users should not have to wonder whether different words, situations, or actions mean the same thing.

**Checkpoints:**
- Same action uses the same label everywhere ("Delete" not sometimes "Remove")
- Interactive elements have consistent visual treatment (all buttons look like buttons)
- Layout patterns repeat across similar pages
- Design tokens applied consistently (not ad-hoc colors or spacing)
- Platform conventions followed (links underlined, external links marked)
- Terminology is consistent across navigation, headings, and actions
- Component behavior is predictable (all dropdowns work the same way)
- Date, time, and number formatting consistent throughout

**Common Violations:**
- "Save" in one place, "Submit" in another for the same action
- Different button styles for the same hierarchy level across pages
- Inconsistent table sorting behavior
- Mix of date formats (MM/DD vs DD/MM)

### 5. Error Prevention

A design that prevents problems is better than good error messages.

**Checkpoints:**
- Input constraints match expected format (type="email", maxlength, pattern)
- Dangerous zones are visually distinct (red delete section, warning banners)
- Smart defaults reduce chance of wrong input
- Confirmation required before irreversible actions
- Auto-save prevents data loss
- Inline validation catches errors before submission
- Duplicate submission prevented during loading states
- Search suggestions prevent misspellings

**Common Violations:**
- Free-text input where a dropdown would prevent errors
- No maxlength on text inputs
- Double-submit possible on slow connections
- Destructive action button styled the same as safe actions

### 6. Recognition Rather Than Recall

Minimize the user's memory load by making elements, actions, and options visible.

**Checkpoints:**
- Recent items and search history accessible
- Labels visible alongside icons (not icons only)
- Instructions visible in context, not requiring memorization
- Autocomplete and search suggestions present
- Related options visible when making a choice
- Previously entered data pre-populated in forms
- Breadcrumbs show path for deep navigation
- Tooltips explain non-obvious controls

### 7. Flexibility and Efficiency of Use

Accelerators — unseen by novice users — may speed up interaction for expert users.

**Checkpoints:**
- Keyboard shortcuts for common actions (documented and discoverable)
- Bulk actions available for repetitive operations
- Customizable dashboard or workspace
- Progressive disclosure hides advanced features initially
- Quick filters and saved searches available
- Recently used items prioritized in lists
- Command palette or search for power users
- Drag-and-drop for reordering alongside accessible alternatives

### 8. Aesthetic and Minimalist Design

Dialogues should not contain information that is irrelevant or rarely needed.

**Checkpoints:**
- Primary action visually dominant, secondary actions subdued
- Non-essential content hidden or in progressive disclosure
- Whitespace used to separate groups and reduce clutter
- No competing calls-to-action on the same screen
- Data density appropriate for the use case
- Decorative elements don't interfere with content
- Modal content focused on single task
- Navigation shows only relevant items for current context

### 9. Help Users Recognize, Diagnose, and Recover from Errors

Error messages should be expressed in plain language, precisely indicate the problem, and suggest a solution.

**Checkpoints:**
- Error messages in plain language (not codes or stack traces)
- Error specifically identifies which field or action failed
- Error suggests how to fix the problem
- Inline errors positioned adjacent to the source
- Error state visually distinct (color + icon, not color alone)
- Error persists until resolved (no auto-dismiss)
- Form errors summarized at top with links to each field
- Network errors suggest retry or alternative actions

### 10. Help and Documentation

Even though it's better if the system can be used without documentation, help should be easy to search and focused on the user's task.

**Checkpoints:**
- Contextual help available (tooltips, info icons, inline hints)
- Onboarding tour for first-time users
- Searchable help documentation
- Empty states provide guidance and call-to-action
- FAQ or knowledge base accessible from within the app
- Contact support option clearly available
- Feature announcements for new functionality
- Keyboard shortcut reference accessible (usually `?`)

---

## Industry-Specific Adaptations

### SaaS Dashboards

Additional evaluation criteria:
- **Data density**: Can users scan key metrics in under 5 seconds?
- **Multi-tenant**: Is organization context always clear (which workspace/team)?
- **Onboarding**: Does first use guide toward the "aha" moment?
- **Permissions**: Are unavailable actions hidden or clearly disabled with explanation?
- **Bulk operations**: Can users manage data at scale (select all, bulk edit, export)?
- **Notifications**: Are alerts prioritized and actionable, not overwhelming?

### E-Commerce

Additional evaluation criteria:
- **Product discovery**: Are filters, search, and sorting intuitive?
- **Cart persistence**: Does the cart survive session changes, device switches?
- **Checkout friction**: Minimum steps to complete purchase?
- **Trust signals**: Security badges, return policies, reviews visible at decision points?
- **Price clarity**: Total cost (including shipping/tax) visible before final step?
- **Guest checkout**: Can users purchase without creating an account?

### Mobile Applications

Additional evaluation criteria:
- **Thumb zones**: Are primary actions in comfortable reach (bottom 1/3 of screen)?
- **Gesture discoverability**: Are swipe/pinch actions discoverable without instruction?
- **Offline handling**: Can users continue working without connectivity?
- **Notification relevance**: Are push notifications timely, actionable, and dismissible?
- **Input adaptation**: Does the keyboard type match the input (numeric, email, URL)?
- **Deep linking**: Do shared links open to the correct content?

---

## Severity Rating Calibration

### Calibration Guide

| Severity | User Impact | Frequency | Workaround Available? |
|----------|------------|-----------|----------------------|
| 0 — Not a problem | None | — | — |
| 1 — Cosmetic | Annoying but no task impact | Rare | Not needed |
| 2 — Minor | Slows task completion slightly | Occasional | Easy workaround exists |
| 3 — Major | Prevents task completion for some users | Common | Difficult workaround |
| 4 — Catastrophic | Prevents task completion for all users | Always | No workaround |

### Calibration Formula

`Severity = Impact (1-3) + Frequency (0-2) - Workaround (0-1)`

- **Impact**: 1 = cosmetic, 2 = efficiency loss, 3 = task failure
- **Frequency**: 0 = edge case, 1 = common flow, 2 = every user every time
- **Workaround**: 0 = no workaround, 1 = easy workaround available

Score 0-1 → Severity 1, Score 2 → Severity 2, Score 3 → Severity 3, Score 4-5 → Severity 4

---

## Cognitive Walkthrough Methodology

### Process

1. **Define the user**: Who is performing the task? What is their experience level?
2. **Define the task**: What is the user trying to accomplish? What is the happy path?
3. **Walk each step**: For each action the user must take, ask:
   - Will the user know what to do? (Is the next action visible and labeled?)
   - Will the user see the correct action? (Is it visually prominent?)
   - Will the user associate the action with their goal? (Does the label match their intent?)
   - Will the user see progress? (Is there feedback after the action?)
4. **Document failures**: Record any step where the answer is "no" with the specific breakdown point.

### Walkthrough Template

```markdown
**User Persona**: [Name, role, experience level]
**Task**: [What the user wants to accomplish]
**Starting Point**: [Where the user begins]

| Step | User Action | Visible? | Labeled? | Matches Goal? | Feedback? | Notes |
|------|------------|----------|----------|---------------|-----------|-------|
| 1 | Click "New Project" | Yes | Yes | Yes | Shows form | — |
| 2 | Fill project name | Yes | No | — | — | Label says "Title" not "Project Name" |
```

---

## Competitive UX Analysis Framework

### Analysis Matrix

| Criterion | Our Product | Competitor A | Competitor B | Best in Class |
|-----------|------------|-------------|-------------|---------------|
| Task completion time | | | | |
| Error rate | | | | |
| Onboarding steps | | | | |
| Feature discoverability | | | | |
| Mobile experience | | | | |
| Accessibility compliance | | | | |
| Loading performance | | | | |

### Process

1. **Select competitors**: 2-3 direct competitors + 1 best-in-class from adjacent market
2. **Define tasks**: 5-7 core tasks that all products support
3. **Evaluate**: Complete each task on each product, recording metrics above
4. **Identify gaps**: Where competitors excel and our product falls short
5. **Prioritize**: Focus on gaps that affect core user tasks, not edge features
