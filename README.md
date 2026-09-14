## Functional Requirements
- Users can create profiles and specify their information.
- Employers can create, publish, update, and close job listings.
- Candidates can apply for jobs and track their application status.
- Candidates and employers can query items with multi-attribute filtering (skills, pay range, experience level, text keywords).
- The system asynchronously generates ranked job recommendations for candidates and candidate leads for employers.
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

### Employer
Stores employer/company information.

- EmployerID
- UserID
- Company information
- Industry
- Location

### Job
Represents a job published by an employer.

- JobID
- EmployerID
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

This allows us to track the complete hiring pipeline and maintain an audit trail.

### Skill & CandidateSkill
Skills are modeled separately so they can be shared across candidates and jobs.

- SkillID
- Skill name
- CandidateID
- Skill level

### JobSkill
Represents the skills required or preferred for a job.

- JobID
- SkillID
- Requirement type
- Required level

This structure makes skill matching and skill-gap analysis easier.

### SkillGap / CareerGoal
Stores a candidate's target role and the result of comparing their current skills with the target job requirements.

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
