# Jitsi Meet video consultation

Video calls use **Jitsi Meet**: patient in app, doctor in browser, same room.

## Backend (Laravel)

- **Config:** `config/services.php` → `video_meeting.jitsi.server_url`.
- **Env (optional):** `JITSI_SERVER_URL=https://meet.jit.si` (default). Use your own Jitsi server if you have one.
- **API:** `GET /api/appointments/{id}/meeting-token` returns `room_id`, `user_id`, `user_name`, `jitsi_server_url` (no token needed).
- **Doctor join:** `GET /doctor/appointments/{id}/join-meeting` (session as doctor) loads the Blade view that embeds Jitsi with the same room name (`appointment_<id>`).

## Flutter (patient app)

- **Dependency:** `jitsi_meet_flutter_sdk`.
- **Flow:** App calls the meeting-token API; response includes `jitsi_server_url`. App joins via Jitsi:
  - **Android/iOS:** In-app Jitsi SDK (camera/mic permissions).
  - **Web:** Opens the Jitsi meeting URL in the browser (SDK does not support web).
- **Room name:** Same as backend: `appointment_<appointmentId>` so patient and doctor are in the same call.

## Doctor portal (browser)

- **View:** `backend/resources/views/doctor_join_meeting.blade.php`.
- Loads Jitsi Meet External API from the configured server, joins with `roomName = appointment_<id>` and the doctor’s display name. “Back to dashboard” returns to the doctor dashboard.
