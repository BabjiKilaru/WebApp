Gait Lab Application Migration - Daily Development & Activity Log
July 27, 2026 - September 28, 2026
Date	Day	Work Performed / Activities
07/27/2026	Monday	•	Joined Nemours Children’s Health and began onboarding for the Gait Lab migration project.
•	Attempted to log in to the assigned Nemours laptop and was unable to access the system.
•	Opened a support ticket in MyTech for the laptop login issue.
•	Coordinated with IT regarding troubleshooting and next steps.
07/28/2026	Tuesday	•	Completed required onboarding and compliance courses in Nemours University while waiting for the laptop issue to be resolved.
•	IT determined that the laptop needed to be returned for repair/reconfiguration.
•	Coordinated with Candi Moore to send the laptop back to IT and shipped it the same day.
•	Requested updated signatures on Form I-983 from Jackie and Chris because the work location was in Delaware while I was still located in Kansas City.
07/29/2026	Wednesday	•	Spoke with Joan Calpin regarding New Hire Orientation and scheduled orientation for August 10 and August 11.
•	Candi received the laptop and sent it to the IT team for repair/reconfiguration.
•	After IT completed the work, Candi received the laptop back and dropped it off at FedEx for shipment to me.
•	Received the signed updated I-983 from Jackie and forwarded it to Chris for his signature.
•	Requested installation of the development software required for the project.
•	IT installed applications available through the Software Center and advised that separate support tickets were required for software not available there.
07/30/2026	Thursday	•	Tracked shipment of the repaired/configured laptop.
•	Reviewed development software and access needed for the Gait Lab project.
•	Prepared additional IT support requests for software not available through the Software Center.
•	Continued onboarding activities while waiting for the laptop to arrive.
07/31/2026	Friday	•	Received the repaired/configured Nemours laptop.
•	Successfully began accessing the Nemours corporate environment.
•	Connected with Chris Church.
•	Chris introduced me to Kam Hazelwood, the previous developer who had worked on the new Gait Lab application.
•	Started discussing the existing project, prior development work, source-code location, and overall migration effort.
08/03/2026	Monday	•	Chris contacted Michelle West regarding access to Kam’s email account so the team could access the GitHub account containing the new application source code.
•	Discussed the new GOAL questionnaire requirement and how it would fit into the migration project.
•	Received access to Kam’s email.
•	Successfully signed in to the existing GitHub account/repository.
•	Began reviewing the available new-application source code and previous development work.
08/04/2026	Tuesday	•	Opened IT service tickets for development software and tools not available through the Software Center.
•	Reviewed the legacy Gait Lab website with Chris.
•	Chris demonstrated the major legacy application functions and workflows.
•	Identified functionality that needed to be retained or recreated in the new application.
•	Began documenting legacy modules and migration requirements.
08/05/2026	Wednesday	•	Chris contacted Christopher Pennington to arrange a discussion regarding the legacy Gait Lab application.
•	Chris introduced me to Beatriz, the server team lead.
•	Started discussing infrastructure requirements for the new application.
•	Identified the need for new server resources and coordination with the server/infrastructure team.
•	Continued reviewing legacy application dependencies and technical questions.
08/06/2026	Thursday	•	Scheduled a meeting for August 13 with Beatriz/server team to discuss the new server requirements.
•	Worked with Jones O'Ronde on installation of required development software and tools.
•	Reviewed remaining software dependencies.
•	Followed up on open service requests needed to complete the development environment.
08/07/2026	Friday	•	Continued working on development workstation preparation.
•	Worked on software installation and configuration.
•	Followed up on IT support tickets.
•	Verified installed applications.
•	Identified remaining software or access requirements needed for development.
08/10/2026	Monday	•	Attended New Hire Orientation - Day 1.
•	Completed required orientation sessions and organizational onboarding activities.
•	Continued tracking outstanding development-environment and access requirements.
08/11/2026	Tuesday	•	Attended New Hire Orientation - Day 2.
•	Completed the scheduled New Hire Orientation program.
•	Reviewed remaining development setup, source-code access, and software requirements.
08/12/2026	Wednesday	•	Continued development workstation setup.
•	Worked on software installation and configuration.
•	Followed up on outstanding IT service requests.
•	Verified access to available development tools.
•	Prepared for the upcoming server/infrastructure discussion.
08/13/2026	Thursday	•	Participated in the scheduled meeting with Beatriz and the server/infrastructure team.
•	Discussed server requirements for the new Gait Lab application.
•	Reviewed the need for a new development/application server.
•	Discussed access requirements, environment requirements, deployment considerations, and infrastructure coordination.
•	Identified next steps for provisioning and preparing the server environment.
08/14/2026	Friday	•	Continued migration planning after the infrastructure discussion.
•	Reviewed database integration requirements and application dependencies.
•	Reviewed how existing PostgreSQL tables could be reused in the new application.
•	Documented technical assumptions and areas where legacy business rules needed additional verification.
08/17/2026	Monday	•	Defined the primary technology direction for the new application: Angular frontend, Java 17/Spring Boot backend, and PostgreSQL database.
•	Reviewed how the frontend and backend would communicate through REST APIs.
•	Planned how the new application could be deployed independently of the legacy JSP/Struts application.
•	Continued mapping legacy functionality to the new architecture.
08/18/2026	Tuesday	•	Created and configured the Spring Boot backend project.
•	Configured Spring Boot 3.3.5, Java 17, and Maven.
•	Verified that the initial Spring Boot application started successfully.
•	Reviewed dependency configuration.
•	Organized backend structure for controllers, services, repositories, models/entities, and configuration.
08/19/2026	Wednesday	•	Continued backend architecture and API planning.
•	Reviewed how patient, visit, diagnosis, scheduling, and questionnaire functionality should be exposed to the Angular frontend.
•	Mapped legacy operations to REST API endpoints.
•	Reviewed validation, service-layer logic, and database-access patterns.
08/20/2026	Thursday	•	Continued development of the Angular/Spring Boot application structure.
•	Organized frontend pages/components for patient-related workflows.
•	Reviewed backend responsibilities for patient and visit functionality.
•	Reviewed routing/navigation requirements and overall application structure.
08/21/2026	Friday	•	Continued migration development.
•	Reviewed PostgreSQL entity/table requirements for patient and visit functionality.
•	Investigated patient identifiers, visit relationships, and fields required by the frontend.
•	Planned patient search and patient-detail retrieval through the new backend.
08/24/2026	Monday	•	Continued Angular and Spring Boot integration work.
•	Reviewed how frontend forms and tables would consume backend APIs.
•	Worked on patient and visit data flow.
•	Identified database queries needed for patient search and patient-detail retrieval.
•	Compared new workflows with the legacy application.
08/25/2026	Tuesday	•	Worked on the new on-premises VM/server environment.
•	Confirmed server access and sudo privileges.
•	Reviewed hostname/IP, authentication method, SSH requirements, VPN connectivity, Java/runtime requirements, PostgreSQL connectivity, ports, disk space, and memory.
•	Began documenting deployment prerequisites for the new application.
08/26/2026	Wednesday	•	Continued server/environment configuration and deployment planning.
•	Reviewed how Angular frontend and Spring Boot backend builds would be deployed to the new VM.
•	Reviewed application ports, environment configuration, database connection settings, logging, and future HTTPS requirements.
08/27/2026	Thursday	•	Worked on patient-management functionality.
•	Reviewed patient search by MRN, last name, and first name.
•	Worked on patient-detail loading, demographics, visits, and related navigation.
•	Reviewed New Patient requirements and demographic fields.
•	Prepared technical questions for discussion with Bill, the original legacy application/database developer.
08/28/2026	Friday	•	Continued legacy system analysis using information gathered from previous developers and stakeholders.
•	Reviewed patient file/image handling.
•	Reviewed report-generation logic and database relationships.
•	Reviewed background services, Perl scripts, scheduled imports, and external dependencies.
•	Identified functionality to migrate directly versus functionality that could be redesigned.
08/31/2026	Monday	•	Continued development of patient and visit modules.
•	Connected frontend functionality with Spring Boot backend endpoints and PostgreSQL data.
•	Reviewed patient search behavior, visit information, New Patient functionality, and General Diagnosis requirements.
•	Worked through data-mapping and user-interface issues.
09/01/2026	Tuesday	•	Continued frontend/backend integration and testing.
•	Worked on patient search, demographics, visits, and database retrieval.
•	Reviewed case-insensitive patient-search behavior.
•	Reviewed MRN-to-patient-ID relationships.
•	Verified that updates from Angular reached Spring Boot APIs and were reflected in PostgreSQL.
09/02/2026	Wednesday	•	Established a stable baseline for the Angular frontend and Spring Boot backend.
•	Verified working patient search and demographics functionality.
•	Verified visits list and schedule-new-visit functionality.
•	Verified new-visit test selection persistence.
•	Verified New Patient creation.
•	Verified General Diagnosis editing and ICD9 mapping.
•	Confirmed Angular needed to run with the configured proxy for backend API routing.
09/03/2026	Thursday	•	Continued refining patient and visit features.
•	Worked on UI alignment and field behavior.
•	Reviewed validation and database updates.
•	Refined visit-list presentation and scheduling workflow.
•	Verified saved data refreshed correctly in the frontend.
09/04/2026	Friday	•	Continued application testing and stabilization.
•	Reviewed patient search workflow.
•	Tested demographic edits, visit retrieval, new-visit scheduling, and New Patient creation.
•	Corrected frontend/backend integration issues.
•	Prepared for questionnaire and historical-data development.
09/07/2026	Monday	•	Paid Holiday - No project work performed.
09/08/2026	Tuesday	•	Continued questionnaire-response module development.
•	Reviewed how baseline responses should be represented in the new application.
•	Reviewed how visit-specific questionnaire responses should be displayed.
•	Analyzed questionnaire response categories, visit-date selection, edit behavior, and historical response viewing.
09/09/2026	Wednesday	•	Redesigned the Questionnaire Responses page.
•	Created Baseline Answers and Per Visit Answers sections.
•	Consolidated per-visit answers, seizure medication, and questionnaire details into a single per-visit view.
•	Worked on PT Frequency, OT Frequency, devices/braces, gait concerns, pain, follow-up date, and general concerns.
•	Refined edit-window behavior, fixed-height sections, internal scrolling, dropdown behavior, and UI consistency.
•	Prepared a concise monthly project update for the team meeting.
09/10/2026	Thursday	•	Performed detailed analysis of the legacy questionnaire implementation.
•	Reviewed History, Adult, Children, Hip, and Sport questionnaire types.
•	Reviewed English and Spanish questionnaire versions.
•	Investigated field naming such as q2_ha_*, q2_sr_*, and q2_ch_*.
•	Determined where questions and answer options were stored or hard-coded.
•	Began analyzing the Admin/Print Patient Visit Forms workflow.
09/11/2026	Friday	•	Continued analysis of the legacy printable-form and report workflow.
•	Reviewed how patient and visit information is populated into printable forms.
•	Reviewed Analysis Information sheets and PT evaluation forms.
•	Investigated how HTML/JSP output is converted to PDF.
•	Reviewed the role of legacy scripts/services in report generation.
09/14/2026	Monday	•	Performed a broader frontend/backend project review.
•	Checked Angular project compatibility and build issues.
•	Reviewed backend configuration.
•	Separated issues requiring fixes from intentionally deferred items such as authentication and unfinished questionnaire functionality.
•	Continued work on printable-form preview and legacy Admin print workflow analysis.
09/15/2026	Tuesday	•	Worked on GitHub Copilot licensing and development-tool approval.
•	Contacted the Help Desk regarding the license.
•	Confirmed the Help Desk could not directly provide the required recurring payment/license approval.
•	Created an internal support ticket.
•	Coordinated with management/IT regarding Finance/Harmony approval requirements.
•	Prepared the technical/business justification for GitHub Copilot.
09/16/2026	Wednesday	•	Redesigned the Patient Demographics page.
•	Integrated demographics and patient visits into the same screen using side-by-side panels.
•	Moved Save/Cancel controls inside the demographics section.
•	Adjusted validation and error-message placement.
•	Added patient-selection empty states.
•	Aligned visit-panel height with demographics.
•	Reduced unnecessary spacing.
•	Adjusted General Diagnosis and ICD9 field widths.
•	Restored patient-loaded feedback.
•	Implemented temporary success/error messaging.
•	Ran Angular frontend build/test checks after the UI changes.
09/17/2026	Thursday	•	Continued application development.
•	Reviewed Microsoft 365 Copilot capabilities.
•	Reviewed Microsoft Copilot Studio.
•	Compared Microsoft 365 Copilot/Copilot Studio with GitHub Copilot for development use.
•	Considered AI-agent use cases for development assistance, documentation, and internal workflows.
09/18/2026	Friday	•	Worked on integrating questionnaire functionality from existing questionnaire source code into the current application.
•	Reviewed questionnaire Java/JSP/backend files.
•	Identified changes required for integration with the Angular/Spring Boot project.
•	Analyzed logic that determines which questionnaires are shown.
•	Reviewed General History, Gait History, first-visit questions, and questionnaire-selection conditions.
09/21/2026	Monday	•	Performed PostgreSQL data work to support development and testing.
•	Worked on copying data from an existing table/database into the local/development database.
•	Reviewed source and destination table schemas and column mappings.
•	Resolved SQL/data-transfer issues.
•	Verified copied records in pgAdmin and through SQL queries.
09/22/2026	Tuesday	•	Continued database integration and application testing after the data-copy work.
•	Verified required patient and visit records could be queried by the application.
•	Reviewed visit relationships and scheduling data.
•	Continued frontend/backend validation using local/development data.
09/23/2026	Wednesday	•	Worked on scheduling-page functionality and frontend design.
•	Reviewed a calendar-based schedule view where a date could be selected and patients for that day displayed in an agenda-style list.
•	Refined patient rows to focus on Name, MRN, and Visit Type.
•	Simplified the screen to show a total appointment count.
•	Reviewed Microsoft Copilot Studio and possible AI-agent use cases for development and operational support.
09/24/2026	Thursday	•	Worked on legacy application SSL certificate renewal.
•	Reviewed the existing SSL configuration on the Linux server.
•	Identified certificate/key files and certificate-chain requirements.
•	Investigated the required GlobalSign intermediate certificate.
•	Updated/replaced the server certificate.
•	Reloaded/restarted the necessary service.
•	Validated that the legacy application continued to work correctly over HTTPS.
•	Reviewed Microsoft 365 options for architecture diagrams.
•	Performed SQL/database queries needed for application analysis.
09/25/2026	Friday	•	Completed SSL renewal follow-up and validation.
•	Verified the renewed certificate was functioning correctly.
•	Cleaned up unnecessary backup files while retaining the appropriate previous certificate backup.
•	Documented the SSL renewal procedure for future use.
•	Included certificate preparation, backup, server replacement, service reload/restart, and validation steps in the documentation.
•	Communicated completion of the SSL update to IT/management.
•	Continued GitHub Copilot approval work and security/intake documentation.
09/28/2026	Monday	•	Focused on formal Gait Lab migration/project documentation.
•	Defined the Project and Migration Overview document.
•	Defined legacy and target architecture documentation.
•	Defined database/table mapping documentation.
•	Defined module and feature documentation.
•	Defined business rules and decision-log documentation.
•	Defined development change-log and known-issues tracking.
•	Created an ongoing daily development/activity log structure that can be maintained throughout development, testing, deployment, and future support.
<img width="601" height="633" alt="image" src="https://github.com/user-attachments/assets/77392a88-007e-43e2-982b-124eaf8f480e" />
