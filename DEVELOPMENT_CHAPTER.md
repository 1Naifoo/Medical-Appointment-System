# Development — Main Features (Code Snippets for Report)

This document highlights the main features of the system with code snippets and short explanations. Use it to decide what to include in your report’s **Development** chapter.

---

## 1. Online Booking (In-Clinic and Online Appointments)

The system supports two consultation types: **in_clinic** (physical visit) and **online** (video consultation). The user selects a clinic, then either “In-clinic” or “Online consultation,” then doctor and time slot. The backend reserves the slot and creates a payment record.

### 1.1 Backend: Reserve appointment (slot + consultation type)

The reserve endpoint creates an appointment in `pending_payment` and a linked payment. It validates `consultation_type` as `in_clinic` or `online`, checks for slot conflicts, and uses a short reservation window (e.g. 15 minutes) for payment.

**File:** `backend/app/Http/Controllers/Api/AppointmentController.php`

```php
public function reserve(Request $request)
{
    $user = $request->user();

    $validated = $request->validate([
        'clinic_id' => 'required|exists:clinics,id',
        'doctor_id' => 'required|exists:doctors,id',
        'starts_at' => 'sometimes|date_format:Y-m-d H:i',
        'date' => 'required_without:starts_at|date_format:Y-m-d H:i|date_format:Y-m-d',
        'time' => 'sometimes|date_format:H:i',
        'consultation_type' => 'required|in:in_clinic,online',
    ]);

    $startsAtRaw = $validated['starts_at']
        ?? (isset($validated['time']) && strlen($validated['date']) === 10
            ? ($validated['date'].' '.$validated['time'])
            : $validated['date']);

    $slot = Carbon::createFromFormat('Y-m-d H:i', $startsAtRaw, config('app.timezone'))->seconds(0);
    if ($slot->isPast()) {
        return response()->json(['message' => 'date must be in the future'], 422);
    }

    $expiresAt = now()->addMinutes(15);
    // ... conflict check with existing appointments ...

    $appointment = Appointment::create([
        'user_id' => $user->id,
        'clinic_id' => $validated['clinic_id'],
        'doctor_id' => $validated['doctor_id'],
        'date' => $slot->format('Y-m-d H:i:s'),
        'consultation_type' => $validated['consultation_type'],
        'status' => 'pending_payment',
        'reserved_until' => $expiresAt,
    ]);

    $payment = Payment::create([
        'appointment_id' => $appointment->id,
        'user_id' => $user->id,
        'amount_expected_cents' => $amountExpectedCents,
        'amount_charged_cents' => $amountChargedCents,
        'currency' => $currency,
        'status' => 'requires_payment',
        'stripe_payment_intent_id' => null,
    ]);

    return response()->json([
        'appointment_id' => $appointment->id,
        'payment_id' => $payment->id,
        'reserved_until' => $appointment->reserved_until ? $appointment->reserved_until->toIso8601String() : null,
        'expires_at' => $appointment->reserved_until ? $appointment->reserved_until->toIso8601String() : null,
        'amount_charged_cents' => (int) $payment->amount_charged_cents,
        'currency' => (string) $payment->currency,
    ], 201);
}
```

**Explanation:** Validation ensures `consultation_type` is either `in_clinic` or `online`. The same reserve flow is used for both; the type is stored on the appointment and used later (e.g. to allow “Join” only for online consultations).

---

### 1.2 Backend: Available time slots per doctor

Slots are generated in the clinic’s working window (e.g. 09:00–17:00 Malaysia time), excluding already booked or reserved slots.

**File:** `backend/app/Http/Controllers/Api/AppointmentController.php`

```php
public function doctorSlots(Doctor $doctor, Request $request)
{
    $date = $request->query('date'); // YYYY-MM-DD
    if (!$date) {
        return response()->json(['message' => 'date is required (YYYY-MM-DD)'], 422);
    }

    $malaysiaTz = new \DateTimeZone('Asia/Kuala_Lumpur');
    $start = new \DateTime("$date 09:00:00", $malaysiaTz);
    $end   = new \DateTime("$date 17:00:00", $malaysiaTz);

    $bookedAppointments = Appointment::where('doctor_id', $doctor->id)
        ->whereDate('date', $date)
        ->where(function ($q) {
            $q->whereIn('status', ['upcoming', 'completed'])
                ->orWhere(function ($q) {
                    $q->where('status', 'pending_payment')
                        ->where('reserved_until', '>', now());
                });
        })
        ->get();

    // Build blocked ranges (30 min per slot: 25 min + 5 min break)
    // ... then iterate 30-minute intervals, skip past times and blocked slots ...
    $slots[] = $cursor->format('H:i');

    return response()->json($slots);
}
```

**Explanation:** Slots are 30-minute intervals. Appointments in `upcoming`, `completed`, or `pending_payment` (still within `reserved_until`) block those slots so double-booking is avoided.

---

### 1.3 Frontend: Consultation type and navigation to booking

The app lets the user choose **in-clinic** or **online** before selecting a doctor. `DoctorsPage` receives `consultationType` and passes it to the reserve API.

**File:** `frontend/lib/online_consultation_clinics_page.dart` (online path)

```dart
onTap: () async {
  final result = await Navigator.push(
    context,
    MaterialPageRoute(
      builder: (_) => DoctorsPage(
        clinicId: clinic['id'] as int,
        clinicName: clinic['name'] ?? '',
        consultationType: 'online', // Set consultation type to online
      ),
    ),
  );
  if (result == true) {
    widget.onAppointmentBooked?.call();
  }
},
```

**File:** `frontend/lib/doctors_page.dart` (widget definition)

```dart
class DoctorsPage extends StatefulWidget {
  final int clinicId;
  final String clinicName;
  final String consultationType; // 'in_clinic' or 'online'

  const DoctorsPage({
    super.key,
    required this.clinicId,
    required this.clinicName,
    this.consultationType = 'in_clinic',
  });
}
```

**Explanation:** From “Online consultation” the user goes to `DoctorsPage` with `consultationType: 'online'`. From regular clinics, the same page is used with `'in_clinic'`. One booking flow supports both types.

---

### 1.4 Frontend: Reserve and open payment screen

After the user picks doctor and slot, the app calls the reserve API and navigates to the payment screen.

**File:** `frontend/lib/services/api_service.dart`

```dart
Future<Map<String, dynamic>> reserveAppointmentForPayment({
  required int clinicId,
  required int doctorId,
  required String date, // "yyyy-MM-dd HH:mm"
  String consultationType = 'in_clinic',
}) async {
  final trimmed = date.trim();
  final hasDateAndTime = trimmed.contains(' ');
  final startsAt = hasDateAndTime ? trimmed : null;

  final res = await http.post(
    Uri.parse('${ApiService.baseUrl}/appointments/reserve'),
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
      'Authorization': 'Bearer ${AuthState.token}',
    },
    body: jsonEncode({
      'clinic_id': clinicId,
      'doctor_id': doctorId,
      if (startsAt != null) 'starts_at': startsAt,
      if (startsAt == null) 'date': trimmed,
      'consultation_type': consultationType,
    }),
  );

  if (res.statusCode == 201) {
    return _expectJsonMap(res);
  }
  // ... error handling ...
}
```

**Explanation:** The client sends `consultation_type` and the chosen datetime (as `starts_at` or `date`). The backend returns `appointment_id`, `payment_id`, and `expires_at` so the app can show the payment gateway.

---

## 2. Consultation Feature (Online Video Consultation)

For appointments with `consultation_type = 'online'`, the patient and doctor join a video call in the same Jitsi Meet room (`appointment_<id>`). The backend provides meeting data (room id, display name, Jitsi server URL); the app uses it to open the call.

### 2.1 Backend: Meeting token for joining the call

Only online appointments can join. The API creates/updates a `ConsultationSession` and returns the room id and Jitsi server URL. Doctor and patient are distinguished via query or ownership.

**File:** `backend/app/Http/Controllers/Api/AppointmentController.php`

```php
/**
 * GET /api/appointments/{appointment}/meeting-token
 * Optional query: ?doctor_id=... when the doctor is joining.
 */
public function meetingToken(Request $request, Appointment $appointment)
{
    $user = $request->user();
    if (($appointment->consultation_type ?? '') !== 'online') {
        return response()->json(['message' => 'This appointment is not an online consultation.'], 400);
    }

    $roomId = 'appointment_' . $appointment->id;
    ConsultationSession::updateOrCreate(
        ['appointment_id' => $appointment->id],
        ['meeting_url' => $roomId, 'status' => 'scheduled']
    );

    $jitsiServerUrl = config('services.video_meeting.jitsi.server_url', 'https://meet.jit.si');

    // Doctor joining (with ?doctor_id=...)
    $doctorIdParam = $request->query('doctor_id');
    if ($doctorIdParam !== null && (int) $doctorIdParam === (int) $appointment->doctor_id) {
        $appointment->load(['doctor', 'user']);
        $doctor = $appointment->doctor;
        $patientName = $appointment->user ? ($appointment->user->name ?? 'Patient') : 'Patient';
        $joinUserId = 'doctor_' . $appointment->doctor_id;
        $joinUserName = $doctor ? $doctor->name : 'Doctor';
        return response()->json([
            'room_id' => (string) $roomId,
            'user_id' => (string) $joinUserId,
            'user_name' => (string) $joinUserName,
            'patient_name' => (string) $patientName,
            'jitsi_server_url' => (string) $jitsiServerUrl,
        ]);
    }

    // Patient joining (must be appointment owner)
    if ($appointment->user_id != $user->id) {
        return response()->json(['message' => 'Unauthorized: This appointment does not belong to you.'], 403);
    }
    $patientUserId = 'patient_' . $user->id;
    $patientName = $user->name ?? 'Patient';
    return response()->json([
        'room_id' => (string) $roomId,
        'user_id' => (string) $patientUserId,
        'user_name' => (string) $patientName,
        'jitsi_server_url' => (string) $jitsiServerUrl,
    ]);
}
```

**Explanation:** Consultation feature is gated by `consultation_type === 'online'`. Same room id for both roles ensures patient and doctor join the same Jitsi meeting.

---

### 2.2 Backend: Video meeting configuration

**File:** `backend/config/services.php`

```php
'video_meeting' => [
    'jitsi' => [
        'server_url' => rtrim(env('JITSI_SERVER_URL', 'https://meet.jit.si'), '/'),
    ],
],
```

**Explanation:** Jitsi server URL is configurable via env; default is public Meet. Doctor portal and app use this to build the meeting URL.

---

### 2.3 Frontend: Fetch meeting data and open Jitsi

The app requests meeting data for the appointment, then opens the Jitsi URL (e.g. in external browser) so the user joins the consultation.

**File:** `frontend/lib/video_call_page.dart`

```dart
class VideoCallPage extends StatefulWidget {
  final int appointmentId;
  final int? doctorId;

  const VideoCallPage({super.key, required this.appointmentId, this.doctorId});
}

// In state:
Future<void> _fetchMeetingToken() async {
  setState(() { _loading = true; _error = null; _meetingData = null; });
  try {
    final data = await ApiService().getMeetingToken(
      appointmentId: widget.appointmentId,
      doctorId: widget.doctorId,
    );
    if (mounted) {
      setState(() { _meetingData = data; _loading = false; });
    }
  } catch (e) {
    // ... set error state ...
  }
}

// When opening the call (conceptually):
// final roomId = _meetingData!['room_id'] ?? 'appointment_${widget.appointmentId}';
// final serverUrl = _meetingData!['jitsi_server_url'];
// final uri = Uri.parse('$serverUrl/${Uri.encodeComponent(roomId)}');
// await launchUrl(uri, mode: LaunchMode.externalApplication);
```

**Explanation:** `VideoCallPage` is used when the user taps “Join consultation.” It calls the meeting-token API and then launches Jitsi with the returned `room_id` and `jitsi_server_url` so patient and doctor are in the same room.

---

### 2.4 Frontend: Join action only for online upcoming appointments

**File:** `frontend/lib/AppointmentsPage.dart` (conceptually)

```dart
// Only for online + upcoming
final isOnline = (consultationTypeRaw ?? 'in_clinic') == 'online';
if (isOnline && status == 'upcoming') {
  _showJoinConsultationConfirm(context, appointmentId);
  return;
}
// After confirm:
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (_) => VideoCallPage(appointmentId: appointmentId),
  ),
);
```

**Explanation:** “Join” is shown only when the appointment is online and upcoming; tapping it confirms and opens `VideoCallPage`, which fetches the meeting token and starts the consultation.

---

## 3. Payment Feature (Stripe)

After reserving a slot, the user pays via Stripe Payment Sheet. The backend creates a PaymentIntent, the app presents the sheet, and payment is finalized either by webhook or by an explicit confirm call from the app.

### 3.1 Backend: Create Stripe PaymentIntent

The app requests a `client_secret` for the payment sheet. The backend creates (or reuses) a Stripe PaymentIntent and stores its id on the payment record.

**File:** `backend/app/Http/Controllers/Api/PaymentController.php`

```php
// POST /payments/{payment}/stripe-intent
public function createStripeIntent(Request $request, Payment $payment)
{
    if ((int) $payment->user_id !== (int) $request->user()->id) {
        abort(404);
    }
    $appointment = $payment->appointment()->first();
    if ($appointment->status !== 'pending_payment' || !$appointment->reserved_until || $appointment->reserved_until->isPast()) {
        return response()->json(['message' => 'Reservation expired or invalid.'], 409);
    }
    Stripe::setApiKey(config('services.stripe.secret'));

    if ($payment->stripe_payment_intent_id) {
        $intent = PaymentIntent::retrieve($payment->stripe_payment_intent_id);
        if ($intent->status === 'succeeded') {
            return response()->json(['message' => 'Payment already completed.'], 409);
        }
        if ($intent->status !== 'canceled') {
            return response()->json(['client_secret' => $intent->client_secret, ...], 200);
        }
    }

    $intent = PaymentIntent::create([
        'amount' => (int) $payment->amount_charged_cents,
        'currency' => strtolower((string) $payment->currency),
        'automatic_payment_methods' => ['enabled' => true],
        'metadata' => [
            'payment_id' => (string) $payment->id,
            'appointment_id' => (string) $payment->appointment_id,
            'user_id' => (string) $payment->user_id,
        ],
    ]);
    $payment->stripe_payment_intent_id = $intent->id;
    $payment->status = 'processing';
    $payment->save();

    return response()->json([
        'client_secret' => $intent->client_secret,
        'payment_intent_id' => $intent->id,
    ], 201);
}
```

**Explanation:** Only the payment owner can create the intent. Reservation must still be valid. Reusing a non-terminal intent avoids duplicate intents; metadata links Stripe events to our payment and appointment.

---

### 3.2 Backend: Confirm payment (app-triggered fallback)

When the webhook is not used, the app calls confirm after the sheet succeeds. The backend checks the intent status and, if `succeeded`, updates payment and appointment in a transaction.

**File:** `backend/app/Http/Controllers/Api/PaymentController.php`

```php
// POST /payments/{payment}/confirm
public function confirmStripePayment(Request $request, Payment $payment)
{
    $intent = PaymentIntent::retrieve($payment->stripe_payment_intent_id);
    if ((string) ($intent->status ?? '') !== 'succeeded') {
        return response()->json(['message' => 'Payment not yet succeeded.'], 409);
    }
    DB::transaction(function () use ($payment) {
        $payment->status = 'succeeded';
        $payment->paid_at = now();
        $payment->save();
        $appointment = $payment->appointment;
        $appointment->status = 'upcoming';
        $appointment->paid_at = now();
        $appointment->save();
    });
    return response()->json([
        'payment_status' => $payment->status,
        'appointment_status' => $appointment->status,
    ], 200);
}
```

**Explanation:** This gives a reliable way to move the appointment to `upcoming` when webhooks are not configured or are delayed.

---

### 3.3 Backend: Stripe webhook (payment_intent.succeeded)

**File:** `backend/app/Http/Controllers/Api/StripeWebhookController.php`

```php
public function handle(Request $request)
{
    $event = Webhook::constructEvent(
        $request->getContent(),
        $request->header('Stripe-Signature'),
        env('STRIPE_WEBHOOK_SECRET')
    );
    if ($event->type === 'payment_intent.succeeded') {
        $pi = $event->data->object;
        DB::transaction(function () use ($pi) {
            $payment = Payment::where('stripe_payment_intent_id', $pi->id)->lockForUpdate()->first();
            if (!$payment || $payment->status === 'succeeded') return;
            $payment->status = 'succeeded';
            $payment->paid_at = now();
            $payment->save();
            $appointment = $payment->appointment;
            $appointment->status = 'upcoming';
            $appointment->paid_at = now();
            $appointment->save();
        });
    }
    return response()->json(['received' => true], 200);
}
```

**Explanation:** Webhook verifies the signature and, on `payment_intent.succeeded`, marks the payment and linked appointment as paid. Route is unauthenticated; security is via `STRIPE_WEBHOOK_SECRET`.

---

### 3.4 Frontend: Payment screen — init sheet and finalize

The payment screen requests the Stripe intent, initializes the Payment Sheet with the `client_secret`, presents it, then finalizes on the backend (with retries).

**File:** `frontend/lib/payment_gateway_page.dart`

```dart
Future<void> _payWithStripe() async {
  if (_submitting) return;
  setState(() => _submitting = true);

  try {
    final intentRes = await ApiService().createStripePaymentIntent(
      paymentId: widget.paymentId,
    );

    final clientSecret = intentRes['client_secret']?.toString();
    if (clientSecret == null || clientSecret.trim().isEmpty) {
      throw Exception('Missing Stripe client secret from backend');
    }

    await Stripe.instance.initPaymentSheet(
      paymentSheetParameters: SetupPaymentSheetParameters(
        paymentIntentClientSecret: clientSecret,
        merchantDisplayName: 'Medical App',
        style: themeMode,
      ),
    );

    await Stripe.instance.presentPaymentSheet();

    if (!mounted) return;
    setState(() => _paymentSheetSucceeded = true);

    final outcome = await _finalizePaymentOnBackend(); // confirmStripePayment with retries
    if (!mounted) return;

    if (outcome == _FinalizeOutcome.confirmed) {
      Navigator.pop(context, 'succeeded');
      return;
    }
    if (outcome == _FinalizeOutcome.expired) {
      Navigator.pop(context, 'expired');
      return;
    }
    // ... handle failure / retry ...
  } finally {
    if (mounted) setState(() => _submitting = false);
  }
}
```

**Explanation:** Flow is: get `client_secret` → init Payment Sheet → present → then call backend confirm (with backoff retries) so the appointment status is updated even if the webhook is not used.

---

## Summary for Report

| Feature | Backend | Frontend |
|--------|--------|----------|
| **Booking (in_clinic / online)** | `reserve()` with `consultation_type`; `doctorSlots()` for availability | `DoctorsPage(consultationType)`, `reserveAppointmentForPayment()` |
| **Consultation (video)** | `meetingToken()` for online appointments; Jitsi config | `VideoCallPage`, `getMeetingToken()`, “Join” only for online upcoming |
| **Payment** | `createStripeIntent()`, `confirmStripePayment()`, Stripe webhook | `PaymentGatewayPage`, Stripe SDK init + present sheet + confirm API |

You can copy the snippets and explanations you need into your report and shorten or merge sections as required.
