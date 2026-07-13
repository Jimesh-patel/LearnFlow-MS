# LearnFlow Microservices Database Schema

Because LearnFlow follows a **Database-per-Service** architecture, there are no hard foreign-key constraints linking tables across different databases. Instead, relationships are maintained logically using `UUID` references.

Here is the class diagram representing the JPA Entities across all microservices:

```mermaid
classDiagram
    direction TB

    namespace user-db {
        class User {
            +UUID id [PK]
            +String name
            +String email [Unique]
            +String password
            +Role role
            +String photoUrl
            +List~UUID~ enrolledCourseIds
            +Instant createdAt
            +Instant updatedAt
        }
    }

    namespace course-db {
        class Course {
            +UUID id [PK]
            +String courseTitle
            +String subTitle
            +String description
            +String category
            +CourseLevel courseLevel
            +Double coursePrice
            +String courseThumbnail
            +Boolean isPublished
            +UUID creatorId
            +List~UUID~ enrolledStudentIds
            +List~UUID~ lectureIds
            +Instant createdAt
            +Instant updatedAt
        }
    }

    namespace lecture-db {
        class Lecture {
            +UUID id [PK]
            +String lectureTitle
            +String videoUrl
            +String publicId
            +Boolean isPreviewFree
            +UUID courseId
            +UUID creatorId
            +Instant createdAt
            +Instant updatedAt
        }
    }

    namespace payment-db {
        class CoursePurchase {
            +UUID id [PK]
            +UUID courseId
            +UUID userId
            +String paymentId
            +Double amount
            +Status status
            +Instant createdAt
            +Instant updatedAt
        }
    }

    namespace progress-db {
        class CourseProgress {
            +UUID id [PK]
            +UUID userId
            +UUID courseId
            +boolean completed
        }
        
        class LectureProgress {
            <<Embeddable>>
            +UUID lectureId
            +Boolean viewed
        }
    }

    %% Logical Relationships (Cross-Database)
    User "1" ..> "*" Course : creatorId
    Course "1" ..> "*" Lecture : contains (lectureIds)
    Lecture "*" ..> "1" Course : belongs to (courseId)
    
    CoursePurchase "*" ..> "1" User : userId
    CoursePurchase "*" ..> "1" Course : courseId
    
    CourseProgress "*" ..> "1" User : userId
    CourseProgress "*" ..> "1" Course : courseId
    CourseProgress "1" *-- "*" LectureProgress : composition
```

### Key Takeaways for Interviews:
1. **No Distributed Joins:** Notice that `User` does not have a `@OneToMany` mapping to `CoursePurchase`. It just stores a `List<UUID>`. This is intentional to prevent tight coupling.
2. **Eventual Consistency:** When a `CoursePurchase` is created in the `payment-db`, a Kafka event is fired so the `user-service` can append the `courseId` to the `enrolledCourseIds` array in the `user-db`.
3. **Embeddable Types:** In the `progress-db`, `LectureProgress` isn't its own standalone entity; it's an `@Embeddable` object stored alongside the `CourseProgress`, meaning it's efficiently loaded without complex joins.
