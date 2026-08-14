# Jam.py Project Internals

## Some Flows in Jam.py

### Application Startup

The framework opens admin.sqlite and loads admin metadata.

- App startup goes through wsgi.py:87 and wsgi.py:145.
- The App constructor calls create_admin: wsgi.py:164, admin.py:57.
- AdminTask receives the admin database path: admin.py:61.
- Then init_admin runs: admin.py:43.

### Building the Project Task Tree

A runtime Task with task tree objects is built from admin metadata.

- create_task reads sys_items from the admin task and looks for the root task record: task.py:9.
- load_task builds the full object tree: task.py:331.
- fill_rec_dicts reads sys_items, sys_fields, sys_filters, sys_report_params: task.py:307.
- create_groups/create_items map rows to Group/Item instances and fill server_code from f_server_module: task.py:109, task.py:138, task.py:158.
- After that, bind_items and compile_all connect and compile server code: task.py:339.
- Compilation executes server_code and attaches functions to item/task: items.py:665.

### Selecting the Main Project Database

- Admin builder reads sys_tasks and sets task_db_info/task_db_type: builder.py:96.
- load_task transfers that into the runtime task and creates the DB adapter/pool: task.py:342.

### User Selects an Object from the Task Tree

- HTTP entry goes through App.call and routes to index/api: wsgi.py:206, wsgi.py:241, wsgi.py:300.
- Frontend initially sends load via /api: task.js:330, task.js:110.
- Backend on_api parses the JSON packet and calls get_response: wsgi.py:667, wsgi.py:726.
- For load, it returns task.get_info plus templates/settings/locale/language: wsgi.py:520.

### When the User Opens a Specific Object

- Frontend sends open: item.js:1778.
- Backend get_response for open calls select_records: wsgi.py:727, items.py:232.
- select_records runs hooks and SQL read through execute_open: items.py:240, items.py:178.

### When Frontend Calls task.server("function_name")

- JS server method is here: abstr_item.js:373.
- It sends the server request: abstr_item.js:389.
- Backend routes this to server_func/getattr and executes the function by name: wsgi.py:731, wsgi.py:743.

Core idea:

admin.sqlite is the metadata database from which the framework builds task tree objects (Task/Group/Item) and their server code; after that, the client works through /api, and for a specific object it uses open or server calls through item_id and function name.

## /api Endpoint

Through wsgi.py, there is effectively one API entry point: /api.

The endpoint receives:

- POST JSON packet in the format [method, task_id, item_id, params, modification]
- Parsing is done in wsgi.py

Exposed methods are:

- load
- open
- apply
- server
- print

That branching is in wsgi.py.

How mapping works:

1. task_id selects task context:
   - 0 means admin task
   - > 0 means project task

2. item_id selects object:
   - task.item_by_ID(item_id)

3. method selects operation:
   - open -> item.select_records(...)
   - apply -> item.apply_changes(...)
   - server -> server_func(item, func_name, params)
   - print -> item.print_report(...)
   - load -> init_client(...)

4. Named server call:
   - server_func does getattr(obj, func_name) and invokes the function.

5. Frontend side targeting /api:
   - JSON POST goes from process_request in task.js
   - URL is explicitly api in task.js
   - task.server(...) packs server method call in abstr_item.js

6. Endpoints outside /api:
   - upload at /upload: wsgi.py
   - ext at /ext: wsgi.py
   - index/login/logout routing in wsgi.py

### Example 1: load (task initialization on client)

Request body to /api:

```json
["load", 1, 1, null, 0]
```

- method = load
- task_id = 1 (project task)
- item_id = 1 (task object)
- params = null
- modification = 0

Backend processing:

- Parsing: wsgi.py
- load branch: wsgi.py
- Return task_info: wsgi.py

Typical response shape:

```json
{
  "result": {
    "status": 9,
    "data": [
      {
        "task": { "id": 1, "name": "test", "items": [] },
        "templates": "<html...>",
        "settings": { "SAFE_MODE": true },
        "locale": {},
        "language": {},
        "user_info": {},
        "privileges": null
      },
      ""
    ],
    "modification": 123
  },
  "error": null
}
```

### Example 2: open (loading item rows)

Frontend builds params with keys:

- __expanded
- __fields
- __open_empty
- __order
- __filters
- __limit
- __offset

This is visible in item.js.

Example request:

```json
[
  "open",
  1,
  57,
  {
    "__expanded": true,
    "__fields": ["id", "customer", "total"],
    "__open_empty": false,
    "__order": [["id", false]],
    "__filters": [["id", 1, 10]],
    "__limit": 50,
    "__offset": 0
  },
  123
]
```

Backend processing:

- open => select_records: wsgi.py
- select_records pipeline: items.py

Typical response:

```json
{
  "result": {
    "status": 9,
    "data": [
      [
        [10, 3, 1250.5],
        [11, 5, 990.0]
      ],
      ""
    ],
    "modification": 123
  },
  "error": null
}
```

### Example 3: server (calling a server function by name)

Frontend call:

- task.server("prepare_users") in JS
- server method: abstr_item.js
- It sends: send_request("server", [func_name, params]): abstr_item.js

Request body:

```json
["server", 1, 1, ["prepare_users", []], 123]
```

Backend processing:

- server => server_func: wsgi.py
- getattr and function call: wsgi.py

Typical response:

```json
{
  "result": {
    "status": 9,
    "data": [null, ""],
    "modification": 123
  },
  "error": null
}
```

Core idea:

- /api receives one uniform JSON packet.
- method determines what happens (load/open/apply/server/print).
- For server calls, function name is params[0], arguments are params[1].

## Record Save Flow (edit form -> database)

User clicks OK in the form.

### apply_record

apply_record is called in item.js:3327.

- apply_record first runs post: item.js:3339.
  - post does:
    - record validation (check_record_valid),
    - on_before_post,
    - post for details currently in edit/insert,
    - write change into change_log,
    - state switch to browse,
    - on_after_post.

    All of that is in item.js:2325.

    Important: post does not go to server. post is a local commit in client dataset/change_log. There is no /api call in that function.

- After post comes apply, apply_record calls apply with callback: item.js:3353.
  - apply checks:
    - if detail has master_applies, it propagates to master,
    - if item is still in edit/insert, it runs post,
    - it takes delta changes from change_log.

    Main logic is in item.js:2402.
    Before sending apply, hook params are packed.

    - on_before_apply is collected through caller -> master chain: item.js:2378.
    - Send to server.
    - If there are changes, it sends apply request with payload [changes, params_dict]: item.js:2447.
    - That request goes to /api through process_request: task.js:86, task.js:110.

- Backend dispatch:
  - /api parses method/task_id/item_id/params/modification: wsgi.py:667.
  - For method apply it calls item.apply_changes(...): wsgi.py:734.
  - Server-side apply_changes:
    - reconstructs delta from client changes,
    - runs on_apply hooks (task then item),
    - if hook does not return result, default apply_delta runs,
    - commits connection at the end.

    See items.py:289, items.py:301, items.py:306, items.py:308.

    - apply_delta (default path):
      - validates delta records,
      - calls db.process_changes for SQL operations,
      - returns update packet for client change_log.

      In items.py:272.

- Back to frontend:
  - _process_apply receives response,
    - checks error,
    - updates change_log,
    - triggers on_after_apply,
    - refreshes controls.

    All that in item.js:2462.

In short:

- post = local closing of edit and delta preparation.
- apply = sending delta to server + DB commit + returning update.

## COPY Item

COPY does not create a new table; it marks item as a copy (f_copy_of), and backend intentionally skips DDL.

What COPY actually does in v7:

1. COPY in UI creates a new sys_items record as a copy of existing item.
   - click copy button: admin.js
   - sets f_copy_of to original item: admin.js
   - immediately starts selecting/filling fields from source: admin.js

2. New copy item inherits source technical attributes, including table.
   - on copy click, almost all item fields are copied (except some system references): admin.js
   - this includes f_table_name/f_gen_name, so copy points to the same physical table as source.

3. During save, backend intentionally disables DDL for copy item.
   - key line: manual_update = params['manual_update'] or delta.f_copy_of.value in builder.py
   - when f_copy_of is set, manual_update becomes true.

4. When manual_update is true, table is not created/altered in database.
   - insert_item_query runs create_table only if not manual_update: admin.py
   - update_item_query runs alter/recreate only if not manual_update: admin.py
   - delete_item_query runs drop table only if not manual_update: admin.py

5. Metadata is still saved normally.
   - apply_delta and commit still run: builder.py
   - sys_items/sys_fields definitions of the copy are saved, but physical table remains the same.

Why it looks like everything goes into the existing item/table:

- copy item and source share same f_table_name, and copy mode forbids DDL schema changes.
- practically, COPY is an alternative definition/interface over the same table, not a clone with a new table.

Additional code signals for this concept:

- in several lookup/detail places, copy uses owner_id = f_copy_of (original) instead of its own id: admin.js
- there is also protection used in copy definitions when deleting fields: builder.py

Conclusion:

- the behavior you described is intentional; COPY in AppBuilder is a copy definition over existing table, not automatic creation of a new physical table.
- normal behavior (create/alter/drop table) returns only when item is not a copy, i.e., no f_copy_of and manual_update is false.

## Routing

In Jam.py there are practically 2 levels of routing:

- HTTP routing (very small, hardcoded)
  This is classic: index, login, logout, plus api/upload/ext.
  App.call branches input to index/api, and index/login/logout are separate endpoints in wsgi.
  So there is no large router like in Django/Flask apps.

- Application routing (most of the work)
  After page load, frontend mostly POSTs to /api.
  Inside one uniform JSON packet, method decides operation (load/open/apply/server/print), and task_id + item_id choose the target object.

So application flow is not driven by URL routes but by RPC dispatch over task/item objects.

How this looks for index/login/logout:

- index: entry web page (client bootstrap)
- login: authentication and session establishment endpoint
- logout: session termination endpoint

After that, almost all real work goes through /api, not through new URL routes.

Core point: in Jam.py, index/login/logout are infrastructure routes, while business navigation and logic are moved to /api + task/item mechanism.

You get operational stability in web mode, not prettier routing.

Short version: in newer Jam.py, routing is still minimal, but it now has a broader protection layer around it that v5 largely did not expose in this form.

### What Was Actually Gained

- Stable session and authentication flow.

- Clearly separated entry points for index, login, and logout.

- Session cookie is stored with safer rules (HttpOnly, SameSite), with optional IP/UUID validation
  in safe mode.
  Effect: lower session abuse risk and fewer odd user states.

- Stale client version detection.

- Server returns PROJECT_MODIFIED when build or metadata change.
  Client recognizes this and does not proceed blindly with stale state.
  Effect: less data corruption when multiple users work while system is changing.

- Maintenance mode without breaking user flow.
  During maintenance, server returns maintenance status, client shows message and retries load.
  Effect: controlled behavior instead of random failures.

- One uniform RPC-style API dispatch.
  Business calls go through one point with methods load/open/apply/server/print instead of many routes.
  Effect: simpler client-server protocol and easier centralized logging/error handling.

- More precise apply model in Master/Details chains.
  master_applies is an explicit part of metadata and client logic.
  If enabled, Details apply does not finish partially; it propagates into Master flow.
  Effect: better transactional consistency for documents.

- Controlled project reload while running.
  Build/version mechanism checks whether code/metadata changed and reloads task if needed.
  Effect: fewer manual restarts and fewer inconsistent process instances.

- Better connection scalability.
  Connection pool model (QueuePool/NullPool) with clear commit boundary in apply.
  Effect: more predictable behavior under load.

### Why It Feels Like None of This Existed in V5

In V5, more things were implicit and monolithic within framework flow.

In the newer model, the same or similar needs are surfaced explicitly because web runtime is treated more seriously as a distributed system (multiple tabs, multiple users, hot changes, maintenance).

### Main Changes

Top 3 changes most relevant for your Master/Details + PostgreSQL scenario:

- Master-driven apply via master_applies.
  Gain: Details changes are not saved partially; they enter the same flow as the Master record.
  Why it matters in PostgreSQL: lowers chance of orphan Details rows and bad save order with FK, NOT NULL, unique constraints.
  Practical: header + lines document is saved as one unit.

- Centralized delta apply with one commit.
  Gain: server reconstructs delta and applies through one transaction path.
  Why it matters in PostgreSQL: ACID works in your favor; errors are more consistent and easier to rollback/retry.
  Practical: fewer half-saved states when multiple tables are involved.

- Protection from stale client and session.
  Gain: statuses like not logged/modified prevent stale tabs/clients from writing wrong delta packets.
  Why it matters in PostgreSQL: fewer conflicting writes and fewer cases where user unknowingly works on changed structure/data.
  Practical: more stable multi-user and hot-change operation.

Concrete end-to-end example for one document (Master + Details):

1. User enters application.
   HTTP entry goes through index/login/logout layer, not directly to business routes.

2. Client initializes task.
   Front sends load through single API point.

3. Opening Master object.
  open call is sent for that item; Details are not auto-loaded in the same step.

4. User edits Master + Details rows.
   Edit runs locally in client dataset/change_log.
   Important difference: post does not go to server immediately; it closes local edit and records delta.

5. Click OK / apply_record.
   Client first does local post, then apply.

6. master_applies decision.
  If a Details item is configured with master_applies, apply propagates to Master flow.
   This is key for saving document as one unit.

7. Sending delta packet to server.
   Client sends apply through API with changes and hook params.
   API packet is uniform: method/task/item/params/modification.

8. Server apply pipeline.
   Server reconstructs delta, runs on_apply logic, and if no override exists runs default apply_delta, then commit.

9. Response return and client sync.
   Client processes response, updates change_log, triggers on_after_apply, and refreshes UI.

Most important practical points:

1. Post is local; apply is server-side transactional.
2. master_applies decides whether Details save independently or under a Master transaction.
3. If you want a document as one unit, master propagation is essential.

### Which Relationship Style to Use

Practical test: decide between V5-style and strict FK style.

- Choose V5-style if your primary goal is flexible document flow.
  - One Details table should serve multiple different Master types.
  - You want faster modeling via OWNER_ID/OWNER_REC_ID logic and the task tree.
  - You accept that part of integrity is maintained through framework rules and event code.

- Choose strict FK style if your primary goal is strict database integrity.
  - You want PostgreSQL to enforce relationship integrity.
  - You run serious analytics/integrations/SQL outside Jam.py.
  - You need every relationship explicit and verifiable at DB level.

Recommendation for your case:

- For a new business app on PostgreSQL: go FK-first.
- Use V5-style only where you truly need polymorphic Details scenarios that become impractical with pure FK.

Pragmatic hybrid (often best):

- Core tables: strict FK, unique, not null, check.
- Document UI flow: master_applies enabled where atomic save is needed.
- Event code: business rules only, not primary referential integrity.

Red flags you went too far with V5-style:

- You cannot write clear SQL JOINs without special OWNER interpretation.
- Integrity depends on whether events happened.
- It is hard to test what relations are actually allowed.

Red flags you went too rigid with FK style:

- Too many helper tables for naturally document-like behavior.
- Every minor workflow change requires schema migration.
- UI gets more complex than business needs.

### When to Use Which in Practice

If Jam.py controls schema through its model, do not rely on strict FK as your primary mechanism.

If you manage PostgreSQL schema manually, you can have FK, but you must test Jam.py save/delete flows so they do not collide.

Why this difference:

Your documentation notes that framework historically runs without FK in its core logic and admin schema (Understanding the Excellence of the Jam.py Framework.md:168).

Jam.py often uses DELETED-based soft delete and its OWNER_ID/OWNER_REC_ID model, which is not the same as classic DB referential integrity logic.

That is why FK-first is an architectural goal for PostgreSQL only when you control DB design, not when you 100% delegate relation handling to Jam.py.

Practical decision rule:

- Jam.py-first project: minimal FK or no FK, rules in Jam.py events.
- DB-first project: introduce FK, but:
  - use deferrable constraints where needed,
  - watch delete flows (soft-delete vs hard FK),
  - test apply Master/Details scenarios.

So the point was not Jam.py natively supports FK, just turn it on.
The point was: you can have FK in PostgreSQL if you manage the schema, but that is an explicit integration and must be tested.

Practical checklist for safe FK introduction in Jam.py + PostgreSQL:

1. Start with one pilot relationship.
  Do not introduce FK everywhere at once. Pick one Master/Details relation and validate full flow.

2. Always enable master_applies for document relations.
   For header + lines, the goal is one apply flow and one transaction unit.

3. Clean existing data before adding FK.
   Before ALTER TABLE, check orphan rows, nulls, duplicates.

4. Add indexes before FK.
   Create index on child columns before FK to avoid slowdowns and lock issues.

5. Separate soft-delete and hard-delete rules.
   If Jam.py uses DELETED flag, do not rely on ON DELETE CASCADE as primary mechanism.

6. Use DEFERRABLE for critical relations.
   For complex writes in one transaction, use DEFERRABLE INITIALLY DEFERRED where needed.

7. Test four mandatory scenarios.
  
   - insert Master + Details
   - update both
   - delete Master
   - delete Details

   Also test UI apply and direct SQL rollback.

8. Keep business rules in events, not base integrity.
   Let DB enforce referential integrity; use events for domain logic.

9. Introduce migrations step by step with rollback plan.
   One FK per migration with clear rollback path.

10. Enable FK error monitoring from day one.
    Log and track constraint violations immediately after deploy.

## Master-Details Relations

### Loading Master and Details Data

For open on Master, framework does not load Details records in the same API call. Details open separately when UI needs them.

How this appears in code:

- open request loads dataset only for current item.
- Front sends open: item.js:1778.
- Backend handles it as select_records for that one item: wsgi.py:731, items.py:232.
- After open, _do_after_load fills only this._dataset of that item.
- There is no automatic loop that opens details at that moment: item.js:1787.
- Details open later on UI events.
- When master row changes (on_after_scroll), active detail runs detail.open(true): item.js:3664.
- On detail tab selection, detail.open(true) also runs: item.js:3634.
- __expanded is not load details, but expanded lookup/render fields.
  Parameter goes via QueryData.expanded: common.py:435.
  DB uses it in lookup/order/group SQL branching, not for Master/Details eager loading: db.py:762, db.py:801, db.py:861.

Conclusion:

- Master and Details are not one joined payload in regular open.
- Master opens first.
- Details open separately (lazy, on demand in UI flow).

### master_applies Attribute on Details Objects

In Jam.py there are 2 steps that are often mixed up:

- post: local close of edit state and write into client change_log
- apply: send delta changes to server and DB commit

The point of master_applies is who owns apply operation in a Master/Details relation.

Your documentation reflects this rule: when a Details item has master_applies, apply propagates to Master.

How it works in practice:

- master_applies = 1 (true)
  - If you are on a Details item and click apply, a separate apply request is not sent from the Details context; flow propagates to Master (in V5, UX often felt like nothing happened).
  - Master and all relevant details enter same apply cycle.
  - One consolidated delta is sent to server.
  - Practically this gives one transaction unit for entire document (header + lines).

- master_applies = 0 (false)
  - Details can apply independently.
  - Master and Details are not forced into the same apply unit.
  - Useful when Details have their own lifecycle, but risk of inconsistent save ordering is higher when they depend on each other.

### What Happens to Details When You Delete Master Without FK in DB?

- If you use standard Jam.py flow delete_record then apply, master record goes to soft delete (DELETED = 1) when soft_delete is enabled.
- Framework then traverses related Details sets and marks them DELETED = 1 as well.
  This is recursive for deeper detail levels.
- If soft_delete is disabled, it does hard delete for both Master and corresponding Details rows.

So in the standard flow, it does not leave live Details behind a deleted Master.

### Is DELETED Automatically Filtered for Lookup Retrieval?

- For regular open on an item: yes, DELETED = 0 is auto-added unless you explicitly filter by DELETED.
- For lookup list (typeahead/select): lookup item opens as regular dataset, so in practice it also gets DELETED = 0 and deleted rows are not offered for new selection.

Important nuance: in expanded join rendering of lookup values in grid, join itself does not automatically add DELETED = 0 filter on lookup table. That means:

- Old record can still show lookup value even if lookup row is soft-deleted.
- New selection via lookup list usually no longer offers that row.
- This is likely intentional for compatibility and historical data display.

For your idea to keep master_applies always enabled on Details:
For a document model (Master + Details), this is the safest choice and I agree with it. It gives a uniform apply flow and fewer partial states.
