### **Project Requirement Document (PRD): Patient Notification System**

*   **Version:** v1.0.0
*   **Date:** 2025-09-21

### 1. Document Purpose

This document provides a definitive outline of the goals, scope, functional requirements (FRs), and non-functional requirements (NFRs) for the Patient Notification System. It is intended for all stakeholders, including project managers, engineers, and QA teams, to establish a common understanding of the system to be built. All development and testing shall be validated against the criteria defined herein.

### 2. Scope

#### 2.1. In Scope
*   The complete, automated backend system for ingesting patient appointment data.
*   The backend logic for processing and dispatching notifications for both scheduled and immediate events.
*   Integration with third-party services for the final delivery of Email and SMS messages.
*   The creation of a secure API for receiving real-time appointment change events.
*   All infrastructure definitions required to deploy the system on a Kubernetes cluster.
*   Monitoring and alerting for system health and critical failures.

#### 2.2. Out of Scope
*   User Interface (UI) for managing patient notification preferences.
*   Analytics dashboards for tracking notification engagement (open/click rates).
*   The third-party delivery services themselves (e.g., Twilio, AWS SES).
*   The external patient management system and its API.
*   Billing and cost-management systems.

### 3. Business Goals

*   **G-1 (Patient Adherence):** Reduce the rate of missed appointments by providing timely and reliable reminders.
*   **G-2 (Administrative Efficiency):** Eliminate manual effort associated with patient outreach for appointment reminders and changes.
*   **G-3 (Platform Foundation):** Establish a scalable and extensible communications platform that can be leveraged for future patient engagement initiatives.

### 4. Functional Requirements (FRs)

| ID | Requirement | Description | Acceptance Criteria / Metric |
| :--- | :--- | :--- | :--- |
| **FR-1** | Scheduled Appointment Reminders | The system shall dispatch reminder notifications for appointments scheduled **3 days in the future**. Reminders must only be sent within the **9:00 am to 12:00 pm** window. | The system correctly identifies all eligible appointments for a given day. Dispatch timestamps for all notifications fall within the specified 3-hour window. The system must handle timezone information if provided by the external API; otherwise, a configurable default timezone shall be used. |
| **FR-2** | Immediate Appointment Updates | The system shall process and dispatch notifications for ad-hoc appointment changes as soon as they are received. | For 95% of requests (P95), a notification shall be dispatched to the third-party gateway within **30 seconds** of being received by the system's API. |
| **FR-3** | Multi-Channel Capability | The system must be capable of dispatching notifications to a patient's **Email**, **SMS**, or both channels based on the contact information provided in the ingested data. | Given a patient with both an email and phone number, the system correctly enqueues a job for both the Email Worker and the SMS Worker. |
| **FR-4** | Automated Data Ingestion | The system shall periodically ingest patient and appointment data from the external API. This process must be automated and run without manual intervention. | The ingestion schedule is configurable (e.g., every 4 hours). A successful run populates the local database with all required fields: `patient_id`, `appointment_id`, `appointment_time`, `patient_email`, `patient_phone`, etc. |
| **FR-5** | Event-Driven API | The system shall provide a secure RESTful API endpoint to accept `POST` requests for immediate appointment updates. | The API endpoint is exposed via an Ingress. It validates incoming requests and, upon success, creates a job in the message queue. A valid request must contain `appointment_id`, `change_type`, and `patient_id`. |
| **FR-6** | Architectural Extensibility | The system's core components shall be designed to allow the addition of new notification channels without requiring modification of the core pipeline. | Adding a new channel (e.g., "Push Notification") only requires adding a new Worker deployment and a corresponding queue. The `API Gateway` and `Notification Dispatcher` logic shall not require changes. |

### 5. Non-Functional Requirements (NFRs)

| ID | Category | Requirement | Acceptance Criteria / Metric |
| :--- | :--- | :--- | :--- |
| **NFR-1**| Reliability | The system shall achieve a **99% successful handoff rate** to third-party delivery gateways for all notifications it attempts to send. | The system logs a "success" status for 99% of jobs consumed by the workers. This metric does not include final delivery to the end-user, which is outside system control. |
| **NFR-2**| Availability | The system's services, including the real-time API, shall maintain **99.9% uptime**. | The system's health checks pass 99.9% of the time over a rolling 30-day window. The system is deployed with N+2 redundancy (3 replicas) for all core components. |
| **NFR-3**| Resilience | The system must tolerate failures from the external data ingestion API (specified at 90% success rate) without impacting core operations. | A single failed ETL run does not prevent the Scheduler from successfully processing notifications using data from the last successful run. An alert is triggered if ETL fails **3 consecutive times**. |
| **NFR-4**| Observability | The system must provide actionable alerts for critical failures. | Alerts are configured and will trigger if: 1) Any core service is unavailable for >5 minutes. 2) The notification handoff rate (NFR-1) drops below 95% for >1 hour. 3) The message queue depth exceeds 10,000 messages. |
| **NFR-5**| Scalability | The system must process a daily batch of **8,000 appointments** efficiently and scale for larger future loads. | The entire daily batch of 8,000 scheduled reminders is processed and enqueued within **30 minutes** of the Scheduler's trigger. |
| **NFR-6**| Latency | Immediate notifications must be processed with minimal delay. | End-to-end processing latency from API Ingress to third-party gateway handoff shall be **under 30 seconds at the 95th percentile (P95)**. |
| **NFR-7**| Elasticity | All stateless services must automatically scale up and down to meet demand. | Horizontal Pod Autoscalers (HPAs) are configured for all Worker and Service deployments. They will trigger a scale-up when CPU utilization exceeds 75% and will scale down after a cool-down period. |
| **NFR-8**| Portability | The system must be deployable on any CNCF-compliant Kubernetes cluster. | All infrastructure is defined as Kubernetes manifests (`Deployment`, `StatefulSet`, `Ingress`, etc.) using a standard Helm chart. No cloud-provider-specific services (e.g., SQS, Lambda) are required for core functionality. |
| **NFR-9**| Security | The public-facing API must be secured against unauthorized access. | The API endpoint must enforce TLS 1.2 or higher. Access to the API must be protected by a validated API Key or equivalent token-based authentication mechanism. |
