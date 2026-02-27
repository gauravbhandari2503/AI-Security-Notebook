---
description: Teams template for PR review notifications
---

# PR Review Template

This import defines the `pr_review` message template and usage guidance.

teams_template:
message_type: pr_review
themeColor: '0078d4'
activityTitle: 'Pull Request Raised'
summary: 'Pull Request Raised'
sections: - name: pr_details
fields: [pr_author, pr_number, pr_title, target_branch, files_changed] - name: plain_summary
fields: [plain_summary] - name: critical_comments
fields: [critical_comments]

usage:

- workflow: code-review-agent.md
  set: message_type: pr_review
  provide: [plain_summary, critical_comments]
