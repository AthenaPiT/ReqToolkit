# ReqTookit — Lightweight Requirements Management

ReqTookit is a single-file, client-side requirements management web app built for

automotive / embedded systems engineering workflows (ISO 26262-style ASIL

decomposition). It ships as one self-contained `index.html` with no build step,

no backend and no database, and deploys as static assets to Cloudflare Pages.



***

## 1. Overview

The application models software/system requirements as a **hierarchical tree**

grouped into **projects**. Each requirement carries safety-relevant attributes

(ASIL, cybersecurity relevance, status, planned dates) plus free-form description

and verification criteria. Users authenticate locally, manage multiple projects,

and inspect requirements in either a **list (3-pane editor)** view or an

interactive **graph** view.

Everything runs in the browser; all data is persisted to the visitor's

`localStorage`. The dark-blue UI is in English and responsive.



***

## 2. Features

### 2.1 Authentication & accounts



* **Login gate** — every visit lands on the sign-in screen (Account + Password).

* **Registration** — "Create an account" to add a new user.

* **Session** — login persists across reloads; "Sign out" ends it.

* **Profile** — top-right avatar opens a panel with:


  * **Upload logo** — choose a local image (≤ 1 MB), stored inline as a data URL and used as the avatar.

  * Edit display name, email; set a new password.

  * Account role and creation date shown.

* Default seeded account: `admin`**&#x20;/&#x20;**`admin123` (change it in Profile).

### 2.2 Projects (multi-project workspace)



* **Project home** — a card grid of all projects, each showing description,

  requirement count and creation date.

* **New Project** — creates an empty project with its own root requirement node.

* **Rename / Delete** — hover a card for the action buttons.

* Entering a project opens its dedicated requirements workspace.

### 2.3 Requirements workspace — List view (3-pane)



* **Left pane — Requirements Tree**


  * Header with live requirement **item count**.

  * Three dropdown filters: **All status**, **All ASIL**, **All types** (combined with the top search box).

  * Each row shows: expand chevron, requirement **ID** (mono, accent blue), title,

    coloured **ASIL chip** (QM/A/B/C/D), and a **status dot + label**.

  * Expand/collapse branches; filters auto-expand matching nodes.

* **Center pane — tabbed editor**


  * Header shows the selected requirement's ID and title.

  * Tabs: **Description** · **Comments (n)** · **Change History (n)**.

  * *Description* tab: **Title**, **Description**, **Verification Criteria** fields.

  * *Comments* tab: add/view threaded comments (author + date + text).

  * *Change History* tab: an automatic audit log of every field edit

    (who, when, old value → new value).

* **Right pane — Properties**


  * Read-only **ID**.

  * **Status** dropdown, **Planned Start / End** date pickers.

  * **Special Characteristic** (Functional / Non-functional),

    **Functional Safety ASIL** (QM / A / B / C / D).

  * **Cybersecurity Relevant** — Yes / No segmented buttons.

  * Actions: **+ Child**, **Save**, **Delete**.

All edits auto-save on change and are written to `localStorage` immediately.

### 2.4 Requirements workspace — Graph view



* Toggle **List / Graph** in the top bar.

* Renders the requirement hierarchy as an interactive tree graph (ECharts).

* Nodes are **colour-coded by status**.

* **Drag to pan, scroll/pinch to zoom** (0.3×–4×); collapse/expand branches by clicking nodes.

* Clicking a node opens a detail drawer (right) summarising the requirement and its comments.

### 2.5 Requirement data model

Each requirement node stores:



| Field                         | Values                                                           |
| ----------------------------- | ---------------------------------------------------------------- |
| `id`                          | Machine identifier, e.g. `REQ-SYS-001`                           |
| `title`                       | Requirement headline                                             |
| `description`                 | Free text                                                        |
| `status`                      | Draft · In Review · Approved · Implemented · Verified · Released |
| `plannedStart` / `plannedEnd` | Dates                                                            |
| `specialChar`                 | Functional · Non-functional                                      |
| `asil`                        | QM · A · B · C · D                                               |
| `cyber`                       | Relevant · Not Relevant                                          |
| `verificationCriteria`        | Free text                                                        |
| `comments[]`                  | `{ author, date, text }`                                         |
| `history[]`                   | `{ date, author, field, old, new }`                              |
| `children[]`                  | Nested requirement nodes                                         |

Root (`isRoot`) and category nodes (`isFolder`) are non-leaf organisational nodes.



***

## 3. Software architecture

### 3.1 Shape



* **One file**: `index.html` contains all markup, CSS and JavaScript inline.

  No bundler, no `npm install`, no framework runtime beyond a single CDN script.

* **Vanilla JS SPA** — plain objects hold state; view functions re-render the DOM.

* **Hash routing** — client-side views are selected by URL fragment:


  * `#/login`

  * `#/projects`

  * `#/project/<projectId>`

    Refreshing or deep-linking lands directly on the right screen.

* **Single CDN dependency**: [ECharts](https://echarts.apache.org) via

  `https://cdn.jsdelivr.net/npm/echarts@5.5.0/dist/echarts.min.js`, used only

  for the Graph view. Everything else is hand-written.

### 3.2 Runtime data store

There is no server. All state lives under one `localStorage` key

(`reqtooki.v2`) as a JSON document:



```
{

&#x20; users:    \[ { username, password, displayName, email, role, createdAt, profileLogo? } ],

&#x20; session:  { username } | null,

&#x20; projects: \[ { id, name, description, createdAt, tree: \<requirement-node> } ]

}
```



* On boot the app loads this object; if absent it seeds a demo

  `admin` account and one brake-control example project.

* Every mutation (login, profile edit, project CRUD, requirement edit, comment,

  field change) calls `save()`, which serialises the whole document back to

  `localStorage`.

* Profile pictures are stored as inline base64 data URLs on the user record.

### 3.3 View / control flow



```
boot → route()

&#x20;      ├─ no session → renderLogin()

&#x20;      ├─ #/projects → renderProjects()

&#x20;      └─ #/project/:id → renderWorkspace()

&#x20;                         ├─ viewMode 'list' → renderListMode()

&#x20;                         │    ├─ renderTreeSidebar()  (filtered tree)

&#x20;                         │    └─ renderCenterAndProps() (tabs + properties)

&#x20;                         └─ viewMode 'graph' → renderGraphMode() (ECharts)
```



* `findInTree(id)` walks the active project's tree to resolve the selected node.

* Editable fields use on-change handlers that call `logChange()` (for the audit

  trail), update the node, `save()`, and re-render the affected panes.

* The ECharts instance is created lazily and disposed on view switch;

  `roam: true` gives pan/zoom out of the box.

### 3.4 UI layers



* CSS custom properties define the dark-blue palette (`--bg-0…3`, `--line`,

  `--accent`, status colours) so the theme can be re-toned in one place.

* Layout is flexbox: top bar fixed, workspace splits into three panes

  (tree / center editor / properties) on desktop and stacks on narrow screens.

* No external fonts; system font stack + monospace for IDs.

### 3.5 Security model (important)

This is a **client-side demonstration app**. Accounts, passwords and project

data all live in the browser's `localStorage` in plain text. It is suitable for

local use, internal demos, and as a UI/requirements-prototype — **not** for

multi-tenant production use. To add real authentication, front the static site

with **Cloudflare Access** (zero-trust SSO) or pair it with a backend.



***

## 4. Files



```
requirements-manager/

├── index.html        # the entire application (markup + CSS + JS)

├── image/

│   └── icon.png      # app logo (square, referenced by index.html)

└── README.md         # this file
```



***

## 5. Run locally

No server is required. Just open `index.html` in a modern browser

(Chrome, Edge, Firefox, Safari). The app loads ECharts from a CDN, so keep an

internet connection for the **Graph** view; all other screens work offline.

Sign in with the seeded demo account `admin`**&#x20;/&#x20;**`admin123` (create your own

account or change the password in Profile).



***

## 6. Deploy to Cloudflare Pages

The site is pure static assets (`index.html` + `image/`), so deployment is

trivial.

### Option A — Dashboard (easiest)



1. Cloudflare dashboard → **Workers & Pages** → **Create** → **Pages** → **Upload assets**.

2. Name the project (e.g. `reqtooki`).

3. Drag the **entire&#x20;**`requirements-manager`**&#x20;folder** (so `image/icon.png` travels

   with `index.html`).

4. Click **Deploy** — you get `https://<project>.pages.dev`.

### Option B — Wrangler CLI



```
npm install -g wrangler

wrangler login

cd requirements-manager

wrangler pages deploy . --project-name reqtooki
```

### Option C — Git integration



1. Push this folder to a GitHub/GitLab repo.

2. In Cloudflare Pages, **Connect to Git** and pick the repo.

3. **Build command:** leave empty. **Build output directory:** `/`.

4. Deploy.

### Custom domain (already on Cloudflare)



1. Pages project → **Custom domains** → **Set up a custom domain**.

2. Enter e.g. `req.yourcompany.com`.

3. Because the zone already lives on Cloudflare, the CNAME/record is created

   automatically — no external DNS changes.

4. HTTPS is provisioned automatically.

> Note: because data is per-browser 
>
> `localStorage`
>
> , each visitor sees their own
> projects. There is no shared server-side database; clearing browser storage
> resets the app.



***

## 7. Seed data & next steps



* The bundled project is an automotive **Brake Control System (BCS)** example

  decomposed into System / Software / Hardware requirements per ISO 26262.

* Replace it by deleting the seed project and creating your own, or by editing

  the `seedTree()` function in `index.html`.

* Natural extensions for a production build: a backend/database (e.g. Cloudflare

  D1 + Workers), real auth via Cloudflare Access, and multi-user sync.