# GitHub MCP Server: Tool-zu-API-Endpunkt-Mapping

Analyse welche MCP Tools mehrere GitHub API Calls abstrahieren und welche REST-Endpunkte (bzw. GraphQL-Operationen) sie intern aufrufen.

---

## Tools die mehrere API-Endpunkte abstrahieren

### Actions

#### `actions_get` (1 Tool -> 6 Endpunkte)

Dispatcht uber den `method`-Parameter an verschiedene Endpunkte:

| method | REST Endpunkt |
|--------|--------------|
| `get_workflow` | `GET /repos/{owner}/{repo}/actions/workflows/{workflow_id}` |
| `get_workflow_run` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}` |
| `get_workflow_run_usage` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}/timing` |
| `get_workflow_run_logs_url` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}/logs` |
| `download_workflow_run_artifact` | `GET /repos/{owner}/{repo}/actions/artifacts/{artifact_id}/{archive_format}` |
| `get_workflow_job` | `GET /repos/{owner}/{repo}/actions/jobs/{job_id}` |

#### `actions_list` (1 Tool -> 5 Endpunkte)

| method | REST Endpunkt |
|--------|--------------|
| `list_workflows` | `GET /repos/{owner}/{repo}/actions/workflows` |
| `list_workflow_runs` (mit workflow_id) | `GET /repos/{owner}/{repo}/actions/workflows/{workflow_id}/runs` |
| `list_workflow_runs` (ohne workflow_id) | `GET /repos/{owner}/{repo}/actions/runs` |
| `list_workflow_jobs` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}/jobs` |
| `list_workflow_run_artifacts` | `GET /repos/{owner}/{repo}/actions/runs/{run_id}/artifacts` |

#### `actions_run_trigger` (1 Tool -> 4 Endpunkte)

| method | REST Endpunkt |
|--------|--------------|
| `run_workflow` | `POST /repos/{owner}/{repo}/actions/workflows/{workflow_id}/dispatches` |
| `cancel_workflow_run` | `POST /repos/{owner}/{repo}/actions/runs/{run_id}/cancel` |
| `rerun_workflow` | `POST /repos/{owner}/{repo}/actions/runs/{run_id}/rerun` |
| `rerun_failed_jobs` | `POST /repos/{owner}/{repo}/actions/runs/{run_id}/rerun-failed-jobs` |

#### `get_job_logs` (1 Tool -> 2+ Endpunkte)

Wenn `failed_only=true`: muss intern erst Jobs listen, dann Logs fur jeden fehlgeschlagenen Job holen.

| Szenario | REST Endpunkte |
|----------|---------------|
| Einzelner Job | `GET /repos/{owner}/{repo}/actions/jobs/{job_id}/logs` |
| Alle fehlgeschlagenen Jobs | `GET /repos/{owner}/{repo}/actions/runs/{run_id}/jobs` + `GET /repos/{owner}/{repo}/actions/jobs/{job_id}/logs` (pro Job) |

---

### Issues

#### `issue_read` (1 Tool -> 4 Endpunkte)

| method | REST Endpunkt |
|--------|--------------|
| `get` | `GET /repos/{owner}/{repo}/issues/{issue_number}` |
| `get_comments` | `GET /repos/{owner}/{repo}/issues/{issue_number}/comments` |
| `get_sub_issues` | `GET /repos/{owner}/{repo}/issues/{issue_number}/sub_issues` |
| `get_labels` | `GET /repos/{owner}/{repo}/issues/{issue_number}/labels` |

#### `issue_write` (1 Tool -> 2 Endpunkte)

| method | REST Endpunkt |
|--------|--------------|
| `create` | `POST /repos/{owner}/{repo}/issues` |
| `update` | `PATCH /repos/{owner}/{repo}/issues/{issue_number}` |

#### `sub_issue_write` (1 Tool -> 3 Endpunkte)

| method | REST Endpunkt |
|--------|--------------|
| `add` | `POST /repos/{owner}/{repo}/issues/{issue_number}/sub_issues` |
| `remove` | `DELETE /repos/{owner}/{repo}/issues/{issue_number}/sub_issue` |
| `reprioritize` | `PATCH /repos/{owner}/{repo}/issues/{issue_number}/sub_issues/priority` |

---

### Pull Requests

#### `pull_request_read` (1 Tool -> 8 Endpunkte)

Die komplexeste Read-Abstraktion. Einige Methoden erfordern intern einen vorgelagerten Call um den HEAD-SHA des PRs zu ermitteln.

| method | REST Endpunkt |
|--------|--------------|
| `get` | `GET /repos/{owner}/{repo}/pulls/{pull_number}` |
| `get_diff` | `GET /repos/{owner}/{repo}/pulls/{pull_number}` (mit `Accept: application/vnd.github.diff`) |
| `get_status` | `GET /repos/{owner}/{repo}/pulls/{pull_number}` (HEAD-SHA holen) + `GET /repos/{owner}/{repo}/commits/{ref}/status` |
| `get_files` | `GET /repos/{owner}/{repo}/pulls/{pull_number}/files` |
| `get_review_comments` | `GET /repos/{owner}/{repo}/pulls/{pull_number}/comments` |
| `get_reviews` | `GET /repos/{owner}/{repo}/pulls/{pull_number}/reviews` |
| `get_comments` | `GET /repos/{owner}/{repo}/issues/{issue_number}/comments` |
| `get_check_runs` | `GET /repos/{owner}/{repo}/pulls/{pull_number}` (HEAD-SHA holen) + `GET /repos/{owner}/{repo}/commits/{ref}/check-runs` |

#### `pull_request_review_write` (1 Tool -> 3 Endpunkte)

| method | REST Endpunkt |
|--------|--------------|
| `create` | `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews` |
| `submit` | `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}/events` |
| `delete` | `DELETE /repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}` |

#### `update_pull_request` (1 Tool -> 2 Endpunkte)

Aktualisiert den PR und kann optional Reviewer zuweisen (separater Endpunkt):

| Aktion | REST Endpunkt |
|--------|--------------|
| PR updaten | `PATCH /repos/{owner}/{repo}/pulls/{pull_number}` |
| Reviewer zuweisen (wenn `reviewers` angegeben) | `POST /repos/{owner}/{repo}/pulls/{pull_number}/requested_reviewers` |

---

### Labels

#### `label_write` (1 Tool -> 3 Endpunkte)

| method | REST Endpunkt |
|--------|--------------|
| `create` | `POST /repos/{owner}/{repo}/labels` |
| `update` | `PATCH /repos/{owner}/{repo}/labels/{name}` |
| `delete` | `DELETE /repos/{owner}/{repo}/labels/{name}` |

---

### Repositories

#### `push_files` (1 Tool -> 5+ Endpunkte)

Die komplexeste Abstraktion insgesamt -- fasst den gesamten Git-Push-Workflow in einen einzigen Tool-Aufruf zusammen:

| Schritt | REST Endpunkt |
|---------|--------------|
| 1. Aktuellen Branch-Ref lesen | `GET /repos/{owner}/{repo}/git/refs/heads/{branch}` |
| 2. Blob(s) erstellen (pro Datei) | `POST /repos/{owner}/{repo}/git/blobs` |
| 3. Tree erstellen | `POST /repos/{owner}/{repo}/git/trees` |
| 4. Commit erstellen | `POST /repos/{owner}/{repo}/git/commits` |
| 5. Ref aktualisieren | `PATCH /repos/{owner}/{repo}/git/refs/heads/{branch}` |

#### `create_branch` (1 Tool -> 2 Endpunkte)

| Schritt | REST Endpunkt |
|---------|--------------|
| 1. SHA der Quell-Branch holen | `GET /repos/{owner}/{repo}/branches/{branch}` |
| 2. Neue Ref erstellen | `POST /repos/{owner}/{repo}/git/refs` |

---

### Notifications

#### `dismiss_notification` (1 Tool -> 2 Endpunkte)

| state | REST Endpunkt |
|-------|--------------|
| `read` | `PATCH /notifications/threads/{thread_id}` |
| `done` | `DELETE /notifications/threads/{thread_id}` |

#### `manage_notification_subscription` (1 Tool -> 2 Endpunkte)

| action | REST Endpunkt |
|--------|--------------|
| `ignore` / `watch` | `PUT /notifications/threads/{thread_id}/subscription` |
| `delete` | `DELETE /notifications/threads/{thread_id}/subscription` |

#### `manage_repository_notification_subscription` (1 Tool -> 2 Endpunkte)

| action | REST Endpunkt |
|--------|--------------|
| `ignore` / `watch` | `PUT /repos/{owner}/{repo}/subscription` |
| `delete` | `DELETE /repos/{owner}/{repo}/subscription` |

---

### Context

#### `get_teams` (1 Tool -> 2 Endpunkte)

| Szenario | REST Endpunkt |
|----------|--------------|
| Teams des authentifizierten Users | `GET /user/teams` |
| Teams eines bestimmten Users | `GET /orgs/{org}/teams` + Filterung |

---

### Projects (primaer GraphQL)

#### `projects_get` (1 Tool -> 3+ Operationen)

| method | Zugrunde liegende Operation |
|--------|---------------------------|
| `get_project` | GraphQL Query / `GET /orgs/{org}/projectsV2/{project_number}` |
| `get_project_field` | GraphQL Query / `GET /orgs/{org}/projectsV2/{project_number}/fields/{field_id}` |
| `get_project_item` | GraphQL Query / `GET /orgs/{org}/projectsV2/{project_number}/items/{item_id}` |
| `get_project_status_update` | GraphQL Query |

#### `projects_list` (1 Tool -> 4+ Operationen)

| method | Zugrunde liegende Operation |
|--------|---------------------------|
| `list_projects` | GraphQL / `GET /orgs/{org}/projectsV2` |
| `list_project_fields` | GraphQL / `GET /orgs/{org}/projectsV2/{project_number}/fields` |
| `list_project_items` | GraphQL / `GET /orgs/{org}/projectsV2/{project_number}/items` |
| `list_project_status_updates` | GraphQL Query |

#### `projects_write` (1 Tool -> 4+ Operationen)

| method | Zugrunde liegende Operation |
|--------|---------------------------|
| `add_project_item` | GraphQL Mutation / REST POST |
| `update_project_item` | GraphQL Mutation / REST PATCH |
| `delete_project_item` | GraphQL Mutation / REST DELETE |
| `create_project_status_update` | GraphQL Mutation |

---

### Discussions (rein GraphQL)

Alle Discussions-Tools nutzen ausschliesslich die GraphQL API, da es keine REST-Endpunkte fur GitHub Discussions gibt:

| MCP Tool | GraphQL Operation |
|----------|------------------|
| `get_discussion` | GraphQL Query: Discussion by number |
| `get_discussion_comments` | GraphQL Query: Discussion comments |
| `list_discussions` | GraphQL Query: Discussions list |
| `list_discussion_categories` | GraphQL Query: Discussion categories |

---

### Copilot (proprietaere API)

Diese Tools haben keine entsprechenden Endpunkte in der oeffentlichen OpenAPI Spec:

| MCP Tool | API |
|----------|-----|
| `assign_copilot_to_issue` | Proprietaere/interne API |
| `request_copilot_review` | Proprietaere/interne API |

---

## 1:1-Mappings (ein Tool = ein Endpunkt)

| MCP Tool | REST Endpunkt |
|----------|--------------|
| **Context** | |
| `get_me` | `GET /user` |
| `get_team_members` | `GET /orgs/{org}/teams/{team_slug}/members` |
| **Issues** | |
| `add_issue_comment` | `POST /repos/{owner}/{repo}/issues/{issue_number}/comments` |
| `list_issues` | `GET /repos/{owner}/{repo}/issues` |
| `search_issues` | `GET /search/issues` |
| `list_issue_types` | `GET /orgs/{org}/issue-types` |
| `get_label` | `GET /repos/{owner}/{repo}/labels/{name}` |
| `list_label` | `GET /repos/{owner}/{repo}/labels` |
| **Pull Requests** | |
| `create_pull_request` | `POST /repos/{owner}/{repo}/pulls` |
| `list_pull_requests` | `GET /repos/{owner}/{repo}/pulls` |
| `merge_pull_request` | `PUT /repos/{owner}/{repo}/pulls/{pull_number}/merge` |
| `search_pull_requests` | `GET /search/issues` (mit `is:pr` Qualifier) |
| `update_pull_request_branch` | `PUT /repos/{owner}/{repo}/pulls/{pull_number}/update-branch` |
| `add_comment_to_pending_review` | `POST /repos/{owner}/{repo}/pulls/{pull_number}/comments` |
| `add_reply_to_pull_request_comment` | `POST /repos/{owner}/{repo}/pulls/{pull_number}/comments/{comment_id}/replies` |
| **Repositories / Git** | |
| `get_repository_tree` | `GET /repos/{owner}/{repo}/git/trees/{tree_sha}` |
| `get_file_contents` | `GET /repos/{owner}/{repo}/contents/{path}` |
| `create_or_update_file` | `PUT /repos/{owner}/{repo}/contents/{path}` |
| `delete_file` | `DELETE /repos/{owner}/{repo}/contents/{path}` |
| `create_repository` | `POST /user/repos` |
| `fork_repository` | `POST /repos/{owner}/{repo}/forks` |
| `get_commit` | `GET /repos/{owner}/{repo}/commits/{ref}` |
| `list_commits` | `GET /repos/{owner}/{repo}/commits` |
| `list_branches` | `GET /repos/{owner}/{repo}/branches` |
| `list_releases` | `GET /repos/{owner}/{repo}/releases` |
| `get_latest_release` | `GET /repos/{owner}/{repo}/releases/latest` |
| `get_release_by_tag` | `GET /repos/{owner}/{repo}/releases/tags/{tag}` |
| `list_tags` | `GET /repos/{owner}/{repo}/tags` |
| `get_tag` | `GET /repos/{owner}/{repo}/git/tags/{tag_sha}` |
| `search_code` | `GET /search/code` |
| `search_repositories` | `GET /search/repositories` |
| **Notifications** | |
| `list_notifications` | `GET /notifications` |
| `get_notification_details` | `GET /notifications/threads/{thread_id}` |
| `mark_all_notifications_read` | `PUT /notifications` |
| **Gists** | |
| `create_gist` | `POST /gists` |
| `get_gist` | `GET /gists/{gist_id}` |
| `list_gists` | `GET /gists` |
| `update_gist` | `PATCH /gists/{gist_id}` |
| **Code Security** | |
| `get_code_scanning_alert` | `GET /repos/{owner}/{repo}/code-scanning/alerts/{alert_number}` |
| `list_code_scanning_alerts` | `GET /repos/{owner}/{repo}/code-scanning/alerts` |
| **Dependabot** | |
| `get_dependabot_alert` | `GET /repos/{owner}/{repo}/dependabot/alerts/{alert_number}` |
| `list_dependabot_alerts` | `GET /repos/{owner}/{repo}/dependabot/alerts` |
| **Secret Protection** | |
| `get_secret_scanning_alert` | `GET /repos/{owner}/{repo}/secret-scanning/alerts/{alert_number}` |
| `list_secret_scanning_alerts` | `GET /repos/{owner}/{repo}/secret-scanning/alerts` |
| **Security Advisories** | |
| `get_global_security_advisory` | `GET /advisories/{ghsa_id}` |
| `list_global_security_advisories` | `GET /advisories` |
| `list_org_repository_security_advisories` | `GET /orgs/{org}/security-advisories` |
| `list_repository_security_advisories` | `GET /repos/{owner}/{repo}/security-advisories` |
| **Stargazers** | |
| `list_starred_repositories` | `GET /user/starred` |
| `star_repository` | `PUT /user/starred/{owner}/{repo}` |
| `unstar_repository` | `DELETE /user/starred/{owner}/{repo}` |
| **Users / Orgs** | |
| `search_users` | `GET /search/users` (mit `type:user`) |
| `search_orgs` | `GET /search/users` (mit `type:org`) |

---

## Ranking: Staerkste Abstraktionen

| Rang | Tool | Anzahl Endpunkte | Bemerkung |
|------|------|-----------------|-----------|
| 1 | `push_files` | 5+ | Kompletter Git-Push-Workflow (Blob, Tree, Commit, Ref) |
| 2 | `pull_request_read` | 8 | Verschiedenste Endpunkte inkl. Cross-API (Issues, Checks) |
| 3 | `actions_get` | 6 | Workflows, Runs, Jobs, Artifacts |
| 4 | `actions_list` | 5 | Verschiedene Listen-Endpunkte |
| 5 | `actions_run_trigger` | 4 | Dispatch, Cancel, Rerun |
| 6 | `issue_read` | 4 | Issue + Comments + Sub-Issues + Labels |
| 7 | `projects_list` | 4+ | GraphQL/REST Hybrid |
| 8 | `projects_write` | 4+ | GraphQL/REST Hybrid |
| 9 | `projects_get` | 3+ | GraphQL/REST Hybrid |
| 10 | `pull_request_review_write` | 3 | Create, Submit, Delete |
| 11 | `sub_issue_write` | 3 | Add, Remove, Reprioritize |
| 12 | `label_write` | 3 | Create, Update, Delete |
