---
layout: post
title: "System Design: Learning Platform Integration with People Management"
date: 2024-08-20
categories: [System Design]
tags: [learning platform, system design, people management, APIs]
image: assets/img/posts/20240820/banner.png
---

## Introduction

In today's fast-paced corporate environment, continuous learning is essential for people development and organizational growth. Designing a learning platform that seamlessly integrates with existing people management systems is a critical challenge. This post outlines a system design for a learning platform tailored to a company's needs, focusing on different personas: employees, managers, and course creators.


## 1. Clarify Requirements

Before designing the system, it's crucial to clarify the requirements:

### Functional Requirements:
- **Employees**: Browse and enroll in courses, track progress, and receive certificates upon completion.
- **Managers**: View reports on employee learning progress, compliance, and engagement.
- **Course Creators**: Create, manage, and analyze courses with the ability to upload multimedia content and assessments.

### Non-Functional Requirements:
- **Scalability**: The system should handle a growing number of users, courses, and data.
- **Security**: Sensitive data, such as employee progress and personal information, must be securely stored and transmitted.
- **Availability**: The system must be highly available, especially during peak usage times.

### Constraints:
- **Integration**: The learning platform must integrate with an existing employee management system to sync user data and learning stats.
- **Real-time Data**: Managers need near real-time reporting on employee progress.

## 2. Define API

Next, let's define the key APIs that the system will expose:

### Employee APIs:
- `GET /courses`: Fetch a list of available courses.
- `POST /enroll`: Enroll an employee in a course.
- `GET /progress`: Retrieve the progress of an employee in a specific course.
- `GET /certificate`: Download the certificate after course completion.

### Manager APIs:
- `GET /team-progress`: Get learning progress for all team members.
- `GET /compliance-report`: Fetch a compliance report for mandatory training.

### Course Creator APIs:
- `POST /create-course`: Create a new course with content and assessments.
- `PUT /update-course`: Update existing course content or assessments.
- `GET /course-analytics`: Retrieve analytics on course engagement and completion.

## 3. High-Level Design

### Overview

We'll design the system using a microservices architecture. The system will consist of the following components:

1. **Frontend Application**: A web interface for employees, managers, and course creators.
2. **User Service**: Manages user authentication, authorization, and roles.
3. **Course Service**: Handles course creation, enrollment, and progress tracking.
4. **Analytics Service**: Provides reports and analytics to managers and course creators.
5. **Data Integration Service**: Syncs data between the learning platform and the company's employee management system.

### High-Level Architecture Diagram

![High-Level Architecture](assets/img/posts/20240820/High-Level_Architecture.svg)

## 4. Detailed Design

### User Service
- **Responsibilities**: User authentication, authorization, and managing user roles.
- **Database Schema**: Stores user profiles, roles, and session tokens.
- **Tech Stack**: Use OAuth 2.0 for authentication, with JWT tokens for session management.

### Course Service
- **Responsibilities**: Manages course content, enrollment, and tracks employee progress.
- **Database Schema**:
  - **Courses Table**: Stores course details (ID, title, description, creator ID).
  - **Enrollments Table**: Tracks which employees are enrolled in which courses.
  - **Progress Table**: Tracks employee progress in each course (employee ID, course ID, completed modules).

### Analytics Service
- **Responsibilities**: Generates reports and dashboards for managers and course creators.
- **Data Storage**: Use a time-series database to efficiently store and query engagement metrics.
- **Tech Stack**: Utilize tools like Apache Kafka for real-time data streaming and ELK Stack (Elasticsearch, Logstash, Kibana) for analytics and reporting.

### Data Integration Service
- **Responsibilities**: Syncs employee data with the company's employee management system.
- **API Interactions**: Periodically fetches updates from the employee management system and updates the learning platform’s user database.

## Flows

### Course Consumption Flow

This section details the course consumption flow for the **Employee** persona, covering the steps for searching courses, enrolling in them, tracking progress, and consuming content.

**_NOTE:_** Let's assume the user is already loggedin and has the proper autorizations.

#### 1. Search Courses
- **User Action**: The employee searches for courses using keywords or filters (e.g., category, difficulty level).
- **System Process**:
  1. The frontend sends a `GET /courses?search=<keywords>&filters=<filters>` request to the Course Service.
  2. The Course Service queries the database for matching courses and returns a list of results.
  3. The frontend displays the results to the employee.

#### 2. Enroll in a Course
- **User Action**: The employee selects a course and clicks the "Enroll" button.
- **System Process**:
  1. The frontend sends a `POST /enroll` request to the Course Service with the employee's ID and the course ID.
  2. The Course Service updates the Enrollments table in the database to record the enrollment.
  3. The system sends a confirmation response back to the frontend.
  4. The frontend confirms enrollment and updates the UI, perhaps by adding the course to the employee's "My Courses" section.

#### 3. Course Progress Tracking
- **User Action**: As the employee progresses through the course, their progress is automatically saved (e.g., after completing a video or quiz).
- **System Process**:
  1. The frontend sends a `POST /progress` request to the Course Service with the employee ID, course ID, and progress details (e.g., module completed).
  2. The Course Service updates the Progress table in the database.
  3. The system sends a confirmation response back to the frontend, and the progress bar or status is updated on the UI.

#### 4. Watch Course Content
- **User Action**: The employee watches course videos, reads materials, and completes quizzes.
- **System Process**:
  1. When the employee starts a video or content module, the frontend sends a `GET /course-content?courseId=<courseId>&moduleId=<moduleId>` request to fetch the content.
  2. The Course Service retrieves the content from storage and streams or delivers it to the frontend.
  3. As the employee watches, the system periodically tracks engagement (e.g., how much of the video was watched) and sends this data to the backend via `POST /engagement` requests.
  4. This engagement data can be stored in a separate analytics database to track course effectiveness and user engagement.
- **Optimizations**:
  - Add cache of the cource metadata to relase the load from the databases
  - Consider the usage of a CDN. CDN's have the data closer to the end user, but at have high costs. Some optimization is required to push only the necessary data to the CDN, related to the SLA's in place.

#### Diagram for Course Consumption

Below is a the diagram that illustrates the interaction between the user and the system during the course consumption process.

![Course Consumption Diagram](assets/img/posts/20240820/Consumer_flow.svg)

### Manager Flow

This section explores the flow for the **Manager** persona, focusing on how managers interact with the system to view learning stats, monitor compliance, and generate reports.

To make this flow possible, we need to add some new features to the previous flow, and to the current one. The main suggestion is to add events publication to the enrollment and progress features. This will allow metrics/stats to be calculated in the manager flow. 

#### 1. View Team Learning Stats
- **User Action**: The manager logs in to the platform and navigates to the dashboard to view the learning progress of their team members.
- **System Process**:
  1. The frontend sends a `GET /team-progress?managerId=<managerId>` request to the Reporting and Analytics Service.
  2. The Reporting and Analytics Service queries the database to retrieve progress data for all employees managed by the manager.
  3. The system aggregates the data and sends a summarized report back to the frontend.
  4. The frontend displays the learning stats in the manager’s dashboard.

#### 2. Monitor Compliance
- **User Action**: The manager needs to ensure that their team has completed mandatory compliance training.
- **System Process**:
  1. The frontend sends a `GET /compliance-report?managerId=<managerId>` request to the Reporting and Analytics Service.
  2. The Reporting and Analytics Service filters the data to show only mandatory courses and their completion statuses.
  3. The system sends a compliance report back to the frontend.
  4. The frontend displays a list of employees who have or haven’t completed the mandatory training.

#### 3. Generate and Download Reports
- **User Action**: The manager wants to generate a detailed report of the team's learning activities and download it for further analysis.
- **System Process**:
  1. The manager selects the reporting parameters (e.g., date range, specific courses) and clicks "Generate Report".
  2. The frontend sends a `POST /generate-report` request with the selected parameters to the Reporting and Analytics Service.
  3. The Reporting and Analytics Service generates the report in the requested format (e.g., PDF, CSV) and stores it temporarily.
  4. The system sends a download link back to the frontend.
  5. The manager clicks the link to download the report.

### Diagram for Manager Flow

Below is a diagram that illustrates the interactions between the manager and the system during these operations.

![Manager Flow](assets/img/posts/20240820/Manager_flow.svg)

### Course Creation Flow

This section explores the flow for the **Course Creator** persona, focusing on how they create a course, upload videos, manage course materials, and publish the course.

#### 1. Create a New Course
- **User Action**: The course creator initiates the creation of a new course by providing basic details such as the course title, description, and category.
- **System Process**:
  1. The frontend sends a `POST /create-course` request to the Course Management Service with the provided course details.
  2. The Course Management Service creates a new entry in the Courses table of the database with a unique Course ID.
  3. The system sends a confirmation response back to the frontend with the Course ID.
  4. The frontend displays the next steps for adding content to the course.

#### 2. Upload Course Videos
- **User Action**: The course creator uploads video files as part of the course content.
- **System Process**:
  1. The frontend allows the course creator to select video files and initiates the upload process.
  2. The frontend sends a `POST /upload-video` request to the Video Upload Service with the video file and associated metadata.
  3. The Video Upload Service stores the video file in cloud storage and triggers a video transcoding job by publishing an event into the queue.
  4. The system sends a response back to the frontend confirming the upload and indicating that the video is being processed.

#### 3. Video Transcoding
- **User Action**: The course creator is notified that the video is being processed and can continue adding other content while waiting.
- **System Process**:
  1. The Video Upload Service sends a message to a Transcoding Service to start the transcoding process.
  2. The Transcoding Service converts the video into multiple formats and resolutions.
  3. Once transcoding is complete, the Transcoding Service stores the transcoded files in cloud storage and updates the Video Upload Service with the status.
  4. The Video Upload Service updates the database with the URLs of the transcoded video files and notifies the Course Management Service that the video is ready.
  5. The Course Management Service sends a notification to the frontend, which updates the course creator's interface to show that the video is ready for use.

#### 4. Add Additional Course Materials
- **User Action**: The course creator adds additional materials such as PDF documents, quizzes, and interactive content.
- **System Process**:
  1. The frontend sends a `POST /upload-material` request to the Course Management Service with the materials and associated metadata.
  2. The Course Management Service stores the materials in the appropriate storage service and updates the course content structure in the database.
  3. The system sends a confirmation response back to the frontend.

#### 5. Publish the Course
- **User Action**: Once all content is uploaded and organized, the course creator publishes the course.
- **System Process**:
  1. The frontend sends a `POST /publish-course` request to the Course Management Service.
  2. The Cource Management Service copies the objects to the final object storage and if necessary to the CDN.
  2. The Course Management Service updates the course status to "Published" in the database and makes it available to employees.
  3. The system sends a confirmation response back to the frontend, and the course is now accessible to employees on the platform.

### Sequence Diagram for Course Creation Flow

Below is a diagram that illustrates the interactions between the course creator and the system during these operations.

![Course Creation Sequence](assets/img/posts/20240820/create_course_flow.svg)


## 5. Scaling and Trade-offs

### Scalability
- **Horizontal Scaling**: Each microservice can be independently scaled horizontally to handle increased load.
- **Load Balancing**: Use load balancers to distribute traffic evenly across service instances.
- **Caching**: Implement caching for frequently accessed data, such as course lists, to reduce database load.
- **Stream processing**: Implement queueing into heavy processes to allow async processing.

### Trade-offs
- **Consistency vs. Availability**: Given the requirement for real-time reporting, prioritize availability. Use eventual consistency for non-critical data.
- **Cost vs. Performance**: Opt for managed cloud services for database and analytics to reduce operational overhead but at a higher cost.

### Bottlenecks
- **API Rate Limiting**: Ensure the APIs are rate-limited to prevent abuse and manage load effectively.
- **Data Sync**: The Data Integration Service might become a bottleneck if syncing large volumes of data. Use batch processing and prioritize critical updates.

## Conclusion

This system design provides a robust and scalable learning platform integrated with an existing employee management system. By focusing on clear APIs, a microservices architecture, and considering scaling trade-offs, this platform can effectively meet the diverse needs of employees, managers, and course creators within a corporate environment.

---