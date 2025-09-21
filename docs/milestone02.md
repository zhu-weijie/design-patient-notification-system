#### 1. Logical View (C4 Component Diagram)

This diagram now shows two parallel paths for notification delivery after the dispatcher.

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

#### 2. Physical View (AWS Deployment Diagram)

This view adds the new SMS worker deployment and the external SMS service.

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

#### 3. Component-to-Resource Mapping Table (New or Modified Components)

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **SMS Worker** | Kubernetes `Deployment` with a `Container` | This is a new, stateless worker. A `Deployment` is the standard for managing its lifecycle and allows for easy scaling, identical to the `Email Worker`. |
| **Notification Dispatcher**| (Modified) Kubernetes `Deployment` | The physical resource remains a `Deployment`. The container image will be updated with the new logic to handle channel selection and routing. No infrastructure change is needed. |
| **Message Queue** | (Modified) `StatefulSet` running RabbitMQ | The physical resource is unchanged. A new queue/exchange will be configured within the existing RabbitMQ service to handle SMS messages, demonstrating a software configuration change. |
