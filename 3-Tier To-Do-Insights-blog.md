# Insights from Building and Deploying a 3-Tier To-Do List Web App Using Flutter, Spring Boot, and Docker
## Introduction
Exploring backend technologies and cloud deployment has been an exciting journey. In this article, I will share the insights and lessons learned while building a 3-tier To-Do List web application. This project utilizes **Flutter for the frontend**, **Spring Boot for the backend**, **MySQL for the database**, and **Docker & Docker Compose for orchestrating the deployment**. For those interested in the full code, you can find the complete project on [GitHub](https://github.com/sabalessandip/to_do).

## Tools I Used
- **Flutter SDK:** For developing the frontend.
- **Java Development Kit (JDK) 21:** For running the Spring Boot application.
- **Visual Studio Code:** As the integrated development environment (IDE).
- **Docker and Docker Compose:** For containerization and orchestration.

## Project Overview
The architecture of the To-Do List application is based on the three-tier design:

1. Frontend (Presentation Layer): Developed using Flutter Web to provide a dynamic and responsive user interface.
2. Backend (Application Layer): Built with Spring Boot to handle business logic and API requests.
3. Database (Data Layer): Utilized MySQL to store and manage task data.

## Frontend with Flutter Web

### Design and Development
I have selected Flutter for the frontend to create a simple yet functional UI for task management as I had explored it earlier, understands it better than React, and wanted to build the frontend quickly to focus on other tiers. Flutter's declarative syntax and state management are relatively easy to understand, making the development process smoother.

### Key Insights:

**Configuration:** Used following command to enable web support in Flutter project,
```sh
flutter config --enable-web.
```
**HTTP Requests:** Used the http package to communicate with the backend, making asynchronous API calls to fetch and manipulate task data.

**Environment Variables:** The '[dart-define](https://dartcode.org/docs/using-dart-define-in-flutter/)' option can be utilized to supply environment variables to a Flutter Web project. However, I have not applied it in this project.

## Backend with Spring Boot

### Design and Development
Spring Boot was selected for the backend due to its rapid development capabilities and ease of integration with other technologies. The backend is responsible for handling CRUD operations for tasks, exposed via RESTful APIs.

By default, if a Spring Boot application contains Spring Data JPA, it will automatically attempt to create a database connection. However, this can cause issues if the database is not available when the application starts. I faced this issue while deploying with Docker Compose. To resolve it, I added health checks on the database and service health conditions on database dependency in the Docker Compose file. To continue without a database, Spring Boot can be configured to skip database initialization with the following properties:

```java
spring.jpa.properties.hibernate.temp.use_jdbc_metadata_defaults=false
spring.jpa.hibernate.ddl-auto=none
```

### Key Insights:

**Entity Design:** Created a Task entity representing the task data model, annotated with JPA for ORM (Object Relational Mapping).

**Repository Layer:** Implemented Repository layer to interacts with the database and performs CRUD operations.

**Service Layer:** Implemented a service layer to encapsulate business logic, promoting a clean separation of concerns.

**Controller Layer:** Developed REST controllers to expose APIs for task management.

**Spring Profiles:** Utilized Spring Profiles to manage different configurations, ensuring smooth transitions between development and production environments.

## Database with MySQL
Instead of setting up a local MySQL instance, Docker was used to create a MySQL container. This approach simplified the setup process. The application needs a database ```todo_list```, a user ```todo_user``` with password ```user@123```, and a ```task``` table to store the tasks in the MySQL database. Although Spring JPA can handle these tasks automatically, I have manually created them.

To create mysql container, I used,
```sh
docker run --name local-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=<rootpassword> mysql
```

opened a terminal session to the MySQL container,

```sh
docker exec -it local-mysql /bin/bash
```

logged in to MySQL with the root user,
```sh
mysql -u root -p
```

Then executed the following SQL commands one by one. Also, When a mysql container is started for the first time, it executes files with extensions ```.sh```, ```.sql``` and ```.sql.gz``` that are found in ```/docker-entrypoint-initdb.d```. directory. So, to automate the process, created a sql script with these commands while containerizing the database.

```sql
# Create the database
CREATE DATABASE todo_list;

# Create 'todo_user' user
CREATE USER 'todo_user'@'%' IDENTIFIED BY 'user@123';

# Grant privileges on the database to the user 'todo_user'. 'select, insert, delete, update' can be used to grant the minimum privileges the application needs.
GRANT ALL PRIVILEGES ON todo_list.* TO 'todo_user'@'%';

# Apply the changes
FLUSH PRIVILEGES;

# Select the database to use
USE todo_list;

# Create the tasks table
CREATE TABLE tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    completed BOOLEAN DEFAULT FALSE
);
```

## Manual Testing of the 3-Tier App
Before deploying the application using Docker Compose, I manually tested it to ensure each component functioned correctly.

### Steps Taken:
**Database:** Ensured the MySQL container was running and accessible, and the required database, user, and table were created.

**Backend:** Launched the Spring Boot application with the local profile to connect to the MySQL database.
```sh
./gradlew bootRun --args='--spring.profiles.active=local'
```

**Frontend:** Configured the localhost backend URL as the base URL in the API service and executed the Flutter web project locally to interact with the backend.

This also illustrates the sequence in which the containers for each respective layer should be created.

## Containerizing with Docker
I have packaged the frontend and backend on my machine and then created the docker images with the built packages to simulate real-world scenarios. However, we could write a Dockerfile to build and package during image creation using multi-stage Docker builds. So, Before creating frontend and backend images or running the docker compose, package them using the following commands.

With [multi-stage builds](https://docs.docker.com/build/building/multi-stage/), we can use multiple FROM statements in Dockerfile. Each FROM instruction can use a different base, and each of them begins a new stage of the build. We can selectively copy artifacts from one stage to another, leaving behind everything we don't want in the final image.

### Frontend Packaging Commands:
```sh
flutter config --enable-web
flutter build web
```

### Backend Packaging Commands:
```sh
./gradlew clean build -Dspring.profiles.active=deploy
```

### Dockerfiles

**Frontend:** Used Nginx to serve the Flutter website. The packaged website from the ```/build/web``` directory is copied to the ```/usr/share/nginx/html``` directory, which is the default HTML directory inside the Nginx container.

**Backend:** Created an arguments and environment variable to accept the database URL from Docker Compose and used the deploy spring profile to use these environment variables in the code.

**Database:** Copied the SQL script to the ```/docker-entrypoint-initdb.d``` directory inside the MySQL container to create the database prerequisites at container startup.


### Docker Compose:
By default, Compose sets up a single [network](https://docs.docker.com/reference/cli/docker/network/create/) for your app. Each container from services joins the default network and is both reachable by other containers on that network, and discoverable by the service's name. I have leveraged this default network to configure the services for the database, backend, and frontend in docker compose, ensuring they communicate seamlessly.

**Build and start all services:**
```sh
docker-compose up -d
```

**Stop all services:**
```sh
docker-compose down
```

**Access the application:**
Once the containers are running, the application can be accessed via http://localhost.

## Conclusion
Building a 3-tier To-Do List web application using Flutter, Spring Boot, and MySQL provided a comprehensive learning experience. The use of Docker and Docker Compose simplified the deployment process, making it scalable and consistent across different environments.

This journey enhanced my understanding of backend technologies, cloud deployment, and containerization. For a detailed view of the project, feel free to explore the [GitHub repository](https://github.com/sabalessandip/to_do). Happy coding!