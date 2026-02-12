# Campus Book Exchange Platform (BookSwap) - Formative Assignment

## 1. Application Domain Description
This application is a dynamic book exchange system designed specifically for campus use. Students can list their idle books and browse items posted by others to initiate swaps. The system aims to facilitate resource circulation within the campus through a Single Page Application (SPA) to provide a smooth user experience.

## 2. Entity Type Identification
To implement the core functionality, the system defines the following two related entities:

* **Book**: Stores the unique identifier (ID), Title, Author, Category, and the Owner's User ID.
* **User**: Stores the unique identifier (ID), Name, Dormitory information, and contact details (e.g., Email).
* **Relationship**: The Book entity is linked to the owner via "User ID," implementing the "one user owns multiple books" logic.

## 3. REST API Design (GET & POST Methods)
The server will provide JSON data through the following interfaces:

### Book Entity
* **GET `/api/books`**: Returns a list of IDs and titles for all available books.
* **GET `/api/books/:id`**: Returns detailed information about a specific book, including the owner's contact profile.
* **POST `/api/books`**: Allows users to submit information for a new book to be exchanged.

### User Entity
* **GET `/api/users/:id`**: Retrieves a specific user's public profile and their list of posted books.
