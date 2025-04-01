# Video to MP3 Converter - Microservice Architecture

## Overview

This project is a scalable, microservices-based application that converts video files to MP3 audio format. It demonstrates modern cloud-native architecture principles using Python, Kubernetes, RabbitMQ, MongoDB, and MySQL. The system is designed to handle asynchronous video processing in a distributed manner with proper authentication, notifications, and file management.

## Architecture

<!-- ![Architecture Diagram](assets/architectureDiagram.svg) -->

```mermaid
flowchart TD
    classDef external fill:#ffffff,stroke:#333,stroke-width:2px
    classDef k8s fill:#e6f3ff,stroke:#0078d7,stroke-width:2px,stroke-dasharray: 5 5
    classDef gateway fill:#c5e1a5,stroke:#33691e,stroke-width:2px
    classDef auth fill:#bbdefb,stroke:#1565c0,stroke-width:2px
    classDef rabbit fill:#ffccbc,stroke:#bf360c,stroke-width:2px
    classDef converter fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
    classDef notification fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    classDef user fill:#d9d9d9,stroke:#333,stroke-width:2px

    %% External User
    User[User]
    class User user

    %% Ingress
    Ingress{Ingress\nmp3converter.com}

    %% Kubernetes Cluster
    subgraph K8sCluster["Kubernetes Cluster"]
        %% Services
        Gateway[Gateway Service\n- User Interface\n- File Upload/Download\n- Request Routing]
        Auth[Auth Service\n- Authentication\n- JWT Generation\n- Token Validation]
        RabbitMQ[(RabbitMQ\n- Message Broker\n- Video Queue\n- MP3 Queue)]
        Converter[Converter Service\n- Video Processing\n- MP3 Conversion\n- File Storage]
        Notification[Notification Service\n- Email Notifications\n- User Alerts\n- Status Updates]
    end
    class K8sCluster k8s
    class Gateway gateway
    class Auth auth
    class RabbitMQ rabbit
    class Converter converter
    class Notification notification

    %% External Resources
    MySQL[(External MySQL\nUser Auth Data)]
    MongoDB[(External MongoDB\nFile Storage GridFS)]
    class MySQL,MongoDB external

    %% Connections
    User <--> Ingress
    Ingress <--> Gateway
    Gateway <--> Auth
    Gateway <--> RabbitMQ
    Auth <--> MySQL
    Gateway <--> MongoDB
    RabbitMQ <--> Converter
    RabbitMQ <--> Notification
    Converter <--> MongoDB
    Notification --> User

    %% Legend
    subgraph Legend
        GatewayLegend[Gateway Service]
        AuthLegend[Auth Service]
        RabbitLegend[(RabbitMQ)]
        ConverterLegend[Converter Service]
        NotifLegend[Notification Service]
        ExternalLegend[(External Services)]
    end
    class GatewayLegend gateway
    class AuthLegend auth
    class RabbitLegend rabbit
    class ConverterLegend converter
    class NotifLegend notification
    class ExternalLegend external

```


The application is composed of several microservices:

1. **Auth Service**: Handles user authentication and JWT token generation/validation
2. **Gateway Service**: Entry point for all client requests, manages file uploads/downloads
3. **Converter Service**: Processes videos and converts them to MP3 format
4. **Notification Service**: Sends email notifications when conversions are complete
5. **RabbitMQ**: Message broker for asynchronous communication between services
6. **MongoDB**: Stores video and MP3 files using GridFS
7. **MySQL**: Stores user authentication information

## Service Communication Flow

1. User authenticates via the Gateway service
2. Upon successful authentication, the user receives a JWT token
3. User uploads a video file through the Gateway service
4. Gateway validates the JWT token and stores the video in MongoDB
5. Gateway sends a message to RabbitMQ's video queue
6. Converter service consumes the message, processes the video, and creates an MP3 file
7. Converter stores the MP3 in MongoDB and sends a message to RabbitMQ's MP3 queue
8. Notification service consumes the message and sends an email to the user
9. User can download the MP3 file through the Gateway service

## Services Breakdown

### Auth Service

- **Purpose**: Manage user authentication and authorization
- **Endpoints**:
  - `POST /login`: Authenticates users and generates JWT tokens
  - `POST /validate`: Validates JWT tokens
- **Database**: MySQL
- **Technologies**: Flask, JWT, MySQL

### Gateway Service

- **Purpose**: API gateway for client applications, handling file uploads/downloads
- **Endpoints**:
  - `POST /login`: Proxies authentication requests to Auth service
  - `POST /upload`: Accepts video uploads, stores in MongoDB, enqueues processing
  - `GET /download`: Streams MP3 files to clients
- **Storage**: MongoDB (GridFS for file storage)
- **Technologies**: Flask, PyMongo, RabbitMQ

### Converter Service

- **Purpose**: Consumes video processing tasks and converts videos to MP3
- **Process**:
  - Listens to the video queue in RabbitMQ
  - Retrieves video from MongoDB
  - Converts video to MP3 using MoviePy
  - Stores MP3 in MongoDB
  - Sends a message to the MP3 queue
- **Technologies**: Pika, PyMongo, MoviePy

### Notification Service

- **Purpose**: Notifies users when their MP3 is ready for download
- **Process**:
  - Listens to the MP3 queue in RabbitMQ
  - Sends email notifications to users
- **Technologies**: Pika, SMTP

### Data Flow
```mermaid
flowchart TD
    classDef process fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    classDef datastore fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef entity fill:#f5f5f5,stroke:#333,stroke-width:2px
    classDef queue fill:#ffccbc,stroke:#bf360c,stroke-width:2px

    %% External Entity
    User[User]
    class User entity

    %% Data Stores
    D1[(D1 MySQL\nUser Authentication Data)]
    D2[(D2 MongoDB\nVideo Files GridFS)]
    D3[(D3 MongoDB\nMP3 Files GridFS)]
    class D1,D2,D3 datastore

    %% Processes
    P1((1.0\nAuthentication))
    P2((2.0\nUpload Video))
    P3((3.0\nConvert Video))
    P4((4.0\nMessage Queue))
    P5((5.0\nSend Email))
    P6((6.0\nDownload MP3))
    class P1,P2,P3,P5,P6 process
    class P4 queue

    %% Authentication Flow
    User -->|Login Credentials| P1
    P1 -->|Verify Login| D1
    D1 -->|Auth Result| P1
    P1 -->|JWT Token| User

    %% Upload Flow
    User -->|Video File + JWT| P2
    P2 -->|Store Video File| D2
    P2 -->|Video Message| P4

    %% Processing Flow
    P4 -->|Video Task| P3
    P3 -->|Read Video| D2
    P3 -->|Store MP3| D3
    P3 -->|MP3 Ready Message| P4
    P4 -->|MP3 Ready Task| P5

    %% Notification Flow
    P5 -->|Get MP3 Info| D3
    P5 -->|Email Notification| User

    %% Download Flow
    User -->|Request MP3 + JWT| P6
    P6 -->|Retrieve MP3 File| D3
    D3 -->|MP3 Data| P6
    P6 -->|MP3 File| User

    %% Add subgraph for process descriptions
    subgraph "Data Flow Process"
        desc1[1. Authentication Flow:
        - User submits email/password to Gateway's /login endpoint
        - Gateway forwards credentials to Auth service
        - Auth service verifies credentials against MySQL database
        - If valid, Auth service generates JWT token and returns it to user]
        
        desc2[2. Video Processing Flow:
        - User uploads video with JWT token to Gateway's /upload endpoint
        - Gateway validates token, stores video in MongoDB and sends message to RabbitMQ
        - Converter service consumes message, retrieves video, converts to MP3
        - Converter stores MP3 in MongoDB and sends message to RabbitMQ
        - Notification service consumes message and sends email to user
        - User requests MP3 download from Gateway with JWT token and file ID
        - Gateway validates token, retrieves MP3 from MongoDB and sends to user]
    end
```

### Infrastructure Components

#### RabbitMQ

- **Purpose**: Message broker for asynchronous communication
- **Queues**:
  - `video`: For video conversion requests
  - `mp3`: For MP3 ready notifications
- **Deployment**: StatefulSet in Kubernetes with persistent storage

#### MongoDB

- **Purpose**: Store video and MP3 files
- **Collections**:
  - `videos.files` & `videos.chunks`: Store uploaded videos using GridFS
  - `mp3s.files` & `mp3s.chunks`: Store converted MP3 files using GridFS
- **Note**: External MongoDB instance   (host.minikube.internal:27017)

#### MySQL

- **Purpose**: Store user authentication data
- **Tables**:
  - `user`: Contains user email and password
- **Note**: External MySQL instance   (host.minikube.internal:3306)

## Kubernetes Deployment

All services are containerized and deployed on Kubernetes with the following resources:

- **Deployments** for stateless services (Auth, Gateway, Converter, Notification)
- **StatefulSet** for RabbitMQ (with persistent volume)
- **Services** for internal communication
- **ConfigMaps** for configuration
- **Secrets** for sensitive information
- **Ingress** for external access

### Deployment Configuration

- **Auth Service**: 2 replicas
- **Gateway Service**: 2 replicas
- **Converter Service**: 4 replicas (CPU-intensive work)
- **Notification Service**: 4 replicas
- **RabbitMQ**: Single instance with persistent storage

### Network Configuration

- **Ingress**:
  - `mp3converter.com` → Gateway Service
  - `rabbitmq-manager.com` → RabbitMQ Management UI
- **Internal Services**:
  - `auth:5000`: Auth service
  - `gateway:8080`: Gateway service
  - `rabbitmq:5672`: RabbitMQ AMQP
  - `rabbitmq:15672`: RabbitMQ Management UI

## Security

The application implements several security measures:

1. **Authentication**: JWT-based authentication system
2. **Authorization**: Admin role check for sensitive operations
3. **Secrets Management**: Kubernetes Secrets for passwords and sensitive data
4. **HTTPS**:Handled by Ingress 

## Setup and Installation

### Prerequisites

- Kubernetes cluster (Minikube for local development)
- MongoDB instance
- MySQL instance

### MySQL Setup

Run the initialization script to set up the database:

```sql
CREATE USER 'auth_user'@'localhost' IDENTIFIED BY 'Auth123';
CREATE DATABASE auth;
GRANT ALL PRIVILEGES ON auth.* TO 'auth_user'@'localhost';
USE auth;
CREATE TABLE user (
  id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL
);
INSERT INTO user (email, password) VALUES ('georgio@email.com', 'Admin123');
```

### Kubernetes Deployment

1. Apply RabbitMQ manifests:
   ```bash
   kubectl apply -f ./src/rabbit/manifests/
   ```

2. Apply Auth service manifests:
   ```bash
   kubectl apply -f ./src/auth/manifests/
   ```

3. Apply Gateway service manifests:
   ```bash
   kubectl apply -f ./src/gateway/manifests/
   ```

4. Apply Converter service manifests:
   ```bash
   kubectl apply -f ./src/converter/manifests/
   ```

5. Apply Notification service manifests:
   ```bash
   kubectl apply -f ./src/notification/manifests/
   ```

### Local Development

For local development using Minikube:

1. Enable Ingress addon:
   ```bash
   minikube addons enable ingress
   ```

2. Update `/etc/hosts` to point domains to Minikube IP:
   ```
   <minikube-ip> mp3converter.com rabbitmq-manager.com
   ```

3. Build and push Docker images:
   ```bash
   docker build -t <your-docker-id>/auth:latest ./src/auth/
   docker build -t <your-docker-id>/gateway:latest ./src/gateway/
   docker build -t <your-docker-id>/converter:latest ./src/converter/
   docker build -t <your-docker-id>/notification:latest ./src/notification/
   
   docker push <your-docker-id>/auth:latest
   docker push <your-docker-id>/gateway:latest
   docker push <your-docker-id>/converter:latest
   docker push <your-docker-id>/notification:latest
   ```

## Usage

### Authentication

To authenticate and get a JWT token:

```bash
curl -X POST http://mp3converter.com/login -u "georgio@email.com:Admin123"
```

### Upload Video

To upload a video:

```bash
curl -X POST -F 'file=@./video.mp4' -H "Authorization: Bearer <your-jwt-token>" http://mp3converter.com/upload
```

### Download MP3

To download the converted MP3:

```bash
curl -X GET -H "Authorization: Bearer <your-jwt-token>" "http://mp3converter.com/download?fid=<mp3-file-id>" -o output.mp3
```

## Technical Details

### Video Processing

- Videos are processed using MoviePy library
- Processing happens in a temporary directory
- Files are stored in MongoDB using GridFS

### Message Flow

1. Video Upload:
   - Gateway stores video in MongoDB
   - Message format: `{"video_fid": "<id>", "mp3_fid": null, "username": "<email>"}`

2. MP3 Ready:
   - Converter updates message with MP3 file ID
   - Message format: `{"video_fid": "<id>", "mp3_fid": "<id>", "username": "<email>"}`

### Email Notifications

The notification service sends emails using Gmail SMTP when an MP3 is ready:
- From: System gmail account
- To: User's email address
- Subject: "MP3 Download"
- Content: MP3 file ID and download instructions


## Technologies Used

- **Python 3.10**: Programming language for all services
- **Flask**: Web framework for Auth and Gateway services
- **MoviePy**: Video processing library
- **PyMongo** and **GridFS**: MongoDB client and file storage
- **Flask-MySQLdb**: MySQL client
- **PyJWT**: JWT token generation and validation
- **Pika**: RabbitMQ client
- **Kubernetes**: Container orchestration
- **Docker**: Container technology
- **RabbitMQ**: Message broker
- **MongoDB**: NoSQL database and file storage
- **MySQL**: Relational database for authentication

## License

