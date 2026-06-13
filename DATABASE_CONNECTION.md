# Database Connection Between Mobile App and Web Portal

## Overview

The mobile app and web portal both use the **same database**.

The database and Laravel backend are stored inside the **WAMP server**.

Both platforms communicate with the Laravel backend, and the backend handles all database operations.

---

# Simple Connection Flow

```text
Mobile App  ──►  Laravel Backend (WAMP Server)  ──►  Database
Web Portal  ──►  Laravel Backend (WAMP Server)  ──►  Database
```

---

# Mobile App Connection

The Flutter mobile app connects to the Laravel backend using API requests.

**File:** `frontend/lib/services/api_service.dart`

```dart
static const String baseUrl = 'http://127.0.0.1:8000/api';
```

The app sends requests to the backend running on the WAMP server.

The app also sends an authentication token:

```dart
'Authorization': 'Bearer ${AuthState.token}'
```

---

# Web Portal Connection

The web portal files are stored inside the WAMP server directory.

Example:

```text
wamp64/www/project-folder/
```

When the admin or doctor opens the website in the browser, the browser accesses the Laravel project through the WAMP server.

Example URL:

```text
http://localhost/project-folder/public
```

The Laravel backend then reads and writes data from the database.

---

# Backend Database Access

The Laravel backend handles all communication with the database.

**File:** `backend/app/Http/Controllers/AdminController.php`

```php
'total_users' => User::count(),
'total_appointments' => Appointment::count(),
```

This code reads information directly from the database.

---

# Database Configuration

The database connection is configured inside Laravel.

**File:** `backend/.env`

```env
DB_CONNECTION=sqlite
```

The database file is stored inside the backend project on the WAMP server.

---

# Final Summary

> The mobile app and web portal both connect to the Laravel backend hosted on the WAMP server.  
> The backend manages all communication with the shared database.