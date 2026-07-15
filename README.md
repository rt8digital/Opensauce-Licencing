# Opensauce-Licencing
An open source, easy and free to use Licencing system using Google sheets + Appscript



___________________

# SheetLicense 🔑

An elegant, open-source, serverless licensing server powered entirely by **Google Sheets** and **Google Apps Script**. 

Perfect for indie hackers, SaaS startups, desktop apps (Electron, Tauri), plugins, and mobile apps (React Native) who want a zero-cost, robust database-backed licensing system without managing servers or paying database fees.

---

## Features

- 💸 **100% Free:** Hosted entirely on Google Cloud infrastructure via Google Apps Script.
- 📊 **Sheets as your Admin Panel:** View, edit, filter, and manage licenses directly within an easy-to-use spreadsheet interface.
- 🧠 **AI-Friendly Header:** Designed with a structural system prompt block at the top of `code.gs`. You can feed this entire script straight into any LLM (Claude, ChatGPT, Gemini) when building client-side integrations, and the AI will immediately know how to interface with your license server.
- 📅 **Auto-Calculating Subscription Dates:** Using native Google Sheets formulas, the server dynamically calculates whether a monthly, yearly, or lifetime (once-off) license is still valid.
- 📡 **Bi-directional Sync:** 
  - **Register/Update** licenses via `POST`
  - **Verify** license validity via `GET`
  - **Retrieve/Fetch** lost keys or full profiles via `GET` (by email)

---

## 📋 Google Sheet Structure

Your Google Sheet needs to have its columns organized in a specific layout so that the automated date calculations and retrieval systems work flawlessly. 

When you run the script for the first time, **it will automatically generate these headers for you**! If you are editing an existing sheet, make sure your columns map to this layout:

| Column | Header Header | Data Type | Notes |
| :--- | :--- | :--- | :--- |
| **A** | `Email` | String (Email) | Core unique identifier |
| **B** | `Name` | String | User's first name |
| **C** | `Surname` | String | User's last name |
| **D** | `Phone` | String | User's contact number |
| **E** | `Organization` | String | Enterprise / Company name |
| **F** | `Licence Key` | String (UUID) | Unique license key (auto-generated if omitted) |
| **G** | `Payment Plan` | Option String | `Monthly`, `Yearly`, or `Once-Off` |
| **H** | `Payment Method` | String | e.g., `Stripe`, `PayPal`, `EFT` |
| **I** | `Last Payment` | Date (`YYYY-MM-DD`) | Date of last payment transaction |
| **J** | `Next Payment Due`| Date Formula | **Auto-Calculated by Sheet** (No manual input required) |
| **K** | `Product Name` | String | App identifier (e.g. `Raydius`, `OpenSauce POS`) |
| **L** | `Price` | String/Number | Transaction value |

---

## ⚙️ Quick Setup Guide

Follow these steps to deploy your licensing server in less than 5 minutes:

### Step 1: Create a Google Sheet
1. Go to [Google Sheets](https://sheets.google.com) and create a brand-new blank spreadsheet.
2. Give your spreadsheet a name (e.g., `My App Licenses`).
3. Note: You don't need to manually type the headers. The script will initialize the sheet on its first run.

### Step 2: Open Google Apps Script
1. Inside your new Google Sheet, click **Extensions** in the top menu bar.
2. Select **Apps Script**. This will open the cloud editor.

### Step 3: Paste the Code
1. Erase any default code in the editor (`function myFunction() {}`).
2. Copy the entire code block from the `code.gs` section below.
3. Paste it directly into the `Code.gs` file inside the editor.
4. Press `Ctrl + S` (or `Cmd + S` on Mac) to save your changes.

### Step 4: Deploy as a Web App (Crucial Step!)
To make this script reachable by your external apps, you must deploy it as a public Web App endpoint:
1. In the top-right corner of the Apps Script editor, click **Deploy** > **New deployment**.
2. Click the **Gear icon (Select type)** and choose **Web app**.
3. Configure the fields exactly as follows:
   - **Description:** `Initial License Server Deployment`
   - **Execute as:** `Me (your-email@gmail.com)` (This gives the script permission to read/write to your Sheet on behalf of your apps)
   - **Who has access:** `Anyone` (This is mandatory so your client apps can connect to the API without needing Google sign-in)
4. Click **Deploy**.
5. Grant permissions: Click **Authorize access**, choose your Google account, click **Advanced**, click **Go to Untitled project (unsafe)**, and hit **Allow**.
6. **Copy the Web App URL** displayed under "Web app". This is your main licensing API endpoint!

---

## 📄 The Code (`code.gs`)

Copy this code and use it inside your Google Apps Script workspace:

```javascript
/**
 * =========================================================================
 * UNIVERSAL LICENSING SERVER - GOOGLE APPS SCRIPT
 * =========================================================================
 * AI CONTEXT & INSTRUCTION BLOCK FOR FUTURE DEVELOPMENT:
 * 
 * When building a new client application that requires licensing synchronization 
 * against this server, follow these API specifications:
 *
 * 1. API ENDPOINT: Use the Web App URL generated after deploying this script.
 * 
 * 2. VERIFY LICENSE (GET):
 *    Fetch: GET {WEB_APP_URL}?action=verify&key=YOUR_LICENSE_KEY
 *    Response (JSON): { "status": "success", "valid": boolean, "message": "...", "data": { ... } }
 *
 * 3. FETCH USER DATA & LICENSE KEY (GET):
 *    Fetch: GET {WEB_APP_URL}?action=fetch&email=user@domain.com
 *    Response (JSON): { "status": "success", "data": { "licence_key": "...", "organization": "...", ... } }
 * 
 * 4. REGISTER/UPDATE LICENSE (POST):
 *    Fetch: POST to {WEB_APP_URL}
 *    Body (JSON):
 *    {
 *      "action": "register",
 *      "email": "user@domain.com",
 *      "name": "First",
 *      "surname": "Last",
 *      "phone": "123456789",
 *      "organization": "Rotate Group",
 *      "licence_key": "XXXX-XXXX-XXXX-XXXX", // Optional: Auto-generates if omitted
 *      "payment_plan": "Monthly", // Options: "Monthly", "Yearly", "Once-Off"
 *      "payment_method": "Stripe", // e.g., "Stripe", "PayPal", "EFT"
 *      "last_payment": "2026-07-15",
 *      "product_name": "OpenSauce POS", 
 *      "price": "500"
 *    }
 * =========================================================================
 */

const SHEET_NAME = "Licenses";

function doGet(e) {
  setupSheet(); 
  
  const action = e.parameter.action;
  
  if (action === "verify") {
    const keyToVerify = e.parameter.key;
    if (!keyToVerify) return jsonResponse({ status: "error", message: "No license key provided." });
    return verifyLicense(keyToVerify);
  }
  
  if (action === "fetch") {
    const emailToFetch = e.parameter.email;
    if (!emailToFetch) return jsonResponse({ status: "error", message: "No email provided." });
    return fetchUserData(emailToFetch);
  }
  
  return jsonResponse({ status: "error", message: "Invalid action. Use ?action=verify or ?action=fetch" });
}

function doPost(e) {
  setupSheet(); 
  
  try {
    const data = JSON.parse(e.postData.contents);
    if (data.action === "register") return registerLicense(data);
    return jsonResponse({ status: "error", message: "Invalid POST action." });
  } catch (error) {
    return jsonResponse({ status: "error", message: "Failed to parse JSON body." });
  }
}

// =========================================================================
// CORE LOGIC FUNCTIONS
// =========================================================================

function registerLicense(data) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
  
  const email = data.email || "";
  const name = data.name || "";
  const surname = data.surname || "";
  const phone = data.phone || "";
  const organization = data.organization || "";
  const licenceKey = data.licence_key || generateUUID();
  const paymentPlan = data.payment_plan || "Once-Off"; 
  const paymentMethod = data.payment_method || "";
  const lastPayment = data.last_payment || new Date().toISOString().split('T')[0];
  const productName = data.product_name || "Unknown Product";
  const price = data.price || "0";

  // Append data in the exact order of the columns
  sheet.appendRow([
    email,          // A (0)
    name,           // B (1)
    surname,        // C (2)
    phone,          // D (3)
    organization,   // E (4)
    licenceKey,     // F (5)
    paymentPlan,    // G (6)
    paymentMethod,  // H (7)
    lastPayment,    // I (8)
    "",             // J (9) - Placeholder for Next Payment Due formula
    productName,    // K (10)
    price           // L (11)
  ]);
  
  const lastRow = sheet.getLastRow();
  const formulaCell = sheet.getRange(lastRow, 10); // Column J is index 10 (1-based for getRange)
  
  // Formula: G is Payment Plan, I is Last Payment
  const formula = `=IF(G${lastRow}="Monthly", I${lastRow}+30, IF(G${lastRow}="Yearly", I${lastRow}+365, "Lifetime"))`;
  formulaCell.setFormula(formula);
  
  return jsonResponse({ 
    status: "success", 
    message: "License registered successfully.",
    licence_key: licenceKey
  });
}

function verifyLicense(key) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
  const data = sheet.getDataRange().getValues();
  
  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const rowKey = row[5]; // Column F (Index 5)
    
    if (rowKey === key) {
      const paymentPlan = row[6]; // Column G
      const nextPaymentDue = row[9]; // Column J
      const today = new Date();
      
      let isValid = true;
      let message = "License is valid.";
      
      if (paymentPlan !== "Once-Off" && nextPaymentDue !== "Lifetime") {
        const dueDate = new Date(nextPaymentDue);
        if (today > dueDate) {
          isValid = false;
          message = "License has expired. Payment required.";
        }
      }
      
      return jsonResponse({
        status: "success",
        valid: isValid,
        message: message,
        data: {
          email: row[0],
          name: row[1],
          organization: row[4],
          product: row[10],
          plan: paymentPlan,
          next_due: nextPaymentDue
        }
      });
    }
  }
  
  return jsonResponse({ status: "error", valid: false, message: "License key not found." });
}

function fetchUserData(email) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
  const data = sheet.getDataRange().getDisplayValues(); 
  
  for (let i = data.length - 1; i > 0; i--) {
    const row = data[i];
    
    if (row[0].toLowerCase() === email.toLowerCase()) {
      return jsonResponse({
        status: "success",
        message: "User record located.",
        data: {
          email: row[0],
          name: row[1],
          surname: row[2],
          phone: row[3],
          organization: row[4],
          licence_key: row[5],
          payment_plan: row[6],
          payment_method: row[7],
          last_payment: row[8],
          next_payment_due: row[9],
          product_name: row[10],
          price: row[11]
        }
      });
    }
  }
  
  return jsonResponse({ status: "error", message: "No record found for the provided email." });
}

// =========================================================================
// UTILITY FUNCTIONS
// =========================================================================

function setupSheet() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(SHEET_NAME);
  
  if (!sheet) sheet = ss.insertSheet(SHEET_NAME);
  
  if (sheet.getLastRow() === 0) {
    const headers = [
      "Email", "Name", "Surname", "Phone", "Organization", 
      "Licence Key", "Payment Plan", "Payment Method", 
      "Last Payment", "Next Payment Due", "Product Name", "Price"
    ];
    
    const headerRange = sheet.getRange(1, 1, 1, headers.length);
    headerRange.setValues([headers]);
    headerRange.setFontWeight("bold");
    headerRange.setBackground("#f3f3f3");
    
    sheet.setFrozenRows(1);
    sheet.autoResizeColumns(1, headers.length);
  }
}

function jsonResponse(object) {
  return ContentService.createTextOutput(JSON.stringify(object))
    .setMimeType(ContentService.MimeType.JSON);
}

function generateUUID() {
  return 'xxxx-xxxx-xxxx-xxxx'.replace(/[x]/g, function(c) {
    var r = Math.random() * 16 | 0;
    return r.toString(16).toUpperCase();
  });
}
```

---

## 🔌 Client-Side Integration Examples (JavaScript / Node.js)

Here is how you can perform standard licensing actions using Javascript `fetch` inside your application or website backend.

### 1. Registering a New License (e.g., in a Stripe checkout Webhook)

Use this script to automatically insert a new customer into your Google Sheet right after they complete a payment.

```javascript
const API_ENDPOINT = "YOUR_WEB_APP_URL"; // Replace with your Google Web App URL

async function registerCustomer() {
  const payload = {
    action: "register",
    email: "customer@company.com",
    name: "John",
    surname: "Doe",
    phone: "+27821234567",
    organization: "Rotate Group",
    payment_plan: "Monthly",
    payment_method: "Stripe",
    product_name: "OpenSauce POS",
    price: "80.80"
  };

  try {
    const response = await fetch(API_ENDPOINT, {
      method: "POST",
      redirect: "follow", // Crucial: Google Apps Script uses redirects!
      headers: {
        "Content-Type": "text/plain" // Prevents CORS preflight pre-checks on simplified clients
      },
      body: JSON.stringify(payload)
    });

    const result = await response.json();
    console.log("Registration Response:", result);
    // Returns: { status: "success", message: "...", licence_key: "XXXX-XXXX-XXXX-XXXX" }
  } catch (error) {
    console.error("Failed to register license:", error);
  }
}
```

### 2. Verifying a License on Application Boot

Paste this function into your main application startup files (e.g., React `useEffect`, Electron Main Process, or Tauri startup) to verify that the user's license key is still valid.

```javascript
const API_ENDPOINT = "YOUR_WEB_APP_URL"; 

async function checkLicenseValidity(licenseKey) {
  try {
    const url = `${API_ENDPOINT}?action=verify&key=${encodeURIComponent(licenseKey)}`;
    
    const response = await fetch(url, {
      method: "GET",
      redirect: "follow" // Crucial for App Script redirect resolution
    });

    const result = await response.json();
    
    if (result.status === "success" && result.valid) {
      console.log("License is active! Welcome, " + result.data.name);
      return true;
    } else {
      console.warn("License check failed:", result.message);
      return false;
    }
  } catch (error) {
    console.error("Could not communicate with licensing server:", error);
    // Best Practice: Handle offline mode gracefully depending on your app's needs
    return false;
  }
}
```

### 3. Fetching User Data & Key by Email (License Recovery)

If a user forgets their license key, you can build a simple "Recover My Key" portal on your website that queries the database using their registration email.

```javascript
const API_ENDPOINT = "YOUR_WEB_APP_URL";

async function fetchLicenseDetails(email) {
  try {
    const url = `${API_ENDPOINT}?action=fetch&email=${encodeURIComponent(email)}`;
    
    const response = await fetch(url, {
      method: "GET",
      redirect: "follow"
    });

    const result = await response.json();
    
    if (result.status === "success") {
      console.log("Record Found! Here is your key:", result.data.licence_key);
      console.log("Next Payment Due on:", result.data.next_payment_due);
      return result.data;
    } else {
      console.warn("User lookup failed:", result.message);
      return null;
    }
  } catch (error) {
    console.error("Lookup error:", error);
    return null;
  }
}
```

---

## 🔒 Security Best Practices

Because this system runs serverlessly and utilizes client-side triggers directly to your Web App URL, keep these best practices in mind:

1. **Protect your Web App URL:** Do not leak your Google Apps Script URL in open Github repositories. Inject it as an environment variable (`.env` or `process.env.LICENSE_SERVER_URL`) during building.
2. **CORS / Post Handling:** Apps Script endpoints use strict redirects. Always include the `redirect: "follow"` parameter in your Javascript `fetch` setups to ensure the API receives the redirection to the final target page gracefully.
3. **Local Fallback Cache:** If the user is offline, avoid blocking them entirely on boot. Cache a local hashed timestamp of their last successful validation inside the local storage (or local SQLite/SQLite encrypted DB), allowing them to use the app offline for up to 3–7 days before requiring an active internet sync.
4. **Obfuscation:** For high-security client applications, obfuscate or pack your production client-side JS build (e.g., using Terser or standard minifiers) to avoid easy script manipulation or verification bypasses.

---

## 📄 License

This project is open-source software licensed under the **MIT License**. Feel free to modify, white-label, distribute, and embed this licensing engine inside commercial applications!
