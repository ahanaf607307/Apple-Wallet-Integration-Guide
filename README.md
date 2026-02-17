# 🍎 The Ultimate Master Guide to Apple Wallet Integration
> **Objective**: Building a production-grade Loyalty Pass system with automatic background updates.

---

## 🎨 1. Phase One: The "Gifts" From Apple (Certification)
Before you write a single line of code, you need to collect these files from your [Apple Developer Account](https://developer.apple.com/account/).

### A. The Identifiers
1.  Go to **Certificates, Identifiers & Profiles** -> **Identifiers**.
2.  Click **(+)** and select **Pass Type IDs**.
3.  **Description**: "Belbeda Loyalty Card".
4.  **Identifier**: `pass.com.belbeda.loyalty` (This is your **Pass Type ID**).

### B. The Signing Certificates (To prove you are you)
1.  Click the **Pass Type ID** you just created -> Click **Create Certificate**.
2.  Follow instructions to upload a CSR (Certificate Signing Request) from your Mac/PC.
3.  Download the resulting `.cer` file.
4.  **Important**: Open this file on a Mac (Keychain Access) and export it as a **.p12** file. 
    *   *If you don't have a Mac*: Use OpenSSL to combine the `.crt` (from `.cer`) and your private key into a `.p12`.
5.  **Conversion**: Convert the `.p12` to `.pem` for Node.js using OpenSSL:
    ```bash
    # 1. Extract Certificate
    openssl pkcs12 -in your_file.p12 -clcerts -nokeys -out signerCert.pem -passin pass:YOUR_P12_PASSWORD
    
    # 2. Extract Private Key (Encrypted or Decrypted)
    openssl pkcs12 -in your_file.p12 -nocerts -out signerKey.pem -passin pass:YOUR_P12_PASSWORD -passout pass:YOUR_P12_PASSWORD
    ```

### C. The Push Notification Key (APNs)
1.  Go to **Keys** sidebar -> Click **(+)**.
2.  **Name**: "Apple Wallet Push".
3.  **Enable**: **Apple Push Notifications service (APNs)**.
4.  **Download**: This will give you a `.p8` file (e.g., `AuthKey_ABC123.p8`). 
5.  **Copy the Key ID**: (e.g., `ABC123DEFG`). This is your **`APPLE_APNS_KEY_ID`**.

---

## 🏗️ 2. Phase Two: Backend Setup (The Engine)

### The .env Configuration
Place these in your server environment:
```bash
SERVER_URL=https://api.yourdomain.com
APPLE_TEAM_ID=10_CHAR_ID
APPLE_PASS_TYPE_ID=pass.com.belbeda.loyalty
APPLE_P12_PASSWORD=your_secure_password
APPLE_APNS_KEY_ID=ABC123DEFG
APPLE_APNS_KEY_PATH=certs/AuthKey_ABC123.p8
```

### The Database (Prisma)
You need to store **who** has **what** device so you can send them updates.
```prisma
model ApplePass {
  serialNumber        String    @unique // Unique per customer
  authenticationToken String              // Secret shared with iPhone
  lastUpdated         DateTime  @default(now())
}

model AppleDevice {
  deviceLibraryIdentifier String @unique
  pushToken               String // Where we send the push!
}
```

---

## 🔄 3. Phase Three: The Update Lifecycle (How it works)

Apple Wallet updates follow a strict **6-step sequence**:

1.  **Download**: Frontend calls `GET /api/customer/wallet/apple-wallet-pass/:customerId/:cardId`.
    - Backend generates a `.pkpass` file.
    - Inside the file, we hide the `webServiceURL` and `authenticationToken`.
2.  **Registration**: When the user clicks "Add to Wallet", the **iPhone** (not your app) calls:
    - `POST /api/customer/apple-wallet-wws/v1/devices/:id/registrations/...`
    - Backend saves the `pushToken`.
3.  **Event**: A customer earns points.
4.  **Wake Up**: Backend sends an **EMPTY PUSH** to the `pushToken` via APNs.
5.  **Query**: The iPhone wakes up and asks: *"Which passes updated?"*
    - `GET /v1/devices/:id/registrations/:type?updatedSince=...`
6.  **Refresh**: The iPhone downloads the latest version:
    - `GET /v1/passes/:type/:serial`

---

## 📱 4. Phase Four: The Frontend (Flutter) Journey

Your Flutter developer has the easiest job. They don't need to know anything about APNs or Web Services.

### A. Data Needed from Backend
- The absolute URL to the binary file: `https://.../api/customer/wallet/apple-wallet-pass/123/456`

### B. Flutter Implementation Details

1.  **Add Plugin**: Add `apple_passkit` to `pubspec.yaml`.
2.  **Download Logic**:
    ```dart
    final response = await http.get(Uri.parse('https://api.yourdomain.com/v1/passes/...'));
    if (response.statusCode == 200) {
        final directory = await getApplicationDocumentsDirectory();
        final file = File('${directory.path}/loyalty.pkpass');
        await file.writeAsBytes(response.bodyBytes);
        
        // Use Apple Passkit to open
        await ApplePasskit().addPassFromFile(file.path);
    }
    ```
3.  **Automatic Updates**: Your Flutter developer does **NOT** need to write code for updates. iOS manages this via the `webServiceURL` provided in the `.pkpass` file.

---

## 🛠️ 5. Phase Five: Testing & Troubleshooting

### How to test with iOS Simulator:
1.  Open Xcode.
2.  Run an iOS Simulator (iPhone 14 or higher).
3.  Drag and drop the generated `.pkpass` file onto the Simulator screen.
4.  It will open the "Wallet" preview. Click **Add**.

### Common Production Errors:
| Error | Probable Cause | Fix |
| :--- | :--- | :--- |
| **Pass Invalid** | Missing `icon.png` or `pass.json` syntax error. | Validate `pass.json` structure. |
| **No Updates** | `webServiceURL` is not HTTPS or wrong path. | Must be `https://` and match mounting path. |
| **Registration 404** | Missing `/v1` in the registration url. | Fix router paths in `index.js`. |
| **Push Failed** | Invalid `.p8` key or wrong `Bundle ID/Topic`. | Topic must be the **Pass Type ID**. |

---

## 🎨 6. Phase Six: Visual Identity (Images)

Every `.pkpass` file is actually a ZIP folder. To make it look beautiful, you must provide these images:

| Image Name | Size (approx) | Purpose |
| :--- | :--- | :--- |
| `icon.png` | 29x29 | Shown on the lock screen and in notifications. **Mandatory**. |
| `logo.png` | 160x50 | Shown in the top-left corner of the card. |
| `strip.png` | 375x123 | Background image for the card (StoreCard style). |
| `background.png`| 375x400 | Full background (Optional). |

### Pro-Tip: Retina Images
To make your pass look sharp on high-end iPhones, always include `@2x` and `@3x` versions:
- `icon.png` (29x29)
- `icon@2x.png` (58x58)
- `icon@3x.png` (87x87)

The backend handles this automatically using the `pass.addBuffer('icon.png', buffer)` method in the `appleWallet.service.js`.

---

## 🛑 Final Warning for Production
Always use a **Test Device** first. If you send too many push notifications to a device that has already unregistered, Apple may temporarily **blacklist** your IP address. Always clean up your `AppleRegistration` table when a device unregisters.

---

## � 7. Official References & Documentation

To build with confidence, always refer to the source. Here are the official Apple documents we used to design this system:

1.  **[Introduction to Apple Wallet](https://developer.apple.com/wallet/)**: High-level overview of what's possible.
2.  **[PassKit Package Structure](https://developer.apple.com/documentation/passkit/wallet/creating_a_wallet_pass/creating_the_package_for_a_wallet_pass)**: Details on the `.pkpass` ZIP structure (images, `pass.json`, manifest).
3.  **[PassKit Web Service Reference](https://developer.apple.com/documentation/passkit/wallet/creating_a_wallet_pass/signing_and_distributing_a_wallet_pass/adding_a_web_service_to_update_passes)**: This is the **Holy Grail** for backend developers. It defines the `/v1/devices/...` API endpoints.
4.  **[APNs Overview](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/sending_notification_requests_to_apns)**: How to communicate with the Push Notification servers.
5.  **[Wallet Design Guidelines](https://developer.apple.com/design/human-interface-guidelines/technologies/wallet/)**: How to design your card to get approved by Apple.

---

## 🔬 8. Technical Deep-Dive (Under the Hood)

### The `pass.json` Secret Sauce
When the backend generates the pass, it constructs a JSON file. Here are the most critical fields:

```json
{
  "passTypeIdentifier": "pass.com.belbeda.loyalty",
  "serialNumber": "cust123_card456",
  "teamIdentifier": "ABC123DEFG",
  "webServiceURL": "https://api.yourdomain.com/customer/apple-wallet-wws/v1",
  "authenticationToken": "a_long_random_secret_token",
  "sharingProhibited": true,
  "barcodes": [{
    "message": "CUST_ID_123",
    "format": "PKBarcodeFormatQR",
    "messageEncoding": "iso-8859-1"
  }],
  "storeCard": {
    "primaryFields": [
      {
        "key": "points",
        "label": "REWARD POINTS",
        "value": 150
      }
    ]
  }
}
```

### WWS Protocol Headers
Every time the iPhone calls your backend, it sends specific headers. You **must** verify them:

- **Header**: `Authorization: ApplePass <authenticationToken>`
- **Logic**: Find the `ApplePass` in your DB by `serialNumber`. If the token doesn't match, return **401 Unauthorized**.

### The "Tag" Logic (State Management)
When iOS asks for updates (`GET /v1/devices/...`), it sends a query parameter called `passesUpdatedSince`.
- **Backend Role**: Compare this "tag" with your `ApplePass.lastUpdated`. 
- **If newer**: Return the serial number.
- **If older**: Return **204 No Content**. This saves server bandwidth!

---

## 🛠️ 9. Developer Tools
- **[Passkit-Generator (npm)](https://www.npmjs.com/package/passkit-generator)**: The library we used in this project to sign passes in Node.js.
- **[PKPass Validator](https://pkpassvalidator.com/)**: Use this to upload your `.pkpass` file and check for errors if it won't open on your phone.
