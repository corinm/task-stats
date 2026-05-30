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

