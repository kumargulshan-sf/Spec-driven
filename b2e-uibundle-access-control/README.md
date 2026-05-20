# B2E UIBundle Access Control — PR Analysis

Analysis of PR #85409 against the B2X App Access Control spec.

## Documents (Read in Order)

| # | File | What You'll Learn |
|---|------|-------------------|
| 1 | [01-background.md](./01-background.md) | Why this work exists, Salesforce concepts explained simply |
| 2 | [02-data-model.md](./02-data-model.md) | The new entities and relationships being created |
| 3 | [03-pr-file-guide.md](./03-pr-file-guide.md) | Every file in the PR, grouped by concern, with reading order |
| 4 | [04-flow-diagram.md](./04-flow-diagram.md) | How the pieces connect at runtime (ASCII diagrams) |
| 5 | [05-pr-summary.md](./05-pr-summary.md) | Quick reference: stats, decisions, potential issues |
| 6 | [06-spec-vs-pr.md](./06-spec-vs-pr.md) | Gap analysis: what spec requires vs what PR delivers |
| 7 | [07-url-and-runtime.md](./07-url-and-runtime.md) | URL routing, runtime access gap, Slack thread decisions, what AFS needs to do |
| 8 | [08-pr-18025-understanding.md](./08-pr-18025-understanding.md) | PR analysis for core platform changes |
| 9 | [09-what-we-can-do.md](./09-what-we-can-do.md) | Options analysis for access control approaches |
| 10 | [10-runtime-access-control-diagram.md](./10-runtime-access-control-diagram.md) | Runtime flow diagrams: direct (blocked) vs CustomApp (allowed) |
| **11** | [**11-proposal-runtime-enforcement.md**](./11-proposal-runtime-enforcement.md) | **Implementation proposal: the agreed design** |

## Source Links

- PR: https://gitcore.soma.salesforce.com/core-2206/core-262-public/pull/85409
- Spec: https://git.soma.salesforce.com/tconn/custom-app-spec/blob/main/Raw/Documents/B2X%20app%20access%20control.md
- GUS: W-22478585
