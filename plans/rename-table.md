# Rename table

This plan is for the same grader and the same maintainer as the role index. The grader still gets `dependency_graph.json` from the catalog path. The maintainer gets two lookups, an exact name index and a small rename table, instead of a description classifier.

These checks are the pass conditions for code that is not written yet. None of them have been run.

## Decisions

The same catalog measurements apply. `issue_number` and `pull_number` never appear as output properties. Producers emit `number` on `Issue` and `PullRequest`. `owner` is required on 442 tools. The output wrapper is `data`, `error`, and `successful` on every tool.

This plan resolves the open choices differently.

- The edge label is still the consumer's parameter name, because that is the README example.
- A producer is kept only when its supply is the response's top-level field or the primary array item. Nested objects such as `Milestone` are not supplies. List, get, and create tools can all be producers. Search tools are not producers.
- `owner`, `repo`, `org`, and `username` are not edges.
- An optional input becomes an edge only when its name is already an exact output name, such as `sha`. `issue_number` is required, so this rule does not create the README chains. It does create edges for optional `sha`.
- `service` is set only when the tool has exactly one tag outside the hint set `openWorldHint`, `readOnlyHint`, `idempotentHint`, `updateHint`, `destructiveHint`, `createHint`, `mcpIgnore`, `important`, and `deprecated`. The value is that tag, lowercased. Otherwise `service` is omitted. List-issues can become `issues`. Create-issue, tagged `Repository Management`, does not get `issues`.
- No model call. Names that miss both indexes are written to `unmatched.json` and produce no edge.

## Components

`loadCatalog` returns untyped JSON. `parseCatalog` returns tools with required names, optional names, and supplies limited to the top level and the primary array item. `exactIndex` maps a leaf name to producer slugs and is used only when the consumer's parameter name equals that leaf. `renameTable` maps a consumer name to a parent definition and a leaf. `matchNames` uses the exact index on a name hit, and the table otherwise. `toGraph` builds nodes and edges. `main` writes `dependency_graph.json` and `unmatched.json`. The viewer reads the graph and writes `graph.html`.

The rename table is data, one array in one file. Adding `comment_id` means adding one row, not a new branch.

## Steps

### 1. Parse top-level and primary-item supplies

Add `parseCatalog` with a narrower walk than the role index. Follow `data.$ref`. Keep properties of that response object. If a property is an array of `$ref` items, keep that item's properties too. Do not walk into properties of those items. Record parent name, leaf name, and JSON type. Ignore descriptions.

Check. Run `node --import tsx src/parse-catalog.ts github_catalog.json GITHUB_LIST_REPOSITORY_ISSUES`. The output includes `Issue.number` from the `issues` array. It does not include `Milestone.number`. It does not include `data`, `error`, or `successful`.

Comment. A passing check shows nested milestone numbers never enter the index, so they cannot become edges later. It does not show a connection to `issue_number`. If a real id is nested two levels down, this parser will miss it, and the check will stay green.

### 2. Build the exact index and the rename table

`exactIndex` maps each supply leaf name to the slugs that emit it. `renameTable` is a constant list. The first rows are `{ input: "issue_number", parent: "Issue", leaf: "number" }` and `{ input: "pull_number", parent: "PullRequest", leaf: "number" }`. Add a row only when the input name never occurs as an output property. `run_id`, `repository_id`, `gist_id`, `sha`, `review_id`, and `project_id` stay on the exact index and have no table row.

Check. Run `node --import tsx src/rename-table.ts`. It prints the two rows above. A lookup of `issue_number` returns only supplies whose parent is `Issue` and whose leaf is `number`. A lookup of `sha` uses the exact index and does not consult the table. A lookup of the leaf `number` is not used for `issue_number`.

Comment. A passing check shows `issue_number` is not joined to every field named `number`. The table is a hand-written list, so a new toolkit with `ticket_number` will miss until a row is added. The check does not prove the rows are complete.

### 3. Emit edges from the two indexes

For a required slot, use the exact index when the slot name is a key there. Otherwise use the one table row with that input name, and keep producers whose parent and leaf equal the row. `label` is the slot name. `from` is the producer. `to` is the consumer. Skip same-slug pairs. For an optional slot, emit an edge only on an exact-index hit. `sha` on merge can match a producer that emits `sha`. It does not use the rename table.

Check. Run `node --import tsx src/generate.ts github_catalog.json`. The JSON contains the list-issues to create-comment edge labeled `issue_number`, and the list-pulls to merge edge labeled `pull_number`. It also contains at least one edge labeled `sha` whose `to` is `GITHUB_MERGE_A_PULL_REQUEST`. `npm run selfcheck` prints provenance 1 and an edge count above 0.

Comment. A passing check shows both README chains and the optional-`sha` decision in the file the grader reads. Selfcheck will not notice a reversed edge or a wrong label, so the check has to name the three edges. An extra `sha` edge can be a commit sha or a pull head sha. This check does not tell those apart.

### 4. Record misses and set service

Any required identifier that hits neither index is appended to `unmatched.json` with its slug and input name. The graph run still exits 0. Set `service` from the single-tag rule. Tools with zero or several resource tags omit `service`.

Check. Run the generator. The list-issues node has `"service": "issues"`. The create-issue node has `"service": "repository management"`. The create-comment node has `"service": "comments"`. Those strings are the single non-hint tag on each tool, lowercased. The tags were read from `github_catalog.json` in this session. `unmatched.json` is an array. `issue_number` and `pull_number` are absent from it.

Comment. A passing check shows a miss is written down instead of guessed, and that `service` follows the tool's own tag. Create-issue does not become `issues`. The check does not decide whether an unmatched name should have had a table row.

### 5. Show the same edges

Write `graph.html` from the graph object. Show the node id, the service when present, and each edge label.

Check. Open `graph.html`. One edge shows `GITHUB_LIST_REPOSITORY_ISSUES`, `GITHUB_CREATE_AN_ISSUE_COMMENT`, and `issue_number`. Another shows `GITHUB_LIST_PULL_REQUESTS`, `GITHUB_MERGE_A_PULL_REQUEST`, and `pull_number`. The list-issues node shows `issues`.

Comment. A passing check shows the connections and the service choice in one view. The page must be generated from `toGraph`, not from a second walk of the catalog. It does not prove edges that are off screen.
