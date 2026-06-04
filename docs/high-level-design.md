# Initial high level design

## Scope (copied from readme)

This project is done when:

- Completed tasks are retrieved from Todoist's API
- A frontend displays a GitHub-style heat map and current streak
- Real data is protected by authentication
- A demo mode seeds fake data for anyone viewing the public URL
- Running in Azure with IaC and continuous deployment

Everything beyond this is optional.

## Functional requirements

### MVP

- The primary user should be able to see their current streak
- The system shall calculate the streak in the following way:
  - The number of consecutive days where at least one task has been completed, counting back from today
  - If no tasks have been completed today, today is not counted towards the streak but the streak isn't considered "broken" i.e. it doesn't reset to 0
- The primary user should be able to see a heat map, similar to GitHub contributions or the heat map in Anki's stats tab that shows how many tasks they've completed per day over the past year
- Demo users should be able to view a "demo" mode (populated with fake data) that allows them to see how the system works and displays data
- The system shall require authentication to see real data
- All users should see statistics based on their browser's timezone
- Demo users should not be able to see the primary user's real tasks
- The system should periodically sync completed tasks from the Todoist API (e.g. every 15 minutes)

### Beyond MVP

- The user should be able to manually refresh to sync against Todoist

## Non-functional requirements

### CAP / Consistency vs availability

- Eventual consistency with respect to accurately reflecting Todoist state is acceptable
- High availability is currently unnecessary. It's a side project with infrequent usage patterns so some downtime is tolerable and likely to go unnoticed.
  - Availability could become more important if reliable demo access becomes a priority later

### Scalability

- Only needs to support a single user with real data
- Needs to support a very minimal (generally zero) number of demo users

### Traffic patterns / read/write symmetry

- Usage by primary user will be at most a few times a day
- Usage by demo users is likely less often than once a month
  - This is so low it could be bumped from the MVP if necessary

### Environmental constraints

- Backend infrastructure will run in a public cloud environment, ideally utilising what's available on the free tier
- UI will run in a web browser

### Security and compliance

- Todoist credentials need to be stored securely (location TBD later)
- Primary user needs secure authentication
- GDPR compliance is out of scope - the system stores data for a single known user (the owner) only

### Data durability

- Not important for MVP as data can be resynced from Todoist

### Latency vs throughput

- Not a concern at this stage due to current scale

## High level design of a functional system

Note this is a first-pass and may change as I do more research

![High level design](diagrams/high-level-design.svg)

### Explanation of initial design

#### Todoist API

| Options | Pros | Cons | Outcome |
| --- | --- | --- | --- |
| REST API | Relatively simple. Can query by date. | Requires polling | 🚧 Good option - prototype and compare |
| REST API's Sync Endpoint | Designed for syncing data. Supports incremental syncing. Used by Todoist's own app. | Requires polling | 🚧  Good option - prototype and compare |
| Webhooks | Event-driven / no polling needed. Can subscribe to relevant events. | More complex setup. Requires a public HTTPS endpoint. Requires use of OAuth flow to use. | 🔮 Potential future improvement for better consistency and to avoid polling |
| Python and Node.js SDKs | Simpler - abstraction layer over REST API | Limited to Python and Node. I would prefer to use Golang. | ❌ Ruled out |
| CLI | | N/A - don't plan to run from shell | ❌ Ruled out |
| Agent skills / MCP | | N/A - don't plan to use agentic AI | ❌ Ruled out |

Note: Todoist API rate limits are not a factor due to only supporting one user.

> For each user, you can make a maximum of 1000 partial sync requests within a 15 minute period.
> For each user, you can make a maximum of 100 full sync requests within a 15 minute period.

[Source](https://developer.todoist.com/api/v1/#tag/Request-limits)

#### Synchroniser

| Options | In Azure free tier? | Pros | Cons | Outcome |
| --- | --- | --- | --- | --- |
| Serverless function (e.g. Azure Function) | Yes | Scale to zero / saves compute / cost-efficient. Good fit for trigger on schedule / ad-hoc (for manual sync). Good fit with free tier requirement. | Cold start latency. Only an issue for an ad-hoc sync triggered on user request | ✅ Preferred approach |
| Azure App Service (PaaS) | Yes | Relatively little ops overhead. | More appropriate for long-running app. Would need to handle schedule myself. Wasted compute / cost. | ❌ Ruled out |
| Azure Container App | Yes | Simple. Familiar workflow. Portable. | Will need to manage triggers / schedule. Wasted compute / cost. Need to manage container runtime and health-checks. Overkill. | ❌ Ruled out |
| App running on VM | First 12-months | | Ops overhead. Don't need this much control. Wasted compute / cost. Overkill. | ❌ Ruled out |

#### Database - SQL vs NoSQL

Specific to this design:

- With a single user there's no complex relations
- The data structure is known up-front and unlikely to change often

General trade-offs:

- NoSQL would be more flexible and make fast iteration easier
- SQL generally provides better off-the-shelf tooling for transactions, schemas and migrating schemas over time

Outcome: ✅ SQL

Why:

- Either option would work
- I have a personal preference for a stricter data model that provides clarity when working with data

#### Database - specific database

| Options | In Azure free tier? | Pros | Cons | Outcome |
| --- | --- | --- | --- | --- |
| Azure DB for PostgreSQL | First 12 months | Widely used, good documentation in community. A good default option. | | ❌ Ruled out. |
| Azure DB for MySQL | First 12 months | | | ❌ Ruled out |
| Azure SQL Database | Yes | | | ✅ Preferred option - free tier |
| Azure SQL Managed Instance | First 12 months | | Enterprise grade. Very expensive. | ❌ Ruled out |

Notes:

- I'm ruling out non-Azure cloud offerings for now. Mostly to keep the decision-making simple and to keep the solution within a single cloud environment. I'm also ruling out self-managed databases in containers or VMs due to the ops overhead.
- At this scale the main differences between the SQL DB options are pricing, community adoption and documentation.

#### Web UI

TBD

#### Backend for UI

| Options | In Azure free tier? | Pros | Cons | Outcome |
| --- | --- | --- | --- | --- |
| Azure Functions | Yes | Scale to zero / cost-efficient. Consistent with synchroniser choice — same deployment model. | Cold start latency will be user-facing and likely noticeable | 🚧 Uncertain - prototype |
| Azure App Service | Yes | Designed for long-running web apps. No cold start. | Wasted compute when idle. Limited to 1hr/day compute | 🚧 Uncertain - prototype and compare |
| Azure Container App | Yes | Good for a long-lived app. Portable. | Need to manage container runtime and health checks. | 🚧 Good option - prototype and compare |
| Azure Static Web Apps | Yes | Bundles frontend hosting and backend API into one service and deployment. Less infrastructure to manage. Built-in auth | API is Azure Functions under the hood / cold starts. Less flexible. | 🚧 Uncertain - prototype and compare |
| App running on VM | First 12 months | Full control | Ops overhead. Overkill. | ❌ Ruled out |

There's a lot of options for this and the UI and it needs further research and prototyping to better understand suitability and trade-offs.

#### Authentication

| Options | In Azure free tier? | Pros | Cons | Outcome |
| --- | --- | --- | --- | --- |
| Microsoft Entra ID or Azure Active Directory | Yes | Uses a robust managed service | Some setup complexity | ✅ Preferred option |
| Pre-hashed password passed in via env var | N/A | Very simple | Would never scale. Manual ops effort to update it. Not a best practice. | ❌ Backup option |
| Custom solution in e.g. Container App + DB | Yes | | Additional complexity and effort. Overkill for a single user. | ❌ Ruled out |

Note: I'm also ruling out third-party auth services due to wanting to stay in the Azure ecosystem for now.

#### Seed demo data - location

| Options | In Azure free tier? | Pros | Cons | Outcome |
| --- | --- | --- | --- | --- |
| Inline in frontend | N/A | Very simple. No additional infrastructure. No network request needed. | Increases initial load size slightly even when demo data not needed. Requires a frontend deployment to change demo data. | 🚧 Prototype and compare |
| Statically-hosted in Azure Blob and loaded in by frontend | First 12 months | | Additional network request. Additional artifact to host and manage. | 🚧 Prototype and compare |
| Served by backend | N/A | Exercises real backend as part of demo. | Adds some complexity. Additional network request and dependency on backend. Mixes fake data with real data. | ❌ Ruled out |

Notes:

- Some pros/cons mirror each other. Where this is the case I've only included them once.
- Blob storage would be my preferred option if it was permanently free. Since it's not I will prototype and defer the decision.

## Things to consider later

- Error handling / retry strategy for syncing
- How demo users enter the demo mode
- Database schema
- Database storage requirements
- Database migration strategy
- Frontend wireframe
- Backend API design
- IaC
- Deployment
