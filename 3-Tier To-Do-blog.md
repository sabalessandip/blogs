# Building and Deploying a 3-Tier To-Do List Web App Using Flutter, Spring Boot, and Docker

## Introduction
Exploring backend technologies and cloud deployment has been an exciting journey. To bridge my knowledge to full-stack of web applications, I embarked on creating a 3-tier To-Do List/Task manager web application using ***Flutter for the frontend***, ***Spring Boot for the backend***, and ***MySQL as the database***. Additionally, I leveraged Docker to containerize the application and Docker Compose to manage the deployment of the entire stack.

In this blog, I'll walk you through the process, from setting up the development environment to deploying the application using Docker Compose.

## GitHub Repository
You can find the complete source code for this project on [GitHub](https://github.com/sabalessandip/blogs/blob/main/3-Tier%20To-Do-blog.md).

## Prerequisites
Before diving into the project, ensure you have the following installed:
- [Flutter](https://flutter.dev/docs/get-started/install)
- [Java Development Kit (JDK) 21](https://www.oracle.com/java/technologies/downloads/#java21)
- [Visual Studio Code](https://code.visualstudio.com/download)
- [Docker](https://www.docker.com/get-started)
- [Docker Compose](https://docs.docker.com/compose/install/)

## Project Overview
The To-Do List application is divided into three tiers:

**Frontend**: Built with Flutter Web.
**Backend**: Developed using Spring Boot to handle API requests.
**Database**: MySQL to store task data.

## Setting Up the Frontend with Flutter Web
### Step 1: Create a New Flutter Project
```sh
flutter create todo_flutter_web
cd todo_flutter_web
```

### Step 2: Enable Flutter Web
```sh
flutter config --enable-web
```

### Step 3: Modify pubspec.yaml to Add Dependencies

Add the http package for API calls:

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.2
```

### Step 4: Create the Flutter Web UI

Create a simple UI to add and display tasks and model and service classes to make api calls as following,

```dart
# todo_page.dart
import 'package:flutter/material.dart';
import 'api_service.dart';
import 'task.dart';
import 'task_detail.dart';

class ToDoHomePage extends StatefulWidget {
  const ToDoHomePage({super.key});

  @override
  State<ToDoHomePage> createState() => _ToDoHomePageState();
}

class _ToDoHomePageState extends State<ToDoHomePage> {
  final ApiService _apiService = ApiService();
  List<Task> _tasks = [];
  final TextEditingController _titleController = TextEditingController();

  @override
  void initState() {
    super.initState();
    _fetchTasks();
  }

  Future<void> _fetchTasks() async {
    try {
      final tasks = await _apiService.getTasks();
      setState(() {
        _tasks = tasks;
      });
    } catch (e) {
      print(e);
    }
  }

  void _addTask() async {
    try {
      final newTask = TaskDetail(title: _titleController.text, description: '');
      final createdTask = await _apiService.createTask(newTask);
      setState(() {
        _tasks.add(createdTask);
        _titleController.clear();
      });
    } catch (e) {
      print(e);
    }
  }

  void _updateTask(Task task) async {
    try {
      final updatedTask = await _apiService.updateTask(task);
      setState(() {
        final index = _tasks.indexWhere((t) => t.id == task.id);
        _tasks[index] = updatedTask;
      });
    } catch (e) {
      print(e);
    }
  }

  void _deleteTask(int id) async {
    try {
      await _apiService.deleteTask(id);
      setState(() {
        _tasks.removeWhere((task) => task.id == id);
      });
    } catch (e) {
      print(e);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Task Manager'),
      ),
      body: Column(
        children: [
          Expanded(child: _buildTaskList()),
          _buildTaskInput(),
        ],
      ),
    );
  }

  Widget _buildTaskList() {
    return ListView.builder(
      itemCount: _tasks.length,
      itemBuilder: (context, index) {
        final task = _tasks[index];
        return Container(
          color: index % 2 == 0 ? Colors.grey[200] : Colors.white,
          child: ListTile(
            leading: Checkbox(
              value: task.completed,
              onChanged: (value) {
                setState(() {
                  task.completed = value!;
                });
                _updateTask(task);
              },
            ),
            title: Text(
              task.title,
              style: TextStyle(
                decoration: task.completed
                    ? TextDecoration.lineThrough
                    : TextDecoration.none,
                color: task.completed ? Colors.grey : Colors.black,
              ),
            ),
            trailing: IconButton(
              icon: const Icon(Icons.delete),
              onPressed: () => _deleteTask(task.id),
            ),
          ),
        );
      },
    );
  }

  Widget _buildTaskInput() {
    return Padding(
      padding: const EdgeInsets.all(8.0),
      child: ListTile(
        shape: RoundedRectangleBorder(
          side: const BorderSide(color: Colors.black, width: 1),
          borderRadius: BorderRadius.circular(5),
        ),
        title: TextField(
          controller: _titleController,
          decoration: const InputDecoration(
            labelText: 'New Task',
          ),
        ),
        trailing: IconButton(
          icon: const Icon(Icons.add),
          onPressed: _addTask,
        ),
      ),
    );
  }
}
```

```dart
# api_service.dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import 'task.dart';
import 'task_detail.dart';

class ApiService {
  // static const String baseUrl = 'http://localhost:5000';
  static const String baseUrl = 'http://backend:5000';

  Future<List<Task>> getTasks() async {
    final response = await http.get(Uri.parse('$baseUrl/tasks'));
    if (response.statusCode == 200) {
      List<dynamic> body = json.decode(response.body);
      return body.map((dynamic item) => Task.fromJson(item)).toList();
    } else {
      throw Exception('Failed to load tasks');
    }
  }

  Future<Task> createTask(TaskDetail task) async {
    final response = await http.post(
      Uri.parse('$baseUrl/task'),
      headers: {'Content-Type': 'application/json'},
      body: json.encode(task.toJson()),
    );
    print(response.toString());
    if (response.statusCode == 200) {
      return Task.fromJson(json.decode(response.body));
    } else {
      throw Exception('Failed to create task');
    }
  }

  Future<Task> updateTask(Task task) async {
    final response = await http.put(
      Uri.parse('$baseUrl/task/${task.id}'),
      headers: {'Content-Type': 'application/json'},
      body: json.encode(task.toJson()),
    );
    if (response.statusCode == 200) {
      return Task.fromJson(json.decode(response.body));
    } else {
      throw Exception('Failed to update task');
    }
  }

  Future<void> deleteTask(int id) async {
    final response = await http.delete(Uri.parse('$baseUrl/task/$id'));
    if (response.statusCode != 200) {
      throw Exception('Failed to delete task');
    }
  }
}
```

```dart
# task.dart
class Task {
  final int id;
  final String title;
  final String description;
  bool completed;

  Task({
    required this.id,
    required this.title,
    required this.description,
    this.completed = false,
  });

  factory Task.fromJson(Map<String, dynamic> json) {
    return Task(
      id: json['id'],
      title: json['title'],
      description: json['description'],
      completed: json['completed'],
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'description': description,
      'completed': completed,
    };
  }
}
```
```dart
# task_detail.dart
class TaskDetail {
  final String title;
  final String description;
  bool completed;

  TaskDetail({
    required this.title,
    required this.description,
    this.completed = false,
  });

  Map<String, dynamic> toJson() {
    return {
      'title': title,
      'description': description,
      'completed': completed,
    };
  }
}
```

## Setting Up the Backend with Spring Boot
### Step 1: Initialize the Spring Boot Project
Use [Spring Initializr](https://start.spring.io) to create a new project with **Spring Web**, **Spring Data JPA**, **MySQL Driver**, **Lombok** dependencies.

### Step 2: Create the Task Entity
```java
# com.demo.todoapp.entity.Task
@Entity
@Table(name = "tasks")
@NoArgsConstructor
@AllArgsConstructor
public class Task {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String title;
    private String description;
    private boolean completed = false;
    public Long getId() {
        return id;
    }
    public String getTitle() {
        return title;
    }
    public String getDescription() {
        return description;
    }
    public boolean isCompleted() {
        return completed;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public void setTitle(String title) {
        this.title = title;
    }
    public void setDescription(String description) {
        this.description = description;
    }
    public void setCompleted(boolean completed) {
        this.completed = completed;
    }
    
    public Task(String title, String description, boolean completed) {
        this.title = title;
        this.description = description;
        this.completed = completed;
    }
}
```

### Step 3: Create the Task Repository
```java
# com.demo.todoapp.repository.TaskRepository
@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {
}
```

### Step 4: Create the Task Service
```java
# com.demo.todoapp.service.impl.TaskServiceImpl;
@Service
public class TaskServiceImpl implements TaskService {
    @Autowired
    TaskRepository repository;

    @Override
    public Task createTask(String title, 
                            String description, 
                            boolean completed) {
        try {
            Task newTask = new Task(title, description, completed);
            return this.repository.save(newTask);
        } catch (Exception e) {
            throw new RuntimeException("Unable to create Task");
        }
        
    }

    @Override
    public List<Task> getAllTasks() {
        return this.repository.findAll();
    }

    @Override
    public Task getTaskById(Long id) {
        try {
            return this.repository.getReferenceById(id);
        } catch (Exception e) {
            throw new RuntimeException("Unable to find Task by " + id);
        }
    }

    @Override
    public Task updateTaskById(Long id, Task task) {
        try {
            Task existingTask = this.getTaskById(id);
            existingTask.setTitle(task.getTitle());
            existingTask.setDescription(task.getDescription());
            existingTask.setCompleted(task.isCompleted());
            return this.repository.save(existingTask);
        } catch (Exception e) {
            throw new RuntimeException("Unable to update Task by " + id);
        }
    }

    @Override
    public void removeTaskById(Long id) {
        try {
            this.repository.deleteById(id);
        } catch (Exception e) {
            throw new RuntimeException("Unable to delete Task by " + id);
        }
    }
}
```

### Step 5: Create the Task Controller
```java
# com.demo.todoapp.controller.TaskController
@RestController
@CrossOrigin
public class TaskController {
    @Autowired
    TaskService service;

    @GetMapping("/tasks")
    public List<Task> getAllTask() {
        return this.service.getAllTasks();
    }
    
    @PostMapping("/task")
    public Task createTask(@RequestBody Task task) {
        return this.service.createTask(task.getTitle(), task.getDescription(), task.isCompleted());
    }

    @PutMapping("task/{id}")
    public Task updateTask(@PathVariable("id") Long id, @RequestBody Task task) {        
        return this.service.updateTaskById(id, task);
    }

    @DeleteMapping("task/{id}")
    public void deleteTask(@PathVariable("id") Long id) {
        this.service.removeTaskById(id);
    }
}
```

### Step 6: Configure Spring Profiles and MySQL in application.properties

Create application.properties files as following:
```java
# application-deploy.properties
spring.application.name=ToDoApp
spring.jpa.hibernate.ddl-auto=update
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
server.port=5000
```

```java
# application-local.properties
spring.datasource.url=jdbc:mysql://localhost:3306/todo_list
spring.datasource.username=todo_user
spring.datasource.password=user@123
```

```java
# application-deploy.properties
spring.datasource.url=${SPRING_DATASOURCE_URL}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD}
```

## Setting Up the Database with MySQL
Instead of installing a local MySQL instance, we will create a Docker container from official MySQL image to set up the required database for the To-Do application.

### Step 1: Create a mysql container
docker run --name local-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=<rootpassword> mysql

### Step 2: Create database to store tasks
Create a database ```todo_list```, a user ```todo_user``` with password ```user@123``` to stores the tasks by creating a terminal session to local-mysql container,
```sql
docker exec -it local-mysql /bin/bash
```

login to mysql with root user (provide the rootpassword you have choosen in step 1),
```sql
mysql -u root -p 
```

and execute the commands from ```create_database.sql``` file from ```database``` directory from final project or refer the [Step 3: Backend - Create Dockerfile for database from Containerizing with Docker section](https://github.com/sabalessandip/blogs/blob/main/3-Tier%20To-Do-blog.md#step-3-backend---create-dockerfile-for-database-as-following) below.

## Test the 3-Tier App Manually
Before deploying the application using Docker Compose, we can manually test the setup.

### Step 1: Database - Ensure the database is already running as a Docker container using the commands from the previous section.

### Step 2: Backend - Run the backend Spring Boot project by setting the Spring profile to local.
```sh
java -jar build/libs/application.jar --spring.profiles.active=local
```

### Step 3: Frontend - Modify the base URL in api_service.dart and run the Flutter project.
```dart
static const String baseUrl = 'http://localhost:8080';
```











## Containerizing with Docker
I have packaged the frontend and backend on my machine and created the docker images with the built packages to simulate real-world scenarios. However, we could write a Dockerfile to build and package during image creation using multi-stage Docker builds. So, Before creating frontend and backend images, package them using the following commands.

**Frontend:**
```sh
flutter config --enable-web
flutter build web
```

**Backend:**

```sh
./gradlew clean build
```

### Step 1: Frontend - Create Dockerfile in the Flutter project root as following,
```Dockerfile
# Dockerfile for Flutter Web
# Base image
FROM nginx:stable-alpine

# Set working directory in the container
WORKDIR /usr/share/nginx/html

# Clear working directory
RUN rm -rf ./*

# Copy packaged flutter app
COPY /build/web .

# Setup nginx
CMD ["nginx", "-g", "daemon off;"]
```

### Step 2: Backend - Create Dockerfile in the Spring Boot project root as following,
```Dockerfile
# Dockerfile for Spring Boot
# Set base image
FROM openjdk:21-jdk

# Set the working directory in the container
WORKDIR /app

# Copy the JAR file into the container
COPY build/libs/todoapp-1.0.0.jar app.jar

# Define arguments
ARG SPRING_DATASOURCE_URL
ARG SPRING_DATASOURCE_USERNAME
ARG SPRING_DATASOURCE_PASSWORD

# Set environment variables
ENV SPRING_DATASOURCE_URL=${SPRING_DATASOURCE_URL}
ENV SPRING_DATASOURCE_USERNAME=${SPRING_DATASOURCE_USERNAME}
ENV SPRING_DATASOURCE_PASSWORD=${SPRING_DATASOURCE_PASSWORD}

# Set spring boot application profile
ENV SPRING_PROFILES_ACTIVE=deploy

# Expose the port the application runs on
EXPOSE 5000

# Run the JAR file
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Step 3: Backend - Create Dockerfile for database as following,
Earliyer, we manually executed commands to create database, user and table for storing task on mysql instance we created from official docker image. 
When a mysql container is started for the first time, it executes files with extensions ```.sh```, ```.sql``` and ```.sql.gz``` that are found in ```/docker-entrypoint-initdb.d```. directory. So, to automate the process, create ```create_database.sq``` file and ```Dockerfile``` in database directory as following,

```sql
-- Create the database
CREATE DATABASE todo_list;

-- Create 'todo_user' user
CREATE USER 'todo_user'@'%' IDENTIFIED BY 'user@123';
-- Grant all privileges on the database to the user 'todo_user'
GRANT ALL PRIVILEGES ON todo_list.* TO 'todo_user'@'%';

-- Apply the changes
FLUSH PRIVILEGES;

-- Select the database to use
USE todo_list;

-- Create the tasks table
CREATE TABLE tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    completed BOOLEAN DEFAULT FALSE
);
```

```Dockerfile
# Set base image
FROM mysql:latest

# Copy sql file to required directory
COPY create_database.sql /docker-entrypoint-initdb.d/

# Accept the root password for mysql
ARG MYSQL_ROOT_PASSWORD

# Set the root password to mysql environment variable
ENV MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
```
### Step 4: Create Docker Compose File
Create ```docker-compose.yml``` in the project root i.e parent of frontend, backend and database folder as following,

```yaml
version: '3.9'
services:
  database:
    container_name: database
    build:
      context: ./database
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: "root@123"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  backend:
    container_name: backend
    build:
      context: ./backend
    ports:
      - "5000:5000"
    environment:
      SPRING_DATASOURCE_URL: "jdbc:mysql://database:3306/todo_list"
      SPRING_DATASOURCE_USERNAME: "todo_user"
      SPRING_DATASOURCE_PASSWORD: "user@123"
    depends_on:
      database:
        condition: service_healthy

  frontend:
    container_name: frontend
    build:
      context: ./frontend
    ports:
      - 80:80
    depends_on:
      - backend
```

### Step 5: Deploy the Application stack
Navigate to the directory containing docker-compose.yml and run:

```sh
docker-compose up -d
```

This command builds and starts the MySQL database, Spring Boot backend, and Flutter web frontend containers.

### Step 6: Access the Application
Once the containers are up and running, you can access the To-Do List application by navigating to http://localhost in your web browser.

## Conclusion
Creating a 3-tier To-Do List web application using Flutter, Spring Boot, and MySQL, and deploying it with Docker Compose, provided a comprehensive learning experience. This project allowed me to explore different aspects of full application stack, from frontend design with Flutter Web to backend development with Spring Boot, and finally, containerized deployment using Docker.

I hope this guide helps you in your journey to explore backend technologies and containerized deployments. Happy coding!