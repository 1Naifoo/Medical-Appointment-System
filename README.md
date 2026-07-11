# Medical Appointment System

A unified, cross-platform digital healthcare platform designed to streamline appointment scheduling, remote teleconsultations, secure payment processing, and clinical record management. The project bridges patient interaction via a mobile application with hospital administration via a web-based portal.

---

## 📋 Table of Contents
- [Project Overview](#-project-overview)
- [System Architecture](#%EF%B8%8F-system-architecture)
- [Core Features & User Roles](#-core-features--user-roles)
- [Technical Stack & Integrations](#-technical-stack--integrations)
- [Database Design](#-database-design)
- [Core Business Logic & Workflows](#-core-business-logic--workflows)
- [Testing & Quality Assurance](#-testing--quality-assurance)
- [Project Boundaries (Out of Scope)](#-project-boundaries-out-of-scope)

---

## 🌟 Project Overview
Traditional healthcare scheduling faces severe bottlenecks, including manual coordination errors, long physical waiting queues, and restricted access for individuals in remote areas or with physical limitations. 

This project delivers a robust, secure, and user-centered solution that digitizes clinical workflows. It supports four distinct stakeholder groups (Patients, Doctors, Pharmacists, and Administrators) to guarantee accurate information flow from initial slot booking to electronic prescription fulfillment.

---

## 🗺️ System Architecture
The platform implements a modular client-server architecture with strict separation of layers to maximize portability, maintainability, and horizontal scalability.
┌────────────────────────────────────────────────────────┐
│                   PRESENTATION LAYER                   │
│   ┌──────────────────────────┐  ┌──────────────────┐   │
│   │   Patient Mobile App     │  │   Staff Portal   │   │
│   │        (Flutter)         │  │     (Laravel)    │   │
│   └──────────────────────────┘  └──────────────────┘   │
└───────────────────────────┬────────────────────────────┘
│ HTTPS / REST JSON
┌───────────────────────────▼────────────────────────────┐
│                  BUSINESS LOGIC LAYER                  │
│               ┌──────────────────────────┐             │
│               │       Backend API        │             │
│               │        (Laravel)         │             │
│               └────────────┬─────────────┘             │
└────────────────────────────┼───────────────────────────┘
│ SQL Connect
┌────────────────────────────▼───────────────────────────┐
│                       DATA LAYER                       │
│               ┌──────────────────────────┐             │
│               │       MySQL Server       │             │
│               └──────────────────────────┘             │
└────────────────────────────────────────────────────────┘

1. **Presentation Layer (Client Side):** A cross-platform Flutter application tailored for end-user patients alongside a responsive Laravel blade-rendered web portal dedicated to hospital workers.
2. **Business Logic Layer (Server Side):** A centralized RESTful API framework handling role-based access management, security validation, third-party handshakes, and transaction execution.
3. **Data Layer (Storage):** A structured MySQL relational instance storing profiles, historical appointments, logs, and cryptographic hashes safely.

---

## 👥 Core Features & User Roles

### 📱 1. Patient Module (Mobile App)
* **Account Provisioning:** Real-time validated sign-up and login utilizing localized configurations for English/Arabic bilingual view and Light/Dark dynamic themes.
* **Unified Appointment Flow:** Multi-step wizard to selectively pick clinical departments, verify available specialists, calendar select dates, and choose generated timeslots.
* **Integrated Telemedicine:** Join temporary virtual video conference appointments with medical providers using direct deep-linking.
* **History & Ledger:** Review complete medical timelines including past invoices, canceled receipts, and prescriptions.

### 💻 2. Doctor Module (Web Portal)
* **Schedule Coordinator:** Aggregated operational view classifying daily schedules into *Upcoming*, *Completed*, or *Canceled* states.
* **Tele-Consultation Interface:** Direct single-click button connection to the active Jitsi call with full display-role identification.
* **Digital Prescription Desk:** Create structural prescription documents (JSON-formatted arrays containing medication instructions and administrative comments) transmitted instantly to pharmacy queues.

### 💊 3. Pharmacist Module (Web Portal)
* **Patient Lookup:** Fast indexing capability to retrieve active patient logs by string names, registered emails, or official government ID numbers.
* **Prescription Dispatch:** Access real-time digital medical notes drafted by clinicians to fill precise orders safely, mitigating manual reading mistakes.

### 🛠️ 4. Administrator Module (Web Portal)
* **System CRUD Controller:** Master structural configuration authority to build, read, update, or deactivate entities inside the system (Clinics, Doctors, and Patient Accounts).
* **Audit Dashboard:** Analytical overview aggregating core business metrics (e.g., Total Users, Active Invoices, Cancelled Slots) with complete financial transaction records.

---

## 🛠 Technical Stack & Integrations

* **Frontend:** Flutter SDK (Dart) for high-performance platform-agnostic compilation (Android 10.0+ / iOS 14.0+).
* **Backend:** Laravel Framework (PHP) providing modular REST API bridges with structured snake_case notation.
* **Database:** MySQL Server 8.0 implementing ACID compliance, referential constraints, and optimized indices.
* **Payment Processing (Stripe Integration):** Employs the native Stripe SDK via a **Reserve-Pay-Confirm** pattern. Includes backend `PaymentIntent` architecture paired with asynchronous background server-side webhooks (`payment_intent.succeeded`) to safely resolve edge cases like sudden client disconnection.
* **Video Conferencing (Jitsi Meet Integration):** Integrated open-source RTC video infrastructure. Rooms are named dynamically after appointment primary keys (`appointment_123`). A specialized server-side endpoint generates secure short-lived entry tokens checking ownership metrics before authorization. Sessions are strictly live and never persisted on disk to guarantee compliance with medical privacy policies.

---

## 🗄️ Database Design
The relational model enforces high data integrity across key entities:

### 🧩 Entity Relationship Summary
* `USERS` ───[ 1 : N ]───► `APPOINTMENTS` *(One patient can book multiple sessions)*
* `CLINICS` ───[ 1 : N ]───► `DOCTORS` *(Clinics house specialized medical specialists)*
* `DOCTORS` ───[ 1 : N ]───► `APPOINTMENTS` *(Doctors manage a schedule of visits)*
* `APPOINTMENTS` ───[ 1 : 1 ]───► `PAYMENTS` *(Each valid booking maps to exactly one Stripe transaction)*
* `APPOINTMENTS` ───[ 1 : 1 ]───► `CONSULTATION_SESSIONS` *(Tracks live teleconsultation duration/links)*
* `DOCTORS` ───[ 1 : N ]───► `PRESCRIPTIONS` *(Clinicians log drugs directly inside a JSON array field)*

---

## 🔄 Core Business Logic & Workflows

### ⏳ 1. Expiration Slot Locking (Concurrency Management)
To prevent double-booking conflicts during client checkout checkout, the Laravel backend uses a 15-minute expiration block window:
1. When a patient chooses a time, the server generates an entry with status `pending_payment` and sets a `reserved_until` column to `now()->addMinutes(15)`.
2. Standard slot searches exclude records locked by active, unexpired pending reservations.
3. Pre-intent validation checks this timestamp again directly before firing external payment sheets to avert expired checkout actions.

### 🛑 2. Cancellation Policy Validation
* Cancellations alter record flags to `canceled` to preserve history logs for administrative audits rather than removing rows permanently.
* The software architecture limits refund entitlement rules via an automated validation logic: cancellations must be processed at least 24 hours prior to slot times, otherwise execution paths are blocked.

---

## 🧪 Testing & Quality Assurance
The platform was validated using rigorous manual execution procedures tracking logic compliance back to software specifications:

* **Functional Testing (Black Box):** Evaluated explicit boundary parameters, password notation metrics (minimum 8 text characters containing symbols), and role interfaces.
* **Integration Testing:** Verified multi-system communication loops verifying that external Stripe authorization calls properly update the MySQL data storage layer and trigger real-time mobile app dashboard changes.
* **Core Defect Resolution:** Caught an authentication gap where expired slot locking structures allowed payment sheet execution. Resolved this by implementing immediate pre-flight server validation checks inside `AppointmentController.php` to drop expired slots gracefully with a `409 Conflict` API code.

---

## ⚠️ Project Boundaries (Out of Scope)
To ensure a structured prototype release cycle, the following parameters are strictly out of scope:
* **Automated External Recovery:** Reset workflows for user passwords via automated external mail server routing are bypassed; adjustments must happen internally on user profile panels.
* **Automated Stripe Cash Reversals:** Refund database changes register inside storage blocks automatically, but actual asset reversal processes must be manually initiated inside the official Stripe Dashboard by hospital managers.
* **Advanced Biometrics & Hardware Security:** Complex multi-factor validation, fingerprint recognition (Touch ID), or specialized hardware-level data encryption is deferred to subsequent updates.
* **Clinical Diagnostic Systems:** The portal does not replace formal Electronic Health Record environments, provide automated computer-generated symptom medical advice, or manage delivery logistics for physical pharmacy distribution.
