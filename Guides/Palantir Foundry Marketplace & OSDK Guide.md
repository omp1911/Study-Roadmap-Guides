# OSDK + Marketplace: Hands-On Practice Guide

**Starting point:** you already have a Workshop application in Foundry, built on some Ontology object types.
**Goal today:** rebuild one small piece of that Workshop app as your own React app, running on `localhost`, reading and writing the same Ontology.
**Goal after that:** package something into a Marketplace product and install it, or be able to explain the process convincingly.

Time needed: ~90 minutes for Part A, ~60 minutes for Part B.

---

# PART A — Local OSDK app (do this first)

## Step 0: Note what you already have

Open your Workshop app and write down:

- The **Ontology object type** it displays (e.g. `Flight`, `Order`, `Equipment`)
- Its **API name** — not the display name. Find it in Ontology Manager → object type → API name
- 2–3 **property API names** you want to show in a table
- One **Action type** the app uses (e.g. "Update status", "Create record")

You will reuse exactly these in your OSDK app. Using the same entities is the point — you'll see the two apps hitting the same data.

If your Workshop app has no Action, create a trivial one in Ontology Manager (e.g. modify a single string property). You want at least one write path.

---

## Step 1: Check your local machine

```bash
node --version   # must be 18 or higher
npm --version
```

If Node is old, install Node 20 LTS (or use `nvm`).

---

## Step 2: Create the application in Developer Console

Developer Console is a separate app in the Foundry portal from Workshop. If you don't see it, it needs enabling at the enrollment level — ask your admin.

1. Open **Developer Console** → **+ New application**
2. **Basic information**: give it a name and an icon. The icon appears on the OAuth consent screen users see.
3. **Application type**: choose **Client-facing application** (this creates a *public* OAuth client — correct for a browser app). Do **not** also tick Backend service, or web hosting will be unavailable later.
4. **Redirect URL**: `http://localhost:8080/auth/callback`
5. **Resources** page → **Yes, generate an Ontology SDK**
   - Select your Ontology
   - Select the **object types** and **action types** from Step 0
   - ⚠️ The Ontology binding **cannot be changed** after the SDK is generated. Object/action selections *can* be changed later by regenerating.
6. Finish. You land on the application **Overview** page.

**From the Overview page, copy these four values into a scratch file — you need all of them:**

| Value | Where |
|---|---|
| Client ID | Overview page |
| Foundry URL | e.g. `https://yourorg.palantirfoundry.com` |
| npm registry URL | Overview page, under SDK install instructions |
| Ontology RID | Ontology Manager → your ontology → Ontology configuration → Ontology metadata |

---

## Step 3: Fix CORS *before* you write any code

This is where most first attempts fail with a confusing browser error.

**Option A (preferred):** Control Panel → CORS policy → add origin `http://localhost:8080`.

**Option B (if you don't have Control Panel rights — common on Developer Tier / restricted enrollments):**
Change the redirect URL in Developer Console to `https://localhost:8080/auth/callback` (note **https**) and run your dev server with HTTPS enabled. In Vite:

```js
// vite.config.js
export default {
  server: { port: 8080, https: true }
}
```

You'll get a browser certificate warning on first load. Click through it.

---

## Step 4: Scaffold the project

**Easiest path:** Developer Console → your app → **Start developing** tab. It shows a generated scaffold command and install instructions pre-filled with *your* client ID, registry URL, and package name. Copy-paste from there — it's always current and correct for your enrollment.

That scaffold (built on `@osdk/create-app`) gives you a Vite + React + TypeScript project with auth already wired.

**Manual path**, if you prefer to see every moving part:

```bash
npm create vite@latest my-osdk-app -- --template react-ts
cd my-osdk-app
```

Create a `.npmrc` in the project root — this lets npm pull your private SDK package from Foundry:

```
//<REGISTRY-URL-FROM-OVERVIEW-PAGE>:_authToken=${FOUNDRY_TOKEN}
<PACKAGE-NAME>:registry=https://<REGISTRY-URL-FROM-OVERVIEW-PAGE>
```

Export the token in your shell (generate a scoped token from the Developer Console app page):

```bash
export FOUNDRY_TOKEN=<your-token>
```

⚠️ Never commit `FOUNDRY_TOKEN` or the token value to git. Add `.env` to `.gitignore`.

Install:

```bash
npm install @osdk/client @osdk/oauth
npm install <YOUR-GENERATED-PACKAGE-NAME>
```

---

## Step 5: Wire up the client

Create `src/foundryClient.ts`:

```ts
import { createClient } from "@osdk/client";
import { createPublicOauthClient } from "@osdk/oauth";
import { $ontologyRid } from "<YOUR-GENERATED-PACKAGE-NAME>";

const FOUNDRY_URL = import.meta.env.VITE_FOUNDRY_URL;
const CLIENT_ID   = import.meta.env.VITE_CLIENT_ID;
const REDIRECT    = import.meta.env.VITE_REDIRECT_URL; // http://localhost:8080/auth/callback

export const auth = createPublicOauthClient(CLIENT_ID, FOUNDRY_URL, REDIRECT);
export const client = createClient(FOUNDRY_URL, $ontologyRid, auth);
```

`.env.local` (gitignored):

```
VITE_FOUNDRY_URL=https://yourorg.palantirfoundry.com
VITE_CLIENT_ID=xxxxxxxx
VITE_REDIRECT_URL=http://localhost:8080/auth/callback
```

> If `$ontologyRid` isn't exported from your package, just paste the Ontology RID string directly.

---

## Step 6: Login flow

Create `src/AuthCallback.tsx` and route `/auth/callback` to it, or handle it inline. The minimal pattern:

```tsx
import { useEffect, useState } from "react";
import { auth } from "./foundryClient";

export function useLogin() {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    if (auth.getTokenOrUndefined()) {
      setReady(true);
      return;
    }
    auth.refresh()
      .then(() => setReady(true))
      .catch(() => auth.signIn()); // redirects to Foundry login
  }, []);

  return ready;
}
```

**Checkpoint 1:** run `npm run dev`, open `http://localhost:8080`. You should be bounced to your Foundry login page and then back. If you're not, the problem is redirect URL mismatch or CORS — nothing else.

---

## Step 7: Read data (the payoff moment)

```tsx
import { useEffect, useState } from "react";
import { client } from "./foundryClient";
import { MyObject } from "<YOUR-GENERATED-PACKAGE-NAME>";
import type { Osdk } from "@osdk/client";

export function ObjectTable() {
  const [rows, setRows] = useState<Osdk.Instance<typeof MyObject>[]>([]);

  useEffect(() => {
    client(MyObject)
      .fetchPage({ $pageSize: 25 })
      .then(res => setRows(res.data));
  }, []);

  return (
    <table>
      <tbody>
        {rows.map(r => (
          <tr key={r.$primaryKey}>
            <td>{r.someProperty}</td>
            <td>{r.anotherProperty}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Checkpoint 2:** the same records your Workshop app shows now render in your own table. That's the whole concept proven.

---

## Step 8: Filter, link, aggregate

```ts
// filter
client(MyObject).where({ status: { $startsWith: "OPEN" } }).fetchPage({ $pageSize: 20 });

// sort
client(MyObject).fetchPage({ $pageSize: 20, $orderBy: { createdAt: "desc" } });

// dates are plain ISO 8601 strings — no special type
client(MyObject).where({ createdAt: { $gt: "2025-01-01T00:00:00Z" } });

// traverse a link
const obj = await client(MyObject).fetchOne("pk-123");
const related = await obj.$link.relatedThings.fetchPage({ $pageSize: 50 });

// aggregate
const agg = await client(MyObject).aggregate({
  $select: { $count: "unordered" },
  $groupBy: { status: "exact" },
});
```

---

## Step 9: Write data (Action)

```ts
import { updateStatus } from "<YOUR-GENERATED-PACKAGE-NAME>";

await client(updateStatus).applyAction(
  { myObject: "pk-123", newStatus: "CLOSED" },
  { $returnEdits: true }
);

// validate without executing
await client(updateStatus).applyAction({ ...params }, { $validateOnly: true });

// bulk
await client(updateStatus).batchApplyAction([{ ...p1 }, { ...p2 }]);
```

**Checkpoint 3:** apply the action from your app, then refresh your **Workshop** app. The change appears there. Same Ontology, two frontends. This is the thing to demo to your team.

---

## Step 10 (optional): Host it on Foundry

Developer Console supports hosting static frontends on a Foundry subdomain — no external hosting needed. It serves static assets only (like GitHub Pages); no Node server or SSR.

1. `npm run build`
2. Zip the **contents** of `dist/` — not the `dist` folder itself
3. Upload in Developer Console → Web hosting
4. Add `https://<subdomain>.<enrollment>.palantirfoundry.com/auth/callback` as an additional redirect URL in Developer Console

---

## Part A troubleshooting

| Symptom | Cause |
|---|---|
| CORS error in browser console | `localhost:8080` not in CORS policy → use the https fallback |
| Redirect loop / "invalid redirect_uri" | Redirect URL in code ≠ redirect URL in Developer Console. Must match exactly, including `/auth/callback` |
| `401 Unauthorized` on npm install | `FOUNDRY_TOKEN` not exported, or token expired |
| Object type not found in package | Not selected on the Resources page → add it and regenerate the SDK |
| Property is `undefined` | Using display name instead of API name |
| Empty result set, no error | Your user lacks permission on that object type, or the app's scope excludes it |
| TypeScript type mismatch | You need `Osdk.Instance<MyObject>`, not `MyObject` |

---

# PART B — Marketplace

## The concept in one paragraph

Marketplace is how Foundry content moves between environments — dev to prod, or one team to another. You build a **product** in **Foundry DevOps**: it has **outputs** (the resources that get installed — datasets, pipelines, ontology types, Workshop apps, functions, Developer Console apps) and **inputs** (dependencies the installer must map to their own local resources, typically the raw source datasets). You version the product and publish it to a **store**. Someone else opens Marketplace, picks the product, maps the inputs, and installs. Their environment now has a working copy.

Analogy: DevOps is `npm publish`, Marketplace is `npm install`, inputs are peer dependencies you must supply yourself.

---

## Vocabulary (memorise these five)

| Term | Meaning |
|---|---|
| **Product** | A packaged bundle of Foundry resources |
| **Store** | A collection of products. Local (lives in a Project/folder) or remote (shared across enrollments) |
| **Output** | Content that gets created on install |
| **Input** | A dependency the installer must map to an existing resource |
| **Installation** | A live instance of an installed product, which can be upgraded |

---

## Practice B1: Install something (30 min, low risk)

Do this first — it takes no permissions beyond Viewer.

1. Open **Marketplace** from the Foundry portal
2. Search for a tutorial product (e.g. "Marketplace Getting Started Tutorial")
3. Select **Create new installations**, tick the product cards, confirm
4. Map any required inputs
5. Choose the target Project
6. Watch the installation job run, then open the installed resources

Note as you go: which screen asks for inputs, what the dependency graph looks like, what appears in the target Project afterwards.

---

## Practice B2: Build and install your own product (the real exercise)

This is the version to do if **Foundry DevOps** appears in your portal.

### 1. Prepare your Workshop app for packaging
Workshop settings → enable **Installation configuration**. Also run the **packaging error linter** before you try to package — it catches unsupported constructs early. (Static/object-backed scenarios are not supported for packaging.)

### 2. Create the product
Foundry DevOps → **New product** → give it a name → choose or create a **local store** (a store living in a Project you own).

### 3. Add outputs
- Add your Workshop application (**Add files** → navigate to it in the filesystem)
- Add the Ontology object types and action types it uses
- Optionally add your Developer Console OSDK app from Part A

DevOps auto-detects dependencies — watch what it pulls in. This is the most instructive moment: you see the true dependency graph of something you built.

### 4. Define inputs
Your source dataset(s) become inputs. The installer will point these at their own data.

### 5. Version and publish
Create a version → publish to the store. Some stores require an approver different from the author.

### 6. Install into a different Project
Marketplace → find your product → install into a fresh Project. Map the input dataset. Verify the installed Workshop app opens and shows data.

### 7. Deliberately break it (best learning)
Rename an object type's **API name** in your source Ontology, republish, reinstall. Watch the OSDK app fail. This teaches the single biggest real-world gotcha, below.

---

## The gotchas that matter in interviews and in production

**1. API names are not remapped on install.**
OSDK references Ontology entities by API name. Entities must have **identical API names** in source and destination Ontologies or the installed application breaks. Website URLs and client IDs *are* auto-mapped; API names are not. This is the #1 cause of "it worked in dev, broke in prod."

**2. Developer Console apps package cleanly.**
Packaging a Developer Console app bundles its metadata, OAuth client config (client type, grant types, redirect URLs), and resource restrictions. Installing auto-creates the Developer Console resource and configures OAuth on the target. Two packaging styles:
- **Application packaging** — ships the deployed website. Use for production deployment.
- **Source code packaging** — ships the code repository as a template. Use for bootcamps, standardised starters, or when the target needs its own Dev Console app.

**3. OSDK app packaging constraints.**
- Application package name must not contain `@` or a trailing `/sdk`
- OSDK version in `package.json` should be `latest` or `0.1.0` for the template to work out of the box
- Auto-mapping of URLs/client IDs only works if the app reads them from **meta tags in `index.html`**, not from bundled code. Apps scaffolded with `@osdk/create-app` v2.1.3+ or from a Foundry VS Code workspace already do this. Older apps need the meta tags added manually.

**4. Functions lose source visibility.**
TypeScript V1 functions package as executables with no viewable source after install. Python and TypeScript V2 functions do include viewable source, but still can't be edited post-install in production mode.

**5. Permissions vocabulary.**
- `marketplace:read-local-marketplace` — see a local store (Viewer role)
- `marketplace:install-from-local-marketplace` — install products
- `marketplace:use-resource-as-input` — needed on every resource used as an input
- `marketplace:finalize-block-set` — approve/publish a version (store owners and editors)

**6. Installation lifecycle.**
Installations track a **release channel** and can be set to auto-upgrade when new versions publish, with maintenance windows. That's the operational answer to "how do you push a fix to 20 deployments."

---

## If someone asks you about Marketplace on day one

A safe, accurate 60-second answer:

> "Marketplace is the delivery layer. We build a product in Foundry DevOps that bundles the resources for a use case — ontology types, pipelines, Workshop apps, Dev Console apps — and declares which upstream datasets are inputs. We version it and publish to a store. The target environment installs it and maps inputs to their local data. The main thing to watch is that OSDK references ontology entities by API name and those aren't remapped on install, so API names have to be kept consistent across environments. Installations track a release channel, so upgrades can be pushed centrally."

---

## Suggested order this week

| Day | Task | Outcome |
|---|---|---|
| 1 | Part A steps 1–7 | Local app reading your Ontology |
| 2 | Part A steps 8–9 | Filters, links, one working Action |
| 3 | Part A step 10 | Hosted on Foundry, shareable link |
| 4 | Practice B1 | Installed a Marketplace product |
| 5 | Practice B2 | Built, published, installed your own product |

If DevOps isn't available to you, do B1 and read this Part B twice — the vocabulary and the API-name gotcha are what people actually get asked about.
