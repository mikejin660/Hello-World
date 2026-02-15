# Campus Book Exchange Platform (BookSwap) 

## 1. Application Domain Description
This application is designed to facilitate resource circulation within a university campus by allowing students to exchange books efficiently. Student can borrow and post books that have vacancies to the online server to ensure our views of glancing it. The application will be implemented as a Single Page Application (SPA), ensuring a smooth and dynamic user experience without constant page reloads. 
Users can browse available books, post books for exchange, and initiate swaps with other users.The application emphasizes simplicity and transparency.


## 2. Entity Type Identification
To implement the core functionality, the system defines the following two related entities:

* **Book**: Stores the unique identifier (ID), Title, Author, Category, and the Owner's User ID.
* **User**: Stores the unique identifier (ID), Name, College information, and contact details (e.g., Email).
* **Relationship**: The Book entity is linked to the owner via "User ID," implementing the "one user owns multiple books" logic.

## 3. REST API Design (GET & POST Methods)
The server will provide JSON data through the following interfaces:

### Book Entity
* **GET `/api/books`**: Returns a list of IDs and titles for all available books.
* **GET `/api/books/:id`**: Returns detailed information about a specific book, including the owner's contact profile.
* **POST `/api/books`**: Allows users to submit information for a new book to be exchanged.

### User Entity
* **GET `/api/users/:id`**: Retrieves a specific user's public profile and their list of posted books.
