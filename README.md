# Personal Document Organizer

Spring Boot midterm project. Users create **folders**, store **documents** in them (invoices, contracts, IDs, ...),
organize documents with **tags** and upload the actual **file** for each document.

**Tech:** Java 21, Spring Boot 4.1 (Web MVC, Data JPA, Validation), H2 database.

## Running

```bash
./mvnw spring-boot:run
```

The app runs on `http://localhost:8080`.

| What               | Where                                                                       |
|--------------------|-----------------------------------------------------------------------------|
| H2 database console | `http://localhost:8080/h2-console` (JDBC URL `jdbc:h2:file:./data/documents-db`, user `sa`, no password) |
| HTML upload form   | `http://localhost:8080/upload.html`                                         |
| Uploaded files     | `./uploads` folder                                                          |

## Project structure

```
com.documentorganizer
├── config        FileStorageConfig        (@Configuration – creates the upload folder)
├── controller    Folder/Document/TagController (@RestController)
├── service       Folder/Document/Tag/FileStorageService (@Service, @Transactional)
├── repository    Folder/Document/TagRepository (@Repository, extend JpaRepository)
├── entity        Folder, Document, Tag, DocumentType (@Entity)
├── dto           request/response records with validation annotations
└── exception     custom exceptions + GlobalExceptionHandler (@RestControllerAdvice)
```

## Data model

| Entity   | Fields                                                              |
|----------|---------------------------------------------------------------------|
| Folder   | id, name, description, createdAt                                    |
| Document | id, title, description, type, expiryDate, filePath, folder          |
| Tag      | id, name, color                                                     |

Document types: `INVOICE`, `RECEIPT`, `CONTRACT`, `CERTIFICATE`, `IDENTITY`, `OTHER`.

- `Folder` **@OneToMany** `Document` — deleting a folder deletes its documents
- `Document` **@ManyToOne** `Folder`
- `Document` **@ManyToMany** `Tag` (join table `document_tags`)

## API

### Folders

| Method | URL                                          | Description                                    |
|--------|----------------------------------------------|------------------------------------------------|
| POST   | `/api/folders`                               | Create a folder                                |
| GET    | `/api/folders?page=0&size=10&sort=name,asc`  | List folders (paged, newest first by default)  |
| GET    | `/api/folders/{id}`                          | Get one folder                                 |
| PUT    | `/api/folders/{id}`                          | Update a folder                                |
| DELETE | `/api/folders/{id}`                          | Delete a folder and its documents              |
| POST   | `/api/folders/{folderId}/documents`          | Create a document inside a folder              |
| GET    | `/api/folders/{folderId}/documents?type=INVOICE` | Documents of a folder (type is optional)   |

### Documents

| Method | URL                                                | Description                                   |
|--------|----------------------------------------------------|-----------------------------------------------|
| GET    | `/api/documents?page=0&size=10&sort=expiryDate,asc` | List documents (paged and sorted)            |
| GET    | `/api/documents/{id}`                              | Get one document                              |
| PUT    | `/api/documents/{id}`                              | Update a document                             |
| DELETE | `/api/documents/{id}`                              | Delete a document                             |
| GET    | `/api/documents/search?title=passport&type=IDENTITY` | Search by title, optionally by type (paged) |
| GET    | `/api/documents/expiring?days=30`                  | Documents expiring within the next N days (default 30) |
| GET    | `/api/documents/by-tag?name=important`             | Documents that have the given tag             |
| PUT    | `/api/documents/{id}/tags`                         | Assign tags, body `{ "tagIds": [1, 2] }`      |
| POST   | `/api/documents/{id}/file`                         | Upload the file (`multipart/form-data`, field `file`) |

### Tags

| Method | URL         | Description                |
|--------|-------------|----------------------------|
| POST   | `/api/tags` | Create a tag               |
| GET    | `/api/tags` | List all tags (by name)    |

### Example requests

```bash
# Create a folder
curl -X POST localhost:8080/api/folders -H "Content-Type: application/json" \
     -d '{"name": "Personal", "description": "Private papers"}'

# Create a document inside folder 1
curl -X POST localhost:8080/api/folders/1/documents -H "Content-Type: application/json" \
     -d '{"title": "Passport", "description": "Scan of my passport", "type": "IDENTITY", "expiryDate": "2027-06-30"}'

# Create a tag and assign it to document 1
curl -X POST localhost:8080/api/tags -H "Content-Type: application/json" \
     -d '{"name": "important", "color": "#FF5733"}'
curl -X PUT localhost:8080/api/documents/1/tags -H "Content-Type: application/json" -d '{"tagIds": [1]}'

# Upload the file for document 1
curl -X POST localhost:8080/api/documents/1/file -F "file=@/path/to/passport.pdf"

# Documents that expire within the next 365 days
curl "localhost:8080/api/documents/expiring?days=365"
```

### Postman

`postman/Personal-Document-Organizer.postman_collection.json` has 33 requests covering every endpoint, each with
tests. With the app running, run the whole collection in order: it creates data, checks every feature, then
deletes what it created.

- **Postman app:** Import the file → open the collection → **Run**. If the upload request shows the file as
  missing, select `postman/sample-document.txt` in its Body tab.
- **Postman CLI** (from the project folder): `postman collection run postman/Personal-Document-Organizer.postman_collection.json`

Invalid input returns `400` with the failing fields:

```json
{ "status": 400, "detail": "Validation failed", "errors": { "name": "Folder name is required" } }
```

## Assignment requirements → implementation

| Requirement                         | Where                                                                 |
|-------------------------------------|-----------------------------------------------------------------------|
| Layered packages + stereotypes      | `controller`, `service`, `repository`, `entity`, `config` packages    |
| Constructor injection only          | Every controller/service receives dependencies through its constructor |
| 3+ entities, 2 relationship types   | `Folder`, `Document`, `Tag` — `@OneToMany`/`@ManyToOne` and `@ManyToMany` |
| 2+ method-name queries              | `DocumentRepository`: `findByTitleContainingIgnoreCase`, `findByTitleContainingIgnoreCaseAndType`, `findByFolderId`, `findByFolderIdAndType` |
| 2+ `@Query` queries                 | `DocumentRepository`: `findExpiringBetween`, `findByTagName`          |
| `@Transactional`                    | All service methods that change data (read methods use `readOnly = true`) |
| HTTP method annotations             | `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` in controllers |
| `@PathVariable` / `@RequestParam` / `@RequestBody` | e.g. `DocumentController.uploadFile`, `searchDocuments`, `updateDocument` |
| Validation + `@Valid`               | `dto` records (`@NotBlank`, `@NotNull`, `@Size`, `@Pattern`, `@NotEmpty`) and `@Min` on `days` in `DocumentController.getExpiringDocuments` |
| File upload (`MultipartFile`)       | `DocumentController.uploadFile` → `FileStorageService`; path saved in `Document.filePath`; `upload.html` form uses `enctype="multipart/form-data"` |
| Pagination & sorting                | `Pageable` + `@PageableDefault` on list/search endpoints, `Sort` in `TagService` and `findExpiringBetween` |
