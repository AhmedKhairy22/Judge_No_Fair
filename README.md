## Functional Requirements
- Users can create profiles and specify their information.
- Employers can create, publish, update, and close job listings.
- Candidates can apply for jobs and track their application status.
- Candidates and employers can query items with multi-attribute filtering (skills, pay range, experience level, text keywords).
- The system asynchronously generates ranked job recommendations for candidates, and candidate leads for employers.
- The system enforces unique constraints on (CandidateID, JobID) pairs
- The system prevents invalid application actions, such as moving a rejected application back to the interview stage.
- Employers can manage the hiring pipeline from application screening to interviews, offers, or rejection.
- Candidates can perform skill-gap analysis by comparing their skills with the requirements of a target job.
- The system dispatches push and email notifications asynchronously for application state changes, recommended matches, and milestone progress based on user preferences.

## Non-Functional Requirements
- **Scalability**: System components scale horizontally to handle $1\text{M+}$ users, $100\text{k}$ active job postings, and high read-to-write ratios ($100:1$ for search vs. application submissions).
- **Performance**: 95% of profile, job search, and job details requests should be completed within 300 ms. Applying for a job should complete within 500 ms.
- **Availability & Reliability**: The system should provide at least 99.9% availability for core services. Applications and application status updates must not be lost or duplicated, even if a service fails.
**Security**: User and employer data must be protected in transit and at rest. The system must enforce authentication and role-based access control so users can only access and modify resources they are authorized to use.

# Data Model
I chose PostgreSQL as the primary source of truth because the system contains highly related and transactional data, including users, candidates, companies, jobs, and applications. It also allows us to enforce important constraints, such as preventing a candidate from applying to the same job more than once

## Core Entities
- User: Common account and authentication information for candidates and employers.
- Candidate Profile: Candidate education, experience, skills, career goals, and portfolio information.
- Company & Employer: Companies and the employer users who manage their job postings.
- Job: Job details, requirements, skills, eligibility, and publishing status.
- Application: Connects candidates with jobs and tracks the current application status.
- Application Status History: Maintains the history of application state changes for tracking and auditing.
- Skills: A shared skill catalog used by both candidates and jobs to support matching and skill-gap analysis.
- Skill Gap: Stores the results of comparing a candidate's skills with a target job or role.
- Recommendation: Stores generated job or candidate recommendations and their scores.
- Notification: Tracks notifications and their delivery status.

For search and filtering, I use a search index such as Elasticsearch/OpenSearch rather than relying entirely on PostgreSQL. It indexes relevant candidate and job data for fast text search and filtering.

Recommendations, skill-gap analysis, notifications, and other non-critical workloads are handled asynchronously through a message queue. These services consume data from the core system and generate results, but they do not become the source of truth.

# API Design

I chose a RESTful HTTP architecture for client-facing operations because the domain revolves around well-defined, persistent resources (Jobs, Profiles, Applications). REST allows us to cleanly decouple clients, scale stateless API servers horizontally behind a load balancer, and leverage standard HTTP caching for high-frequency queries like job listings.

However, operations that require heavy computation (such as candidate matching feeds) or external delivery (such as push/email notifications) do not run synchronously inside the request-response cycle. Instead, REST endpoints accept or mutate state, emit domain events via an asynchronous message broker, and return immediate responses.

| Endpoint | Request | Response | Purpose |
|---|---|---|---|
| `POST /users/{userId}/profile` | Profile data: education, skills, experience, goals | Profile | Create/update candidate profile |
| `POST /jobs` | Job details, requirements, salary, etc. | Job ID + status | Create a job |
| `PATCH /jobs/{jobId}` | Fields to update / status | Updated job | Update, publish, or close a job |
| `GET /jobs` | Filters: keywords, skills, pay, experience, pagination | Paginated jobs | Job search |
| `GET /candidates` | Filters: keywords, skills, experience, pagination | Paginated candidates | Candidate search |
| `POST /jobs/{jobId}/applications` | Candidate ID | Application ID + status | Apply for a job |
| `GET /candidates/{candidateId}/applications` | Pagination/filter | Applications + statuses | Track applications |
| `PATCH /applications/{applicationId}/status` | New status | Updated application | Manage hiring pipeline |
| `GET /candidates/{candidateId}/recommendations` | Pagination | Ranked jobs | Get job recommendations |
| `GET /jobs/{jobId}/recommendations` | Pagination | Ranked candidates | Get candidate recommendations |
| `POST /candidates/{candidateId}/skill-gap` | Target job ID | Missing/improvable skills | Perform skill-gap analysis |

# High-Level Architecture
To handle the load and maintain transactional integrity and deliver fast search and recommendations, I chose a Modular Monolith transitioning to Microservices. I keep the core transactional operations-User Profiles, Job Management, and Applications under strong ACID guarantees, while I decouple high-throughput operations such as Search and background workloads like Matching, Skill-Gap Analysis, and Notifications through an Event Bus / Message Broker.

![High-Level Architecture](./High-Level%20Architecture.png)

-----
- [Deep Dives](./DeepDives.md)
- [Back-Of-The-Envelope Estimation](./Back-Of-The-Envelope.md)
