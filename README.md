# OAuth2 for Apps Script

OAuth2 for Apps Script is a library for Google Apps Script that provides the
ability to create and authorize OAuth2 tokens as well as refresh them when they
expire. This library uses Apps Script's new
[StateTokenBuilder](https://developers.google.com/apps-script/reference/script/state-token-builder)
and `/usercallback` endpoint to handle the redirects.


## Setup

This library is already published as an Apps Script, making it easy to include
in your project. To add it to your script, do the following in the Apps Script
code editor:

1. Click on the menu item "Resources > Libraries..."
2. In the "Find a Library" text box, enter the script ID
   `1B7FSrk5Zi6L1rSxxTDgDEUsPzlukDsi4KGuTMorsTQHhGBzBkMun4iDF` and click the
   "Select" button.
3. Choose a version in the dropdown box (usually best to pick the latest
   version).
4. Click the "Save" button.

Alternatively, you can copy and paste the files in the [`/dist`](dist) directory
directly into your script project.

## Redirect URI

Before you can start authenticating against an OAuth2 provider, you usually need
to register your application and retrieve the client ID and secret. Often
these registration screens require you to enter a "Redirect URI", which is the
URL that users will be redirected to after they've authorized the token. For
this library (and the Apps Script functionality in general) the URL will always
be in the following format:

    https://script.google.com/macros/d/{SCRIPT ID}/usercallback

Where `{SCRIPT ID}` is the ID of the script that is using this library. You
can find your script's ID in the Apps Script code editor by clicking on
the menu item "File > Project properties".

Alternatively you can call the service's `getRedirectUri()` method to view the
exact URL that the service will use when performing the OAuth flow:

```js
/**
 * Logs the redict URI to register.
 */
function logRedirectUri() {
  var service = getService();
  Logger.log(service.getRedirectUri());
}
```

## Usage

Using the library to generate an OAuth2 token has the following basic steps.

### 1. Create the OAuth2 service

The OAuth2Service class contains the configuration information for a given
OAuth2 provider, including its endpoints, client IDs and secrets, etc. This
information is not persisted to any data store, so you'll need to create this
object each time you want to use it. The example below shows how to create a
service for the Google Drive API.

```js
function getDriveService() {
  // Create a new service with the given name. The name will be used when
  // persisting the authorized token, so ensure it is unique within the
  // scope of the property store.
  return OAuth2.createService('drive')

      // Set the endpoint URLs, which are the same for all Google services.
      .setAuthorizationBaseUrl('https://accounts.google.com/o/oauth2/auth')
      .setTokenUrl('https://accounts.google.com/o/oauth2/token')

      // Set the client ID and secret, from the Google Developers Console.
      .setClientId('...')
      .setClientSecret('...')

      // Set the name of the callback function in the script referenced
      // above that should be invoked to complete the OAuth flow.
      .setCallbackFunction('authCallback')

      // Set the property store where authorized tokens should be persisted.
      .setPropertyStore(PropertiesService.getUserProperties())

      // Set the scopes to request (space-separated for Google services).
      .setScope('https://www.googleapis.com/auth/drive')

      // Below are Google-specific OAuth2 parameters.

      // Sets the login hint, which will prevent the account chooser screen
      // from being shown to users logged in with multiple accounts.
      .setParam('login_hint', Session.getActiveUser().getEmail())

      // Requests offline access.
      .setParam('access_type', 'offline')

      // Forces the approval prompt every time. This is useful for testing,
      // but not desirable in a production application.
      .setParam('approval_prompt', 'force');
}
```

### 2. Direct the user to the authorization URL

Apps Script UI's are not allowed to redirect the user's window to a new URL, so
you'll need to present the authorization URL as a link for the user to click.
The URL is generated by the service, using the function `getAuthorizationUrl()`.

```js
function showSidebar() {
  var driveService = getDriveService();
  if (!driveService.hasAccess()) {
    var authorizationUrl = driveService.getAuthorizationUrl();
    var template = HtmlService.createTemplate(
        '<a href="<?= authorizationUrl ?>" target="_blank">Authorize</a>. ' +
        'Reopen the sidebar when the authorization is complete.');
    template.authorizationUrl = authorizationUrl;
    var page = template.evaluate();
    DocumentApp.getUi().showSidebar(page);
  } else {
  // ...
  }
}
```

### 3. Handle the callback

When the user completes the OAuth2 flow, the callback function you specified
for your service will be invoked. This callback function should pass its
request object to the service's `handleCallback` function, and show a message
to the user.

```js
function authCallback(request) {
  var driveService = getDriveService();
  var isAuthorized = driveService.handleCallback(request);
  if (isAuthorized) {
    return HtmlService.createHtmlOutput('Success! You can close this tab.');
  } else {
    return HtmlService.createHtmlOutput('Denied. You can close this tab');
  }
}
```

If the authorization URL was opened by the Apps Script UI (via a link, button, 
etc) it's  possible to automatically close the window/tab using 
`window.top.close()`. You can see an example of this in the sample add-on's 
[Callback.html](samples/Add-on/Callback.html#L47).

### 4. Get the access token

Now that the service is authorized you can use its access token to make
reqests to the API. The access token can be passed along with a `UrlFetchApp`
request in the "Authorization" header.

```js
function makeRequest() {
  var driveService = getDriveService();
  var response = UrlFetchApp.fetch('https://www.googleapis.com/drive/v2/files?maxResults=10', {
    headers: {
      Authorization: 'Bearer ' + driveService.getAccessToken()
    }
  });
  // ...
}
```

## Compatiblity

This library was designed to work with any OAuth2 provider, but because of small
differences in how they implement the standard it may be that some APIs
aren't compatible. If you find an API that it does't work with, open an issue or
fix the problem yourself and make a pull request against the source code.

## Other features

See below for some features of the library you may need to utilize depending on
the specifics of the OAuth provider you are connecting to. See the [generated
reference documentation](http://googlesamples.github.io/apps-script-oauth2/Service_.html)
for a complete list of methods available.

#### Resetting the access token

If you have an access token set and need to remove it from the property store
you can remove it with the `reset()` function. Before you can call reset you
need to set the property store.

```js
function clearService(){
  OAuth2.createService('drive')
      .setPropertyStore(PropertiesService.getUserProperties())
      .reset();
}
```

#### Setting the token format

OAuth services can return a token in two ways: as JSON or an URL encoded
string. You can set which format the token is in with
`setTokenFormat(tokenFormat)`. There are two ENUMS to set the mode:
`TOKEN_FORMAT.FORM_URL_ENCODED` and `TOKEN_FORMAT.JSON`. JSON is set as default
if no token format is chosen.

#### Setting additional token headers

Some services, such as the FitBit API, require you to set an Authorization
header on access token requests. The `setTokenHeaders()` method allows you
to pass in a JavaScript object of additional header key/value pairs to be used
in these requests.

```js
.setTokenHeaders({
  'Authorization': 'Basic ' + Utilities.base64Encode(CLIENT_ID + ':' + CLIENT_SECRET)
});
```

See the [FitBit sample](samples/FitBit.gs) for the complete code.

#### Modifying the access token payload

Similar to Setting additional token headers, some services, such as the 
Smartsheet API, require you to 
[add a hash to the access token request payloads](http://smartsheet-platform.github.io/api-docs/?javascript#oauth-flow).
The `setTokenPayloadHandler` method allows you to pass in a function to modify 
the payload of an access token request before the request is sent to the token 
endpoint:

```js
// Set the handler for modifying the access token request payload:
.setTokenPayloadHandler(myTokenHandler)
```

See the [Smartsheet sample](samples/Smartsheet.gs) for the complete code.

#### Service Accounts

This library supports the service account authorization flow, also known as the
[JSON Web Token (JWT) Profile](https://tools.ietf.org/html/draft-ietf-oauth-jwt-bearer-12).
This is a two-legged OAuth flow that doesn't require a user to visit a URL and
authorize access.

One common use for service accounts with Google APIs is
[domain-wide delegation](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#delegatingauthority).
This process allows a Google Apps for Work/EDU domain administrator to grant an
application access to all the users within the domain. When the application
wishes to access the resources of a particular user, it uses the service account
authorization flow to obtain an access token. See the sample
[`GoogleServiceAccount.gs`](samples/GoogleServiceAccount.gs) for more
information.

## Breaking changes

* Version 20 - Switched from using project keys to script IDs throughout the 
library. When upgrading from an older version, ensure the callback URL 
registered with the OAuth provider is updated to use the format 
`https://script.google.com/macros/d/{SCRIPT ID}/usercallback`.
