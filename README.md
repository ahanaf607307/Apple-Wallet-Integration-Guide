# 🍎 Apple Wallet Pass Integration Guide (Production Grade)

This guide provides a step-by-step technical blueprint for a backend engineer to implement Apple Wallet passes with **Automatic Background Updates**.

---

## 🏗️ 1. Architecture Overview
Apple Wallet updates work via a "Pull-to-Refresh" model triggered by a "Push" notification.

1.  **Backend** sends an empty push to Apple (APNs).
2.  **iOS device** receives push and asks backend: *"What changed for this user?"*
3.  **Backend** returns the serial number of the updated pass.
4.  **iOS device** downloads the newest `.pkpass` file automatically.

---

## 🔑 2. Prerequisites (Apple Developer Account)
You need these 4 things from [developer.apple.com](https://developer.apple.com/):

1.  **Pass Type ID**: An identifier for your card (e.g., `pass.com.belbeda.loyalty`).
2.  **Team ID**: Found in your "Membership" tab.
3.  **Signer Certificates**: 
    -   Generate a `Pass Type ID Certificate`.
    -   Convert it to `.pem` format (`signerCert.pem` and `signerKey.pem`).
    -   Download `WWDR.pem` (Apple World Wide Developer Relations).
4.  **APNs Auth Key (.p8)**:
    -   Go to **Keys** -> Create new key -> Enable **APNs**.
    -   Download the `.p8` file. This is for **Push Notifications**.

---

## 💾 3. Database Schema (Prisma)
You must track which devices have installed which passes to know where to send push notifications.

```prisma
model ApplePass {
  serialNumber        String              @unique // format: customerId_cardId
  authenticationToken String                      // Unique secret per pass
  lastUpdated         DateTime            @default(now())
  registrations       AppleRegistration[]
}

model AppleDevice {
  deviceLibraryIdentifier String              @unique
  pushToken               String              // Needed to send notifications
  registrations           AppleRegistration[]
}

model AppleRegistration {
  deviceLibraryIdentifier String
  serialNumber            String
  device                  AppleDevice @relation(fields: [deviceLibraryIdentifier], references: [deviceLibraryIdentifier])
  pass                    ApplePass   @relation(fields: [serialNumber], references: [serialNumber])
  @@unique([deviceLibraryIdentifier, serialNumber])
}
```

---

## 🚀 4. Backend Implementation (Express.js)

### A. Environment Variables (.env)
```bash
APPLE_TEAM_ID=ABC123DEFG
APPLE_PASS_TYPE_ID=pass.com.yourcompany.loyalty
APPLE_SIGNER_CERT_PATH=signerCert.pem
APPLE_SIGNER_KEY_PATH=signerKey.pem
APPLE_APNS_KEY_ID=XYZ9876543
APPLE_APNS_KEY_PATH=certs/AuthKey_XYZ.p8
```

### B. Standard WWS Endpoints (The Protocol)
Apple's iOS Wallet app will call these **automatically**.

| Endpoint Path | Description |
| :--- | :--- |
| `POST /v1/devices/:id/registrations/:type/:serial` | **Register**: iPhone tells backend its `pushToken`. |
| `DELETE /v1/devices/:id/registrations/:type/:serial` | **Unregister**: iPhone tells backend it deleted the pass. |
| `GET /v1/devices/:id/registrations/:type` | **Poll**: iPhone asks for updated serial numbers. |
| `GET /v1/passes/:type/:serial` | **Download**: iPhone downloads the latest binary file. |

---

## 📱 5. Frontend (Flutter) Responsibility
The Flutter developer has very little work to do. They do **not** handle the updates.

1.  **Step 1**: Call your backend API to get the `.pkpass` download URL.
2.  **Step 2**: Download the file.
3.  **Step 3**: Open the file using a plugin like `apple_passkit`.
    ```dart
    // Example Flutter Logic
    await ApplePasskit().addPassFromFile(passFilePath);
    ```

---

## 🔔 6. The Update Flow (Prodcution Level)

When a user's points change (e.g., they buy a coffee):

1.  **Update DB**: `rewardPoints` increments.
2.  **Mark Pass**: Update `ApplePass.lastUpdated = now()`.
3.  **Find Devices**: Query `AppleRegistration` for all `pushTokens` associated with that user's `serialNumber`.
4.  **Send Push**:
    ```javascript
    import apn from "@parse/node-apn";
    
    const notification = new apn.Notification();
    notification.payload = {}; // MUST BE EMPTY
    notification.topic = "pass.com.belbeda.loyalty";
    
    provider.send(notification, pushToken);
    ```

---

## ⚠️ Critical Troubleshooting (The "Gotchas")

1.  **HTTPS ONLY**: Apple Wallet will refuse to connect to `http`. You **must** use `https`.
2.  **v1 Path**: Even if you mount your route at `/api`, Apple expects `.../v1/devices`. **Always** ensure your `webServiceURL` in the pass matches your router setup.
3.  **Authentication**: Every request from Apple includes an `Authorization: ApplePass <Token>` header. You **must** verify this token against your database for security.
4.  **APNs Topic**: For Wallet passes, the `topic` in your push notification must be the **Pass Type ID**, not the App Bundle ID.

---

## 🔄 7. The Integration Process (Step-by-Step)

This is the exact order of execution between the Frontend, Backend, and Apple's servers.

### Step 1: User Downloads the Pass
*   **Action**: User/Frontend hits your download endpoint.
*   **Method**: `GET`
*   **URL**: `/api/customer/wallet/apple-wallet-pass/:customerId/:cardId`
*   **Data Provided**: `customerId`, `cardId` (in URL params).
*   **Process**: 
    1. Backend generates a `serialNumber` (e.g., `user123_card456`).
    2. Backend creates a unique `authenticationToken`.
    3. Backend saves these in the `ApplePass` table.
    4. Backend streams the binary `.pkpass` file back to the browser/app.
*   **Frontend**: Simply opens the downloaded file. iOS will show the "Add to Wallet" view.

### Step 2: Automatic Device Registration
*   **Action**: **iOS System (iPhone)** hits your registration endpoint.
*   **Method**: `POST`
*   **URL**: `/api/customer/apple-wallet-wws/v1/devices/:deviceId/registrations/:passType/:serial`
*   **Auth**: Includes `Authorization: ApplePass <authenticationToken>`.
*   **Payload**: Contains a `pushToken`.
*   **Backend Logic**: Verifies the token and saves the `pushToken` to the `AppleDevice` and `AppleRegistration` tables.

### Step 3: Points Change Event (The Trigger)
*   **Action**: A staff member updates points or a user redeems a reward.
*   **URL**: Your existing `/api/staff/redeem/add-points` or `/api/customer/claim-reward/:rewardId`.
*   **Backend Logic**:
    1. Update points in Database.
    2. Check if the user has an `ApplePass` serial number.
    3. Trigger a push: Send an **EMPTY** notification to Apple (APNs) using the saved `pushToken`.

### Step 4: iOS Background Refresh
*   **Action**: iOS receives the push and calls your "Check for Updates" route.
*   **Method**: `GET`
*   **URL**: `/api/customer/apple-wallet-wws/v1/devices/:deviceId/registrations/:passType?passesUpdatedSince=...`
*   **Backend Logic**: Returns the `serialNumber` because the `ApplePass.lastUpdated` time is newer than the request timestamp.

### Step 5: Final Auto-Update
*   **Action**: iOS hits your "Download Pass" route to get the new version.
*   **Method**: `GET`
*   **URL**: `/api/customer/apple-wallet-wws/v1/passes/:passType/:serial`
*   **Backend Logic**: Generates a **NEW** `.pkpass` file with the updated points and streams it back.
*   **Result**: The user's phone updates the card UI instantly in the background. No user action or Flutter app code needed!
