---
spec_id: 001-persistent-tasks-in-postgres-with-a-web
title: Persistent tasks in Postgres with a web UI
status: approved
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks in Postgres with a web UI

## Problem

The Task API of taskboard22b holds tasks in memory only: every restart or redeploy empties the list, and a second instance would show a different list. People can only reach tasks by calling the API directly, because the web UI has no task features yet. So the taskboard cannot be used for real work today. Keeping tasks in a Postgres database and offering a page to see and create them makes the system usable end to end.

## Stories

As a taskboard user, I want the tasks I created to still be there after the system restarts or is redeployed, so that I can rely on the taskboard to remember my work.
**Repos:** taskboard22b-api, taskboard22b-gitops

As a taskboard user, I want to open the taskboard in my browser and see all tasks with their due dates, so that I know what is to do without calling an API.
**Repos:** taskboard22b-web, taskboard22b-api, taskboard22b-gitops

As a taskboard user, I want to create a task with a title and an optional due date in the browser, so that I can capture work as it comes up.
**Repos:** taskboard22b-web, taskboard22b-api

## Acceptance criteria

- **AC-001.1** Given tasks were created through `POST /tasks`, when the Task API is restarted or redeployed, then `GET /tasks` returns `200` with every one of those tasks, each with the same `id`, `title`, `due_date` and `created_at` it was created with, ordered by creation time oldest first.
- **AC-001.2** Given the Task API runs as 2 instances, when a task is created through one instance, then a `GET /tasks` answered by either instance contains that task.
- **AC-001.3** Given tasks exist, when a user opens the web UI's start page `/`, then the page lists every task returned by `GET /tasks`, oldest first, each showing its title and its due date as `YYYY-MM-DD` or the text "No due date" when it has none; given no tasks exist, the page shows the text "No tasks yet".
- **AC-001.4** Given the start page is open, when the user enters the title "Buy milk", picks the due date 2026-12-24 and submits the create form, then the task "Buy milk" with due date `2026-12-24` appears in the page's list without the user reloading, the form is emptied, and `GET /tasks` on the Task API contains that task.
- **AC-001.5** Given the start page is open, when the user submits the create form with an empty title, a title of only whitespace, or a title longer than 200 characters after trimming, then the page shows the message "Title must be 1 to 200 characters" next to the title field, keeps the entered values, and no task is created.
- **AC-001.6** Given the database is unreachable, when a client sends `GET /tasks` or `POST /tasks`, then the Task API answers `503` with a message that tasks are temporarily unavailable, its readiness check `/ready` answers `503`, and once the database is reachable again the requests succeed with no restart of the Task API.
- **AC-001.7** Given the Task API is unreachable or answers `503`, when a user opens the start page, then the page shows the message "Tasks cannot be loaded right now. Please try again later." instead of the list; and when the user submits the create form, the page shows "The task could not be saved. Please try again." and keeps the entered values.
- **AC-001.8** Given 10,000 tasks exist, when the user submits a valid task in the web UI, then the page shows "The task limit of 10,000 is reached." and keeps the entered values, and the Task API answers the underlying `POST /tasks` with `507` as before.

## Non-goals

- Updating, completing or deleting tasks; showing a single task on its own page.
- Filtering, searching, sorting options or paging the list.
- Users, login, ownership or per-user task lists: create and list stay open to anyone who reaches the system in this version; login is a later spec, due before prod exposure.
- Storing personal or confidential data in tasks.
- Live updates of the list when another user creates a task (a reload shows them).
- Carrying over tasks held in memory by the currently running Task API.
- Backups and restore of the task database (a later spec).
- Changes to the Task API's request and response contract beyond the `503` of AC-001.6.

## Assumptions

- **A1** The Task API keeps its existing contract (fields, validation, `422` errors, ordering oldest first, `201`); only where tasks are held changes (other option: revise the contract together with persistence; affects AC-001.1, AC-001.3, AC-001.4).
- **A2** The cap of 10,000 tasks with `507` stays, so that the existing contract and its tests remain valid (other option: drop the cap now that tasks are not in memory; affects AC-001.8).
- **A3** Tasks held in memory at the moment of the switch are not carried over; they would be lost on the next restart anyway (other option: copy them into the database once; affects AC-001.1).
- **A4** One Postgres database is declared for the system and used by the Task API only; the web UI reaches tasks only through the Task API, never the database (other option: web reads the database directly; affects AC-001.3, AC-001.4).
- **A5** The web UI checks the title rules itself before sending, with the same rules as the Task API (1 to 200 characters after trimming) and the wording "Title must be 1 to 200 characters"; the due date is picked with a date field, so a malformed date cannot be entered (other option: show the Task API's `422` text as is; affects AC-001.5).
- **A6** A newly created task appears in the list at once after the Task API confirmed it; the list is not refreshed for tasks other users create meanwhile (other option: reload the whole list after each create; affects AC-001.4).
- **A7** The database being down answers `503` and makes the Task API not ready, and the Task API reconnects by itself (other option: `500`, or the process exits and restarts; affects AC-001.6).
- **A8** The start page `/` is the task page; due dates are shown as `YYYY-MM-DD`, and the page text is English (other option: a separate `/tasks` page, localised dates; affects AC-001.3, AC-001.4).
- **A9** The feature runs in every environment the components run in (today dev; staging and prod when promoted), and each environment has its own database (other option: none; affects AC-001.1, AC-001.2).
- **A10** Listing and creating tasks, in the web UI and on the Task API, stay open to anyone who can reach the system, with no login in this version; tasks must hold no personal or confidential data, and login comes in a later spec before prod exposure (other option: a login or an API key now; affects AC-001.3, AC-001.4). Answered by the user 2026-11-20.

## Open questions

- ~~Tasks are now kept permanently and the web UI is exposed without login, so anyone who reaches the web UI or the Task API can read and create tasks. Does that stay open, as decided for the in-memory version? (suggested: yes, no login in this version, tasks must hold no personal or confidential data; a later spec adds login before prod exposure; affects AC-001.3, AC-001.4, new criterion)~~ Answered 2026-11-20: yes, open; see A10 and Non-goals.

<!-- Priority levels (spec 196) for the product-manager's PRIORITY line: P0 an outage, data loss, a security problem, or something that blocks all other work; P1 a normal bug or feature; the fallback when nothing signals urgency; P2 cosmetic, a convenience, or something with an easy workaround. A reason is "P<n>: <phrase>", one of: P0: outage, data loss, security problem, blocks all other work; P1: normal bug or feature; P2: cosmetic, convenience, easy workaround. -->

## Changelog

- 2026-11-20 created
- 2026-11-20 priority P1 proposed: P1: normal bug or feature
- 2026-11-20 amended: open question on access answered (open, no login, no personal or confidential data), recorded as A10 and in the non-goals
- 2026-11-20 approved
