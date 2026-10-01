# File Versioning and Recovery System

A college microservices project that accepts files from the browser, stores immutable versions on local disk, keeps version metadata in three separate MySQL databases, and records activity through an API.

## Project layout

    file-versioning-system/
    ├── database/                  MySQL setup and schema scripts
    ├── docs/                      API reference and demo walkthrough
    ├── frontend/
    │   ├── index.html             Page shell and library links
    │   ├── css/fileguard.css      Frontend styling
    │   └── js/app.jsx             React interface and API calls
    ├── services/
    │   ├── api-gateway/           Public API on port 8080
    │   ├── file-service/          File and folder metadata, port 8081
    │   ├── version-service/       Version metadata and workflow, port 8082
    │   ├── activity-service/      Recent Activity database API, port 8083
    │   └── storage-service/       Physical version files, port 8084
    ├── storage/versioned-files/   Actual uploaded version contents
    ├── pom.xml                    Builds all five Maven modules
    └── run-all.bat                Starts service and frontend windows

Each Java service has one Spring Boot application entry point. The controller and entity/repository code is kept in the service source package and has comments describing the service boundaries and APIs.

## Architecture

| Service | Port | Database | Responsibility |
|---|---:|---|---|
| API Gateway | 8080 | None | Browser entry point, service routing, upload/download byte forwarding, CORS |
| File Service | 8081 | file_service_db | File and folder records, duplicate lookup, delete and recovery |
| Version Service | 8082 | version_service_db | Version history, upload, restore, compare, retention |
| Activity Service | 8083 | activity_service_db | Persistent audit events and Recent Activity |
| Storage Service | 8084 | None | Physical file versions on local disk |

The frontend talks only to the Gateway at http://localhost:8080/api. Services communicate with each other through REST. No service reads another service's tables.

## MySQL setup

MySQL remains installed separately at localhost:3306. SQL scripts are in database/.

Run database/01_create_databases.sql, 02_file_service_schema.sql, 03_version_service_schema.sql, and 04_activity_service_schema.sql, then 05_grant_fileversion_user.sql. A MySQL administrator is needed only for these provisioning scripts. Application services must use DB_USERNAME=fileversion_user and DB_PASSWORD from the Windows user environment. Never configure the application as root.

Spring JPA uses update mode for additive schema changes. Storage Service has no fourth database.

## Versioned content and workflows

Actual bytes are stored under storage/versioned-files/file-{fileId}/version-{versionNumber}/{fileName}. Each version has its own directory and file. MySQL stores metadata including file ID, version number, path, byte size, original name, content type, timestamp, restored-from version, and status.

The Open File dialog uses the browser's native file chooser. The browser uploads the chosen file to File Service through the Gateway. File Service checks the active record by SHA-256 and then by file name and folder. A new file receives Version 1. An updated file is saved as a new version without overwriting earlier content.

Restoring a version copies its bytes into a new latest version and keeps previous history. Text files return both contents for line-level comparison in the UI. PDF, Office, archive, and image formats receive a clear metadata-only comparison message.

The maximum-version setting is applied when new versions are created. Old content is removed from disk before its version row is removed, and the Version Service records VERSION_AUTO_DELETED. File delete is a soft delete; Recently Deleted can recover a file while its history is kept.

Activity entries such as FILE_CREATED, VERSION_CREATED, VERSION_RESTORED, VERSION_COMPARED, VERSION_AUTO_DELETED, FILE_DELETED, and FILE_RECOVERED are stored in Activity Service and remain after a browser refresh.

The browser cannot watch arbitrary edits made in other desktop programs. After editing a downloaded file, choose Upload Saved Changes and select the updated copy to create a new version.

## Start the application

1. Start MySQL and make sure the three databases and application grants exist.
2. Set DB_PASSWORD for your Windows user and set DB_USERNAME to fileversion_user (run-all.bat defaults it to that account).
3. From this folder, run run-all.bat. It opens one visible terminal per service and, when Python is installed, a frontend server.
4. Wait for each Spring Boot window to report that it started, then open http://127.0.0.1:5500/.

To build and run the Maven checks without starting the services:

    mvn test

To debug one service, open a terminal at the project root and run:

    cd services/file-service
    mvn spring-boot:run

Replace file-service with api-gateway, version-service, activity-service, or storage-service as needed. Start MySQL and the called downstream services first.

## Test the system

Use the frontend Browse button to select a local file. Check that Version 1 appears, then download/edit and upload a same-named updated copy. Compare versions, restore an older version, change the retention limit, and inspect Recent Activity. Verify metadata in each service's own database and physical content beneath storage/versioned-files.

Additional request/response examples are in docs/API_DOCUMENTATION.md. A guided college demonstration is in docs/MICROSERVICES_DEMO.md.

## Common debugging

- MySQL login failure: confirm DB_USERNAME and DB_PASSWORD are set in the launching Windows session; never substitute root for the application account.
- Unknown database or table: run the matching database SQL script and restart that service.
- Port is already in use: stop the old service window/process or change that service's server.port and its URL in the other services.
- Upload returns 413 or 400: check the browser selected a file and the configured 100 MB upload limit.
- API returns 503: inspect the visible service console and confirm the downstream service is listening on its documented port.
- Frontend does not load API data: serve frontend/index.html over HTTP, use the provided port 5500, and keep the API Gateway on 8080.
- Versioned file missing: compare the version metadata's storagePath with the actual file beneath storage/versioned-files.
- Binary compare: content diffs are intentionally not supported for PDF/Office/image/archive files.

Settings are held in Version Service memory and reset to defaults after that service restarts. Activity records and file/version metadata are stored in MySQL.
