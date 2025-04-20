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

Each object contains the registration and database id of the aircraft in question as show in the example below:

```json
[{ "registration_no": "N123", "id": 15678 }]
```

## File Upload API

SavvyAviation provides two mechanisms for uploading files:

1. **V1 Upload API (Legacy)** - The original direct file upload method
2. **V2 Upload API (BETA - Recommended)** - The new S3 presigned URL upload method with improved status tracking

### V1 Upload API (Legacy)

The V1 "Upload Files" endpoint supports a POST operation with a URLEncoded parameter containing the token and another containing the file. The aircraft ID is included in the URL, as shown in the example below:

```bash
curl -X POST https://apps.savvyaviation.com/upload_files_api/15678/ --form "token=YOUR_API_TOKEN" --form "file=@/Users/flyer/development/engine_data_samples/JPI/U130214.JPI"
```

The endpoint returns an "application/json" response with a single object as shown in the example below:

```json
{ "status": "OK", "id": 550, "logs": "/file_parse_log/550" }
```

The "status" field will be "OK" if the upload was a success. The ID field is the database identifier of the uploaded file. The "logs" field is a relative URL (same base URL as the endpoint) that provides additional information about the endpoint.

### V2 Upload API (BETA - Recommended)

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
  -F "file=@/Users/flyer/development/engine_data_samples/JPI/U130214.JPI"
```

A successful upload will return HTTP 204.

#### Step 3: Check Upload Status

After uploading the file, you can check the status of the processing using the file ID returned in step 1:

```bash
curl -X GET "https://apps.savvyaviation.com/upload_status/[FILE_ID]/?token=YOUR_API_TOKEN"
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
- "failed" (final state) - File processing failed. See logs for details using status call
- "timeout" (final state) - Processing timed out. See logs for details using status call
- "done" (final state) - File processed successfully
- "flight duplication" (final state) - Done, but some flights in the file were duplicates

## Bruno API Examples

Bruno is an API client that allows you to save and share API requests. The following Bruno files are available for testing the SavvyAviation API:

### Get Aircraft

- [Get Aircraft.bru](bruno/Get%20Aircraft.bru) - Get a list of aircraft for the authenticated user

### V2 Upload API

- [Request Upload URL.bru](bruno/Request%20Upload%20URL.bru) - Request an S3 presigned URL for uploading a file
- [Upload File Direct to S3.bru](bruno/Upload%20File%20Direct%20to%20S3.bru) - Upload a file directly to S3 using the presigned URL
- [Check Upload Status.bru](bruno/Check%20Upload%20Status.bru) - Check the status of a file upload

## Support

Report errors with the API, documentation or request help by opening a technical support ticket on SavvyAviation.com. You'll need an account to do this.
