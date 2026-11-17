# Modernisation Plan

Goal: bring Realms of War up to date, make it run end to end, and deploy it so it can serve as a working full-stack portfolio piece.

The plan is ordered so every phase leaves the app in a working state and can ship as its own pull request. Sizes are rough: **S** is an evening, **M** is a weekend, **L** is a few sessions.

## Where things stand (November 2026)

**Done**

- **Phase 1: Make it run** (#20, #21). The Firebase route is mounted, `getUsersDetails` returns every player, the event handler arguments are fixed, the connection string is no longer logged, and the `package.json` links point at `jessicaleaturner`.
- **Phase 2: Dependency and toolchain upgrade** (#22 to #25). The app runs on Node 26 with native ES modules and `node --watch`, Express 5, Mongoose 9 with `async/await`, and the modular Firebase SDK. Axios comes from npm, and unused client libraries have been removed. `npm audit` reports 0 vulnerabilities.
- `index.js` exports the app and only calls `listen()` when run directly, so tests can import it.

**Decisions since the original plan**

- **Firebase is used for sign-in only.** Firebase Authentication handles usernames, emails and passwords. The original design added Firestore as a layer between the Firebase account and the player data in MongoDB. That layer is gone: the Firebase user ID now links the account directly to the player document in MongoDB, which holds all game data such as inventory, units and gold.
- **Players are identified by a `uuid` field, not `firebase_token`.** The old name was misleading, because it held the Firebase user ID, not a token. `uuid` holds the Firebase Auth user ID (`user.uid`), so a signed-in user maps directly to their player document. `src/data/mockUsers.json` already uses this shape.
- **The client is split into services and components.** Auth and data fetching no longer render UI, and UI code no longer talks to Firebase. Components build elements with DOM methods instead of `innerHTML` strings, so player data such as usernames can't inject HTML.
- **Materialize is replaced with native HTML and the project's own CSS, not another framework.** Materialize 1.0.0 is unmaintained, and Pico CSS, the obvious lightweight replacement, was archived in October 2026. The app only needs a few styled elements and two widgets that browsers now provide natively (`<dialog>` and `<details>`), so a single hand-written stylesheet is less to maintain than any dependency.
- **The server moves to TypeScript.** Express works well with TypeScript, and Node 26 can run `.ts` files directly, so the server needs no build step.
- **Tests sit next to the code they test** (`eventController.ts` and `eventController.test.ts` in the same folder). Every module gets tests.

**Still outstanding from Phases 1 and 2** (folded into the phases below)

- The players table is still closed with `</table` (missing `>`), and the View and Attack buttons still use `onclick='#(...)'` placeholders. Both go away in Phase 3.
- The event handlers other than `deleteEvent` are still stubs that only `console.log`.

## Phase 3: Restructure the client (S to M)

`auth.js` currently does four jobs: it initialises Firebase, handles the sign-up, login and logout forms, fetches data, and renders the players and quests lists through `fillUsers` and `setupQuests`. `profile.js` and `questlog.js` build HTML strings and assign them to `innerHTML`, which runs any markup in a player's username (cross-site scripting). `index.js` is entirely commented out but is still loaded by `index.html`.

Split the client so each file has one job:

```
client/
  main.js              the only <script> tag; connects services to components
  services/
    firebase.js        fetches the config, initialises Firebase, wraps auth calls
    api.js             replaces shared.js (getData, postData)
  components/
    Button.js
    UserTable.js
    QuestTable.js
    SignupForm.js
    LoginForm.js
    LogoutButton.js
```

- **Services** don't touch the DOM. `firebase.js` exports `onAuthChange`, `signUp`, `signIn` and `signOut`. Firebase is initialised asynchronously from the `/firebase` config, so each export waits on an internal "ready" promise. Callers never see an uninitialised `auth`.
- **Components** are functions that take data and callbacks and return an element, for example `UserTable(currentUser, users, { onAttack, onView })` or `SignupForm({ onSubmit })`. They don't import services, and they don't choose where they're mounted. They build elements with `createElement` and set player data with `textContent`, never `innerHTML`.
- **`main.js`** is the only file that knows about both sides. It subscribes to auth changes, fetches data, mounts components (`container.replaceChildren(UserTable(...))`) and opens and closes the Materialize modals.
- Imports use the full filename (`./Button.js`), because the browser resolves them without a bundler.
- Delete `index.js`, `auth.js`, `profile.js`, `questlog.js` and `shared.js` once they're replaced, and change `index.html` to load only `main.js`.
- Keep behaviour the same apart from the rendering bugs that this fixes by construction: the `</table` typo, the `onclick='#(...)'` placeholders and the signed-out `fillUsers(response.data)` call with its arguments in the wrong order. Sign-up stays broken until Phase 5.
- Keep Materialize for now. It stays loaded as the global `M` and is removed in Phase 4.

**Done when:** `index.html` loads one script, no client code assigns player data to `innerHTML`, no component imports a service, and the app behaves as it did before.

## Phase 4: Replace Materialize with native elements and own CSS (S to M)

Materialize 1.0.0 is no longer maintained. The app uses little of it: button, colour and form-field classes, one navbar, and two JavaScript widgets, `M.Modal` (five modals) and `M.Collapsible` (four panels). Browsers now provide both widgets natively, so Materialize can be removed entirely rather than swapped for another framework that could also be abandoned.

- Replace the modals with `<dialog>`. `main.js` calls `dialog.showModal()` and `dialog.close()` instead of `M.Modal.getInstance(...)`. The browser provides the backdrop, closing with Esc, and focus handling. Buttons that open a modal get a click handler instead of the `modal-trigger` class and `data-target` attribute.
- Materialize modals also close when the backdrop is clicked, but `<dialog>` doesn't by default. Add `closedby="any"` to each dialog, with a small backdrop click handler as a fallback for browsers that don't support it yet.
- Replace the collapsibles with `<details>` and `<summary>`, which need no JavaScript.
- Add one stylesheet, `client/styles.css`:
  - Custom properties for colours, spacing and corner radius at the top (for example `--colour-primary`), so the look can be changed in one place.
  - Styles on elements rather than classes where possible: `button`, `input`, `label`, `table`, `nav`, `dialog`, `details` and `summary`. Components then need few or no class names, so `Button` no longer sets a class string.
  - A simple layout with grid or flexbox that works at phone width.
  - Visible `:focus-visible` styles, so the forms and buttons work with a keyboard.
- Delete `materialize.min.css` and `materialize.min.js`, remove their tags from `index.html`, and remove every Materialize class (`btn`, `z-depth-0`, `lighten-2`, `input-field`, `collapsible-header` and so on).
- Copying Materialize's look exactly isn't the goal. Keep the layout recognisable and the colours close to the current blue.

**Done when:** nothing references `M` or a Materialize class, the Materialize files are deleted, every modal and panel opens and closes with both mouse and keyboard, and the page works at phone width.

## Phase 5: Switch players to `uuid` (S)

Sign-up is currently half broken. After Firebase creates the account, the client calls `newUserData.slice()`, but `newUserData` is defined in `profile.js` and never exported. It's also an object, so it has no `slice()`. The error is swallowed by an empty `catch`, so the player document is never created in MongoDB. Phase 3 moves this code into `main.js` without fixing it. Fix it here while renaming the field.

- In `userModel.js`, rename `firebase_token` to `uuid`, keep it `required` and make it `unique` with an index.
- Rename the routes from `/user/:token` to `/user/:uuid`, and `getUserByToken` to `getUserByUuid`. Update every `{ firebase_token: ... }` query.
- On the client:
  - Export the new-player defaults from one place, set `uuid` from the signed-in user's `uid` on sign-up, and post them to `/user`.
  - Compare players with `user.uuid !== currentUser.uid` in `UserTable`. At the moment it compares `firebase_token` against a Firebase user object, which never matches.
  - Pass `uuid` to the `onAttack` and `onView` callbacks.
- Include `uuid` in the `/userDetails` response, because the View and Attack buttons need it to identify a player.
- Remove `FIREBASE_DATABASE_URL` and `FIREBASE_STORAGE_BUCKET` from `fireBaseRoute.js` and `projectSetup.md`, because Realtime Database and Storage aren't used.
- Add an optional `npm run database -- --mock-users` flag to `defaultDatabaseSetup.js` that seeds `mockUsers.json` for local development. Decide whether `mockUsers.json` is committed. It's currently untracked, and an earlier commit (`ae05dbc`) removed mock user data.
- Existing documents created with `firebase_token` can be migrated with a one-off `updateMany` using `$rename`, or deleted if they're only test data.

**Done when:** signing up creates a player document keyed by `uuid`, the players list hides the signed-in player, and nothing in the code refers to `firebase_token`.

## Phase 6: Show errors to the player (S)

Firebase rejects bad sign-ups and logins (for example, a password under 6 characters), but nothing catches the errors from `signUp` or `signIn`. The error only reaches the browser console, and the modal just sits there.

- Give `SignupForm` and `LoginForm` an error area, and keep the modal open when something fails.
- Map Firebase error codes to plain messages in a small, testable function in `client/services/authErrors.js`:

  | Code | Message |
  | --- | --- |
  | `auth/weak-password` | Password must be at least 6 characters. |
  | `auth/email-already-in-use` | An account with that email already exists. |
  | `auth/invalid-email` | That email address isn't valid. |
  | `auth/invalid-credential` | Email or password is incorrect. |
  | `auth/too-many-requests` | Too many attempts. Try again in a few minutes. |
  | `auth/network-request-failed` | Couldn't reach the server. Check your connection. |
  | anything else | Something went wrong. Please try again. |

- Add `minlength="6"` to the sign-up password field so the browser catches the most common case before the request is sent.
- Disable the submit button while a request is in flight, so double clicks don't create two requests.
- If Firebase creates the account but saving the player to MongoDB fails, tell the player rather than leaving them signed in with no player data.
- On the server, return real status codes and a consistent `{ error: "message" }` body instead of `res.send(err)` with a 200. Express 5 passes rejected `async` handlers to error middleware, so most `try/catch` blocks can be replaced by one error handler in `index.js`. Use 404 for a missing player or quest, 400 for validation errors, and 500 for anything else, and never send raw Mongoose errors to the client.
- Show API errors (such as a failed quest) in the client in the same way.

**Done when:** each error in the table shows a readable message in the modal, and no API route returns an error with a 200 status.

## Phase 7: Migrate the server to TypeScript (M)

TypeScript works well with Express. Node 26 strips types natively, so `node --watch index.ts` runs TypeScript directly, with no Babel, `ts-node` or build step. Type stripping doesn't check types, so `tsc` runs separately as a check.

- Add `typescript`, `@types/node`, `@types/express` and `@types/supertest` as dev dependencies. Mongoose ships its own types.
- Add a `tsconfig.json` set up for running files directly with Node:
  - `"noEmit": true`, `"strict": true`, `"module": "nodenext"`, `"target": "esnext"`
  - `"allowImportingTsExtensions": true`, so imports keep their `.ts` extension
  - `"erasableSyntaxOnly": true` and `"verbatimModuleSyntax": true`, so the code only uses syntax that Node can strip (no `enum` or `namespace`, and type-only imports use `import type`)
  - `"resolveJsonModule": true`, for the default data JSON files
- Rename the server files from `.js` to `.ts` one folder at a time: models, then controllers, then routes, then `index.ts` and `defaultDatabaseSetup.ts`. Commit after each folder.
- Export each compiled model from its model file (`export const User = model("User", UserSchema)`) instead of calling `mongoose.model()` in every controller, and type it with an interface or `InferSchemaType`.
- Type the request params and bodies on the handlers, for example `Request<{ uuid: string }>`.
- Update the scripts: `"start": "node index.ts"`, `"dev": "node --watch index.ts"`, and add `"typecheck": "tsc"`.
- **Client:** browsers can't run TypeScript, so the client needs a build step to use it. For now, keep the client in JavaScript and add `// @ts-check` to each file. Editors and `tsc` (with `checkJs` in a separate `client/tsconfig.json`) then catch type errors without changing how the files are served. Converting the client fully means adding a bundler such as Vite and installing Firebase from npm. That's worth doing later if the client grows, but it isn't needed now.

**Done when:** there are no `.js` files left in `src/`, `npm run typecheck` passes with `strict` on, and the app behaves as it did before.

## Phase 8: Tests and CI (M)

Every module gets a test file in the same folder, named after the file it tests:

```
index.ts
index.test.ts
src/
  controllers/
    eventController.ts
    eventController.test.ts
    questController.ts
    questController.test.ts
    setupController.ts
    setupController.test.ts
    userController.ts
    userController.test.ts
  models/
    userModel.ts
    userModel.test.ts
    ...
  routes/
    userRoutes.ts
    userRoutes.test.ts
    ...
client/
  services/
    authErrors.js
    authErrors.test.js
    ...
  components/
    UserTable.js
    UserTable.test.js
    ...
```

- Keep Node's built-in test runner (`node:test`) and `supertest`, which are already set up. Set the test script to an explicit glob so it only picks up colocated tests: `"test": "node --test \"**/*.test.ts\" \"client/**/*.test.js\""`.
- Move `test/index.test.js` to `index.test.ts` and delete the `test/` folder.
- Use `mongodb-memory-server` so tests run against a real but throwaway MongoDB without touching the dev database. A shared helper (for example, `src/testUtils/db.ts`) starts it, clears collections between tests and stops it at the end. `.npmrc` has `ignore-scripts=true`, so the MongoDB binary downloads on the first test run rather than at install.
- What to cover:
  - **Models:** required fields, the unique `uuid` index and defaults.
  - **Controllers:** each handler's success path, missing-record path (404) and validation path (400). `mockUsers.json` and `defaultQuests.json` work as fixtures.
  - **Routes:** each route is wired to the right handler and method, through `supertest`.
  - **Setup:** `setupDatabase` seeds quests once and doesn't duplicate them on a second run.
  - **Client:** pure logic such as `authErrors.js` is tested directly. Components return elements and don't depend on the page, so test them with `happy-dom`: render with fixture data, then check the rows, the button callbacks and that a username containing HTML appears as text.
- Turn on Node's built-in coverage reporting so gaps are visible.
- Add a GitHub Actions workflow that runs `npm ci`, `npm run typecheck` and `npm test` on every pull request. Cache the `mongodb-memory-server` binary between runs.
- Add a CI status badge to the README.

**Done when:** every source file has a matching test file, and pull requests run type checks and tests automatically and fail on regressions.

## Phase 9: Auth and API hardening (M)

The API currently trusts whatever `uuid` is in the URL, and `PUT /user/:uuid` writes the request body straight into the document.

- Add middleware that uses the Firebase Admin SDK to verify the ID token from an `Authorization: Bearer` header, and set `req.user.uuid` from the verified token's `uid`.
- Replace `/user/:uuid` for the player's own data with `/me` routes that use `req.user.uuid`, so a player can only change their own record. Viewing another player stays a read-only route that returns public fields only.
- Whitelist updatable fields on user updates, so clients can't set their own `gold`, `level` or `experience`.
- Restrict quest create, update and delete to an admin role (the `type` field on the user).
- Replace the `Accept`-header checks on `/user` and `/firebase` with real authorisation. They aren't a security control.
- Add `helmet`, a request body size limit and basic rate limiting.
- Add tests for each of these alongside the code: missing token, invalid token, another player's `uuid`, a non-admin creating a quest, and a client trying to set `gold`.

**Done when:** a request without a valid token can't read private data or change any player, and tests prove it.

## Phase 10: Finish one vertical slice (L)

A working "start a quest" loop is more convincing than many half-built features.

- Implement `performQuest` for the signed-in player: check they meet the requirements (level, action points and units), deduct action points, apply gold and experience rewards, and handle level-ups.
- Return the updated player and show the result in the client, including failures ("not enough action points").
- Remove the remaining event stubs (`addNewEvent`, `getEvent` and `getEvents`) or implement them, depending on whether timed events are needed for quests.
- Test the quest's cost, reward, level-up and failure cases.
- Optionally seed default units and items (`defaultUnits.json` and `defaultItems.json` already exist) so quests have something to check against.

**Done when:** a new player can sign up, take a quest and see their gold, experience and action points change.

## Phase 11: Deploy (S to M)

- Host the API and static client on a free or low-cost platform (Render, Fly.io and Railway are all options) with MongoDB Atlas.
- Keep all secrets in the host's environment settings, including the Firebase Admin service account. Rotate any credentials that were ever committed or logged.
- Add the live link and a screenshot to the README.

**Done when:** the README links to a working demo.

## Suggested order and pull requests

1. Phase 3 as one PR, client only. It comes first because Phases 4, 5 and 6 all change sign-up and the players list. Doing them after the restructure means they edit the new files once, instead of editing `auth.js` and `profile.js` only for those files to be deleted.
2. Phase 4 as one PR. After the restructure, the markup lives in a few small component files, so the styling change is contained. It's kept separate from Phase 3 so that one PR changes structure without changing the look, and the other changes the look without changing behaviour. Nothing later depends on it, so it can move after Phase 5 if fixing sign-up feels more urgent.
3. Phase 5 as one PR. It fixes sign-up.
4. Phase 6 as one PR, or two if you want to split the client messages from the server status codes.
5. Phase 7 as one PR per folder, or one PR with a commit per folder.
6. Phase 8. The test setup and CI can go in first, followed by tests folder by folder.
7. Phase 9, then Phase 10, then Phase 11.

TypeScript comes before tests so tests are only written once, in TypeScript. The server is small enough that the risk of migrating without tests is low, and `tsc` catches most mistakes a rename would introduce.

## Out of scope for now

Guilds, leaderboards, profile messages, unit art and combat between players stay on [`todoList.md`](todoList.md) until the vertical slice is working.
