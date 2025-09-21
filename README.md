# Patient Notification System

A scalable, cloud-native patient notification system designed to send scheduled reminders and immediate appointment updates via SMS and Email.

## Project Overview

### Requirements

- Send patients reminders about their upcoming appointments (3 days before, from 9am to 12pm)
    - SMS, Email or both
- Update patients about changes (time/venue) to their appointments (~100 a day)
    - Must be done immediately

### Performance Targets

- On average 8,000 appointments in a day
- 99% delivery rate
- Alerts on any failure

### Limitations

- Patient appointment and contact details are obtained via API
    - External System
    - 90% success rate

## Final Architecture Overview

The final design is a decoupled, event-driven system built on a microservices architecture. It is designed to be highly available, scalable, and portable across any standard Kubernetes environment.

**Key Components:**
- **Data Ingestion:** A `CronJob`-based **ETL Service** periodically fetches data from an external API into a local PostgreSQL database, decoupling the system from external dependencies.
- **Trigger Mechanisms:**
    - A **Scheduler** (`CronJob`) triggers scheduled batch reminders.
    - An **API Gateway** (`Ingress`) provides an entry point for immediate, event-driven notifications.
- **Core Pipeline:** A central **Notification Dispatcher** routes notification jobs to the correct **Message Queue** (RabbitMQ).
- **Delivery Workers:** Dedicated, auto-scaling **Email** and **SMS Workers** consume jobs from the queues and communicate with third-party delivery services.

## Technology Stack

- **Orchestration:** Kubernetes
- **Messaging:** RabbitMQ
- **Database:** PostgreSQL
- **Architecture Style:** Microservices, Event-Driven
- **Design Language:** Mermaid (for Architecture-as-Code)

## Logical View (C4 Component Diagram)

### Milestone 01: Core Notification Pipeline for Scheduled Reminders (Email)

```mermaid
C4Component
    title Component Diagram for Core Email Notification Pipeline (v1.3)

    System_Ext(patient_db, "Appointment Database", "Stores patient and appointment info.")
    System_Ext(third_party_email, "Third-Party Email Service", "e.g., AWS SES, SendGrid")

    Container_Boundary(c1, "Notification System") {
        Component(scheduler, "Scheduler", "CronJob/Scheduled Task", "Triggers the notification process daily at 9am.")
        Component(appointment_service, "Appointment Service", "Container (e.g., Go, Python)", "Fetches appointments scheduled 3 days from now.")
        Component(dispatcher, "Notification Dispatcher", "Container (e.g., Go, Python)", "Formats notifications and places them on the queue.")
        ComponentQueue(queue, "Message Queue", "RabbitMQ / SQS", "Buffers notifications to decouple components and handle load.")
        Component(email_worker, "Email Worker", "Container (e.g., Go, Python)", "Processes messages from the queue and sends emails.")
    }

    %% --- Relationships (Re-ordered to guide renderer) ---
    Rel(scheduler, appointment_service, "1. Triggers")
    Rel(appointment_service, dispatcher, "3. Forwards appointments to")
    Rel(dispatcher, queue, "4. Enqueues email job")
    Rel(queue, email_worker, "5. Delivers job to")
    
    %% External System Relationships
    Rel(appointment_service, patient_db, "2. Fetches data from")
    Rel(email_worker, third_party_email, "6. Sends email via")
```

### Milestone 02: Add SMS as a Second Notification Channel

```mermaid
C4Component
    title Component Diagram with Email and SMS Channels (v2.1)

    System_Ext(patient_db, "Appointment Database", "Stores patient and appointment info.")
    System_Ext(third_party_email, "Third-Party Email Service", "e.g., AWS SES, SendGrid")
    System_Ext(third_party_sms, "Third-Party SMS Service", "e.g., Twilio, SNS")

    Container_Boundary(c1, "Notification System") {
        Component(scheduler, "Scheduler", "CronJob/Scheduled Task", "Triggers the notification process daily at 9am.")
        Component(appointment_service, "Appointment Service", "Container", "Fetches appointments scheduled 3 days from now.")
        Component(dispatcher, "Notification Dispatcher", "Container", "Determines channel and enqueues jobs.")
        
        ComponentQueue(email_queue, "Email Queue", "RabbitMQ Topic/Queue", "Buffers email notifications.")
        Component(email_worker, "Email Worker", "Container", "Processes messages and sends emails.")
        
        ComponentQueue(sms_queue, "SMS Queue", "RabbitMQ Topic/Queue", "Buffers SMS notifications.")
        Component(sms_worker, "SMS Worker", "Container", "Processes messages and sends SMS.")
    }

    %% --- Relationships ---
    Rel(scheduler, appointment_service, "1. Triggers")
    Rel(appointment_service, patient_db, "2. Fetches data from")
    Rel(appointment_service, dispatcher, "3. Forwards appointments to")
    
    Rel(dispatcher, email_queue, "4a. Enqueues email job")
    Rel(email_queue, email_worker, "5a. Delivers job to")
    Rel(email_worker, third_party_email, "6a. Sends email via")

    Rel(dispatcher, sms_queue, "4b. Enqueues SMS job")
    Rel(sms_queue, sms_worker, "5b. Delivers job to")
    Rel(sms_worker, third_party_sms, "6b. Sends SMS via")
```

### Milestone 03: Implement Immediate Notifications for Appointment Changes

```mermaid
C4Component
    title Component Diagram with Immediate Notification API (v3.2)

    System_Ext(patient_db, "Appointment Database", "Stores patient and appointment info.")
    System_Ext(third_party_email, "Third-Party Email Service", "e.g., AWS SES, SendGrid")
    System_Ext(third_party_sms, "Third-Party SMS Service", "e.g., Twilio, SNS")
    System_Ext(appointment_manager, "External Appointment Mgt System", "Triggers immediate updates.")

    Container_Boundary(c1, "Notification System") {
        Component(api_gateway, "API Gateway", "Container / Ingress", "Receives real-time notification requests.")
        Component(scheduler, "Scheduler", "CronJob", "Triggers the scheduled notification process.")
        Component(appointment_service, "Appointment Service", "Container", "Fetches appointments for scheduled jobs.")
        Component(dispatcher, "Notification Dispatcher", "Container", "Determines channel and enqueues jobs.")
        
        ComponentQueue(email_queue, "Email Queue", "RabbitMQ", "Buffers email notifications.")
        Component(email_worker, "Email Worker", "Container", "Processes and sends emails.")
        
        ComponentQueue(sms_queue, "SMS Queue", "RabbitMQ", "Buffers SMS notifications.")
        Component(sms_worker, "SMS Worker", "Container", "Processes and sends SMS.")
    }

    %% --- Relationships ---
    
    %% Flow A: Immediate Notification
    Rel(appointment_manager, api_gateway, "A.1. Sends change event")
    Rel(api_gateway, dispatcher, "A.2. Forwards request to")

    %% Flow B: Scheduled Notification
    Rel(scheduler, appointment_service, "B.1. Triggers")
    Rel(appointment_service, patient_db, "B.2. Fetches data from")
    Rel(appointment_service, dispatcher, "B.3. Forwards appointments to")
    
    %% Common Delivery Sub-System (Called by both flows via Dispatcher)
    Rel(dispatcher, email_queue, "C.1a. Enqueues email job")
    Rel(email_queue, email_worker, "C.2a. Delivers job to")
    Rel(email_worker, third_party_email, "C.3a. Sends email via")

    Rel(dispatcher, sms_queue, "C.1b. Enqueues SMS job")
    Rel(sms_queue, sms_worker, "C.2b. Delivers job to")
    Rel(sms_worker, third_party_sms, "C.3b. Sends SMS via")
```

### Milestone 04: External API Integration and Data Ingestion

```mermaid
C4Component
    title Component Diagram with Data Ingestion ETL (v4.2)

    %% External Systems
    System_Ext(external_api, "External Patient API", "Source of patient and appointment data.")
    System_Ext(third_party_email, "Third-Party Email Service", "e.g., AWS SES, SendGrid")
    System_Ext(third_party_sms, "Third-Party SMS Service", "e.g., Twilio, SNS")
    System_Ext(appointment_manager, "External Appointment Mgt System", "Triggers immediate updates.")

    Container_Boundary(c1, "Notification System") {
        Component(etl_service, "ETL Service", "CronJob", "Periodically extracts data from the External API.")
        ComponentDb(local_db, "Appointment Database", "PostgreSQL", "Local, cached copy of appointment data.")

        Component(api_gateway, "API Gateway", "Ingress", "Receives real-time notification requests.")
        Component(scheduler, "Scheduler", "CronJob", "Triggers the scheduled notification process.")
        Component(appointment_service, "Appointment Service", "Container", "Fetches appointments from the LOCAL database.")
        Component(dispatcher, "Notification Dispatcher", "Container", "Determines channel and enqueues jobs.")
        
        ComponentQueue(email_queue, "Email Queue", "RabbitMQ", "Buffers email jobs.")
        Component(email_worker, "Email Worker", "Container", "Processes and sends emails.")
        
        ComponentQueue(sms_queue, "SMS Queue", "RabbitMQ", "Buffers SMS jobs.")
        Component(sms_worker, "SMS Worker", "Container", "Processes and sends SMS.")
    }

    %% --- Relationships (Corrected and Re-ordered) ---
    
    %% Flow 1: Data Ingestion (ETL)
    Rel(etl_service, external_api, "1. Fetches data from")
    Rel(etl_service, local_db, "2. Writes data to")

    %% Flow 2: Scheduled Notifications
    Rel(scheduler, appointment_service, "A. Triggers")
    Rel(appointment_service, local_db, "B. Reads appointments from")
    Rel(appointment_service, dispatcher, "C. Forwards to")
    
    %% Flow 3: Immediate Notifications
    Rel(appointment_manager, api_gateway, "D. Sends change event")
    Rel(api_gateway, dispatcher, "E. Forwards request to")

    %% Flow 4: Common Delivery Pipeline
    Rel(dispatcher, email_queue, "F1. Enqueues email job")
    Rel(email_queue, email_worker, "G1. Delivers job to")
    Rel(email_worker, third_party_email, "H1. Sends email via")

    Rel(dispatcher, sms_queue, "F2. Enqueues SMS job")
    Rel(sms_queue, sms_worker, "G2. Delivers job to")
    Rel(sms_worker, third_party_sms, "H2. Sends SMS via")
```

### Milestone 05: High Availability, Scalability, and Performance (remain the same)

```mermaid
C4Component
    title Component Diagram with Data Ingestion ETL (v4.2)

    %% External Systems
    System_Ext(external_api, "External Patient API", "Source of patient and appointment data.")
    System_Ext(third_party_email, "Third-Party Email Service", "e.g., AWS SES, SendGrid")
    System_Ext(third_party_sms, "Third-Party SMS Service", "e.g., Twilio, SNS")
    System_Ext(appointment_manager, "External Appointment Mgt System", "Triggers immediate updates.")

    Container_Boundary(c1, "Notification System") {
        Component(etl_service, "ETL Service", "CronJob", "Periodically extracts data from the External API.")
        ComponentDb(local_db, "Appointment Database", "PostgreSQL", "Local, cached copy of appointment data.")

        Component(api_gateway, "API Gateway", "Ingress", "Receives real-time notification requests.")
        Component(scheduler, "Scheduler", "CronJob", "Triggers the scheduled notification process.")
        Component(appointment_service, "Appointment Service", "Container", "Fetches appointments from the LOCAL database.")
        Component(dispatcher, "Notification Dispatcher", "Container", "Determines channel and enqueues jobs.")
        
        ComponentQueue(email_queue, "Email Queue", "RabbitMQ", "Buffers email jobs.")
        Component(email_worker, "Email Worker", "Container", "Processes and sends emails.")
        
        ComponentQueue(sms_queue, "SMS Queue", "RabbitMQ", "Buffers SMS jobs.")
        Component(sms_worker, "SMS Worker", "Container", "Processes and sends SMS.")
    }

    %% --- Relationships (Corrected and Re-ordered) ---
    
    %% Flow 1: Data Ingestion (ETL)
    Rel(etl_service, external_api, "1. Fetches data from")
    Rel(etl_service, local_db, "2. Writes data to")

    %% Flow 2: Scheduled Notifications
    Rel(scheduler, appointment_service, "A. Triggers")
    Rel(appointment_service, local_db, "B. Reads appointments from")
    Rel(appointment_service, dispatcher, "C. Forwards to")
    
    %% Flow 3: Immediate Notifications
    Rel(appointment_manager, api_gateway, "D. Sends change event")
    Rel(api_gateway, dispatcher, "E. Forwards request to")

    %% Flow 4: Common Delivery Pipeline
    Rel(dispatcher, email_queue, "F1. Enqueues email job")
    Rel(email_queue, email_worker, "G1. Delivers job to")
    Rel(email_worker, third_party_email, "H1. Sends email via")

    Rel(dispatcher, sms_queue, "F2. Enqueues SMS job")
    Rel(sms_queue, sms_worker, "G2. Delivers job to")
    Rel(sms_worker, third_party_sms, "H2. Sends SMS via")
```

### Overall Logical View

```mermaid
C4Component
    title Component Diagram with Data Ingestion ETL (v4.2)

    %% External Systems
    System_Ext(external_api, "External Patient API", "Source of patient and appointment data.")
    System_Ext(third_party_email, "Third-Party Email Service", "e.g., AWS SES, SendGrid")
    System_Ext(third_party_sms, "Third-Party SMS Service", "e.g., Twilio, SNS")
    System_Ext(appointment_manager, "External Appointment Mgt System", "Triggers immediate updates.")

    Container_Boundary(c1, "Notification System") {
        Component(etl_service, "ETL Service", "CronJob", "Periodically extracts data from the External API.")
        ComponentDb(local_db, "Appointment Database", "PostgreSQL", "Local, cached copy of appointment data.")

        Component(api_gateway, "API Gateway", "Ingress", "Receives real-time notification requests.")
        Component(scheduler, "Scheduler", "CronJob", "Triggers the scheduled notification process.")
        Component(appointment_service, "Appointment Service", "Container", "Fetches appointments from the LOCAL database.")
        Component(dispatcher, "Notification Dispatcher", "Container", "Determines channel and enqueues jobs.")
        
        ComponentQueue(email_queue, "Email Queue", "RabbitMQ", "Buffers email jobs.")
        Component(email_worker, "Email Worker", "Container", "Processes and sends emails.")
        
        ComponentQueue(sms_queue, "SMS Queue", "RabbitMQ", "Buffers SMS jobs.")
        Component(sms_worker, "SMS Worker", "Container", "Processes and sends SMS.")
    }

    %% --- Relationships (Corrected and Re-ordered) ---
    
    %% Flow 1: Data Ingestion (ETL)
    Rel(etl_service, external_api, "1. Fetches data from")
    Rel(etl_service, local_db, "2. Writes data to")

    %% Flow 2: Scheduled Notifications
    Rel(scheduler, appointment_service, "A. Triggers")
    Rel(appointment_service, local_db, "B. Reads appointments from")
    Rel(appointment_service, dispatcher, "C. Forwards to")
    
    %% Flow 3: Immediate Notifications
    Rel(appointment_manager, api_gateway, "D. Sends change event")
    Rel(api_gateway, dispatcher, "E. Forwards request to")

    %% Flow 4: Common Delivery Pipeline
    Rel(dispatcher, email_queue, "F1. Enqueues email job")
    Rel(email_queue, email_worker, "G1. Delivers job to")
    Rel(email_worker, third_party_email, "H1. Sends email via")

    Rel(dispatcher, sms_queue, "F2. Enqueues SMS job")
    Rel(sms_queue, sms_worker, "G2. Delivers job to")
    Rel(sms_worker, third_party_sms, "H2. Sends SMS via")
```

## Physical View (Deployment Diagram)

### Milestone 01: Core Notification Pipeline for Scheduled Reminders (Email)

```mermaid
graph TD
    %% Define External Systems First
    subgraph "External Systems"
        D[Appointment DB]
        G[Third-Party Email Service]
    end

    %% Define Kubernetes Cluster Components
    subgraph "Kubernetes Cluster"
        subgraph "Notification Namespace"
            A(<b>CronJob</b><br/>Scheduler Pod)
            B(<b>Deployment</b><br/>Appointment Service Pod)
            C(<b>Deployment</b><br/>Notification Dispatcher Pod)
            E(<b>StatefulSet</b><br/>RabbitMQ Pod)
            F(<b>Deployment</b><br/>Email Worker Pod)
        end
    end

    %% Define Relationships
    A -- triggers --> B
    B -- fetches data from --> D
    B -- forwards appointments to --> C
    C -- enqueues job to --> E
    E -- delivers job to --> F
    F -- sends email via --> G
```

### Milestone 02: Add SMS as a Second Notification Channel

```mermaid
graph TD
    %% Define External Systems First
    subgraph "External Systems"
        D[Appointment DB]
        G[Third-Party Email Service]
        H[<b>Third-Party SMS Service</b>]
    end

    %% Define Kubernetes Cluster Components
    subgraph "Kubernetes Cluster"
        subgraph "Notification Namespace"
            A(<b>CronJob</b><br/>Scheduler Pod)
            B(<b>Deployment</b><br/>Appointment Service Pod)
            C(<b>Deployment</b><br/>Notification Dispatcher Pod)
            E(<b>StatefulSet</b><br/>RabbitMQ Pod)
            F(<b>Deployment</b><br/>Email Worker Pod)
            I(<b><b>Deployment</b><br/>SMS Worker Pod</b>)
        end
    end

    %% Define Relationships
    A -- triggers --> B
    B -- fetches data from --> D
    B -- forwards appointments to --> C
    C -- enqueues job(s) to --> E
    E -- delivers email job to --> F
    E -- <b>delivers SMS job to</b> --> I
    F -- sends email via --> G
    I -- <b>sends SMS via</b> --> H
```

### Milestone 03: Implement Immediate Notifications for Appointment Changes

```mermaid
graph TD
    %% External Systems
    subgraph "External Systems"
        AppManager("Appointment Mgt System")
        AppointmentDB("Appointment DB")
        EmailService("Third-Party Email Service")
        SMSService("Third-Party SMS Service")
    end

    %% Kubernetes Cluster
    subgraph "Kubernetes Cluster"
        subgraph "Notification Namespace"
            Ingress("Kubernetes Ingress")
            Scheduler("CronJob<br/>Scheduler Pod")
            AppSvc("Deployment<br/>Appointment Service Pod")
            Dispatcher("Deployment<br/>Notification Dispatcher Pod")
            Queue("StatefulSet<br/>RabbitMQ Pod")
            EmailWorker("Deployment<br/>Email Worker Pod")
            SMSWorker("Deployment<br/>SMS Worker Pod")
        end
    end

    %% --- Relationships ---

    %% Immediate Flow
    AppManager -- "HTTPS Request" --> Ingress
    Ingress -- "routes traffic to" --> Dispatcher

    %% Scheduled Flow
    Scheduler -- "triggers" --> AppSvc
    AppSvc -- "fetches data from" --> AppointmentDB
    AppSvc -- "forwards appointments to" --> Dispatcher

    %% Common Delivery Flow
    Dispatcher -- "enqueues job(s) to" --> Queue
    Queue -- "delivers email job to" --> EmailWorker
    Queue -- "delivers SMS job to" --> SMSWorker
    EmailWorker -- "sends email via" --> EmailService
    SMSWorker -- "sends SMS via" --> SMSService
```

### Milestone 04: External API Integration and Data Ingestion

```mermaid
graph TD
    subgraph "External Systems"
        direction TB
        ExtAPI("External Patient API")
        AppManager("Appointment Mgt System")
        EmailService("Third-Party Email Service")
        SMSService("Third-Party SMS Service")
    end

    subgraph "Kubernetes Cluster"
        direction TB
        subgraph "Notification Namespace"
            direction TB

            subgraph " "
                direction LR
                ETL("CronJob<br/>ETL Service Pod")
                Ingress("Kubernetes Ingress")
                Scheduler("CronJob<br/>Scheduler Pod")
            end

            subgraph " "
                direction LR
                AppSvc("Deployment<br/>Appointment Service Pod")
                Dispatcher("Deployment<br/>Notification Dispatcher Pod")
            end

            subgraph " "
                direction LR
                DB("StatefulSet<br/>PostgreSQL Pod")
                Queue("StatefulSet<br/>RabbitMQ Pod")
            end

            subgraph " "
                direction LR
                EmailWorker("Deployment<br/>Email Worker Pod")
                SMSWorker("Deployment<br/>SMS Worker Pod")
            end
        end
    end

    %% --- Relationships ---
    ETL -- "fetches data from" --> ExtAPI
    ETL -- "writes to" --> DB
    
    Scheduler -- "triggers" --> AppSvc
    AppSvc -- "reads from" --> DB
    AppSvc -- "forwards to" --> Dispatcher

    AppManager -- "HTTPS Request" --> Ingress
    Ingress -- "routes to" --> Dispatcher

    Dispatcher -- "enqueues job(s) to" --> Queue
    Queue -- "delivers email job to" --> EmailWorker
    Queue -- "delivers SMS job to" --> SMSWorker

    EmailWorker -- "sends email via" --> EmailService
    SMSWorker -- "sends SMS via" --> SMSService
```

### Milestone 05: High Availability, Scalability, and Performance

```mermaid
graph TD
    subgraph "External Systems"
        direction TB
        ExtAPI("External Patient API")
        AppManager("Appointment Mgt System")
        EmailService("Third-Party Email Service")
        SMSService("Third-Party SMS Service")
    end

    subgraph "Kubernetes Cluster"
        direction TB
        subgraph "Notification Namespace"
            direction TB

            HPA1("HPA"); HPA2("HPA"); HPA3("HPA"); HPA4("HPA")

            subgraph " "
                direction LR
                ETL("CronJob<br/>ETL Service Pod")
                Ingress("Kubernetes Ingress")
                Scheduler("CronJob<br/>Scheduler Pod")
            end

            subgraph " "
                direction LR
                AppSvc("Deployment (x3)<br/>Appointment Service Pods")
                Dispatcher("Deployment (x3)<br/>Notification Dispatcher Pods")
            end

            subgraph " "
                direction LR
                DB("StatefulSet (x3)<br/>PostgreSQL Pods")
                Queue("StatefulSet (x3)<br/>RabbitMQ Pods")
            end

            subgraph " "
                direction LR
                EmailWorker("Deployment (x3)<br/>Email Worker Pods")
                SMSWorker("Deployment (x3)<br/>SMS Worker Pods")
            end
        end
    end

    %% --- Relationships ---
    HPA1 -- "monitors & scales" --> AppSvc
    HPA2 -- "monitors & scales" --> Dispatcher
    HPA3 -- "monitors & scales" --> EmailWorker
    HPA4 -- "monitors & scales" --> SMSWorker

    ETL -- "fetches data from" --> ExtAPI
    ETL -- "writes to" --> DB
    
    Scheduler -- "triggers" --> AppSvc
    AppSvc -- "reads from" --> DB
    AppSvc -- "forwards to" --> Dispatcher

    AppManager -- "HTTPS Request" --> Ingress
    Ingress -- "routes to" --> Dispatcher

    Dispatcher -- "enqueues job(s) to" --> Queue
    Queue -- "delivers email job to" --> EmailWorker
    Queue -- "delivers SMS job to" --> SMSWorker

    EmailWorker -- "sends email via" --> EmailService
    SMSWorker -- "sends SMS via" --> SMSService
```

### Overall Physical View

```mermaid
graph TD
    subgraph "External Systems"
        direction TB
        ExtAPI("External Patient API")
        AppManager("Appointment Mgt System")
        EmailService("Third-Party Email Service")
        SMSService("Third-Party SMS Service")
    end

    subgraph "Kubernetes Cluster"
        direction TB
        subgraph "Notification Namespace"
            direction TB

            HPA1("HPA"); HPA2("HPA"); HPA3("HPA"); HPA4("HPA")

            subgraph " "
                direction LR
                ETL("CronJob<br/>ETL Service Pod")
                Ingress("Kubernetes Ingress")
                Scheduler("CronJob<br/>Scheduler Pod")
            end

            subgraph " "
                direction LR
                AppSvc("Deployment (x3)<br/>Appointment Service Pods")
                Dispatcher("Deployment (x3)<br/>Notification Dispatcher Pods")
            end

            subgraph " "
                direction LR
                DB("StatefulSet (x3)<br/>PostgreSQL Pods")
                Queue("StatefulSet (x3)<br/>RabbitMQ Pods")
            end

            subgraph " "
                direction LR
                EmailWorker("Deployment (x3)<br/>Email Worker Pods")
                SMSWorker("Deployment (x3)<br/>SMS Worker Pods")
            end
        end
    end

    %% --- Relationships ---
    HPA1 -- "monitors & scales" --> AppSvc
    HPA2 -- "monitors & scales" --> Dispatcher
    HPA3 -- "monitors & scales" --> EmailWorker
    HPA4 -- "monitors & scales" --> SMSWorker

    ETL -- "fetches data from" --> ExtAPI
    ETL -- "writes to" --> DB
    
    Scheduler -- "triggers" --> AppSvc
    AppSvc -- "reads from" --> DB
    AppSvc -- "forwards to" --> Dispatcher

    AppManager -- "HTTPS Request" --> Ingress
    Ingress -- "routes to" --> Dispatcher

    Dispatcher -- "enqueues job(s) to" --> Queue
    Queue -- "delivers email job to" --> EmailWorker
    Queue -- "delivers SMS job to" --> SMSWorker

    EmailWorker -- "sends email via" --> EmailService
    SMSWorker -- "sends SMS via" --> SMSService
```
