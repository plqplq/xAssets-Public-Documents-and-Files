# xAssets REST API - Postman Guide

This guide explains how to authenticate and call the xAssets REST API using Postman.

The xAssets API uses a two-step process: first you **log on** to obtain a session token, then you include that token in **subsequent requests**. A Postman post-response script automates passing the session values between requests.

---

## Prerequisites

- Postman installed (desktop or web)
- Your xAssets instance URL (e.g. `https://mycompany.mydomain.xassets.net`)
- An API key and secret, provided by your xAssets administrator

> **Port numbers:** Most hosted instances use the default HTTPS port (443) and no port is needed in the URL. If your administrator has provided a specific port number (common for dev/test environments), include it in the URL, e.g. `https://mycompany.mydomain.xassets.net:61210`.

---

## Step 1 - Create a Postman Environment

Environments store variables that are shared across requests.

1. In the left sidebar, click **Environments**.
2. Click **+** to create a new environment. Name it something meaningful (e.g. `xAssets Production`).
3. Add the following variable:

| Variable | Initial Value |
| -------- | ------------- |
| `xa_db`  | mycompany     |

The value should be your **database name** - typically the first part of your xAssets URL (e.g. if your URL is `https://mycompany.mydomain.xassets.net`, the database name is `mycompany`).

4. Click **Save**.
5. In the top-right environment dropdown, select the environment you just created.

---

## Step 2 - Create a Collection

Collections group related requests and allow scripts and variables to be shared between them.

1. In the left sidebar, click **Collections**.
2. Click **+** to create a new collection. Name it `xAssets API`.
3. Inside the collection, create two requests:
   - `Logon`
   - `Get Assets`

---

## Step 3 - Configure the Logon Request

Select the **Logon** request and configure it as follows.

### Method and URL

| Field  | Value |
| ------ | ----- |
| Method | GET   |
| URL    | `https://mycompany.mydomain.xassets.net/api/api.ashx` |

Replace the URL with your actual xAssets instance URL (including port if applicable).

### Params Tab

| Key             | Value      |
| --------------- | ---------- |
| `command`       | `apilogon` |
| `logonidentity` | `test`     |

The `logonidentity` value identifies the calling application in audit logs. Use any short descriptive name.

### Headers Tab

| Key           | Value                           |
| ------------- | ------------------------------- |
| `xa-apikey`   | `yourapikey^yoursecret`         |
| `xa-database` | `{{xa_db}}`                     |

Replace `yourapikey^yoursecret` with your actual API key and secret, joined with a `^` character.

### Authorization Tab

Set to **No Auth**. Authentication is handled entirely through the custom headers above.

---

## Step 4 - Add the Post-Response Script

This is the key step that makes everything work. The logon response contains session tokens that must be extracted and reformatted for use in subsequent requests. A Postman script does this automatically.

1. With the **Logon** request selected, click the **Scripts** tab.
2. Select the **Post-response** section.
3. Paste the following script:

```javascript
let json = pm.response.json();

let hash = json.data.find(x => x.type === "hash")?.value;
let nonce = json.data.find(x => x.type === "nonce")?.value;
let noncetime = json.data.find(x => x.type === "noncetime")?.value;

// The noncetime arrives as "dd-MMM-yyyy HH:mm:ss.fff" in UTC.
// Reformat directly to avoid timezone issues with JavaScript Date parsing.
let months = {
    Jan: "01", Feb: "02", Mar: "03", Apr: "04", May: "05", Jun: "06",
    Jul: "07", Aug: "08", Sep: "09", Oct: "10", Nov: "11", Dec: "12"
};

let m = noncetime.match(/(\d+)-(\w+)-(\d+)\s+(\d+):(\d+):(\d+)\.(\d+)/);

// Format as yyyy-MM-ddTHH:mm:ssZ
let isoNoMs = m[3] + "-" + months[m[2]] + "-" + m[1].padStart(2, '0') + "T" +
              m[4].padStart(2, '0') + ":" + m[5].padStart(2, '0') + ":" +
              m[6].padStart(2, '0') + "Z";

// Milliseconds as a separate 3-digit value
let ms = m[7].padStart(3, '0');

pm.environment.set("xa_hash", hash);
pm.environment.set("xa_nonce", nonce);
pm.environment.set("xa_isodate", isoNoMs);
pm.environment.set("xa_ms", ms);
```

4. Click **Save**.

---

## Step 5 - Send the Logon Request

Click **Send**. You should receive a `200` response with JSON containing `hash`, `nonce`, and `noncetime` values in the `data` array.

To verify the script worked, open **Environments** in the left sidebar and click your environment. You should see these variables now have values:

| Variable       | Example Value                                                      |
| -------------- | ------------------------------------------------------------------ |
| `xa_hash`      | `User_918bee8e-6018-4ea0-9236-f0a22730c5ec`                       |
| `xa_nonce`     | `198039F154E522E8B312F77039CEEAAA99AE5B7430E8BF8618ADBBFB4C1159B3`|
| `xa_isodate`   | `2026-02-11T18:14:05Z`                                             |
| `xa_ms`        | `193`                                                              |

If these variables are empty, open the Postman Console (bottom-left) and check for script errors.

---

## Step 6 - Configure a Data Request

Select the **Get Assets** request and configure it as follows.

### Method and URL

| Field  | Value |
| ------ | ----- |
| Method | GET   |
| URL    | `https://mycompany.mydomain.xassets.net/api/api.ashx` |

Use the same base URL as the Logon request.

### Params Tab

| Key       | Value       |
| --------- | ----------- |
| `command` | `AssetXML`  |
| `format`  | `XML`       |
| `arg0`    | `LIST_ALL`  |
| `arg1`    | `FIXEDLIST` |
| `arg2`    | `1,2,3,4,5` |

This example retrieves assets with IDs 1 through 5. Adjust `arg2` to match asset IDs that exist in your database.

### Headers Tab

| Key                | Value                                                  |
| ------------------ | ------------------------------------------------------ |
| `xa-Authorization` | `Bearer {{xa_hash}}`                                   |
| `Sec`              | `{{xa_db}}\|{{xa_nonce}}\|{{xa_isodate}}\|{{xa_ms}}`  |

The `Sec` header is a single value with four parts separated by pipe (`|`) characters.

### Authorization Tab

Set to **No Auth**. Do not use Postman's built-in Authorization tab - the xAssets API uses custom headers instead.

---

## Step 7 - Send the Data Request

Click **Send**. You should receive a `200` response containing XML asset data.

---

## Running Both Requests in Sequence

Rather than sending each request manually, you can run them as a sequence:

1. Click your **xAssets API** collection in the left sidebar.
2. Click **Run collection**.
3. Ensure the order is:
   1. Logon
   2. Get Assets
4. Click **Run xAssets API**.

Postman will execute the logon, store the session variables via the script, then execute the data request using those variables.

---

## Session Lifetime

The logon response includes a timeout setting (typically 1200 seconds / 20 minutes). If your data requests start returning `401` or `500` errors after working previously, your session has expired. Simply re-send the Logon request to obtain a fresh session.

---

## SSL Certificate Verification

If you are connecting to a development or test environment that uses a self-signed SSL certificate, you may need to disable certificate verification:

1. Click the **Settings** icon (gear) in Postman.
2. Turn off **SSL certificate verification**.

This should not be necessary for production hosted instances.

---

## Troubleshooting

| Symptom | Likely Cause |
| ------- | ------------ |
| Logon returns `401` | API key or secret is incorrect. Check the `xa-apikey` header value. |
| Logon returns `500` | Database name is wrong. Check the `xa-database` header and `xa_db` environment variable. |
| Data request returns `401` or "token expired" | Session has expired. Re-send the Logon request. |
| Data request returns `500` | Check that all four environment variables (`xa_hash`, `xa_nonce`, `xa_isodate`, `xa_ms`) have values. Check the `Sec` header uses pipe separators. |
| Variables are empty after logon | Check the post-response script is in the correct section (Scripts > Post-response). Open the Postman Console to check for script errors. |
| Connection refused or timeout | Check the URL and port number. Dev environments may use a non-standard port. |

For further debugging, open the **Postman Console** (View > Show Postman Console, or the console icon at the bottom-left). This shows the raw requests and responses including all headers sent.

---

## API Reference Summary

**Authentication flow:**

1. Send a logon request with `xa-apikey` and `xa-database` headers
2. Extract `hash`, `nonce`, and `noncetime` from the JSON response
3. On all subsequent requests, send two headers:

```
xa-Authorization: Bearer <hash>
Sec: <database>|<nonce>|<date in yyyy-MM-ddTHH:mm:ssZ format>|<milliseconds>
```

**Common commands (passed as the `command` parameter):**

| Command      | Description                     | Key Arguments |
| ------------ | ------------------------------- | ------------- |
| `apilogon`   | Authenticate and get a session  | `logonidentity` |
| `AssetXML`   | Retrieve asset records          | `arg0`=mode, `arg1`=filter type, `arg2`=values |
| `FieldTag`   | Get the XML tag name for a field| `arg0`=field display name |
