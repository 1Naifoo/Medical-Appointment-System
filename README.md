# Medical Appointment System

A complete digital healthcare platform built to make booking and managing medical appointments simple and stress-free. The project includes a mobile app for patients and a web portal for hospital staff (doctors, pharmacists, and administrators) so everyone stays safely connected.

---

## 📋 Table of Contents
- [Project Overview](#-project-overview)
- [How the System Works](#-how-the-system-works)
- [Core Features for Everyone](#-core-features-for-everyone)
- [Tools Used to Build the System](#-tools-used-to-build-the-system)
- [Database Structure](#-database-structure)
- [Important Rules & Features](#-important-rules--features)
- [How We Tested the System](#-how-we-tested-the-system)
- [What the System Does Not Do (Out of Scope)](#-what-the-system-does-not-do-out-of-scope)

---

## 🌟 Project Overview
Traditional hospital booking often comes with long waiting lines, accidental scheduling mistakes, and difficulty accessing doctors for patients who live far away or have trouble moving around. 

This project solves those issues by moving everything into a secure digital network. It helps patients book easily from home and helps hospital staff manage daily records and prescriptions without paperwork confusion.

---

## 🗺️ How the System Works
The system is cleanly divided into different parts to keep data safe and ensure the app loads quickly. 

![System Architecture](assets/System%20Architicture)

1. **The User Screens:** The easy-to-use screens that patients see on their phones and hospital staff see on their web browsers.
2. **The Main Brain (API Backend):** The hidden server that double-checks logins, processes bookings, and links everything together safely.
3. **The Secure Storage (Database):** A digital filing cabinet that saves user profiles, timeslots, appointment history, and prescriptions.

---

## 👥 Core Features for Everyone

### Project Functions Blueprint
![Use Case Diagram](assets/Use%20Case)

### 📱 1. For Patients (Mobile App)
* **Easy Account Setup:** Create an account securely with real-time helper guides, English or Arabic language choices, and a comfortable Dark Mode theme.
* **Simple Booking Wizard:** A step-by-step guide to pick a medical clinic, choose a preferred doctor, check open days, and select an open 30-minute timeslot.
* **Video Consultations:** Join a live, secure video call with your doctor directly inside the app so you do not have to drive to the clinic.
* **Personal Medical History:** Keep track of your past hospital bills, cancellation receipts, and active prescriptions anytime.

### 💻 2. For Doctors (Web Portal)
* **Daily Schedule Manager:** A clear dashboard that organizes the doctor's daily calendar into *Upcoming*, *Finished*, or *Canceled* visits.
* **Easy Video Join:** A single-click button that lets the doctor jump right into a video call with the waiting patient.
* **Digital Prescription Pen:** Send digital prescriptions instantly to the pharmacy database as soon as a consultation is completed.

### 💊 3. For Pharmacists (Web Portal)
* **Quick Patient Search:** Easily find a patient's folder by typing their name, email, or government ID card number.
* **Instant Prescription View:** Read clear, digital medicine orders sent directly by the doctor, preventing handwriting mistakes and dispensing medicine faster.

### 🛠️ 4. For System Administrators (Web Portal)
* **Hospital Control Center:** Full power to add, update, or temporarily turn off clinics, doctor profiles, and patient accounts.
* **Live Audit Dashboard:** A visual dashboard tracking crucial hospital numbers like total registered users, current payments, and canceled appointment metrics.

---

## 🛠 Tools Used to Build the System

* **Flutter:** The framework used to build the beautiful mobile app so it works flawlessly on both Android phones and iPhones.
* **Laravel (PHP):** The server programming language used to build the "brain" of the system, keeping all logic organized and secure.
* **MySQL:** The reliable database engine used to run the digital storage filing cabinet.
* **Stripe (Money Gateway):** A globally trusted payment framework integrated into the app. It uses an automated **Reserve-Pay-Confirm** workflow paired with intelligent background safety triggers. This means if your phone battery dies right as you pay, the system safely confirms your booking automatically.
* **Jitsi Meet (Video Engine):** The video system that powers the online consultations. It creates an isolated video chat room for every single appointment to keep calls private. Calls are completely live and are **never recorded or stored** anywhere to guarantee total patient privacy.

---

## 🗄️ Database Structure
The digital filing cabinet organizes data cleanly using safe relational connections so information never gets mixed up:

### Relational Storage Chart
![Entity Relationship Diagram](assets/Entity%20Relationship%20Diagram)

* Each **Patient** can book multiple separate appointments over time.
* Each **Clinic** department supports multiple professional doctors.
* Each **Appointment** connects to exactly one secure Stripe payment receipt and tracks its own live video session details.
* **Prescriptions** are safely tied to the specific appointment history and saved inside optimized data formats for the pharmacist to load instantly.

---

## 🔄 Important Rules & Features

### ⏳ 1. The 15-Minute Booking Lock
To make sure two patients do not accidentally pay for the exact same timeslot at the same time, the system uses an intelligent countdown lock:

![Booking Sequence Diagram](assets/Sequence%20Diagram)

1. The moment you tap on an open time slot, the system locks it just for you and marks it as `Pending Payment`.
2. The system holds the slot and gives you a 15-minute window to enter your card details.
3. Other patients browsing the app cannot see or steal that slot while your timer is running. If the timer runs out before you finish paying, the slot is immediately released for others to book.

### 🛑 2. Simple 24-Hour Cancellation Policy
* If a patient cancels an appointment, the system updates its status text to `Canceled` instead of deleting it, keeping records accurate for auditing.
* To respect the doctors' time, cancellations must be done at least 24 hours before the appointment. If a user tries to cancel at the last minute, the app will politely block the action and notify them of the 24-hour rule.

---

## 🧪 How We Tested the System
We put the entire application through strict real-world testing scenarios to verify that it is reliable and completely bug-free:

* **Screen Testing:** We checked every button, entry form, and language switch to ensure they respond perfectly and securely block invalid passwords.
* **Integration Testing:** We simulated real bank cards using Stripe to ensure that payment confirmations instantly update the hospital dashboard and lock in the correct slots.
* **Fixing Bugs:** During testing, we caught an issue where patients could accidentally finish checking out even if their 15-minute booking timer had run out. We immediately modified the system code to force a final time-check right before money is charged, safely correcting the issue.

---

## ⚠️ What the System Does Not Do (Out of Scope)
To ensure the initial prototype works perfectly, a few advanced options were left out of this version:
* **Automated Password Reset Emails:** There is no automated "Forgot Password" link sent to external emails; password changes are safely managed inside your active profile settings screen.
* **Automated Bank Refunds:** Cancelled bookings register in the database automatically, but actual money refunds must be manually triggered on the official Stripe website by a hospital admin.
* **Fingerprint/Face Recognition:** Biometric logins and multi-factor authentication are saved for future updates.
* **Automated Medical Advice & Delivery:** The app is strictly for scheduling and prescriptions. It does **not** give computer-generated medical diagnoses or manage physical medicine home delivery.
