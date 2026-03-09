# REST API Analyse: `add_comment_to_pending_review` nachbauen

## Ausgangslage

Das MCP Tool `add_comment_to_pending_review` (Quelle: `pkg/github/pullrequests.go:1895-2009`) macht **3 sequenzielle GraphQL Calls**, um einen Kommentar-Thread zu einem bestehenden Pending Review auf einem Pull Request hinzuzufuegen.

Die Calls sind sequenziell abhaengig: Call 2 benoetigt den Login aus Call 1, Call 3 benoetigt die Review-ID aus Call 2.

---

## Mapping: GraphQL -> REST API

### Call 1: Viewer Login abrufen

| | GraphQL | REST |
|---|---|---|
| **Operation** | `query { viewer { login } }` | `GET /user` |
| **operationId** | -- | `users/get-authenticated` |
| **Abdeckung** | -- | **Voll** |

`GET /user` liefert das `login`-Feld des authentifizierten Benutzers. Vollwertiger Ersatz.

---

### Call 2: Letztes Pending Review des Viewers auf dem PR finden

| | GraphQL | REST |
|---|---|---|
| **Operation** | `reviews(first: 1, author: $author)` mit State-Check | `GET /repos/{owner}/{repo}/pulls/{pull_number}/reviews` |
| **operationId** | -- | `pulls/list-reviews` |
| **Abdeckung** | -- | **Teilweise** |

**Einschraenkungen der REST API:**
- Kein serverseitiger Filter nach `author` oder `state`
- Client muss alle Reviews abrufen (paginiert) und selbst filtern nach:
  - `user.login == <viewer_login>`
  - `state == "PENDING"`
- Weniger effizient als der GraphQL-Call, aber funktional aequivalent

---

### Call 3: Review-Kommentar-Thread zum Pending Review hinzufuegen

| | GraphQL | REST |
|---|---|---|
| **Operation** | `addPullRequestReviewThread(input: $input)` | **Kein direkter Endpoint** |
| **Abdeckung** | -- | **Nicht abgedeckt** |

Der GraphQL-Call `addPullRequestReviewThread` fuegt einen Kommentar zu einem **bereits existierenden** Pending Review hinzu, unter Angabe der `pullRequestReviewID`.

#### Untersuchte REST-Kandidaten

**Kandidat A: `pulls/create-review-comment`**
- Endpoint: `POST /repos/{owner}/{repo}/pulls/{pull_number}/comments`
- Parameter: `body`, `commit_id` (required), `path` (required), `line`, `side`, `start_line`, `start_side`, `subject_type`, `in_reply_to`
- Problem: **Kein Parameter** um den Kommentar an ein bestehendes Pending Review zu binden
- Erstellt einen eigenstaendigen, sofort sichtbaren Kommentar

**Kandidat B: `pulls/create-review`**
- Endpoint: `POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews`
- Parameter: `commit_id`, `body`, `event`, `comments[]` (mit `path`, `body`, `line`, `side`, `start_line`, `start_side`)
- Problem: Erstellt immer ein **neues** Review
- Kann nicht an ein bestehendes Pending Review anhaengen
- Wenn `event` leer gelassen wird: neues Pending Review, aber eben ein separates

---

### Parameter-Vergleich: GraphQL vs. REST-Kandidaten

| GraphQL Parameter | `pulls/create-review-comment` | `pulls/create-review` (comments[]) |
|---|---|---|
| `path` | `path` (required) | `path` (required) |
| `body` | `body` (required) | `body` (required) |
| `subjectType` (LINE/FILE) | `subject_type` (line/file) | -- |
| `line` | `line` | `line` |
| `side` (LEFT/RIGHT) | `side` (LEFT/RIGHT) | `side` |
| `startLine` | `start_line` | `start_line` |
| `startSide` | `start_side` | `start_side` |
| `pullRequestReviewID` | **Fehlt** | **N/A** (immer neues Review) |

---

## Umfassende Suche in der OpenAPI Spec

Die gesamte OpenAPI Spec (226.120 Zeilen, Version `2022-11-28`) wurde systematisch durchsucht:

- **`pull_request_review_id`** existiert nur als **Response-Feld** im Schema `pull-request-review-comment` (Zeile 79592). Es zeigt an, zu welchem Review ein Kommentar gehoert, ist aber **kein Input-Parameter** bei der Erstellung.
- **`/repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}/comments`** hat nur **GET** (Zeile 43048). Es gibt kein POST.
- Es existiert kein Endpoint mit Stichworten wie "add review", "append review" etc., der Kommentare zu bestehenden Reviews hinzufuegt.
- Der Pfad-Parameter `review_id` wird nur in Endpoints verwendet, die Reviews lesen, loeschen, submiten oder dismissen -- nie zum Hinzufuegen von Kommentaren.

**Ergebnis:** Es gibt in der gesamten REST API keinen Endpoint, um einen Kommentar an ein bestehendes Pending Review zu binden.

---

## Alle relevanten REST Endpoints

| # | Methode | Pfad | operationId | Zweck |
|---|---|---|---|---|
| 1 | `GET` | `/user` | `users/get-authenticated` | Authentifizierten User abrufen |
| 2 | `GET` | `.../pulls/{n}/reviews` | `pulls/list-reviews` | Alle Reviews eines PRs listen |
| 3 | `POST` | `.../pulls/{n}/reviews` | `pulls/create-review` | Neues Review (mit Kommentaren) erstellen |
| 4 | `GET` | `.../pulls/{n}/reviews/{id}` | `pulls/get-review` | Einzelnes Review lesen |
| 5 | `PUT` | `.../pulls/{n}/reviews/{id}` | `pulls/update-review` | Review-Body aktualisieren |
| 6 | `DELETE` | `.../pulls/{n}/reviews/{id}` | `pulls/delete-pending-review` | Pending Review loeschen |
| 7 | `GET` | `.../pulls/{n}/reviews/{id}/comments` | `pulls/list-comments-for-review` | Kommentare eines Reviews lesen |
| 8 | `POST` | `.../pulls/{n}/reviews/{id}/events` | `pulls/submit-review` | Pending Review einreichen |
| 9 | `POST` | `.../pulls/{n}/comments` | `pulls/create-review-comment` | Eigenstaendigen Diff-Kommentar erstellen |
| 10 | `GET` | `.../pulls/{n}/comments` | `pulls/list-review-comments` | Alle Review-Kommentare eines PRs listen |

---

## Alternative REST-Workflows

### Alternative A: Neues Pending Review mit Kommentar erstellen

Dieser Workflow ist geeignet, wenn **noch kein Pending Review existiert**. Er erstellt ein neues Pending Review und fuegt den Kommentar direkt beim Erstellen hinzu.

#### Ablauf

```
Schritt 1: GET /user
   -> Ergebnis: viewer_login

Schritt 2: GET /repos/{owner}/{repo}/pulls/{pull_number}/reviews
   -> Alle Reviews abrufen
   -> Client-seitig filtern: user.login == viewer_login AND state == "PENDING"
   -> Ergebnis: Kein Pending Review gefunden

Schritt 3: POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews
   Body: {
     "event": null,          // <- kein Event = PENDING State
     "comments": [
       {
         "path": "src/main.ts",
         "line": 42,
         "side": "RIGHT",
         "body": "Hier sollte eine Fehlerbehandlung rein."
       }
     ]
   }
   -> Ergebnis: Neues Pending Review mit dem Kommentar erstellt
```

#### Vorteile
- Einfacher, atomarer Aufruf
- Kein Datenverlust-Risiko
- Kann mehrere Kommentare auf einmal hinzufuegen (`comments` ist ein Array)

#### Einschraenkungen
- Funktioniert nur, wenn **kein** Pending Review existiert
- `comments[]` im `pulls/create-review` Endpoint hat keinen `subject_type` Parameter (im Gegensatz zu `pulls/create-review-comment`), daher sind File-Level-Kommentare moeglicherweise nicht unterstuetzt
- Pro Benutzer kann nur ein Pending Review gleichzeitig existieren -- ein zweites erstellen fuehrt zu einem Fehler

---

### Alternative B: Bestehendes Pending Review loeschen und neu erstellen

Dieser Workflow ist der einzige REST-basierte Weg, um einen Kommentar zu einem **bereits existierenden** Pending Review hinzuzufuegen. Er loescht das bestehende Review, liest dessen Kommentare aus, und erstellt ein neues mit allen alten plus dem neuen Kommentar.

#### Ablauf

```
Schritt 1: GET /user
   -> Ergebnis: viewer_login

Schritt 2: GET /repos/{owner}/{repo}/pulls/{pull_number}/reviews
   -> Alle Reviews abrufen
   -> Client-seitig filtern: user.login == viewer_login AND state == "PENDING"
   -> Ergebnis: pending_review gefunden (id, body)

Schritt 3: GET /repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}/comments
   -> Alle bestehenden Kommentare des Pending Reviews abrufen (paginiert!)
   -> Ergebnis: existing_comments[] (jeweils path, line, side, start_line, start_side, body)

Schritt 4: DELETE /repos/{owner}/{repo}/pulls/{pull_number}/reviews/{review_id}
   -> Pending Review loeschen (nur PENDING Reviews koennen geloescht werden)
   -> Ergebnis: 200 OK, gibt das geloeschte Review zurueck

Schritt 5: POST /repos/{owner}/{repo}/pulls/{pull_number}/reviews
   Body: {
     "event": null,
     "body": "<urspruenglicher review body>",
     "comments": [
       // Alle bestehenden Kommentare aus Schritt 3:
       { "path": "src/auth.ts", "line": 10, "side": "RIGHT", "body": "Alter Kommentar 1" },
       { "path": "src/db.ts",   "line": 55, "side": "LEFT",  "body": "Alter Kommentar 2" },
       // Plus der neue Kommentar:
       { "path": "src/main.ts", "line": 42, "side": "RIGHT", "body": "Neuer Kommentar" }
     ]
   }
   -> Ergebnis: Neues Pending Review mit allen Kommentaren erstellt
```

#### Vorteile
- Ermoeglicht es, Kommentare zu einem bestehenden Pending Review "hinzuzufuegen" (via Neuerstellen)
- Vollstaendig ueber REST API realisierbar

#### Risiken und Einschraenkungen
- **Race Condition:** Zwischen Schritt 4 (DELETE) und Schritt 5 (POST) koennte ein anderer Prozess eingreifen
- **Datenverlust:** Wenn Schritt 5 fehlschlaegt, sind die bestehenden Kommentare verloren
- **Kommentar-IDs aendern sich:** Alle Kommentare bekommen neue IDs -- externe Referenzen brechen
- **Review-Metadaten:** Der Review-Body muss manuell uebernommen werden
- **Paginierung:** Bei vielen Kommentaren muss Schritt 3 paginiert werden
- **Kein `subject_type`:** Das `comments[]` Array beim Erstellen eines Reviews unterstuetzt keinen `subject_type` Parameter, File-Level-Kommentare sind moeglicherweise nicht uebertragbar
- **Komplexe Fehlerbehandlung noetig:** Retry-Logik fuer Schritt 5 ist kritisch

---

## Fazit

| Szenario | Empfehlung |
|---|---|
| Neues Review + erster Kommentar | **Alternative A** -- `pulls/create-review` mit `comments[]` |
| Kommentar zu bestehendem Pending Review | **Nicht sauber via REST moeglich** -- GraphQL (`addPullRequestReviewThread`) ist die einzig zuverlaessige Loesung; Alternative B ist ein fragiler Workaround |

Die REST API deckt den zentralen Use Case von `add_comment_to_pending_review` -- das nachtraegliche Hinzufuegen eines Kommentars zu einem bestehenden Pending Review -- **nicht ab**. Das ist der Grund, warum der GitHub MCP Server fuer diese Operation GraphQL statt REST verwendet.

Fuer einen reinen REST-basierten Nachbau muss entweder:
1. Die Semantik geaendert werden (immer neues Pending Review erstellen statt an bestehendes anhaengen), oder
2. Der riskante Delete-and-Recreate-Workaround (Alternative B) akzeptiert werden.
