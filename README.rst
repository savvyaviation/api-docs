SavvyAviation File Upload API
-----------------------------

SavvyAviation provides an API for 3rd party developers to build applications that can upload files directy into a user's
account.

Authentication
==============
It is not desirable for a user to provide their SavvyAviation credentials (i.e. emailand password) to a third party
service because that would (a) give the application full access to their account and (b) would make it impossible
to easily revoke access from that application.

To get around this, Savvy provides a mechanisms for users to create authentication tokens. Applications in possession
of these tokens can perform the following actions:

a) Retrieve the IDs and registrations of that user's aircraft; and
b) Upload files to a given aircraft ID.

To obtain a token, a user needs to perform the following steps:

a) Log into https://apps.savvyaviation.com/
b) On the main menu select "Account", then "3rd Party Apps"
c) At the bottom of the page, type in a name for the token. For example, the name could be the name of the app they will use this token with
d) Click on "Save"
e) Copy-paste the token and enter it to the appropriate application

Mobile Integration
========================

SavvyAviation provides a mechanism to allow for the automatic generation of these tokens.
An app would issue an HTTP GET with two parameters, one indicating indicating the URL schema of
the mobile application and another one indicating the name of the app. This would redirect them to the SavvyAviation
web-site where (after logging in, if necessary) they would just confirm the creating of the token with a single tap
and then the web-site would redirect them back to a URL (based on the provided schema) that would re-open the app
and provide the token, thus eliminating most steps.

https://apps.savvyaviation.com/request-api-token/?app_name=MyAwesomeMobileApp&callback_url=myawesomeapp://

When the user clicks on the button on the resulting screen, the token will be created and they will be redirected to:

myawesomeapp://?token=XYZ

The app_name is used to name the token. The tokens cane be seen on this page:

https://apps.savvyaviation.com/third-party-apps/

Redirection Support
===================

Applications implementing the APIs in this document MUST support 301 and 302 HTTP redirection, in order to support
any future URL changes.

Get Aircraft Endpoint
=====================

The "Get Aircraft" end-point supports a POST operation with a single URLEncoded parameter containing the token.::

    curl -X POST https://apps.savvyaviation.com/get-aircraft/ --form "token=XYZ"

It returns a "application/json" response, as an array of objects.  The array may be empty.

Each object contains the registration and database id of the aircraft in question as show in the example below.::

    [{"registration_no": "N123", "id": 15678}]

Upload Files Endpoint
=====================

The "Upload Files" end-point supports a POST operation with a URLEncoded parameter containing the token and another
containing the file.  The aircraft ID is included in the URL, as show in in the example below:::

    curl -X POST https://apps.savvyaviation.com/upload_files_api/15678/ --form "token=4724aa85-a61b-4d4f-a0ed-xxxxyyyyzzzz" --form "file=@/Users/flyer/development/engine_data_samples/JPI/U130214.JPI"

The end-point returns an "application/json" response with a single object as shown in the example below:::

    {"status": "OK", "id": 550, "logs": "/file_parse_log/550”}

The "status" field will be "OK" if the upload was a success. The ID field is the database identifier of the uploaded
file.  The "logs" field is a relative URL (same base URL as the endpoint) that provides additional information about
the endpoint.

TODO: Document failure error codes.

Support
=======

Report errors with the API, documentation or request help by opening a technical support ticket on SavvyAviation.com.  You'll need
an account to do this.