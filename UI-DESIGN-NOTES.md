# A clearer way to read the audit

The page should answer a decision before asking the reader to interpret a score. This redesign changes presentation, not the independent audits or their ratings.

| Original difficulty | Design change |
|---|---|
| A long introduction followed by 26 dense rows | A compact overview with the conclusion and three next steps |
| Six scores, symbols and evidence labels in every row | Plain-language side-by-side summaries; original scores under Details |
| Everything on one long page | Four views: Overview, Compare, Recommendations and Auditors |
| Recommendations read like engineering notes | Each starts with an action and benefit; first step, success check and dependency follow |
| Important disagreement was buried in prose | Clear reviewer-difference labels and a short explanation beside the relevant capability |

## The intended reading order

1. **Overview:** Understand the recommendation and the strengths on each side.
2. **Compare:** Start with eight important differences. Search or switch to all 26 capabilities when needed.
3. **Recommendations:** Work through First, Next, Then test and Later. These are proposals, not work already implemented.
4. **Auditors:** Compare the three viewpoints, read original prose, and inspect the audit's limits.

## How recommendation cards should look

Keep the title concrete: “Put active work in one clear view.” Follow it with a sentence about the benefit. Show who provides the idea—our own improvement, borrowed from Omnara, or offered to Omnara. Put effort estimates and implementation detail below the main message. The reader should be able to find the first step, prerequisite and test for success without decoding P0/P1 notation.

The operator-view recommendation also includes a small, clearly labeled interface sketch: task list on the left, current task and timeline in the center, and next action on the right. It illustrates a proposed screen; it is not a connected operational dashboard.

## Visual and interaction rules

- Compact headings, generous spacing between ideas, restrained green accents and readable body text.
- One primary reading task per view; no autoplay, decorative animation or metric wall.
- Original scores stay separate. No average score or invented overall winner.
- Text accompanies color. Unknown and not independently rated remain explicit.
- Details open on demand. Escape and a visible close button return to the comparison.
- Small screens stack comparison cells and recommendation content without horizontal page scrolling.
- The full report, CSV, source links and individual audits remain available.

Acceptance is functional and visual inspection, data-integrity checks and verified publication. No measured user-comprehension improvement is claimed without a user test.
