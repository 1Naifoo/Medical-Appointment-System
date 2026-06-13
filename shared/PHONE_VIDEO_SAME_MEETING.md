# Physical device: join same meeting as doctor (like emulator)

When it worked on **emulator** but **not on physical device**, both sides must use the same Jitsi room and the phone must reach the backend.

## 1. Phone must reach the backend

On a **physical device**, the app cannot use `127.0.0.1` (that’s the phone itself). It must call your **PC’s IP** where the Laravel backend runs.

- **Emulator:** `API_BASE_URL=http://10.0.2.2:8000/api` (emulator’s alias to host).
- **Physical device:** `API_BASE_URL=http://YOUR_PC_IP:8000/api` (e.g. `http://192.168.1.5:8000/api`).

Run the app with the correct base URL, for example:

```bash
flutter run --dart-define=API_BASE_URL=http://192.168.1.5:8000/api
```

(Replace `192.168.1.5` with your PC’s LAN IP. Phone and PC must be on the **same Wi‑Fi**.)

## 2. Same room (Jitsi)

- **Room ID:** Backend and doctor page use `appointment_<id>` (e.g. `appointment_96`). Flutter uses the `room_id` from the API. All join the same Jitsi room.
- No app ID or token config needed for Jitsi (optional: `JITSI_SERVER_URL` in `.env` if using your own server).

## 3. Backend and appointment

- The appointment must be **online** (`consultation_type = 'online'`). Otherwise the API returns 400.
- No Zego or other video credentials required.

## 4. Quick test

1. **PC:** Backend running (e.g. `php artisan serve`), doctor dashboard open in browser.
2. **Phone:** Same Wi‑Fi as PC, app run with `API_BASE_URL=http://PC_IP:8000/api`.
3. **Patient (phone):** Log in, open an **online** appointment, tap Join consultation. Allow camera/mic.
4. **Doctor (browser):** Click Join meeting for that appointment. Allow camera/mic.
5. In **browser console** you should see: `[VIDEO_DEBUG] Doctor JOINING Jitsi: room=appointment_XX ...`
6. In **Flutter debug console** you should see: `[VIDEO_DEBUG] App JOINING Jitsi: room=...`

Same room name means you’re in the same Jitsi meeting.
