# API Call Analyse: `add_comment_to_pending_review`

Das Tool macht **3 sequenzielle GraphQL API Calls**:

| # | Typ | Zweck |
|---|-----|-------|
| 1 | `Query` | Aktuellen Benutzer (Viewer Login) abrufen |
| 2 | `Query` | Letztes Pending Review des Benutzers auf dem PR abrufen |
| 3 | `Mutation` | Review-Kommentar-Thread zum Pending Review hinzufügen |

Die Calls sind sequenziell abhängig: Call 2 benötigt den Login aus Call 1, Call 3 benötigt die Review-ID aus Call 2.

## GraphQL Queries

### 1. Viewer Login abrufen

```graphql
query {
  viewer {
    login
  }
}
```

### 2. Letztes Review des Viewers auf dem PR

```graphql
query ($owner: String!, $name: String!, $prNum: Int!, $author: String!) {
  repository(owner: $owner, name: $name) {
    pullRequest(number: $prNum) {
      reviews(first: 1, author: $author) {
        nodes {
          id
          state
          url
        }
      }
    }
  }
}
```

### 3. Review-Kommentar-Thread hinzufügen

```graphql
mutation ($input: AddPullRequestReviewThreadInput!) {
  addPullRequestReviewThread(input: $input) {
    thread {
      id
    }
  }
}
```

Input-Felder von `$input`:
- `path` — Dateipfad im Diff
- `body` — Kommentartext
- `subjectType` — z.B. `LINE` oder `FILE`
- `line` / `side` — Zielzeile und Seite (`LEFT`/`RIGHT`)
- `startLine` / `startSide` — Für Multi-Line-Kommentare
- `pullRequestReviewID` — ID des Pending Reviews aus Query 2

> Quelle: `pkg/github/pullrequests.go:1895-2009`

## REST API Alternativen (OpenAPI Spec)

| # | GraphQL Operation | REST Endpunkt | operationId | Match |
|---|---|---|---|---|
| 1 | `viewer { login }` | `GET /user` | `users/get-authenticated` | Voll |
| 2 | Reviews nach Author filtern | `GET /repos/{owner}/{repo}/pulls/{pull_number}/reviews` | `pulls/list-reviews` | Teilweise |
| 3 | `addPullRequestReviewThread` | `POST /repos/{owner}/{repo}/pulls/{pull_number}/comments` | `pulls/create-review-comment` | Teilweise |
| 3 | (Alternative) | `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews` | `pulls/create-review` | Teilweise |

### Bewertung

- **Call 1** — `GET /user` ist ein vollwertiger Ersatz. Liefert das `login`-Feld des authentifizierten Benutzers.
- **Call 2** — `GET .../pulls/{pull_number}/reviews` liefert alle Reviews, hat aber **keinen serverseitigen Filter** nach Author oder State. Der Client muss selbst nach `user.login` und `state: PENDING` filtern.
- **Call 3** — Es gibt **keinen direkten REST-Endpunkt**, der einen Kommentar-Thread zu einem **bereits existierenden** Pending Review hinzufügt. `pulls/create-review-comment` erstellt einen eigenständigen Kommentar (nicht an ein Review gebunden). `pulls/create-review` erstellt ein **neues** Review mit Kommentaren, kann aber keine Kommentare an ein bestehendes Pending Review anhängen.

### Fazit

Die REST API deckt Call 3 (`addPullRequestReviewThread`) nicht ab — es fehlt die Möglichkeit, einen Kommentar an ein bestehendes Pending Review zu binden. Das ist der Grund, warum der GitHub MCP Server für diese Operation GraphQL statt REST verwendet.
