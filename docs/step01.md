#### 1. Logical View (C4 Component Diagram)

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

#### 2. Physical View (AWS Deployment Diagram)

*For this design, we will use a generic container orchestration model based on Kubernetes primitives, which are portable to AWS (EKS), GCP (GKE), Azure (AKS), etc.*

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

#### 3. Component-to-Resource Mapping Table

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Scheduler** | Kubernetes `CronJob` | Native Kubernetes resource for time-based job scheduling. Perfectly fits the "run at 9 am" requirement. Portable and standard. |
| **Appointment Service** | Kubernetes `Deployment` with a `Container` | A standard stateless service. A `Deployment` ensures it's running and can be scaled if needed later. |
| **Notification Dispatcher**| Kubernetes `Deployment` with a `Container` | Another core stateless service. A `Deployment` is the appropriate controller for managing its lifecycle. |
| **Message Queue** | `StatefulSet` running RabbitMQ (or a managed service) | RabbitMQ is a lightweight, widely-used message broker perfect for containerized environments. A `StatefulSet` is used for stateful applications like a message broker to ensure stable network identifiers and storage. |
| **Email Worker** | Kubernetes `Deployment` with a `Container` | A scalable, stateless consumer of jobs from the queue. A `Deployment` allows us to easily scale the number of worker pods to handle load. |
| **Appointment Database** | External Database (e.g., Amazon RDS, local PostgreSQL) | Separating the database from the cluster is a best practice for data persistence, management, and security. For this issue, we assume it exists. |
