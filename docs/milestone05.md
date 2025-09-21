#### 1. Logical View (C4 Component Diagram)

**No change.**

The logical architecture remains the same. High availability and scalability are non-functional requirements that are implemented in the *physical view*. The components and their logical interactions do not change when we add more instances of them.

#### 2. Physical View (Deployment Diagram)

This is the only diagram that changes. It is updated to show multiple replicas and the addition of HPAs.

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

#### 3. Component-to-Resource Mapping Table (Updated Rationale)

| Logical Component | Physical Resource | Rationale (Updated for HA/Scalability) |
| :--- | :--- | :--- |
| Appointment Service | Kubernetes `Deployment` | Will be deployed with **3 replicas** for high availability. A **Horizontal Pod Autoscaler (HPA)** will be attached to automatically scale pods based on CPU/memory load. |
| Notification Dispatcher| Kubernetes `Deployment` | Will be deployed with **3 replicas** for high availability. A **Horizontal Pod Autoscaler (HPA)** will be attached to automatically scale pods based on CPU/memory load. |
| Email Worker | Kubernetes `Deployment` | Will be deployed with **3 replicas** for high availability. A **Horizontal Pod Autoscaler (HPA)** will be attached to automatically scale pods based on queue length or CPU. |
| SMS Worker | Kubernetes `Deployment` | Will be deployed with **3 replicas** for high availability. A **Horizontal Pod Autoscaler (HPA)** will be attached to automatically scale pods based on queue length or CPU. |
| Message Queue | `StatefulSet` running RabbitMQ | Will be deployed as a **3-node cluster** using a RabbitMQ operator or Helm chart to ensure high availability of the message bus. |
| Appointment Database | `StatefulSet` running PostgreSQL | Will be deployed as a **3-node cluster** with a primary and standbys using a PostgreSQL operator or Helm chart to ensure high availability of the data store. |
