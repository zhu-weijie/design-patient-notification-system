#### 1. Logical View (C4 Component Diagram)

This view introduces the new ETL service and clarifies that the Appointment DB is now an internal component.

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

#### 2. Physical View (Deployment Diagram)

This view adds a new CronJob for the ETL service and a StatefulSet for the new internal database.

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

#### 3. Component-to-Resource Mapping Table (New or Modified Components)

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **ETL Service** | Kubernetes `CronJob` | This is a periodic, batch-oriented task, making `CronJob` the perfect Kubernetes-native resource for scheduling its execution. |
| **Appointment Database**| Kubernetes `StatefulSet` with `PersistentVolumeClaim` | This is a stateful component that must persist data. A `StatefulSet` is the standard for running databases on Kubernetes, providing stable network IDs and storage. A `PersistentVolumeClaim` ensures the data survives pod restarts. |
| **Appointment Service**| (Modified) Kubernetes `Deployment` | The physical resource is unchanged. The container's code and configuration will be updated to point to the new internal database service address instead of an external one. |
