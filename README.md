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

## Approach

I chose PostgreSQL as the primary source of truth because the system has highly related and transactional data such as candidates, jobs, applications, and hiring states. It also allows us to enforce constraints, such as preventing a candidate from applying to the same job twice.

For search and filtering, I chose a search index such as Elasticsearch/OpenSearch because it is better suited for text search and filtering across jobs and candidate profiles.

For notifications, recommendations, and other non-critical background tasks, I chose an asynchronous message queue. These services consume events from the core system without becoming the source of truth.

## Core Entities

### User
Stores common account information and authentication data.

- UserID
- Name
- Email
- Role (Candidate / Employer)
- Notification preferences

### CandidateProfile
Stores candidate-specific information.

- CandidateID
- UserID
- Education
- Work experience
- Skills
- Career goals
- Portfolio
- Target roles
  
### Company
Represents the employer organization itself
CompanyID
Name
Industry
Location
Description/website

### Employer
Represents an individual employer-side user (e.g., a recruiter or hiring manager) belonging to a company.

- EmployerID
- UserID
- CompanyID (FK → Company)
- Role/title within company (optional, e.g., "Recruiter", "Admin")

### Job
Represents a job published by an employer.

- JobID
- CompanyID
- PostedByEmployerID
- Title
- Description
- Required skills
- Experience requirements
- Salary/pay range
- Location
- Eligibility criteria
- Status (Draft / Published / Closed)

### Application
Represents a candidate's application to a job.

- ApplicationID
- CandidateID
- JobID
- Current status
- CreatedAt
- UpdatedAt

A **unique constraint on `(CandidateID, JobID)`** ensures that a candidate cannot apply to the same job more than once.

### ApplicationStatusHistory
Stores the history of an application's state changes.

- HistoryID
- ApplicationID
- Previous status
- New status
- ChangedAt
- ChangedBy

This allows us to track the complete hiring pipeline and maintain an audit trail.

### Skill
A shared, normalized catalog of all known skills
- SkillID
- Skill name
- Category

### CandidateSkill
- CandidateID
- SkillID
- Skill level
  
### JobSkill
- JobID
- SkillID
- Requirement type
- Required level

This structure makes skill matching and skill-gap analysis easier.

### SkillGap 
Stores the result of comparing a candidate's skills against a target job/role

- CandidateID
- TargetJob/Role
- Missing skills
- Skills to improve
- Analysis timestamp

## Supporting Data

### Recommendation
Stores generated recommendations for candidates and employers.

- RecommendationID
- CandidateID / EmployerID
- JobID / CandidateID
- Score
- GeneratedAt

Recommendations can be regenerated asynchronously without affecting the core transactional data.

### Notification
Stores notification events and delivery status.

- NotificationID
- UserID
- Type
- Channel (Email / Push)
- Status
- CreatedAt
- SentAt
### One small architectural point
I chose to keep candidate, job, profile, and application data in the core relational database as the authoritative source of truth. And the recommendation system is not a separate source of truth. It consumes this data, calculates recommendations, and stores the results. This keeps the core data consistent while allowing recommendation logic to evolve independently

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
To handle Carieeer’s load, maintain transactional integrity, and deliver fast search and recommendations, I chose a **Modular Monolith transitioning to Microservices**. I keep the core transactional operations-User Profiles, Job Management, and Applications- under strong ACID guarantees, while I decouple high-throughput operations such as Search and background workloads like Matching, Skill-Gap Analysis, and Notifications through an Event Bus / Message Broker.

![High-Level Architecture](./High-Level%20Architecture.png)


