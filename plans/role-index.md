# Role index

This plan is for the person who will grade `dependency_graph.json`, and for the next person who edits `src/generate.ts`. The grader gets edges that name the consumer's parameter. The maintainer gets one role table instead of a chain of name comparisons.

The program still reads the catalog path from the last argument and writes `dependency_graph.json` in the working directory. Node ids stay catalog slugs. This plan does not hardcode GitHub slugs.

These checks are the pass conditions for code that is not written yet. None of them have been run.

## Decisions

The catalog pass counted 893 tools. `issue_number` is a required input 25 times and an output property 0 times. `pull_number` is a required input 27 times and an output property 0 times. List issues exposes `Issue.number` ("Issue number within the repository.") and, in the same payload, `Milestone.number`. `GITHUB_CREATE_AN_ISSUE` also emits an issue `number`. `owner` is required on 442 tools and is also an output on 132. Every tool's output wrapper is `data`, `error`, and `successful`.

This plan resolves the open choices as follows.

- The edge label is the consumer's parameter name. The README example uses `issue_number` and `pull_number`.
- Every producer of that role is kept. The README says an issue number can come from more than one tool. Create-issue stays beside list-issues.
- `owner`, `repo`, `org`, and `username` are not edges. They are coordinates the caller already has. Enums, booleans, and free text such as `body` are not edges.
- Optional inputs are not edges. `sha` on the merge tool stays a caller value.
- `service` is omitted. The catalog has no `service` field, and tags do not follow one chain. Selfcheck does not read `service`.
- No model call. The graph is a pure function of the catalog, so a missing `LITMUS_ASSESSMENT_TOKEN` still writes the file.

## Components

`loadCatalog` reads the file and returns untyped JSON. `parseCatalog` turns that JSON into tools, slots, and supplies. `indexRoles` groups supplies by role. `matchSlots` pairs each remaining slot with those supplies. `toGraph` builds nodes and edges. `main` writes the file. `selfcheck` reads the file after `main` returns. A viewer reads the same graph and writes `graph.html`.

`parseCatalog` is the only function that sees raw JSON. Later functions take the parsed tools.

## Steps

### 1. Parse each tool into slots and supplies

Add `parseCatalog`. A slot is a required input name that exists in `properties`. A supply is a field reached from `outputParameters.properties.data` through that tool's `$defs`, including one level of array items. Record the parent definition name, the leaf name, the JSON type, and the description. Do not record `data`, `error`, or `successful` as supplies.

Check. Run `node --import tsx src/parse-catalog.ts github_catalog.json GITHUB_LIST_REPOSITORY_ISSUES GITHUB_CREATE_AN_ISSUE_COMMENT`. The list tool prints a supply whose parent is `Issue`, leaf is `number`, and description is "Issue number within the repository." The comment tool prints a required slot named `issue_number`. Neither print includes `data`, `error`, or `successful`.

Comment. A passing check shows the parser reached the resource field behind the wrapper and kept the required name the README uses as a label. It does not show that those two records should be connected. Milestone's `number` may also appear, and this step should not delete it yet.

### 2. Drop coordinates and caller-supplied values

A slot stays only when its type is `integer` or `string` and its name is an identifier. Identifiers end in `_number`, `_id`, or `_sha`, or are exactly `id`, `number`, or `sha`. Drop `owner`, `repo`, `org`, `username`, enums, booleans, and free text.

Check. Run the parser on `GITHUB_CREATE_AN_ISSUE_COMMENT` and `GITHUB_MERGE_A_PULL_REQUEST` with the filter on. The comment tool keeps `issue_number` and drops `owner`, `repo`, and `body`. The merge tool keeps `pull_number` and drops `owner`, `repo`, and the optional `sha`.

Comment. A passing check shows the graph will not treat a shared repository coordinate as a dependency. It does not show which tool produces `issue_number`. The identifier rule is a heuristic. A future catalog could require an id under another name, and this check would not catch that.

### 3. Match a slot to supplies by role

Give each remaining slot a role of `issue-number`, `pull-number`, or the exact leaf name when the names already agree (`run_id`, `sha`, `review_id`). A supply matches `issue-number` only when the leaf is `number` and the description says it is an issue number. The same rule, with "pull request number", matches `pull-number`. `Milestone.number` matches neither. Exact leaf names match only the same leaf name and the same JSON type.

Check. Run the matcher on the GitHub catalog and print the producers for `issue_number` and `pull_number`. `GITHUB_LIST_REPOSITORY_ISSUES` and `GITHUB_CREATE_AN_ISSUE` appear for `issue_number`. `GITHUB_LIST_PULL_REQUESTS` appears for `pull_number`. No producer is listed because of `Milestone.number`.

Comment. A passing check shows the README chains survive a name mismatch, and that a nested milestone number does not. It does not show the written JSON file, the edge direction, or tools beyond this catalog. The description test is the fragile part. A wording change in a later catalog can drop a true producer.

### 4. Write producer-to-consumer edges

`toGraph` emits one node per parsed slug, with no `service`. For each match, `from` is the producer slug and `to` is the consumer slug. `label` is the consumer's parameter name. Skip a match whose producer and consumer are the same slug. `main` writes `dependency_graph.json` with two-space indentation. Running it twice replaces the file with the same bytes.

Check. Run `node --import tsx src/generate.ts github_catalog.json`, then inspect the JSON. It contains `{ "from": "GITHUB_LIST_REPOSITORY_ISSUES", "to": "GITHUB_CREATE_AN_ISSUE_COMMENT", "label": "issue_number" }` and `{ "from": "GITHUB_LIST_PULL_REQUESTS", "to": "GITHUB_MERGE_A_PULL_REQUEST", "label": "pull_number" }`. `npm run selfcheck` prints a provenance ratio of 1 and an edge count above 0. Both warnings stay silent.

Comment. A passing check shows the file the grader reads has the two documented edges and catalog slugs. Selfcheck does not read `from`, `to`, or `label`, so this check must look at those fields itself. A high edge count alone would not prove the README chains.

### 5. Show the same edges

Write `graph.html` from the graph object `toGraph` already returned. The page lists nodes and draws an edge for each record. Generating again overwrites the file.

Check. Open `graph.html` and find the text `GITHUB_LIST_REPOSITORY_ISSUES`, `GITHUB_CREATE_AN_ISSUE_COMMENT`, and `issue_number` on one edge. Find `GITHUB_LIST_PULL_REQUESTS`, `GITHUB_MERGE_A_PULL_REQUEST`, and `pull_number` on another.

Comment. A passing check shows a person can see the connections without reading JSON. It does not show that every edge in the file is correct. The HTML is a view of the same graph, so a second matching pass here would be a second source of truth.
