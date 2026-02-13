---
name: UX Review
description: This skill should be used when the user asks to "review the UX", "audit the UI", "UX critique", "usability review", "design review", "heuristic evaluation", or "improve the user experience". It covers systematic UX auditing, heuristic evaluation methodology, usability checklist, and prioritized design recommendations.
---

### Heuristic Evaluation Framework

Evaluate interfaces against Nielsen's 10 Usability Heuristics, scoring each from 0-4:

| Score | Severity | Action |
|-------|----------|--------|
| 0 | Not a problem | No action needed |
| 1 | Cosmetic | Fix when convenient |
| 2 | Minor | Low priority fix |
| 3 | Major | High priority, fix before next release |
| 4 | Catastrophic | Must fix immediately, blocks users |

#### The 10 Heuristics Applied

**1. Visibility of System Status**
- Loading indicators present for async operations?
- Progress shown for multi-step processes?
- Current navigation location clear?
- Form submission feedback immediate?

**2. Match Between System and Real World**
- Labels use user language, not technical jargon?
- Icons match conventional meanings?
- Workflow matches user's mental model of the task?
- Date/time/currency formats match user locale?

**3. User Control and Freedom**
- Undo available for destructive actions?
- Cancel/close available on all dialogs?
- Back navigation works as expected?
- Multi-step forms allow revisiting previous steps?

**4. Consistency and Standards**
- Same actions use same labels/icons everywhere?
- Interactive elements look and behave consistently?
- Platform conventions followed (links underlined, buttons look tappable)?
- Design tokens applied consistently?

**5. Error Prevention**
- Confirmation before destructive actions?
- Input constraints prevent invalid data (type, max-length, pattern)?
- Dangerous zones visually distinct (red for delete, warning for irreversible)?
- Smart defaults reduce chance of wrong input?

**6. Recognition Rather Than Recall**
- Recent items/history accessible?
- Labels visible (not just icons)?
- Instructions visible when needed (not memorized)?
- Search suggestions and autocomplete present?

**7. Flexibility and Efficiency of Use**
- Keyboard shortcuts for power users?
- Bulk actions available for repeated operations?
- Customization options for frequent workflows?
- Progressive disclosure for advanced features?

**8. Aesthetic and Minimalist Design**
- Visual hierarchy guides scanning?
- Non-essential information hidden or de-emphasized?
- Whitespace used effectively?
- No competing calls-to-action?

**9. Help Users Recognize, Diagnose, and Recover from Errors**
- Error messages in plain language (not codes)?
- Error messages identify the problem specifically?
- Error messages suggest a solution?
- Inline errors positioned near the source?

**10. Help and Documentation**
- Contextual help available (tooltips, info icons)?
- Onboarding for first-time users?
- Search-friendly help documentation?
- Empty states guide users?

### UX Audit Process

Follow this systematic process when conducting a UX review:

#### Step 1: Define Scope

Identify what is being reviewed:
- Specific feature or flow?
- Entire application?
- Particular user persona's journey?

Document the user's goal, the happy path, and key decision points.

#### Step 2: Walk the Happy Path

Navigate the primary user flow, documenting:
- Each screen/state encountered
- Number of clicks/taps to complete the task
- Any moments of confusion or hesitation
- Missing feedback or confirmation

#### Step 3: Test Edge Cases

For each screen, systematically test:
- Empty states (no data)
- Error states (validation failures, server errors, network errors)
- Overflow states (very long text, many items, large numbers)
- Loading states (slow network simulation)
- Permission states (unauthorized access)

#### Step 4: Evaluate Accessibility

Check each screen against the core accessibility requirements:
- Keyboard navigation (Tab through entire page)
- Screen reader flow (logical reading order)
- Color contrast (use automated tools + manual review)
- Focus management (modal, route change, dynamic content)
- Touch targets (44px minimum on mobile)

#### Step 5: Document Findings

For each issue found, document:

```markdown
### Issue: [Brief Title]

**Heuristic**: [Which heuristic is violated]
**Severity**: [0-4]
**Location**: [Page/Component/State]
**Description**: [What the problem is]
**Impact**: [How it affects users]
**Recommendation**: [Specific fix with implementation guidance]
**Screenshot/Reference**: [File path or description]
```

#### Step 6: Prioritize Recommendations

Group findings into tiers:

| Tier | Criteria | Timeline |
|------|----------|----------|
| P0 - Critical | Blocks core user flow, accessibility failure, data loss risk | Fix immediately |
| P1 - High | Causes significant confusion, frequent user errors, major accessibility gap | Fix in current sprint |
| P2 - Medium | Inconsistency, inefficiency, minor accessibility issue | Fix in next sprint |
| P3 - Low | Cosmetic, nice-to-have improvement, edge case polish | Backlog |

### Quick Usability Checklist

Use this checklist for rapid reviews of individual components or pages:

**Layout and Hierarchy**
- [ ] Primary action is visually prominent
- [ ] Secondary actions are clearly secondary (not competing)
- [ ] Reading flow follows F-pattern or Z-pattern
- [ ] Related items are grouped with adequate whitespace between groups
- [ ] Content fits without horizontal scroll at all supported viewports

**Interactive Elements**
- [ ] All clickable elements look clickable (cursor, affordance)
- [ ] Hover and focus states are visible and distinct
- [ ] Disabled state is visually clear and not confusable with default
- [ ] Button labels describe the action (not "Submit" or "Click here")
- [ ] Links are distinguishable from surrounding text

**Feedback**
- [ ] Success actions are confirmed (toast, redirect, visual change)
- [ ] Error messages are specific, helpful, and positioned near the source
- [ ] Loading states prevent repeated submissions
- [ ] Progress indication for operations >1 second

**Content**
- [ ] Headings describe content accurately
- [ ] Labels are clear without requiring tooltip or documentation
- [ ] Error messages suggest how to fix the problem
- [ ] Empty states provide guidance and a call to action

**Responsiveness**
- [ ] Layout adapts correctly to mobile viewport
- [ ] Touch targets are at least 44x44px on mobile
- [ ] No content is truncated or hidden unexpectedly
- [ ] Navigation is accessible on all viewport sizes

### Review Output Format

Present UX review findings in this structured format:

```markdown
## UX Review: [Feature/Page Name]

### Summary
[2-3 sentence overview of overall quality and key themes]

### Strengths
- [What works well — acknowledge good design]

### Critical Issues (P0-P1)
[Issues listed with full detail per the template above]

### Improvements (P2-P3)
[Issues listed with brief description and recommendation]

### Accessibility Notes
[Specific accessibility findings grouped together]

### Recommendations Summary
| # | Issue | Severity | Effort | Recommendation |
|---|-------|----------|--------|----------------|
| 1 | Title | P0 | Low | Fix description |
| 2 | Title | P1 | Medium | Fix description |
```

## References

- [heuristic-details.md](references/heuristic-details.md) - Expanded heuristic evaluation criteria with real-world examples, common violations per heuristic, and industry-specific adaptations for SaaS, e-commerce, and mobile applications.
- [audit-tools.md](references/audit-tools.md) - Automated UX and accessibility auditing tools, browser extensions, contrast checkers, screen reader testing guides, and performance impact of UX decisions.
