# Stripe Payment Connection (Mobile App)

## Overview

The mobile app uses **Stripe** for appointment payments.

The Flutter app does not process payments directly.  
Instead, it communicates with the Laravel backend, and the backend connects to Stripe.

---

# Simple Payment Flow

```text
Mobile App  ──►  Laravel Backend  ──►  Stripe
       │                                   │
       └──────── Appointment Confirmed ◄───┘
```

---

# Reserve Appointment

Before payment, the app reserves the appointment slot.

**File:** `frontend/lib/doctors_page.dart`

```dart
final reserveRes = await ApiService().reserveAppointmentForPayment(...);

Navigator.push(context, MaterialPageRoute(
  builder: (_) => PaymentGatewayPage(
    paymentId: reserveRes['payment_id'],
  ),
));
```

The app sends a request to the backend to reserve the slot.

---

# Backend Creates Payment

The Laravel backend creates the payment request using Stripe.

**File:** `backend/app/Http/Controllers/Api/PaymentController.php`

```php
$intent = PaymentIntent::create([
    'amount' => 200,
    'currency' => 'myr',
]);

return ['client_secret' => $intent->client_secret];
```

The backend communicates with Stripe and returns a `client_secret` to the mobile app.

---

# Stripe Payment Screen

The Flutter app opens Stripe’s payment screen.

**File:** `frontend/lib/payment_gateway_page.dart`

```dart
await Stripe.instance.initPaymentSheet(
  paymentIntentClientSecret: clientSecret,
  merchantDisplayName: 'Medical App',
);

await Stripe.instance.presentPaymentSheet();
```

The user enters card details and completes the payment securely through Stripe.

---

# Confirm Payment

After successful payment, the backend updates the appointment status.

**File:** `backend/app/Http/Controllers/Api/PaymentController.php`

```php
$payment->status = 'succeeded';
$appointment->status = 'upcoming';
```

The appointment is now confirmed.

---

# Final Summary

> The Flutter mobile app sends payment requests to the Laravel backend, and the backend securely communicates with Stripe to process payments.