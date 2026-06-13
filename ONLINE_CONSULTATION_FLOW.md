# Online Consultation Connection

## Overview

The system allows patients and doctors to join an online video consultation using Jitsi Meet.

The mobile app and web portal both connect to the Laravel backend, and the backend generates the same meeting room for both users.

---

# Simple Connection Flow

```text
Patient Mobile App ──► Laravel Backend ──► Jitsi Meeting Room.
Doctor Web Portal ──► Laravel Backend ──► Jitsi Meeting Room.
```

Both users join the same room using the appointment ID.

---

# Booking Online Consultation

When the patient books an appointment, the mobile app sends the consultation type to the backend.

**File:** `frontend/lib/services/api_service.dart`

```dart
'consultation_type': 'online'
```

The Laravel backend saves the appointment as an online consultation.

**File:** `backend/app/Http/Controllers/Api/AppointmentController.php`

```php
$appointment = Appointment::create([
    'consultation_type' => 'online',
    'status' => 'pending_payment',
]);
```

---

# Create Meeting Room

The backend creates a unique room name for each appointment.

**File:** `backend/app/Http/Controllers/Api/AppointmentController.php`

```php
$roomId = 'appointment_' . $appointment->id;

return response()->json([
    'room_id' => $roomId,
]);
```

Example room:

```text
appointment_5
```

---

# Patient Joins From Mobile App

The mobile app requests the room information from the backend and opens the Jitsi video call.

**File:** `frontend/lib/video_call_page.dart`

```dart
final data = await ApiService().getMeetingToken(
  appointmentId: id
);

await launchUrl(jitsiUrl);
```

The patient joins the video call from the mobile application.

---

# Doctor Joins From Web Portal

The doctor joins from the web portal hosted on the WAMP server.

**File:** `backend/app/Http/Controllers/DoctorJoinMeetingController.php`

```php
$roomId = 'appointment_' . $appointment->id;
```

The web page opens the same Jitsi room in the browser.

**File:** `backend/resources/views/doctor_join_meeting.blade.php`

```javascript
new JitsiMeetExternalAPI(domain, {
  roomName: 'appointment_5',
});
```

---

# Final Summary

> The mobile app and web portal both connect to the Laravel backend, and the backend generates the same Jitsi meeting room so the patient and doctor can join the online consultation together.