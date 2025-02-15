# Task Management App Backend

## Table of Contents
- [Description](#description)
- [Technologies used](#technologies-used)
- [Installation & Configuration](#installation--configuration)
- [Steps to run](#steps-to-run)
- [Design decisions: Approach for the "(Should have)" user stories](#design-decisions-approach-for-the-should-have-user-stories)
- [Design decisions: Mitigating the risk of huge volumes of tasks created](#design-decisions-mitigating-the-risk-of-huge-volumes-of-tasks-created)
- [Design decisions: Building the application for efficient scaling](#design-decisions-building-the-application-for-efficient-scaling)
- [Further improvements](#further-improvements)
- [Tests](#tests)


## Description
This project is a task management application inspired by a hypothetical scenario:

### The Task

You've been assigned to a team working on building out a new task management software. Over the course of a few days, many customer interviews & user mapping flows, you and your product manager arrive together at the following set of user stories.

- User should be able to create a new task, including the following fields **(Required)**
  - Name
  - Description
  - Due date
- User should be able to view all tasks created in a list view, showing all the following details **(Required)**
  - Name
  - Description
  - Due date
  - Create date
  - Status
    - Not urgent
    - Due soon (Due date is within 7 days)
    - Overdue
- User should be able to edit task name, description and due date **(Required)**
- User should be able to sort by due date or create date **(Should have)**
- User should be able to search based on task name **(Should have)**

You also discussed that one of the key risks would be that there may be huge volumes of tasks created, in the 10s of 1000s, and wanted to ensure that the system would still be performant for users.


## Technologies used
The technologies included in this entire project (including the [front-end](https://github.com/AJLandry1000000000/task-management-app-frontend-technical-assessment), [back-end](https://github.com/AJLandry1000000000/task-management-app-backend-technical-assessment), containerisation, database, etc) are...
- Backend: Typescript
- Frontend: React + Vite 
- Package manager: Node Package Manager (npm)
- Containerisation: Docker
- Database: Postgres


## Installation & Configuration
Note: To complete these steps you will need to install NodeJS, Typescript, Docker, and use some IDE.

### Clone the repository
Run the following commands in your terminal to clone the code repository:  
```git clone https://github.com/AJLandry1000000000/task-management-app-backend-technical-assessment.git```

### Install the packages
Run the following commands to your root directory to install the node packages:  
```npm install```

### Pull the Postgres docker image and configure
(Note: Take note of the information you use to configure the database and container because you will need them when filling in your ```.env``` file.)  
Pull the PostgreSQL image:  
```docker pull postgres```  

Create and run a Docker container:  
```docker run --name ***put-your-container-name-here*** -e POSTGRES_PASSWORD=***put-your-db-password-here*** -p 5432:5432 -d postgres```  
When you run a PostgreSQL Docker container, a superuser named postgres is automatically created, and the POSTGRES_PASSWORD you provide is assigned as its password.  

Connect to the Postgres image:  
```docker exec -it ***put-your-container-name-here*** psql -U postgres```

It is best practice to avoid using the superuser for applications. So we will create a user and give them the minimum necessary permissions.  
In the Postgres terminal create a user:  
```CREATE USER ***your-user-name-here*** WITH PASSWORD '***your-user-password-here***';```  

In the Postgres terminal create a new database for our task management application:  
```CREATE DATABASE task_management_db;```  

In the Postgres terminal grant all privileges to the new user so they can create and query from the new database:  
```ALTER USER ***your-user-name-here*** WITH SUPERUSER;```

(Optional) Now in your terminal you should be able to connect to your database using your username and password:  
```docker exec -it ***put-your-container-name-here*** psql -U ***your-user-name-here*** -d task_management_db```

### Put your credentials in the .env folder in the root directory
Your ```.env``` file should contain the information you used to run the above commands:  
```
# General
PORT=9000

# DB
DB_HOST="localhost"
DB_PORT=5432
DB_NAME="task_management_db"
DB_USER="***some user name***"
DB_USER_PASSWORD="***some password***"
DB_PASSWORD="***some password***"

# seed
DEFAULT_PASSWORD="***some password***"
```

### Build your database schema
Run the knex migrations to build your database tables:  
```npm run migrate:latest```  

## Steps to run
### Include the scripts in package.json
Once correctly configured, run the following commands in the root directory of the project:  
  
Start the docker image (if it's not already running):
```docker start ***put-your-container-name-here***```

In your root folder run the following ```package.json``` script to compile and start the application:  
```npm run start```

Your back-end is now running!

## Design decisions: Approach for the "(Should have)" user stories
Both "(Should have)" user stories concern modifying the task list viewable from the frontend. 
### Back-end changes
My approach was to use a single endpoint (found in src/router/index.ts, ```GET /tasks-get/``` endpoint) to handle the sorting and searching functionality required in these user stories. 

The ```/tasks-get/``` endpoint defines the expected parameters in the 'expectedParams' type definition.
By default this endpoint shouldn't search for any task name or filter in any way, unless specified by the request parameters. 

### Front-end changes
The frontend has a button named "Filter Tasks". When clicked, this button should display a dropdown with filtering option. When selected, these filtering option fill a variable, ```getTaskPayload```, for the ```/tasks-get/``` endpoint and retrigger the fetching logic. 
Filling this ```getTaskPayload``` variable with the search/ordering selections, and editing the ```useEffect()``` function in App.jsx (see code edit below) will automatically fill the frontend task list with the tasks filtered by the user. 
```
  useEffect(() => {
    getAllTasks(getTaskPayload).then((data) => {
      setData(data.data)
    }).catch(error => {
      console.error("Error fetching data:", error)
    })

  }, [getTaskPayload])
```
<br>
This satifies the "(Should have)" user stories. (Note that the ```/tasks-get/``` endpoint uses pagination. So adding filtering options to the fetching query will automatically return paginated results, mitigating the risk of huge task volumes causing performance issues.)


## Design decisions: Mitigating the risk of huge volumes of tasks created
My approach was to use pagination and to use an indexed RDS for fast database queries.  

Pagination: This helps to load data in chunks rather than all at once, reducing the load on the frontend and improving the user experience.  
Indexed RDS: Indexing your database can significantly speed up query performance by allowing the database to quickly locate the rows that match the query conditions.

See the "Further improvements" section for more ideas on mitigating the volume related risks.

## Design decisions: Building the application for efficient scaling
### Model.ts super class
For this app, our database model could have just used one class e.g. TaskModel. But as the application grows, and the database tables grow, we will need basic features required from them all. Using a super class to define these common basic functions, then extending this super class to define more specific logic is the efficient design for codebase scalability. 

### Routes
Similarly, routes for this app could have been defined directly in src/index.ts. But since I am coding under the assumption this codebase will grow, I put them in a separate src/routes folder, where each file definded its own routes. 
This organises the logic as our endpoints grow in number and complexity.

### Using Knex for database modifications
The database modifications could be done directly in the Postgres database using SQL. But when a large engineering team is scaling an application with multiple database instances (e.g. dev, staging, production), having a central database management tool is essential. So I used Knex.


## Further improvements
The following methods could be used to further mitigate the risk of volume related issues:
- Caching requests: Use caching tools (e.g., using Redis or Memcached) to store frequently accessed data in memory, reducing the load on the database.
- Query optimisation: Regularly analyze and optimise database queries. Focus on optimising the most used and/or slowest queries first.
- Load balancing: Implement load balancing to distribute incoming traffic across multiple servers, ensuring no single server becomes a bottleneck (critical as our application and user base grows).
- Global databases: Use "global" database tables in the cloud (e.g. Amazon Aurora Global Database) if our app is used globally and needs low latency and high availablility across continents.

The following features should be added to improve our applications features:
- Add a user management system (e.g. with a sign-in page for users and admins)
    - Create multiple user types e.g. users and admins.
    - Create a log-in page for each user type.
    - Create a user UI, which allows users to view and create tasks. Allow them to only edit their own tasks.
    - Create an admin UI, which allows admins to action and close tasks.
- Add a comment section to tasks so users and admins can discuss a raised task.
- Add labelling of tasks so they can be organised. This is critical for large volumes of tasks.
- Build authorisation methods for our server so it is more secure.
- Add more filtering options. e.g. filter tasks by a label, filter tasks by tasks created by the current user, etc.
- Add more tests for the back-end.
- Add a test suite for the front-end.

## Tests
I have included test (in test/taskRoutes.test.ts) for the following endpoints:
- Check server health ```/health```
- Create tasks ```/tasks-create/```
- Update tasks ```/tasks-update/```
- Get tasks ```/tasks-get/:page```

I used Jest and supertest for the testing suite. 
Run the tests with this command:  
```npm run test```

Note: You will have to fill a ```.env.test``` file with dummy data similar to your ```.env``` file.