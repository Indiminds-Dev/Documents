# Azure Key Vault Implementation Guide

**Goal:** Securely store the Firebase Service Account JSON in Azure Key Vault and access it in your Node.js App Service.

---

## ⚠️ Important Prerequisite

To create Key Vaults and secrets, the **"Contributor"** role is not enough. You must have the **"Key Vault Secrets Officer"** (or Administrator) role.

---

## Step 1: Create the Azure Key Vault

1. Go to the **Azure Portal**, search for **Key Vaults**, and click **Create**.

2. **Basics Tab:**
   - **Resource Group:** Select the group where your App Service is (e.g., `GenericAPP`).
   - **Key Vault Name:** `firebase-keyvault` (or your unique name).
   - **Region:** Use the same region as your App Service.
   - **Pricing Tier:** Standard.

3. **Access Configuration:**
   - Select **Azure role-based access control (RBAC)**.

4. Click **Review + Create** → **Create**.

---

## Step 2: Add Secret and Copy the Identifier

1. Open your new **Key Vault** and go to the **Secrets** tab.

2. Click **Generate/Import**.
   - **Name:** `firebase-service-account`
   - **Secret Value:** Paste the **entire raw JSON content with no white spaces in one single line** from your `firebase-service-account.json` file.
   - Click **Create**.

3. **Get the Secret Identifier (Critical Step):**
   - Click on the secret you just created (`firebase-service-account`).
   - Click on the **Current Version** (the long alphanumeric code, e.g., `ae2c2...`).
   - Locate the **Secret Identifier** field.
   - **Copy the full URL** using the copy icon.
   - It should look like this:
     ```
     https://firebase-keyvault.vault.azure.net/secrets/firebase-service-account/ae2c2e3fb649427f94a5363c947e4f4c
     ```

---

## Step 3: Turn On App Service Identity

1. Go to your **App Service** (`friskit-logger-api`).

2. In the left menu, click **Identity**.

3. Under the **System assigned** tab, switch **Status** to **On**.

4. Click **Save**.

---

## Step 4: Grant the App Service Access to the Key Vault

1. Go back to your **Key Vault**.

2. Click **Access control (IAM)** → **Add** → **Add role assignment**.

3. **Role:** Search for and select **Key Vault Secrets User**.

4. **Members Tab:**
   - Choose **Managed Identity**.
   - Click **Select members** → Find and select your App Service (`friskit-logger-api`).

5. Click **Review + assign**.

---

## Step 5: Add the App Setting (Environment Variable)

1. Go to **App Service** → **Configuration** (or Environment Variables).

2. Click **New application setting**.

3. Fill in the details:
   - **Name:** `FIREBASE_SERVICE_ACCOUNT`
   - **Value:** Paste the code below, replacing the URL with your **exact Secret Identifier**:
     ```
     @Microsoft.KeyVault(SecretUri=https://firebase-keyvault.vault.azure.net/secrets/firebase-service-account/ae2c2e3fb649427f94a5363c947e4f4c)
     ```

4. Click **OK**, then click **Save** at the top. The app will restart.

---

## Step 6: Update Your Node.js Code

Your Firebase service should handle **both Azure (production)** and **local development** automatically:

```javascript
import admin from "firebase-admin";
import path from "path";
import fs from "fs-extra";

async function initializeFirebase() {
  let serviceAccount = null;

  // 1️⃣ FIRST: Check Azure Key Vault ENV (Production)
  const firebaseJson = process.env.FIREBASE_SERVICE_ACCOUNT;

  if (firebaseJson && firebaseJson.trim() !== "") {
    try {
      const trimmedJson = firebaseJson.trim();
      
      // Validate JSON format
      if (!trimmedJson.startsWith('{') || !trimmedJson.endsWith('}')) {
        throw new Error("Invalid JSON format");
      }

      serviceAccount = JSON.parse(trimmedJson);
      
      // Validate required fields
      const requiredFields = ['project_id', 'client_email', 'private_key'];
      const missingFields = requiredFields.filter(field => !serviceAccount[field]);
      
      if (missingFields.length > 0) {
        throw new Error(`Missing fields: ${missingFields.join(', ')}`);
      }

      console.log("✅ Firebase loaded from Azure Key Vault");
      
    } catch (err) {
      console.error("❌ Key Vault JSON parsing error:", err.message);
      throw err;
    }
  } 
  // 2️⃣ FALLBACK: Local development file
  else {
    const filePath = path.join(process.cwd(), "firebase-service-account.json");

    if (await fs.pathExists(filePath)) {
      console.log("🟡 Firebase loaded from LOCAL file (development mode)");
      serviceAccount = await fs.readJson(filePath);
    } else {
      throw new Error("❌ No Firebase credentials found (ENV or local file)");
    }
  }

  // 3️⃣ Initialize Firebase Admin SDK
  admin.initializeApp({
    credential: admin.credential.cert({
      projectId: serviceAccount.project_id,
      clientEmail: serviceAccount.client_email,
      privateKey: serviceAccount.private_key.replace(/\\n/g, "\n"),
    }),
  });

  console.log("✅ Firebase Admin SDK initialized successfully!");
}
```

---

## How It Works

| Environment | Credential Source | Console Output |
|-------------|-------------------|----------------|
| **Azure (Production)** | `FIREBASE_SERVICE_ACCOUNT` ENV from Key Vault | `✅ Firebase loaded from Azure Key Vault` |
| **Local (Development)** | `firebase-service-account.json` file in project root | `🟡 Firebase loaded from LOCAL file` |

---

## ⚠️ Important Notes

1. **Local Development:** Place `firebase-service-account.json` in your project root

2. **Git Security:** Add this to your `.gitignore`:
   ```
   firebase-service-account.json
   ```

3. **Azure Secret Format:** When pasting JSON in Key Vault, ensure it's in **one single line with no extra whitespaces**

---

## Final Check

After the app restarts, check your **Log Stream** in Azure Portal:

| Status | Message | Action |
|--------|---------|--------|
| ✅ Success | `Firebase loaded from Azure Key Vault` | All good! |
| ❌ Error | Parsing error or missing fields | Check if Secret Identifier URL is correct |
| ❌ Error | Access denied | Verify App Service has `Key Vault Secrets User` role |

---

## Quick Troubleshooting

| Problem | Solution |
|---------|----------|
| Secret not loading | Check if Secret Identifier URL includes the version ID |
| Access denied | Re-add `Key Vault Secrets User` role to App Service |
| JSON parse error | Ensure JSON is in single line with no extra spaces |
| Local file not found | Place `firebase-service-account.json` in project root |
