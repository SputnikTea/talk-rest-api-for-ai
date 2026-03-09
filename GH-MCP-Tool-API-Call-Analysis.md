# GitHub MCP Server — Tool API-Call Analyse

Analyse der tatsächlichen API-Calls (REST + GraphQL) je Tool-Methode, basierend auf dem Quellcode in `pkg/github/`.

---

## Actions

### `actions_get` — `actions.go:395`

| method | Endpoint | Calls |
|--------|----------|-------|
| `get_workflow` | `GET /repos/{owner}/{repo}/actions/workflows/{id}` | 1 |
| `get_workflow_run` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}` | 1 |
| `get_workflow_run_usage` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}/timing` | 1 |
| `get_workflow_run_logs_url` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}/logs` | 1 |
| `download_workflow_run_artifact` | `GET /repos/{owner}/{repo}/actions/artifacts/{artifact_id}/zip` | 1 |
| `get_workflow_job` | `GET /repos/{owner}/{repo}/actions/jobs/{job_id}` | 1 |

### `actions_list` — `actions.go:200`

| method | Endpoint | Calls |
|--------|----------|-------|
| `list_workflows` | `GET /repos/{owner}/{repo}/actions/workflows` | 1 |
| `list_workflow_runs` | `GET /repos/{owner}/{repo}/actions/runs` (ohne workflow_id) oder `.../workflows/{id}/runs` | 1 |
| `list_workflow_jobs` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}/jobs` | 1 |
| `list_workflow_run_artifacts` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}/artifacts` | 1 |

### `actions_run_trigger` — `actions.go:503`

| method | Endpoint | Calls |
|--------|----------|-------|
| `run_workflow` | `POST /repos/{owner}/{repo}/actions/workflows/{id}/dispatches` | 1 |
| `rerun_workflow_run` | `POST /repos/{owner}/{repo}/actions/runs/{run_id}/rerun` | 1 |
| `rerun_failed_jobs` | `POST /repos/{owner}/{repo}/actions/runs/{run_id}/rerun-failed-jobs` | 1 |
| `cancel_workflow_run` | `POST /repos/{owner}/{repo}/actions/runs/{run_id}/cancel` | 1 |
| `delete_workflow_run_logs` | `DELETE /repos/{owner}/{repo}/actions/runs/{run_id}/logs` | 1 |

### `get_job_logs` — `actions.go:621`

| Szenario | Calls |
|----------|-------|
| Einzelner Job, `return_content=false` | 1 REST |
| Einzelner Job, `return_content=true` | 1 REST + 1 `http.GET` (pre-signed URL) |
| `failed_only=true`, N failed Jobs, `return_content=false` | 1+N REST |
| `failed_only=true`, N failed Jobs, `return_content=true` | 1+N REST + N `http.GET` = **1+2N** |

---

## Issues

### `issue_read` — `issues.go:205`

| method | Implementierung | API Call | Calls |
|--------|----------------|----------|-------|
| `get` | `GetIssue` (:302) | REST: `GET /repos/{owner}/{repo}/issues/{number}` | 1 |
| `get_comments` | `GetIssueComments` (:354) | REST: `GET /repos/{owner}/{repo}/issues/{number}/comments` | 1 |
| `get_sub_issues` | `GetSubIssues` (:414) | REST: `GET /repos/{owner}/{repo}/issues/{number}/sub_issues` | 1 |
| `get_labels` | `GetIssueLabels` (:480) | **GraphQL** Query: `repository { issue { labels { nodes } } }` | 1 |

### `issue_write` — `issues.go:972`

| method | Szenario | Calls |
|--------|----------|-------|
| `create` | — | 1 REST: `POST /repos/{owner}/{repo}/issues` |
| `update` | kein `state` | 1 REST: `PATCH .../issues/{number}` |
| `update` | `state` gesetzt | 3: 1 REST `PATCH` + 1 GraphQL Query `fetchIssueIDs` + 1 GraphQL Mutation (`reopenIssue` / `closeIssue`) |

### `sub_issue_write` — `issues.go:679`

| method | Endpoint | Calls |
|--------|----------|-------|
| `add` | `POST /repos/{owner}/{repo}/issues/{number}/sub_issues` | 1 |
| `remove` | `DELETE /repos/{owner}/{repo}/issues/{number}/sub_issues` | 1 |
| `reprioritize` | `PATCH /repos/{owner}/{repo}/issues/{number}/sub_issues` | 1 |

---

## Pull Requests

### `pull_request_read` — `pullrequests.go:26`

| method | Implementierung | Calls |
|--------|----------------|-------|
| `get` | `GetPullRequest` (:141) | 1 REST: `GET .../pulls/{number}` |
| `get_diff` | `GetPullRequestDiff` (:198) | 1 REST: `GET .../pulls/{number}` (diff media type) |
| `get_status` | `GetPullRequestStatus` (:228) | **2 REST**: `GET .../pulls/{number}` (HEAD-SHA holen) + `GET .../commits/{sha}/status` |
| `get_files` | `GetPullRequestFiles` (:339) | 1 REST: `GET .../pulls/{number}/files` |
| `get_review_comments` | `GetPullRequestReviewComments` (:411) | **1 GraphQL** Query: `repository { pullRequest { reviewThreads { comments } } }` |
| `get_reviews` | `GetPullRequestReviews` (:481) | 1 REST: `GET .../pulls/{number}/reviews` |
| `get_comments` | `GetIssueComments` (issues.go:354) | 1 REST: `GET .../issues/{number}/comments` |
| `get_check_runs` | `GetPullRequestCheckRuns` (:274) | **2 REST**: `GET .../pulls/{number}` (HEAD-SHA holen) + `GET .../commits/{sha}/check-runs` |

### `pull_request_review_write` — `pullrequests.go:1512`

> Ausschliesslich GraphQL — kein REST.

| method | Implementierung | GraphQL Calls |
|--------|----------------|---------------|
| `create` | `CreatePullRequestReview` (:1599) | 2: Query (PR node ID) + Mutation `addPullRequestReview` |
| `submit_pending` | `SubmitPendingPullRequestReview` (:1657) | 3: Query `viewer.login` + Query (pending review suchen) + Mutation `submitPullRequestReview` |
| `delete_pending` | `DeletePendingPullRequestReview` (:1742) | 3: Query `viewer.login` + Query (pending review suchen) + Mutation `deletePullRequestReview` |

### `update_pull_request` — `pullrequests.go:702`

Das abschliessende `PullRequests.Get` wird immer ausgefuehrt.

| Szenario | REST | GraphQL | Total |
|----------|------|---------|-------|
| Nur Metadaten (title/body/state/base) | 2 | 0 | 2 |
| Nur `draft` aendern | 1 | 2 (Query node ID + Mutation convert/markReady) | 3 |
| `draft` bereits im Zielzustand | 1 | 1 (Query only) | 2 |
| Nur `reviewers` | 2 | 0 | 2 |
| Alle Parameter kombiniert | 3 | 2 | **5** |

---

## Labels

### `label_write` — `labels.go:213`

> Ausschliesslich GraphQL — kein REST.

| method | GraphQL Calls |
|--------|---------------|
| `create` | 2: Query `getRepositoryID` + Mutation `createLabel` |
| `update` | 2: Query `getLabelID` + Mutation `updateLabel` |
| `delete` | 2: Query `getLabelID` + Mutation `deleteLabel` |

---

## Repositories / Git

### `push_files` — `repositories.go:1207`

Dateiinhalte werden als Inline-Content in `CreateTree` uebergeben — kein separater Blob-Call pro Datei.

| Pfad | Szenario | REST Calls |
|------|----------|-----------|
| A | Branch existiert (Normalfall) | **5**: `GetRef` → `GetCommit` → `CreateTree` → `CreateCommit` → `UpdateRef` |
| B | Branch nicht gefunden (404) | **8**: + `Repositories.Get` + 2x `GetRef` (default branch) + `CreateRef` |
| C | Leeres Repo (409), branch = default | **8** |
| C | Leeres Repo (409), branch != default | **11** |

### `create_branch` — `repositories.go:1094`

| Szenario | REST Calls |
|----------|-----------|
| `from_branch` angegeben | 2: `Git.GetRef` + `Git.CreateRef` |
| `from_branch` nicht angegeben | 3: `Repositories.Get` + `Git.GetRef` (default) + `Git.CreateRef` |

---

## Notifications

### `dismiss_notification` — `notifications.go:166`

| state | Endpoint | Calls |
|-------|----------|-------|
| `"read"` | `PATCH /notifications/threads/{thread_id}` | 1 |
| `"done"` | `DELETE /notifications/threads/{thread_id}` | 1 |

### `manage_notification_subscription` — `notifications.go:409`

| action | Endpoint | Calls |
|--------|----------|-------|
| `"ignore"` / `"watch"` | `PUT /notifications/threads/{thread_id}/subscription` | 1 |
| `"delete"` | `DELETE /notifications/threads/{thread_id}/subscription` | 1 |

### `manage_repository_notification_subscription` — `notifications.go:505`

| action | Endpoint | Calls |
|--------|----------|-------|
| `"ignore"` / `"watch"` | `PUT /repos/{owner}/{repo}/subscription` | 1 |
| `"delete"` | `DELETE /repos/{owner}/{repo}/subscription` | 1 |

---

## Context

### `get_teams` — `context_tools.go:122`

| Szenario | REST | GraphQL | Total |
|----------|------|---------|-------|
| `user` nicht angegeben | 1x `GET /user` | 1x Query `user { organizations { teams } }` | 2 |
| `user` angegeben | — | 1x Query `user { organizations { teams } }` | 1 |

---

## Projects (REST + GraphQL Hybrid)

> `(+2)` = optionaler `detectOwnerType`-Call wenn `owner_type` nicht angegeben (bis zu 2 zusaetzliche REST Calls).

### `projects_get` — `projects.go:264`

| method | Implementierung | API Calls |
|--------|----------------|-----------|
| `get_project` | `getProject` (:846) | 1 REST (+2) |
| `get_project_field` | `getProjectField` (:882) | 1 REST (+2) |
| `get_project_item` | `getProjectItem` (:917) | 1 REST (+2) |
| `get_project_status_update` | `getProjectStatusUpdate` (:1281) | 1 GraphQL Query: `node(id) { ...ProjectV2StatusUpdate }` |

### `projects_list` — `projects.go:139`

| method | Implementierung | API Calls |
|--------|----------------|-----------|
| `list_projects` (mit `owner_type`) | `listProjects` (:608) | 1 REST |
| `list_projects` (ohne `owner_type`) | `listProjectsFromBothOwnerTypes` (:685) | 2 REST (beide Typen, Ergebnisse gemergt) |
| `list_project_fields` | `listProjectFields` (:735) | 1 REST (+2) |
| `list_project_items` | `listProjectItems` (:781) | 1 REST (+2) |
| `list_project_status_updates` | `listProjectStatusUpdates` (:1202) | 1 GraphQL Query: `org/user { projectV2 { statusUpdates } }` (+2) |

### `projects_write` — `projects.go:401`

| method | Implementierung | API Calls |
|--------|----------------|-----------|
| `add_project_item` | `addProjectItem` (:1066) | **3 GraphQL** (+2): Query issue/PR node ID + Query project node ID + Mutation `addProjectV2ItemById` |
| `update_project_item` | `updateProjectItem` (:960) | 1 REST (+2) |
| `delete_project_item` | `deleteProjectItem` (:999) | 1 REST (+2) |
| `create_project_status_update` | `createProjectStatusUpdate` (:1131) | **2 GraphQL** (+2): Query project node ID + Mutation `createProjectV2StatusUpdate` |

---

## Discussions (rein GraphQL)

Alle vier Tools machen genau **1 GraphQL Query**, kein REST.

| Tool | Implementierung | GraphQL Query |
|------|----------------|---------------|
| `get_discussion` | `GetDiscussion` (discussions.go:279) | `repository { discussion(number) { title, body, ... } }` |
| `get_discussion_comments` | `GetDiscussionComments` (discussions.go:383) | `repository { discussion(number) { comments(first, after) } }` |
| `list_discussions` | `ListDiscussions` (discussions.go:126) | `repository { discussions(...) { nodes, pageInfo } }` (4 Struct-Varianten je nach `category`/`orderBy`) |
| `list_discussion_categories` | `ListDiscussionCategories` (discussions.go:510) | `repository { discussionCategories(first: 25) { nodes { id, name } } }` |

---

## Auffaelligkeiten

**Tools die ausschliesslich GraphQL nutzen (kein REST):**
- `pull_request_review_write` — alle 3 Methoden machen 2–3 GraphQL Calls
- `label_write` — alle 3 Methoden machen je 2 GraphQL Calls (ID-Lookup-Query dann Mutation)
- `get_review_comments` (Methode in `pull_request_read`) — einzige reine GraphQL-Methode in einem sonst REST-basierten Tool

**Versteckte Call-Kaskaden:**
- `issue_write` `update` mit `state`-Parameter: verdreifacht die Calls von 1 auf 3 (REST + 2 GraphQL). Ohne `state` bleibt es bei 1.
- `update_pull_request`: 2 bis 5 Calls je nach uebergebenen Parametern. Das abschliessende `GET` fuer den Rueckgabewert laeuft immer mit.
- `get_job_logs` mit `failed_only=true`: skaliert mit **1+2N** Calls (N = Anzahl fehlgeschlagener Jobs). Die Downloads sind direkte `http.GET` auf pre-signed S3-URLs ausserhalb des API-Clients — also unsichtbar fuer Rate-Limit-Tracking.

**Projects ist echter REST+GraphQL Hybrid:** `add_project_item` benoetigt 3 GraphQL Calls fuer eine konzeptuell einfache Operation — zwei Lookup-Queries fuer Node-IDs vor der eigentlichen Mutation. Dazu kommen bis zu 2 weitere REST Calls fuer `detectOwnerType` wenn `owner_type` nicht explizit angegeben wird.

**`push_files` erstellt keine Blobs:** Dateiinhalte werden direkt als Inline-Content in `CreateTree` eingebettet, nicht ueber separate Blob-Calls. Im Fehlerfall (leeres Repo, fehlende Branch) steigt die Call-Zahl von 5 auf bis zu 11.

---

## Eignung als Arazzo Workflow

Das Tool **`push_files`** (Pfad A, Normalfall) eignet sich am besten als Arazzo Spec Workflow.

**Warum:**
- Klar sequenzielle Schritte mit expliziten Datenabhaengigkeiten: jeder Schritt konsumiert den Output des vorherigen
- Alle 5 Schritte sind Standard-REST-Endpunkte mit vollstaendiger OpenAPI-Spec-Coverage
- Klarer Startpunkt (branch name + file contents) und Endpunkt (updated ref / neuer Commit SHA)
- Die Output-zu-Input-Verkettung ist deterministisch und maschinenlesbar darstellbar

**Arazzo-Workflow-Skizze (Pfad A):**

```
Step 1: GET /repos/{owner}/{repo}/git/ref/refs/heads/{branch}
        outputs: $.object.sha as $parentCommitSha

Step 2: GET /repos/{owner}/{repo}/git/commits/{$parentCommitSha}
        outputs: $.tree.sha as $baseTreeSha

Step 3: POST /repos/{owner}/{repo}/git/trees
        body: { base_tree: $baseTreeSha, tree: [{ path, mode, type, content }] }
        outputs: $.sha as $newTreeSha

Step 4: POST /repos/{owner}/{repo}/git/commits
        body: { message, tree: $newTreeSha, parents: [$parentCommitSha] }
        outputs: $.sha as $newCommitSha

Step 5: PATCH /repos/{owner}/{repo}/git/refs/refs/heads/{branch}
        body: { sha: $newCommitSha }
```

**Alternativer Kandidat:** `get_job_logs` mit `failed_only=true` demonstriert einen dynamischen Fan-out (Liste Jobs → fuer jeden Job Logs holen), was ein interessantes Arazzo-Pattern fuer bedingte Iteration waere — liegt aber ausserhalb der aktuellen Arazzo 1.0 Spec-Moeglichkeiten.
