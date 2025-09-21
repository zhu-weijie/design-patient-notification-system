#### 1. Logical View (C4 Component Diagram)

This diagram introduces the new API Gateway and the external system that triggers it.

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

#### 2. Physical View (Deployment Diagram)

This view adds the Kubernetes Ingress, which acts as our API Gateway.

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

#### 3. Component-to-Resource Mapping Table (New or Modified Components)

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **API Gateway** | Kubernetes `Ingress` & `Service` | `Ingress` is the standard, cloud-agnostic Kubernetes resource for managing external access to services, typically HTTP. It handles routing, SSL/TLS termination, and can be backed by various controllers (Nginx, Traefik, cloud provider LBs). A `Service` provides the stable internal endpoint that the `Ingress` directs traffic to (`Notification Dispatcher` service). |
