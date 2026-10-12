Act as a Senior Full-Stack Software Engineer, Application Security Engineer, and Code Reviewer.

I am migrating a legacy Gait Lab application into a new application using:

* Frontend: Angular, TypeScript, HTML, CSS
* Backend: Spring Boot 3.3.5, Java 17
* Database: PostgreSQL
* Authentication: Currently under development
* Environment: Internal hospital application with some external users

I want you to perform a complete codebase audit of both frontend and backend.

IMPORTANT:

* Skip the Visit Home module and Videos module entirely for now.
* Do not modify existing functionality unless there is a confirmed issue.
* Do not modify the database schema or existing data.
* Do not automatically fix anything. First identify and explain the problems.
* Review actual code, configurations, dependencies, SQL queries, and API integrations. Do not assume something is working without verifying it.

1. Frontend Code Review

Inspect all Angular components, services, routes, guards, models, and configuration files.

Check for:

* TypeScript compilation errors
* HTML template errors
* CSS conflicts
* Incorrect imports or missing dependencies
* Undefined or null values
* Broken navigation and routing
* Incorrect API calls or endpoint mappings
* Incorrect data binding
* Form validation problems
* Subscription and memory leaks
* Duplicate or unnecessary code
* Incorrect error handling
* Problems with loading states
* Potential runtime errors
* Unused variables, functions, and imports

Also check whether UI functionality is consistent across modules.

2. Backend Code Review

Inspect every relevant Spring Boot controller, service, repository, entity, DTO, configuration, and exception handler.

Check for:

* Java compilation errors
* Incorrect Spring annotations
* Missing dependency injection
* Repository query errors
* Incorrect entity relationships
* Incorrect database column mappings
* NullPointerException risks
* Incorrect request and response handling
* Missing input validation
* Duplicate business logic
* Missing transaction management
* Improper exception handling
* Sensitive information exposed in logs
* API endpoints without proper access restrictions
* Database connection and configuration problems

Ensure frontend API requests match the backend endpoints and response models.

3. Login, Authentication, and Security Review — HIGH PRIORITY

Perform a detailed security audit of the current login implementation and any planned or partially implemented authentication code.

Check:

* Whether login genuinely authenticates users on the backend
* Whether protected APIs can be accessed without authentication
* Whether passwords are stored or transmitted insecurely
* Whether password hashing is implemented correctly
* Whether authentication secrets are hardcoded
* Whether Angular route guards are present and sufficient for UI navigation
* Whether authorization is enforced on the backend
* Whether users can access another user’s restricted functionality
* Whether role-based access control is properly implemented
* Whether admin endpoints are protected
* Session management and expiration
* Cookie security, including HttpOnly, Secure, and SameSite
* CSRF protection where applicable
* CORS configuration
* XSS vulnerabilities
* SQL injection vulnerabilities
* Insecure direct object references (IDOR)
* Sensitive data exposure
* Missing security headers
* Brute-force login protection
* Error messages that reveal sensitive information
* Account lockout and logout behavior
* Session fixation risks
* Security risks affecting external users

Evaluate whether the current implementation would be secure enough for a hospital application handling sensitive patient data.

If authentication is incomplete, clearly identify what is missing instead of assuming it exists.

Also evaluate the suitability of server-side session authentication versus JWT for the application’s architecture. Do not implement either approach without approval.

4. Database and Patient Data Security

Review PostgreSQL queries, repositories, entities, and data access logic.

Check:

* SQL injection risks
* Incorrect joins
* Query performance issues
* Missing parameterized queries
* Incorrect type conversions
* Unnecessary exposure of patient data
* Improper update and delete operations
* Missing transactional integrity
* Potential data duplication
* Unauthorized patient record access
* Risks of overwriting existing data

Preserve the existing database structure and any intentionally stored text-based dates.

5. Module-by-Module Review

Review the implemented modules, including:

* Login and authentication
* Patient Search
* Patient Demographics
* Visits List
* Schedule New Visit
* Forms and PDF generation
* History
* Questionnaire Selection
* Questionnaire Welcome
* Questionnaire Runner
* First Visit, Adult, Children, Hip, and Sports Questionnaires
* Questionnaire Responses
* Admin
* User Management
* Lookup Tables
* Other existing implemented functionality

Exclude Visit Home and Videos.

For each module, verify frontend behavior, backend integration, data validation, exception handling, and security.

Clearly distinguish completed functionality from incomplete or placeholder implementations.

6. Dependencies and Configuration

Review:

* Angular and Node.js compatibility
* Java 17 and Spring Boot compatibility
* Maven dependencies
* npm package dependencies
* Known vulnerable dependencies
* Environment configuration
* Hardcoded database credentials
* API URLs
* Secrets and API keys
* Production build settings
* Logging configuration
* Development-only features accidentally enabled in production

7. Build and Testing

Where the environment supports it, run appropriate non-destructive validation commands, including:

* Angular TypeScript checks
* Angular production build
* Available frontend tests
* Maven compile
* Available backend tests
* Dependency vulnerability checks

Do not start destructive tests, modify database records, expose secrets, or change production configurations.

If a test cannot run, state the exact reason.

Do not claim a test passed unless it actually executed successfully.

8. Final Audit Report

After reviewing the application, provide a structured report with these columns:

| Priority | Module | File Path | Line Number | Issue | Impact | Recommended Fix |

Classify each issue as:

* CRITICAL — Security vulnerability or major data exposure
* HIGH — Broken functionality, serious security risk, or data integrity problem
* MEDIUM — Potential runtime failure, validation issue, or performance problem
* LOW — Minor bugs, maintainability, or code quality concerns

For each confirmed issue:

1. Explain the exact problem.
2. Identify the actual file and line number.
3. Explain when and how the issue occurs.
4. Describe the impact.
5. Recommend the safest fix.
6. Identify any dependencies or other files affected.

Separate confirmed problems from possible risks requiring additional verification.

At the end, provide:

1. Overall application health assessment.
2. Authentication and security assessment.
3. Frontend issues summary.
4. Backend issues summary.
5. Database and patient data security assessment.
6. Missing or incomplete functionality.
7. Recommended fixes in priority order.
8. Suggested testing before deployment.
9. Any blockers for production readiness.

Do not make code changes yet.

First complete the audit and present the report. Once I review the findings, we will resolve problems one by one, starting with critical security vulnerabilities and authentication issues.

Do not generate unnecessary new files, restructure the project, or replace working code without approval.
