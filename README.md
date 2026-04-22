# SavvyAviation File Upload API

SavvyAviation provides an API for 3rd party developers to build applications that can upload files directly into a user's account.

## Authentication

It is not desirable for a user to provide their SavvyAviation credentials (i.e. email and password) to a third party service because that would:

- Give the application full access to their account
- Make it impossible to easily revoke access from that application

To get around this, Savvy provides a mechanism for users to create authentication tokens. Applications in possession of these tokens can perform the following actions:

1. Retrieve the IDs and registrations of that user's aircraft
2. Upload files to a given aircraft ID

To obtain a token, a user needs to perform the following steps:

1. Log into [SavvyAviation](https://apps.savvyaviation.com/)
2. On the main menu select "Account", then "3rd Party Apps"
3. At the bottom of the page, type in a name for the token. For example, the name could be the name of the app they will use this token with
4. Click on "Save"
5. Copy-paste the token and enter it to the appropriate application

## Mobile Integration

SavvyAviation provides a mechanism to allow for the automatic generation of these tokens. An app would issue an HTTP GET with two parameters:

- One indicating the URL schema of the mobile application
- Another one indicating the name of the app

This would redirect them to the SavvyAviation web-site where (after logging in, if necessary) they would just confirm the creating of the token with a single tap and then the web-site would redirect them back to a URL (based on the provided schema) that would re-open the app and provide the token, thus eliminating most steps.

`https://apps.savvyaviation.com/account/request-api-token/?app_name=MyAwesomeMobileApp&callback_url=myawesomeapp://`

When the user clicks on the button on the resulting screen, the token will be created and they will be redirected to:

`myawesomeapp://?token=[YOUR_TOKEN]`

The app_name is used to name the token. The tokens can be seen on this page under the Third Party Apps tab:

`https://apps.savvyaviation.com/account/`

## Redirection Support

Applications implementing the APIs in this document MUST support 301 and 302 HTTP redirection, in order to support any future URL changes.

## Get Aircraft Endpoint

The "Get Aircraft" end-point supports a POST operation with a single URLEncoded parameter containing the token:

```bash
curl -X POST https://apps.savvyaviation.com/get-aircraft/ --form "token=[YOUR_TOKEN]"
```

It returns a "application/json" response, as an array of objects. The array may be empty.

Each object contains the following fields:

| Field | Type | Description |
|---|---|---|
| `id` | integer | Database id of the aircraft. |
| `registration_no` | string | Aircraft registration (tail number). |
| `eligibleForBorescopeAnalysis` | boolean | Whether this aircraft is currently covered by a subscription plan that includes borescope image analysis (Analysis, QA, MX, or Prebuy). Clients should use this flag to decide whether to surface the "Request Analysis" action and to prompt the user to upgrade when it is `false`. Recomputed on every call, so the value reflects the aircraft's eligibility at request time. |
| `serviceUpgradeUrl` | string (URI) | Deep link to the SavvyAviation web app plan page for this specific aircraft, where the owner can upgrade to a subscription that includes borescope analysis. Present **if and only if** `eligibleForBorescopeAnalysis` is `false`; when eligibility is `true`, the field is omitted entirely (not returned as `null`). |

Example response when the aircraft is eligible for borescope analysis:

```json
[{ "registration_no": "N123", "id": 15678, "eligibleForBorescopeAnalysis": true }]
```

Example response when the aircraft is not eligible:

```json
[{
  "registration_no": "N123",
  "id": 15678,
  "eligibleForBorescopeAnalysis": false,
  "serviceUpgradeUrl": "https://apps.savvyaviation.com/account/plan/15678"
}]
```

## File Upload API

SavvyAviation provides two mechanisms for uploading files:

1. **V1 Upload API (Legacy)** - The original direct file upload method. No longer supported.
2. **V2 Upload API (Recommended)** - The new S3 presigned URL upload method with improved status tracking

### V2 Upload API (Recommended)

The V2 Upload API provides a more robust upload mechanism using S3 presigned URLs and improved status tracking. The process involves three steps:

1. Request an upload URL
2. Upload the file directly to S3
3. Check the upload status

#### Step 1: Request an Upload URL

To request an upload URL, make a POST request to the following endpoint:

```bash
curl -X POST https://apps.savvyaviation.com/request_upload_url/AIRCRAFT_ID/ --form "token=YOUR_API_TOKEN" --form "filename=U130214.JPI"
```

The response will include a presigned S3 URL and the necessary fields to complete the upload:

```json
{
  "status": "OK",
  "id": "[FILE_ID]",
  "upload_url": "[UPLOAD_URL]",
  "fields": {
    "key": "[KEY]",
    "AWSAccessKeyId": "[AWS_ACCESS_KEY_ID]",
    "x-amz-security-token": "[SECURITY TOKEN]",
    "policy": "[POLICY]",
    "signature": "[SIGNATURE]"
  }
}
```

#### Step 2: Upload the File to S3

Use the presigned URL and fields from the previous step to upload the file directly to S3:

```bash
curl -X POST "[URL]" \
  -F "key=[KEY]" \
  -F "AWSAccessKeyId=[AWS_ACCESS_KEY_ID]" \
  -F "policy=[POLICY]" \
  -F "signature=[SIGNATURE]" \
  _F "x-amz-security-token=[SECURITY TOKEN]" \
  -F "file=@/Users/flyer/development/engine_data_samples/JPI/U130214.JPI"
```

A successful upload will return HTTP 204.

#### Step 3: Check Upload Status

After uploading the file, you can check the status of the processing using the file ID returned in step 1:

```bash
curl -X GET "https://apps.savvyaviation.com/upload_status/[FILE_ID]/" \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

The 200 response will include the current status of the file processing:

```json
{
  "id": "[FILE_ID]",
  "status": "parsing",
}
```

Possible upload_status values include:

- "init" - Initial state, workflow has not started yet
- "waiting_for_file_upload" - Waiting for the file to be uploaded to S3
- "file_not_found" (final state) - File not found in S3. This happens if 10 minutes pass and the file is not uploaded.
- "compute_hash_digest" - Computing file hash digest
- "unable_to_compute_hash" (final state) - Unable to compute file hash digest
- "duplication" (final state) - File is an exact duplicate of another file for this aircraft. See logs for details using status call.
- "parsing" - Parsing the file
- "failure" (final state) - File processing failed. See logs for details using status call
- "timeout" (final state) - Processing timed out. See logs for details using status call
- "success" (final state) - File processed successfully
- "flight_duplication" (final state) - Done, but some flights in the file were duplicates
- "partial (final state)"  - File processed. Some flights succeeded, some failed. See logs for details.

## Borescope Image Management API

This API supports mobile applications that guide aircraft mechanics and owners through capturing standardized borescope inspection images. It handles offline image capture in hangars and syncs data when WiFi becomes available.

### Overview

The borescope inspection workflow follows a 12-view per cylinder protocol:

1. **Offline capture**: App captures images following the 12-view protocol per cylinder
2. **File management**: Auto-names files according to Savvy's naming spec
3. **Backend sync**: Upload to Savvy backend when WiFi is available

### 1. Get User's Aircraft

Use the existing [Get Aircraft](#get-aircraft-endpoint) endpoint to retrieve the list of aircraft the authenticated user can access.

---

### 2. Get Borescope Image Sets for Aircraft

Retrieve all borescope image sets associated with a specific aircraft.

```bash
curl -X POST https://apps.savvyaviation.com/api/borescope/image-sets/ \
  --form "token=YOUR_API_TOKEN" \
  --form "aircraft_id=AIRCRAFT_ID"
```

**Response:**

```json
{
  "image_sets": [
    {
      "bis_id": 42,
      "date": "2025-06-15T10:30:00Z",
      "title": "Annual Inspection 2025",
      "status": "in_progress",
      "images": [
        {
          "image_id": 187,
          "cylinder": 1,
          "view": "piston-crown"
        }
      ]
    }
  ]
}
```

**Image Set Statuses:**

| Status | Description |
|---|---|
| `in_progress` | The image set is being built. Images can only be uploaded while in this status. |
| `analysis_requested` | All images have been uploaded and the owner has requested analysis. A support ticket has been created. No further uploads are allowed. |
| `report_sent` | Analysis is complete and the report has been delivered to the owner. |

---

### 3. Create Borescope Image Set

Initialize a new borescope image set for an aircraft.

**Required fields:**

| Field | Description |
|---|---|
| `token` | API authentication token |
| `aircraft_id` | The aircraft to create the image set for |

**Optional fields:**

| Field | Description |
|---|---|
| `inspection_date` | Date of the inspection (ISO 8601 format) |
| `hobbs_hours` | Hobbs meter reading at time of inspection |

```bash
curl -X POST https://apps.savvyaviation.com/api/borescope/create-image-set/ \
  --form "token=YOUR_API_TOKEN" \
  --form "aircraft_id=AIRCRAFT_ID" \
  --form "inspection_date=2025-06-15" \
  --form "hobbs_hours=1523.4"
```

**Response:**

```json
{
  "bis_id": 42
}
```

---

### 4. Get Signed S3 URL for Image Upload

Request a pre-signed S3 URL for uploading a specific borescope image. The image set must be in `in_progress` status.

If an image already exists for the given `cylinder` and `view` combination, the endpoint returns an error. To overwrite an existing image, set `force` to `true`. The `other` view is exempt from this check — multiple "other" images can be uploaded per cylinder without using `force`.

See [Borescope Views](#borescope-views) for the list of valid `view` values.

```bash
curl -X POST https://apps.savvyaviation.com/api/borescope/request-upload-url/ \
  --form "token=YOUR_API_TOKEN" \
  --form "bis_id=BIS_ID" \
  --form "cylinder=1" \
  --form "view=piston-crown" \
  --form "filename=N123_CYL1_piston-crown.jpg" \
  --form "content_type=image/jpeg" \
  --form "force=false"
```

**Response (success):**

```json
{
  "upload_url": "https://s3.amazonaws.com/bucket/...",
  "image_id": 187,
  "expires_at": "2025-06-15T11:00:00Z"
}
```

**Response (duplicate, when `force` is not `true`):**

```json
{
  "error": "duplicate_view",
  "message": "An image already exists for cylinder 1, view piston-crown. Set force=true to overwrite."
}
```

**Response (wrong status):**

```json
{
  "error": "invalid_status",
  "message": "Image uploads are only allowed when the image set is in_progress."
}
```

The pre-signed URL expires after 15 minutes.

---

### 5. Upload Borescope Image to S3

Upload the actual image file using the pre-signed URL from step 4.

```bash
curl -X PUT "UPLOAD_URL_FROM_STEP_4" \
  -H "Content-Type: image/jpeg" \
  --data-binary @/path/to/image.jpg
```

A successful upload returns HTTP 200.

---

### 6. Request Analysis

Submit a completed borescope image set for analysis. This transitions the image set status from `in_progress` to `analysis_requested` and creates a support ticket.

**Required fields:**

| Field | Description |
|---|---|
| `token` | API authentication token |
| `subject` | Ticket subject line |
| `body` | Ticket body / client comments |
| `aircraft_id` | The aircraft the images belong to |
| `borescope_image_set_id` | The specific image set to analyze |

**Optional fields:**

| Field | Description |
|---|---|
| `priority` | Ticket priority |
| `priority_explanation` | Why this priority was chosen |

```bash
curl -X POST https://apps.savvyaviation.com/api/borescope/request-analysis/ \
  --form "token=YOUR_API_TOKEN" \
  --form "subject=Annual borescope inspection N123" \
  --form "body=Please review cylinder 3 closely, rough running noted." \
  --form "aircraft_id=AIRCRAFT_ID" \
  --form "borescope_image_set_id=BIS_ID" \
  --form "priority=high" \
  --form "priority_explanation=Rough running on cylinder 3"
```

**Response:**

```json
{
  "status": "analysis_requested",
  "ticket_id": 5012,
  "ticket_url": "https://apps.savvyaviation.com/tickets/5012/"
}
```

---

### Data Models

#### Borescope Image Set (BIS)

| Field | Type | Description |
|---|---|---|
| `bis_id` | integer | Unique identifier |
| `aircraft_id` | integer | Associated aircraft |
| `tail_number` | string | Aircraft tail number |
| `inspection_date` | date | Date of inspection (ISO 8601) |
| `hobbs_hours` | number | Hobbs meter reading at time of inspection (optional) |
| `borescope_model` | string | Borescope hardware model |
| `status` | enum | `in_progress`, `analysis_requested`, `report_sent` |
| `created_at` | timestamp | Creation time |
| `submitted_by_type` | enum | `owner`, `mechanic` |
| `submitter_info` | object | Submitter details |

#### Image

| Field | Type | Description |
|---|---|---|
| `image_id` | integer | Unique identifier |
| `bis_id` | integer | Parent image set (foreign key) |
| `cylinder` | integer | Cylinder number (1-n based on engine) |
| `view` | string | One of the 12 [borescope views](#borescope-views) |
| `filename` | string | Original filename |
| `s3_key` | string | S3 storage key |
| `upload_timestamp` | timestamp | Upload time |
| `metadata` | object | Image metadata |

### Borescope Views

Each cylinder has 12 standardized views. The `view` parameter in API requests must use the **slug** value.

| View Name | Slug |
|---|---|
| Piston Crown | `piston-crown` |
| Exhaust Valve Head | `exhaust-valve-head` |
| Exhaust Valve Seat | `exhaust-valve-seat` |
| Exhaust Valve Stem & Guide | `exhaust-valve-stem` |
| Intake Valve Head | `intake-valve-head` |
| Intake Valve Seat | `intake-valve-seat` |
| Intake Valve Stem & Guide | `intake-valve-stem` |
| Cylinder Wall 12 | `cylinder-wall-12` |
| Cylinder Wall 3 | `cylinder-wall-3` |
| Cylinder Wall 6 | `cylinder-wall-6` |
| Cylinder Wall 9 | `cylinder-wall-9` |
| Other | `other` |

### Error Responses

All borescope API endpoints return errors in a consistent format:

```json
{
  "error": "error_code",
  "message": "Human-readable description of what went wrong."
}
```

| Error Code | HTTP Status | Description |
|---|---|---|
| `invalid_token` | 401 | The provided token is invalid or expired |
| `not_found` | 404 | The requested resource (aircraft, image set, etc.) was not found |
| `invalid_status` | 409 | The image set is not in the required status for this operation |
| `duplicate_view` | 409 | An image already exists for this cylinder/view combination (use `force=true` to overwrite) |
| `validation_error` | 400 | A required field is missing or has an invalid value |
| `forbidden` | 403 | The user does not have permission to access this resource |

### OpenAPI Specification

The full machine-readable API specification is available at [openapi.yaml](openapi.yaml).

## Bruno API Examples

Bruno is an API client that allows you to save and share API requests. The following Bruno files are available for testing the SavvyAviation API:

### Get Aircraft

- [Get Aircraft.bru](bruno/Get%20Aircraft.bru) - Get a list of aircraft for the authenticated user

### V2 Upload API

- [Request Upload URL.bru](bruno/Request%20Upload%20URL.bru) - Request an S3 presigned URL for uploading a file
- [Upload File Direct to S3.bru](bruno/Upload%20File%20Direct%20to%20S3.bru) - Upload a file directly to S3 using the presigned URL
- [Check Upload Status.bru](bruno/Check%20Upload%20Status.bru) - Check the status of a file upload

### Borescope Image Management

- [Get Borescope Image Sets.bru](bruno/Get%20Borescope%20Image%20Sets.bru) - Get all borescope image sets for an aircraft
- [Create Borescope Image Set.bru](bruno/Create%20Borescope%20Image%20Set.bru) - Create a new borescope image set
- [Get Signed Upload URL (Borescope).bru](bruno/Get%20Signed%20Upload%20URL%20(Borescope).bru) - Get a pre-signed S3 URL for uploading a borescope image
- [Upload Borescope Image to S3.bru](bruno/Upload%20Borescope%20Image%20to%20S3.bru) - Upload a borescope image directly to S3
- [Request Borescope Analysis.bru](bruno/Request%20Borescope%20Analysis.bru) - Submit a borescope image set for analysis

## Support

Report errors with the API, documentation or request help by opening a technical support ticket on SavvyAviation.com. You'll need an account to do this.
