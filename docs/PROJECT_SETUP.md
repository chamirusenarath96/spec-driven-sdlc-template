# GitHub Project setup

This pipeline assumes a GitHub Project (v2) with a custom **Status** single-select
field driving the state machine in [AGENTS.md](AGENTS.md).

## One-time setup for a new project generated from this template

1. Create a Project: repo → **Projects** tab → **New project** → Board.
2. Add a single-select field named `Status` with these options, in order:
   - `Backlog`
   - `Spec In Progress`
   - `Spec Ready for Review`
   - `Approved for Dev`
   - `In Development`
   - `In Review`
   - `Changes Requested`
   - `Done`
3. Add a workflow (Project → **⋯** → **Workflows**) to auto-add issues to
   `Backlog` when opened, and PRs to `In Review` when opened, so nothing
   silently misses the board.
4. Note the Project's node ID and each Status option's ID — the agents move
   items via the GraphQL API (`updateProjectV2ItemFieldValue`), which needs
   these IDs:

   ```bash
   gh api graphql -f query='
     query($org: String!, $number: Int!) {
       organization(login: $org) {
         projectV2(number: $number) {
           id
           fields(first: 20) {
             nodes {
               ... on ProjectV2SingleSelectField {
                 id
                 name
                 options { id name }
               }
             }
           }
         }
       }
     }' -f org=<your-org-or-user> -F number=<project-number>
   ```

   (Use `user(login: ...)` instead of `organization` if this is a personal
   project.) Store the resulting Project ID, Status field ID, and each
   option's ID as repo variables or reference them directly in the
   `agent-*.md` workflow instructions — this template doesn't hardcode them
   since they're generated per-project.

5. Link the Project to the repo if it isn't already (Project → **Settings**
   → **Manage access**, or `gh project link`).

## Automating status transitions

`gh-aw`'s safe-outputs don't include a first-class "move Project item"
action as of this template's creation — the `agent-review.md` workflow's
`pr-review-decision` custom safe-output job is where you implement the
`updateProjectV2ItemFieldValue` mutation using the IDs from step 4. The
research/dev agent workflows can do the same via a `gh api graphql` call in
their safe-output job, or simply leave item movement to the built-in
"auto-add + status by event" Project workflows from step 3 if that level of
automation is enough for you.
