# ICS Transfer API - Detailed Usage Guide

> **Base URL:** `https://tourchain.icstravelgroup.com/tourchain/api/IcsTransfer`  
> **Content-Type:** `application/json`  
> **Authentication:** Bearer JWT Token (required for all endpoints)

---

## 1. Authentication

All API endpoints require a **Bearer JWT Token** in the request header.

### Required Headers

| Header          | Value                            | Description                  |
|-----------------|----------------------------------|------------------------------|
| `Authorization` | `Bearer <jwt_token>`             | A valid JWT token            |
| `Content-Type`  | `application/json`               | Request content type         |

### JWT Token Structure

The JWT token is signed with `HS256` using the `JwtSecret` configured on the server. The token contains the following claims:

| Claim        | Type     | Description                                      |
|--------------|----------|--------------------------------------------------|
| `Id`         | `string` | User ID                                          |
| `Client`     | `string` | `"true"` if the user is a client login           |
| `IsGuest`    | `string` | `"True"` if the user is a guest manager          |
| `Username`   | `string` | Login username                                   |
| `Password`   | `string` | Authentication code (used for guest logins only) |

### Example Header

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
Content-Type: application/json
```

### Authentication Errors

| HTTP Status | Description                                      |
|-------------|--------------------------------------------------|
| `401`       | Token is invalid, expired, or missing            |

---

## 2. API: Create or Update Transfer Booking

### `POST /api/IcsTransfer/webhook/booking-complete`

Creates a new or updates an existing ICS transfer booking (airport transfer: Bali, Thailand, Vietnam, etc.).

---

### 2.1 Request Body

```json
{
  "number": "B2024-001234",
  "stopFollowUpButton": true,
  "customer_email": "john.doe@example.com",
  "customer_given_name": "John",
  "customer_surname": "Doe",
  "customer_phone": "+84901234567",
  "packageName": "Bali Airport Transfer Package",
  "packageCode": "PKG-ICS-001",
  "transfer_information": {
    "type": "arrival-and-departure",
    "checkInDate": "2024-12-15",
    "checkOutDate": "2024-12-20",
    "fullname": "John Doe",
    "countryCodePhone": "+84",
    "phone": "0901234567",
    "luggage": 2,
    "oversizeLuggage": 0,
    "babyCarSeat": 0,
    "notes": "Additional notes if any",
    "pickUpDescription": "Hotel lobby, ground floor",
    "dropOffDescription": "Terminal 2 - International Airport",
    "arrival": {
      "date": "2024-12-15",
      "time": "14:30",
      "flightNumber": "VN123",
      "adults": 2,
      "children": 1
    },
    "departure": {
      "date": "2024-12-20",
      "time": "09:00",
      "flightNumber": "VN456",
      "adults": 2,
      "children": 1
    }
  },
  "accommodation_items": [
    {
      "id": "hotel-item-id-001",
      "reservation": {
        "check_in": "2024-12-15",
        "check_out": "2024-12-20",
        "hotel_info": {
          "id": "hotel-bali-abc123",
          "name": "The Ritz Carlton Bali",
          "geo_data": {
            "country": "Indonesia",
            "administrative_area_level_1": "Bali",
            "place_id": "ChIJN1t_tDeuEmsRUsoyG83frY4"
          }
        }
      }
    }
  ]
}
```

---

### 2.2 Request Parameters

#### Root fields

| Field                | Type       | Required | Description                                                  |
|----------------------|------------|----------|--------------------------------------------------------------|
| `number`             | `string`   | ✅ Yes   | Booking number. e.g. `"B2024-001234"`                       |
| `stopFollowUpButton` | `boolean`  | ✅ Yes   | **Must be `true`**. Confirms the booking is complete         |
| `customer_email`     | `string`   | ✅ Yes   | Valid customer email address                                 |
| `customer_given_name`| `string`   | ✅ Yes   | Customer's first name                                        |
| `customer_surname`   | `string`   | ✅ Yes   | Customer's last name                                         |
| `customer_phone`     | `string`   | ✅ Yes   | Customer's phone number                                      |
| `packageName`        | `string`   | ❌ No    | Package Name                                                 |
| `packageCode`        | `string`   | ❌ No    | Package Code                                                 |
| `transfer_information`| `object`  | ✅ Yes   | Detailed transfer information. See table below               |
| `accommodation_items`| `array`    | ✅ Yes   | List of accommodation items. At least 1 item required        |

---

#### `transfer_information` object

| Field                | Type      | Required                               | Description                                                                      |
|----------------------|-----------|----------------------------------------|----------------------------------------------------------------------------------|
| `type`               | `string`  | ✅ Yes                                 | Transfer direction: `"arrival"`, `"departure"`, or `"arrival-and-departure"`    |
| `checkInDate`        | `string`  | ❌ No                                  | Check-in date. Format: `"YYYY-MM-DD"`                                           |
| `checkOutDate`       | `string`  | ❌ No                                  | Check-out date. Format: `"YYYY-MM-DD"`                                          |
| `fullname`           | `string`  | ✅ Yes                                 | Full name of the lead passenger                                                  |
| `countryCodePhone`   | `string`  | ❌ No                                  | Country phone code. e.g. `"+84"`                                                |
| `phone`              | `string`  | ❌ No                                  | Phone number (without country code)                                              |
| `luggage`            | `integer` | ❌ No (default: `0`)                   | Number of luggage pieces. Must be >= 0                                           |
| `oversizeLuggage`    | `integer` | ❌ No (default: `0`)                   | Oversize luggage. Accepts only `0` or `1`                                        |
| `babyCarSeat`        | `integer` | ❌ No (default: `0`)                   | Baby car seat required. Accepts only `0` or `1`                                  |
| `notes`              | `string`  | ❌ No                                  | Additional notes                                                                 |
| `pickUpDescription`  | `string`  | ❌ No                                  | Pick-up point description                                                        |
| `dropOffDescription` | `string`  | ❌ No                                  | Drop-off point description                                                       |
| `arrival`            | `object`  | ✅ When `type` includes `"arrival"`   | Arrival transfer details. See `transfer leg` table below                         |
| `departure`          | `object`  | ✅ When `type` includes `"departure"` | Departure transfer details. See `transfer leg` table below                       |

---

#### `arrival` / `departure` (Transfer Leg) object

| Field          | Type      | Required               | Description                                               |
|----------------|-----------|------------------------|-----------------------------------------------------------|
| `date`         | `string`  | ✅ Yes                 | Flight date. Format: `"YYYY-MM-DD"`                       |
| `time`         | `string`  | ❌ No (can be empty)   | Flight time. Format: `"HH:mm"` (24h). e.g. `"14:30"`     |
| `flightNumber` | `string`  | ❌ No (can be empty)   | IATA flight number. e.g. `"VN123"`                        |
| `adults`       | `integer` | ✅ Yes                 | Number of adults. Must be >= 1                            |
| `children`     | `integer` | ❌ No (default: `0`)   | Number of children. Must be >= 0                          |

---

#### `accommodation_items[]` (array)

| Field         | Type     | Required | Description                                        |
|---------------|----------|----------|----------------------------------------------------|
| `id`          | `string` | ✅ Yes   | Accommodation item ID                              |
| `reservation` | `object` | ✅ Yes   | Reservation details. See `reservation` table below |

---

#### `reservation` object

| Field        | Type     | Required | Description                                        |
|--------------|----------|----------|-------------------------------------------------|
| `check_in`   | `string` | ✅ Yes   | Check-in date. Format: `"YYYY-MM-DD"`             |
| `check_out`  | `string` | ✅ Yes   | Check-out date. Format: `"YYYY-MM-DD"`            |
| `hotel_info` | `object` | ✅ Yes   | Hotel information. See `hotel_info` table below   |

---

#### `hotel_info` object

| Field      | Type     | Required | Description                                                  |
|------------|----------|----------|--------------------------------------------------------------|
| `id`       | `string` | ✅ Yes   | Unique hotel ID in the system                                |
| `name`     | `string` | ✅ Yes   | Hotel name                                                   |
| `geo_data` | `object` | ❌ No    | Hotel geographic data                                        |

---

#### `geo_data` object

| Field                          | Type     | Required | Description                                   |
|--------------------------------|----------|----------|-----------------------------------------------|
| `country`                      | `string` | ❌ No    | Country name. e.g. `"Indonesia"`              |
| `administrative_area_level_1`  | `string` | ❌ No    | Province/Region. e.g. `"Bali"`                |
| `place_id`                     | `string` | ❌ No    | Google Place ID                               |

---

### 2.3 Validation Rules

| Field                          | Rule                                                                       |
|--------------------------------|----------------------------------------------------------------------------|
| `number`                       | Must not be empty                                                          |
| `stopFollowUpButton`           | Must be `true`                                                             |
| `customer_email`               | Must not be empty and must be a valid email address                        |
| `customer_given_name`          | Must not be empty                                                          |
| `customer_surname`             | Must not be empty                                                          |
| `customer_phone`               | Must not be empty                                                          |
| `transfer_information`         | Must not be null                                                           |
| `transfer_information.type`    | Must be `"arrival"`, `"departure"`, or `"arrival-and-departure"`          |
| `transfer_information.fullname`| Must not be empty                                                          |
| `arrival.date`                 | Required when type is `arrival` or `arrival-and-departure`                 |
| `arrival.adults`               | Must be >= 1                                                               |
| `departure.date`               | Required when type is `departure` or `arrival-and-departure`               |
| `departure.adults`             | Must be >= 1                                                               |
| `luggage`                      | Must be >= 0                                                               |
| `oversizeLuggage`              | Must be `0` or `1` only                                                    |
| `babyCarSeat`                  | Must be `0` or `1` only                                                    |
| `accommodation_items`          | Must not be empty. At least 1 item required                                |
| `accommodation_items[].id`     | Must not be empty                                                          |
| `reservation.check_in`         | Must not be empty                                                          |
| `reservation.check_out`        | Must not be empty                                                          |
| `hotel_info.id`                | Must not be empty                                                          |
| `hotel_info.name`              | Must not be empty                                                          |

---

### 2.4 Success Response

**HTTP Status: `200 OK`**

```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

| Field     | Type      | Description                              |
|-----------|-----------|------------------------------------------|
| `status`  | `integer` | HTTP status code (`200`)                 |
| `message` | `string`  | Success message                          |

---

### 2.5 Error Response (Validation - 422)

**HTTP Status: `422 Unprocessable Entity`**

```json
{
  "status": 422,
  "error": 1,
  "messages": {
    "type": "error",
    "message": {
      "customer_email": "Valid customer email is required",
      "transfer_information.type": "Transfer type must be 'arrival-and-departure', 'arrival', or 'departure'",
      "accommodation_items[0].id": "Accommodation item id is required"
    }
  }
}
```

| Field                | Type             | Description                                                         |
|----------------------|------------------|---------------------------------------------------------------------|
| `status`             | `integer`        | HTTP status code                                                    |
| `error`              | `integer`        | Always `1` when an error occurs                                     |
| `messages.type`      | `string`         | Error type, always `"error"`                                        |
| `messages.message`   | `string\|object` | A plain error string, or an object with field names as keys         |

---

### 2.6 cURL Example

```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "B2024-001234",
    "stopFollowUpButton": true,
    "customer_email": "john.doe@example.com",
    "customer_given_name": "John",
    "customer_surname": "Doe",
    "customer_phone": "+84901234567",
    "packageName": "Bali Airport Transfer Package",
    "packageCode": "PKG-ICS-001",
    "transfer_information": {
      "type": "arrival",
      "fullname": "John Doe",
      "luggage": 2,
      "oversizeLuggage": 0,
      "babyCarSeat": 0,
      "arrival": {
        "date": "2024-12-15",
        "time": "14:30",
        "flightNumber": "VN123",
        "adults": 2,
        "children": 0
      }
    },
    "accommodation_items": [
      {
        "id": "hotel-item-001",
        "reservation": {
          "check_in": "2024-12-15",
          "check_out": "2024-12-20",
          "hotel_info": {
            "id": "hotel-bali-001",
            "name": "The Ritz Carlton Bali"
          }
        }
      }
    ]
  }'
```

---

## 3. API: Cancel Transfer Booking

### `DELETE /api/IcsTransfer/webhook/cancel/{bookingNumber}`

Cancels an ICS transfer booking by booking number. Sets the booking status to `"Cancelled"` in the system.

---

### 3.1 URL Parameter

| Parameter       | Location | Type     | Required | Description                                           |
|-----------------|----------|----------|----------|-------------------------------------------------------|
| `bookingNumber` | Path     | `string` | ✅ Yes   | The booking number to cancel. e.g. `B2024-001234`     |

### Example URL

```
DELETE /api/IcsTransfer/webhook/cancel/B2024-001234
```

---

### 3.2 Request Body

No request body required. Pass `bookingNumber` directly in the URL path.

---

### 3.3 Processing Flow

When the API is called, the system performs the following steps:

1. Find the `LeadModel` (pipedriveTour) containing a booking with `voucherCode = bookingNumber`.
2. Update `status = "Cancelled"` and `updatedDate = DateTime.UtcNow` for that booking.

> **Note:** If no matching booking is found in the system, the API returns `404 Not Found`.

---

### 3.4 Success Response

**HTTP Status: `200 OK`**

```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

| Field     | Type      | Description               |
|-----------|-----------|---------------------------|
| `status`  | `integer` | HTTP status code (`200`)  |
| `message` | `string`  | Success message           |

---

### 3.5 Error Response (Validation - 422)

**HTTP Status: `422 Unprocessable Entity`** — when `bookingNumber` is empty:

```json
{
  "status": 422,
  "error": 1,
  "messages": {
    "type": "error",
    "message": "Booking number is required"
  }
}
```

**HTTP Status: `404 Not Found`** — when `bookingNumber` does not exist:

```json
{
  "status": 404,
  "error": 1,
  "messages": {
    "type": "error",
    "message": "Booking 'B2024-000000' not found"
  }
}
```

---

### 3.6 cURL Example

```bash
curl -X DELETE https://tourchain.icstravelgroup.com/tourchain/api/IcsTransfer/webhook/cancel/B2024-001234 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4"
```

## 4. HTTP Status Codes Summary

| Status Code | Scenario                                             |
|-------------|------------------------------------------------------|
| `200`       | Success                                              |
| `401`       | Token is invalid, expired, or missing                |
| `404`       | Booking number not found (Cancel endpoint)           |
| `422`       | Input validation failed                              |

---

## 5. Full Example: `arrival-and-departure` Booking

```bash
curl -X POST https://tourchain.icstravelgroup.com/tourchain/api/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "B2024-VN-9988",
    "stopFollowUpButton": true,
    "customer_email": "nguyen.van.a@gmail.com",
    "customer_given_name": "Van A",
    "customer_surname": "Nguyen",
    "customer_phone": "+84912345678",
    "packageName": "Da Nang Airport Transfer Package",
    "packageCode": "PKG-ICS-002",
    "transfer_information": {
      "type": "arrival-and-departure",
      "checkInDate": "2024-12-20",
      "checkOutDate": "2024-12-27",
      "fullname": "Nguyen Van A",
      "countryCodePhone": "+84",
      "phone": "0912345678",
      "luggage": 3,
      "oversizeLuggage": 0,
      "babyCarSeat": 1,
      "notes": "Guest has an 18-month-old baby",
      "pickUpDescription": "Hotel lobby, ground floor",
      "dropOffDescription": "International Terminal T1",
      "arrival": {
        "date": "2024-12-20",
        "time": "10:00",
        "flightNumber": "VN7201",
        "adults": 2,
        "children": 1
      },
      "departure": {
        "date": "2024-12-27",
        "time": "08:30",
        "flightNumber": "VN7202",
        "adults": 2,
        "children": 1
      }
    },
    "accommodation_items": [
      {
        "id": "acc-danang-2024",
        "reservation": {
          "check_in": "2024-12-20",
          "check_out": "2024-12-27",
          "hotel_info": {
            "id": "hotel-intercontinental-dn",
            "name": "InterContinental Danang Sun Peninsula",
            "geo_data": {
              "country": "Vietnam",
              "administrative_area_level_1": "Da Nang",
              "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
            }
          }
        }
      }
    ]
  }'
```

**Response:**

```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

---

## 6. Cancel Booking Example

```bash
curl -X DELETE https://tourchain.icstravelgroup.com/tourchain/api/IcsTransfer/webhook/cancel/B2024-VN-9988 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4"
```

**Response:**

```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

---

## 7. Important Notes

- **`stopFollowUpButton` must always be `true`** — the server will reject the request if the value is `false`.
- **`transfer_information.type`** determines which leg objects are required:
  - `"arrival"` → `arrival` object is required
  - `"departure"` → `departure` object is required
  - `"arrival-and-departure"` → both `arrival` and `departure` objects are required
- **`oversizeLuggage` and `babyCarSeat`** only accept integer values `0` or `1` (not boolean `true`/`false`).
- **JWT Token** must be sent in the correct format: `Bearer <token>` (with a space between `Bearer` and the token).
- The Cancel API returns **`404 Not Found`** when the booking number does not exist in the system.

> **LE requirement:** LE must include the two additional keys, `packageName` and `packageCode`, in the payload sent to this API.