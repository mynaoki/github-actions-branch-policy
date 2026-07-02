name: 🔒 Apply Branch Policy

# One-time workflow to configure branch protection on this repository.
# Run manually from Actions → "Apply Branch Policy" → Run workflow.
# After a successful run the workflow disables itself.
#
# What it does:
#   - Restricts merge to squash-merge only
#   - Enables auto-delete of merged branches
#   - Creates a branch ruleset for main (see docs/branch-policy.md)

on:
  workflow_dispatch:

jobs:
  apply-branch-policy:
    name: Configure repository branch policy
    runs-on: ubuntu-latest
    permissions:
      contents: read
      administration: write
      actions: write

    steps:
      - name: Configure squash-only merge
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh api --method PATCH /repos/${{ github.repository }} \
            -f allow_squash_merge=true \
            -f allow_merge_commit=false \
            -f allow_rebase_merge=false \
            -f delete_branch_on_merge=true \
            -f squash_merge_commit_title=PR_TITLE \
            -f squash_merge_commit_message=PR_BODY \
            --silent
          echo "✅ Squash-only merge configured."

      - name: Create branch ruleset for main
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh api --method POST /repos/${{ github.repository }}/rulesets \
            --input - << 'EOF'
          {
            "name": "Protect main",
            "target": "branch",
            "enforcement": "active",
            "conditions": {
              "ref_name": {
                "include": ["refs/heads/main"],
                "exclude": []
              }
            },
            "rules": [
              { "type": "deletion" },
              { "type": "non_fast_forward" },
              { "type": "required_linear_history" },
              {
                "type": "pull_request",
                "parameters": {
                  "required_approving_review_count": 1,
                  "dismiss_stale_reviews_on_push": true,
                  "require_last_push_approval": true,
                  "required_review_thread_resolution": true,
                  "require_code_owner_review": false
                }
              }
            ]
          }
          EOF
          echo "✅ Branch ruleset for main created."

      - name: Disable this workflow
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh workflow disable "apply-branch-policy.yml"
          echo "✅ Workflow disabled. Branch policy is now active."
