# Apple Wallet Integration Guide 🍎🎟️

This guide explains how to implement Apple Wallet pass generation (`.pkpass`) in a Node.js backend, including certificate setup and secure distribution.

## 1. Apple Developer Account Setup

To generate passes, you need an active **Apple Developer Program** membership.

### A. Create a Pass Type ID
1. Log in to [Apple Developer Certificates, Identifiers & Profiles](https://developer.apple.com/account/resources/identifiers/list/passTypeId).
2. Click **Identifiers (+)** and select **Pass Type IDs**.
3. Provide a description and a unique identifier (e.g., `pass.com.yourcompany.loyalty`).
4. Save the **Pass Type ID** and your **Team ID**.

### B. Generate the Pass Certificate
1. Go to **Certificates (+)**.
2. Select **Pass Type ID Certificate** under the "Services" section.
3. Select the **Pass Type ID** you just created.
4. Upload a **Certificate Signing Request (CSR)**:
   - On Mac: Use **Keychain Access** > **Certificate Assistant** > **Request a Certificate from a Certificate Authority**.
   - On Windows: Use `openssl genrsa -out request.key 2048` and `openssl req -new -key request.key -out request.csr`.
5. Download the resulting `.cer` file from Apple.

### C. Download the WWDR Certificate
Apple Wallet passes must be signed with the **Apple Worldwide Developer Relations Certification Authority** certificate.
- Download the **G4 (PEM)** version from [Apple PKI](https://www.apple.com/certificateauthority/).
- Save it as `WWDR.pem`.

---

## 2. Certificate Transformation (PEM Requirement)

Modern libraries like `passkit-generator` (v3+) require certificates in **PEM format**.

### Step 1: Export to .p12
On a Mac, double-click the `.cer` file from Apple to add it to Keychain. Right-click the certificate and export it as `apple_wallet.p12`. Set a password.

### Step 2: Convert .p12 to .pem (Signer Certificate & Key)
Run these commands using OpenSSL (on Mac or Linux):

```bash
# Extract the Certificate
openssl pkcs12 -in apple_wallet.p12 -clcerts -nokeys -out signerCert.pem

# Extract the Private Key
openssl pkcs12 -in apple_wallet.p12 -nocerts -nodes -out signerKey.pem
```

> [!IMPORTANT]
> If the `.pem` files contain "Bag Attributes" or extra text before `-----BEGIN CERTIFICATE-----`, delete that extra text. The file must start exactly with the BEGIN line.

---

## 3. Project Configuration

### Dependencies
```bash
npm install passkit-generator
```

### Environment Variables (.env)
```env
APPLE_PASS_TYPE_ID=pass.com.yourcompany.loyalty
APPLE_TEAM_ID=ABCDE12345
APPLE_P12_PASSWORD=your_password
SERVER_URL=https://your-api.com
```

---

## 4. Code Implementation

### A. The Service Layer (`appleWallet.service.js`)
Use the `PKPass` class from `passkit-generator` to define your pass structure.

```javascript
import { PKPass } from "passkit-generator";

// Load certificates
const wwdr = fs.readFileSync("./WWDR.pem");
const signerCert = fs.readFileSync("./signerCert.pem");
const signerKey = fs.readFileSync("./signerKey.pem");

export const generatePass = async (data) => {
  const pass = await PKPass.from({
    model: "./models/loyalty.pass", // Folder containing icon.png, logo.png, etc.
    certificates: { wwdr, signerCert, signerKey, signerKeyPassword: "" }
  });

  pass.setPrimaryFields([{ key: "balance", label: "POINTS", value: data.points }]);
  return pass.asBuffer();
};
```

### B. Secure Distribution Strategy (UUID Flow)
To avoid "Unauthorized" errors or exposing login tokens in URLs, use a **Public UUID** strategy:

1. **Request Link (Private)**: User calls `GET /api/wallet-link` (Authenticated).
   - Server generates a unique UUID for this specific enrollment.
   - Server returns: `https://api.com/apple-wallet-pass/UNIQUE_UUID`.
2. **Download Pass (Public)**: User/iPhone clicks the link.
   - Server looks up the UUID and returns the `.pkpass` file.
   - **No token needed** because the UUID is unguessable and private.

---

## 5. File Checklist
Ensure these files are in your project root:
- [ ] `WWDR.pem` (Apple Authority)
- [ ] `signerCert.pem` (Your Public Cert)
- [ ] `signerKey.pem` (Your Private Key)
- [ ] `apple_wallet.p12` (Backup origin)
- [ ] `models/loyalty.pass/` (Icons and JSON template)

---

## 6. Testing on Windows
Since Windows cannot open `.pkpass` files:
1. Download the file.
2. Upload it to [**pkpass.io**](https://pkpass.io/) to see a visual preview and verify validity.
3. Ultimately, test on a physical **iPhone** using Safari.
