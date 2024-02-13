# Integrate 2FA and IVA functionality (Sugar Ant)

**Epic Type:** Implementation Epic

Epic planning and implementation follow the
[Epic Planning and Marathon SOP](https://docs.ghga-dev.de/main/sops/sop001_epic_planning.html).

## Scope

### Outline

This goal of this epic is to implement all the functionality necessary
to support two-factor authentication (2FA) and independent verification addresses (IVAs).

### Included/Required

The implementation must include all changes necessary to support the full functionality described
in [GFRC007: User authentication and identification](https://docs.ghga-dev.de/main/grfcs/grfc007_auth_and_identity.html)
and in the [white paper on 2FA and identity verification](https://docs.ghga-dev.de/main/white_papers/auth_and_identity.html).

These changes are outlined in the following sections. They affect both the frontend and the backend.

It has been decided to implement the frontend changes by extending the existing React-based code and not waiting for the planned migration to an Angular-based frontend. The frontend implementation should also cover required mocks for the new backend APIs.

On the backend side, mostly the auth adapter is affected, but also some other services.

### Optional

Some changes have been specified as optional in the sections below.

### Not included

- support for multiple TOTP tokens
- support for other kinds of 2FA factors
- integration tests (extending the Archive Test Bed)

## Implementation Details

The following sections describe the various parts of the backend and frontend that are affected by the implementation work in this epic, and the corresponding user journeys and new or changed APIs:

### Auth Session Management

The Auth Adapter must be extended with a cookie based auth session management.

This means the Auth Adapter should track user sessions using unique session IDs submitted via cookies. It also maintains an internal session store which stores the current user information and authentication state of the user. The session store should not be used for storing any other information about the state of the application.

The store will be implemented in memory in the first implementation. Another type of cache could be used later that would allow retaining sessions when restarting the auth adapter and support multiple instances of the auth adapter. Therefore the storage mechanism for the session cache should be made easily changeable.

The Auth Adapter creates an auth session and tracks users as soon as they have logged in via LS Login, but not earlier.

Instead of converting the OIDC access token to our internal auth token, the Auth Adapter should now convert the content of the user session into the internal auth token. The internal auth token will change a bit, as outlined in a section below.

The Auth Adapter must also be extended with a mechanism that prevents "session riding" attacks, using the "Cookie-to-header token" method. Thereby, the first request that creates the session responds not only with the session cookie, but also with a "CSRF cookie" which must be a unique and unpredictable string, either as a random string or derived from the session token via HMAC and an application secret. Contrary to the session cookie, the CSRF cookie must not set the "HttpOnly" flag, because it must be read by JavaScript running in the frontend so that it can be passed in the request header as a CSRF token for each request to the backend. The Auth Adapter needs to compare the CSRF token with the unique string set in the CSRF cookie.

The following endpoints will be implemented in the Auth Adapter for managing sessions. These endpoints are not proxied by the API gateway, but respond directly back to the client.

- `POST /rpc/login` - *creates or gets the user session*
  - request body: empty
  - auth header: optionally, the OIDC access token
  - response status:
    - `204 No Content`: session exists or has been created
    - `401 Unauthorized`: invalid access token or CSRF code
  - response header:
    - `X-Session`: JSON-encoded client session object
      (contains all properties that are relevant for the client)

If a user session already exists, this endpoint simply returns the existing session. This is used to request the current session state by the frontend when the page is loaded. If no user session exists, the authorization header is inspected. If a valid OIDC access token is found, the user data is fetched from the userinfo endpoint of the OIDC provider. A new session is created based on that user information. Also, a CSRF token is created and added to the session. The session starts with the state `needs-registration` or higher depending on the state of user registration in the database. This is used after the user logged in via OIDC on the client.

Note that this method does not return the status code `200 OK` because this cannot be directly passed back to the client via the ExtAuth protocol, since it would be interpreted as a successful authentication and the request would be proxied. Therefore, the response status code `204 No Content` is used and the session is passed back in the response header instead of the response body.

- `POST /rpc/logout` - *removes the user session*
  - request body: empty
  - response status:
    - `204 No Content`: session exists or has been created
    - `401 Unauthorized`: already logged out or invalid CSRF code

This endpoint simply removes the auth session that is tracked in the Auth Adapter, thereby effectively logging the user out.

When an auth session already exists and any other request is made, the expiration date of the session should be automatically extended. The timeout for auth sessions and their maximum duration should be made configurable.

See also: [ADR: user session management](https://github.com/ghga-de/adrs/blob/main/docs/adrs/adr004_user_session_management.md)

### Session states

The session should have a state attribute that can have the following values:

- **none**
  There is no session yet, or the user has logged out.
  Set as default and via `POST /rpc/logout`.
- **identified** (frontend only)
  The user has logged in via LS Login. External ID, name and email are known.
- **needs-registration**
  The user still needs to register with GHGA.
  Set via `POST /rpc/login`.
- **needs-re-registration**
  The user is registered, but needs to confirm a name or email change.
  Set via `update_session()`.
- **registered**
  The user is registered with GHGA, but does not have a 2nd factor.
  Set via `update_session()`.
- **needs-totp-token** (frontend only)
  The user needs to create a TOTP token in the next step.
- **new-totp-token**
  A TOTP token for the user was created, but is not yet confirmed by the user.
  Set via `POST /totp-token`.
- **has-totp-token**
  The user has installed a TOTP token and it was confirmed to be working.
- **lost-totp-token** (frontend only)
- Set via `update_session()`.
  The user indicated that the TOTP token was lost and needs to be re-generated.
- **authenticated**
  The user is fully authenticated with two factors.
  Set via `POST /rpv/verify-totp`.

### TOTP Management

To implement two-factor authentication using TOTP, we need a mechanism to create and validate TOTP tokens. This functionality could be implemented as a separate service. However, the outcome of the validation of a TOTP should be stored in the auth session, so the most simple solution is to validate TOTPs inside the Auth Adapter, since it holds the auth session. And then it makes sense to let the Auth Adapter also handle the creation of the TOTP tokens, though technically this does not need to be part of the Auth Adapter and could be handled by a separate service. We decided to use this simple approach, implementing TOTP management in a module that is part of the Auth Adapter. The related endpoints would be intercepted by the Auth Adapter as part of the ExtAuth protocol, and handled by the TOTP management module.

The Auth Adapter should intercept the following four endpoints:

- `POST /totp-token` - *creates a TOTP token*
  - request body:
    - `user_id`: string (the registered User ID)
    - `force`: boolean (whether an existing TOTP can be replaced)
  - response body:
    - `uri`: string (the provisioning URI)
  - alternative response body:
    - `text`: string (the secret as text)
    - `svg`: string (URI as QR-code in SVG format)

This endpoint first verifies that the user has a valid auth session, i.e. has been successfully logged in via LS Login. It then verifies that the session refers to an already registered user, and that the user has the same user ID as specified in the request body. Next, if `force` is not set to `true`, it verifies that this user does not already have an active TOTP token. If any of these verification steps fail, it responds with the HTTP status `401 Unauthorized`. Otherwise, creates a TOTP token and returns its provisioning URI which also contains the secret (seed) used by this token as a query parameter, using the HTTP status `201 Created`.

The implementation only supports a single TOTP token per user. If an activated token (a TOTP token that has already been successfully validated at least once) already exists and the `force` flag is set to `true`, then the existing TOTP token will be replaced by the newly created one.

As a side effect of this endpoint, all existing IVAs associated with this user must be set back to "unverified", as required in the white paper for 2FA and IVAs.

The QR code should be created in the frontend, e.g. using `react-qr-code` or the `qr-code` web component. If during implementation there are any issues with this approach, it can alternatively also be created in the backend, e.g. using the `segno` library.

- `POST /rpc/verify-totp` - *verifies a one-time password*
  - request body:
    - `user_id`: string (the registered User ID)
    - `totp`: string (the one-time-password)

This endpoint first verifies that the user has a valid auth session, i.e. has been successfully logged in via LS Login. It then verifies that the session refers to an already registered user, and that the user has the same user ID as specified in the request body. Next, it verifies that this user has already created a TOTP token. Finally, it verifies the given one-time password in `totp` using the current time and a configurable time window. If all verification steps succeed, the token is activated, and the HTTP status `204 No Content` is send in an empty response. Otherwise, if the one-time password could be verified, it responds with the HTTP status `401 Unauthorized`.

Note again that these endpoints do not respond with the HTTP status code `200 OK` so that the responses are returned directly to the client.

The application logic for the TOTP related endpoints should be implemented in the TOTP Management module. This module should also implement rate limiting and replay attack prevention.

See also: [ADR: custom 2FA micro service](https://github.com/ghga-de/adrs/blob/main/docs/adrs/adr003_custom_2fa_service.md)

### IVA management

The management of IVAs (independent verification addresses) should happen in the User Management service.

For the definition of an IVA, see the section on backend models below.

The following REST-style endpoints should be added to the User Management service:

- `GET /users/{user-id}/ivas`
  - *returns the list of IVAs belonging to the specified user*
  - auth header: internal token (of data steward or same user)
  - response body:
    - list of IVA objects
- `POST /users/{user-id}/ivas`
  - *creates an IVA for the specified user*
  - auth header: internal token (of data steward or same user)
  - request body:
    - `type`: enum (phone, fax, postal address, in-person)
    - `value`: string (the actual phone number)
  - response body:
    - `id`: string (the ID of the newly created IVA)
- `DELETE /users/{user-id}/ivas/{iva_id}`
  - *deletes an existing IVA of the specified user*
  - auth header: internal token (of data steward or same user)

Additionally, the following RPC-style endpoints will be added:

- `POST /rpc/ivas/{iva_id}/unverify`
  - *invalidate the specified IVA*
  - auth header: internal token of a data steward
  - response status:
    - `204 No Content`: state changed to `unverified`
    - `401 Unauthorized`: auth error (e.g. not a data steward)
  - *should also send a notification to the user*
- `POST /rpc/ivas/{iva_id}/request-code`
  - *request the verification of the specified IVA*
  - auth header: internal token
  - response body: empty
  - `200 No Content`: state has been changed to `code_requested`
  - `400 Bad Request`: IVA did not have the state `unverified`
  - `401 Unauthorized`: auth error (e.g. IVA not of current user)
  - *should also send a notification to the user and a data steward*
- `POST /rpc/ivas/{iva_id}/create-code`
  - *create verification for the specified IVA*
  - auth header: internal token of a data steward
  - response body:
    - `verification_code`: string (to be transmitted to the user)
  - response status:
    - `200 OK`: state has been changed to `code_created`
    - `401 Unauthorized`: auth error (e.g. not a data steward)
- `POST /rpc/ivas/{iva_id}/code-transmitted`
  - *confirm the transmission of the verification code for the specified IVA*
  - auth header: internal token of a data steward
  - response status:
    - `204 No Content`: state has been changed to `code_transmitted`
    - `400 Bad Request`: IVA did not have the state `code_created` or `code_transmitted`
    - `401 Unauthorized`: auth error (e.g. not a data steward)
  - *should also send a notification to the user*
- `POST /rpc/ivas/{iva_id}/verify-code`
  - *submit verification code for the specified IVA*
  - auth header: internal token
  - request body:
    - `verification_code`: string (that had been transmitted to the user)
  - response status:
    - `204 No Code`: verification code correct, IVA is now in state `verified`
    - `400 Bad Request`: IVA did not have the state `code_transmitted`
    - `401 Unauthorized`: auth error or verification code was wrong
    - `429 Too Many Requests`: IVA has been reset to unverified
  - *should also send a notification to the data steward*

### Claims Repository

The `Claims` model must provide a new `iva_id` field, as also specified in the section on backend model changes below.

The claims repository currently has an endpoint

- `POST /download-access/users/{user_id}/datasets/{dataset_id}`

This endpoint must be extended to become

- `POST /download-access/users/{user_id}/ivas/{iva_id}/datasets/{dataset_id}`

The claims repository also has endpoints at

- `GET /download-access/users/{user_id}/`

The routes of these endpoints do not need to be changed. However, these endpoints must now also check that the corresponding claims are bound to IVAs that have been verified for the given user.

### Access Request Service

The `AccessRequest` model should have a new optional `iva_id` field, as specified in the section on backend model changes below.

The `PATCH /access-requests/{access_request_id}` endpoint that is used to allow or deny access requests must be extended so that it also accepts the corresponding `iva_id` in the body. The `iva_id` must then also be passed to the claims repository.

### Internal Auth Token

Until now, an internal auth token was created and passed by the Auth Adapter via the API gateway after the user was successfully authenticated via LS Login. The token contained an additional `state` enum (active, inactive, invalid). It could contain the internal or external user id in the fields `id` and `ext_id`.

In the new implementation, we change the internal auth token as follows:

The internal auth token will be only created and passed on by the Auth Adapter if the user is fully authenticated, i.e. logged in via LS Login and the second factor has been validated.

The `state` field will be removed from the auth token. The existence of the token always implies that the user account is active and not invalid.

The `ext_id` field will be removed from the auth token. There are only two exceptional cases where it is needed, and in theses cases it can be stored in the `id` field instead. The `id` field will also be made mandatory and required to be a non-empty string.

There are only the following two exceptions where the auth token will be also added by the Auth Adapter if the user is only logged in via LS Login:

- `POST /users`
  - used to self-register a user
  - the `id` field of the auth context will contain the external id, not the internal id
- `PUT /users/{user_id}`
   when requested by users to confirm a name an email change
  - the user must be already registered
  - the `id` field of the auth context will contain the internal id of the user, and it must correspond to the `user_id` in the path

### Service Commons Library

The `ghga-service-commons` library must be changed to reflect the changes in the internal auth token as outlined above.

### Backend Models

A new `IVA` (independent verification address) model must be added to the User Management service:

- `id`: string (unique internal id)
- `user_id`: string (internal id of the user)
- `type`: enum (`phone`, `fax`, `postal_address`, `in_person`)
- `value`: string (the actual phone number)
- `verification_code_hash`: optional string (hash of actual verification code)
- `verification_attempts`: int (number of attempts to verify the code)
- `state`: enum (`unverified`, `code_requested`, `code_created`, `code_transmitted`, `verified`)
- `created`: date (of creation)
- `changed`: date (date of last change)

The `IVA`s should be maintained in a separate collection by the User Management service.

The `verification_code_hash` and `verification_attempts` fields should only be stored in the database and not be returned via the REST interface. The verification code itself should be created randomly and only be shown to the data steward or transmitted directly to the user, it should not be stored in the database.
The `verification_code_hash` should be created using a random salt and a dedicated password hashing algorithm from the verification code.
The `verification_attempts` field tracks how often the user attempted to send the verification code in the `code_transmitted` state.
After three failed attempts or when the `last_changed` field indicates that the verification process takes too long ago (the number of days should be configurable), the state should be set back to `unverified`.
The `state` transitions from `unverified` (after creation), over `code_requested` (user requested a verification), `code_created` (a verification code has been created) and `code_transmitted` (the verification code has been transmitted to the user) to `verified` (the user confirmed the receipt of the verification code by returning it properly).

The `Claims` model must be extended so that claims in addition to referencing a user, it can optionally also reference an `IVA` via an additional property `iva_id`.

A new `TOTPToken` model must be created that includes all attributes required to operate a TOTP token, like the OTP secret key that it is based upon (details will be defined during implementation).

The `TOTPToken`s will be managed by the Auth Adapter, as explained above.

The `AccessRequest` model must be extended to also include an optional `iva_id` field referencing an IVA that is used to verify the corresponding access grant. This field will be initially set to `None`, and it will be set after access has been allowed.

### Login Flow in the Frontend

The frontend keeps the state of the current user in the session storage. The state in the frontend can be in one of the stages already listed above.

The user starts with an empty client session (state is `null`). Only when the last stage of the two factor authentication process has been reached, the user is considered fully authenticated.

To initiate the authentication process, the user must first log in via LS Login and the OIDC flow must have been completed, at the end of which the frontend receives an OIDC access token. After that, the state is moved to `identified`.

Now let's assume the user is in the state `identified`.

The frontend then requests the user data from the user management service using the `POST /rpc/login` endpoint, passing the the OIDC access token. On the backend side, this will create an auth session, of which a view with the relevant fields is passed back to the frontend via the `X-Session` header.

The state from the backend is now used as frontend session state, and depending on the state, the user can be redirected to a different route.

If the user is not yet registered, the state is set to `needs-registration` and the frontend asks the user to register.

If the user info from LS Login does not match the registered user info, the user will not be considered valid by the backend, the state is set to `needs-re-registration`, and the frontend should show a message accordingly, asking the user to confirm and thereby re-register.

TODO: Should we send a notification to a data steward in this case, or maybe even invalidate existing TOTP tokens and IVAs?

If the state is `needs-registration` or `needs-re-registration`, the user is requested to newly register or confirm the changed user data. Registration of users has already been implemented in the frontend and in the backend and does not need to be changed. After registration, the frontend also gets the user info from the backend and stores it in the session storage.

If the session state is `registered`, the user will be shown the registered data (maye with an option to change them via re-registration) and informed that a second factor needs to be created. If the user confirms, the state progresses to `needs-totp-token`.

The frontend then uses the `POST /totp-token` endpoint to create a provisioning URL, presents it in form of a QR code to the user and asks the user to scan the QR code using an authenticator app. It should also show a button or link to display the secret as text as fallback for manually entering the secret and as backup code.

The frontend should recommend using Aegis (for Android) or 2FAS (for Android and iOS) as authenticator apps. The authenticators provided by Microsoft and Google (both are available for Android and iOS) can also be mentioned, since some users may already have them installed. However, the Google authenticator should not be explicitly recommended, since it does not require unlocking the phone and therefore is less secure. The also popular Authy should not be recommend at all, since it stores the secrets in the cloud and does not provide a means for the user to retrieve them, which makes it impossible to migrate them to another app.

On the same page, the frontend also asks the user to enter the one-time password (six-digit code) shown in the authenticator app to validate the creation of the second factor in a text input field.

If the session has the `has-totp-token` state, a similar text input field should be presented to the user, asking them to enter the one-time-password (six-digit code) shown in the authenticator app for authentication.

The user is also shown a link that allows re-creating the second factor. This link can be used in the case they lost the phone with the authenticator app and do not have a backup of the secrets. Following the link, after a warning that all independent verification addresses will be invalidated, the user state should be set to `lost-topt-token`. In that case, the `POST /totp-token` endpoint should be called with the `force` flag to overwrite an existing TOTP token.

When the user submits the one-time-password, the frontend uses the `POST /rpc/verify-totp` endpoint to check its validity.

If the password is validated, the user is moved to the state `authenticated`, otherwise an authentication error is displayed and the state stays the same.

The frontend must also change the "Logout" button so that it calls the `/rpc/logout` endpoint of the Auth Adapter.

### Access Request Management

A few changes need to be made to the access request functionality implemented in the frontend.

The Access Request Submission Form does not need to be changed, since the IVA is not required at this time. Users can formulate Data Access Requests without entering or verifying an IVA.

The Access Request Browser itself does not need to be changed, either. It may interesting to show the corresponding IVA for allowed accesses, but this does not need to be implemented as part of this epic.

However, the Access Request Details Form that allows granting or denying access needs some changes.

In addition to the "allow" and "deny" buttons, it should also have a selector that shows the types, values and states of the IVAs created by the corresponding user. The "allow" button should only be activated when one of the IVAs is selected. If the user has already been granted access using a given IVA, this should also be clearly indicated.

### IVA Verification

On the user profile page, it should be possible to create one ore more IVAs and to list the existing IVAs of the user.

The IVA creation dialog should allow to select the type and enter the value of the new IVA.

For existing IVAs, their state and the number of bound datasets should be displayed. Each IVA should have a "delete" button. When an IVA is still bound to a dataset, a warning should be displayed before the deletion.

Each existing IVA in the state `unverified` should have a button "request verification". Each IVA in the state `code-transmitted` should have a button
"enter verification code".

After clicking "request verification", the state of the IVA should be moved from `unverified` to `code_requested`, a confirmation email should be sent to the user and a notification email sho0uld be sent to a data steward.

After clicking "verify", the verification code should be requested from the user via an input field, and the state of the IVA should be moved from `code-transmitted` to `verified`. If this does not succeed, a corresponding error message must be shown to the user. After three failed attempts or when the verification code expired, users should be informed that they need to re-request the verification because the verification code expired.

After the user requested verification, a data steward should have received a notification. The data steward should be able to access an "IVA browser" page that lists all users and their IVAs, similar to the "access request browser". The IVAs in the state `code_requested` or `code_created` should have a button "(Re)create code". After clicking the button, the IVA should be moved to the state `code_created` and a dialog should appear that shows the verification code and ask the data steward to send it to the user via the selected IVA. The dialog should have three buttons: "Cancel" would revert the creation of the code and reset the state to `code_requested`, "Send later" would do nothing, but remind the data steward to confirm the transmission of the code later, and "Confirm transmission" would move the IVA to the state `code-transmitted`. This will also notify the user via an email that the code has been transmitted. Of course the code itself should *not* be sent in the notification email since it is expected to be sent via the transmission channel specified in the IVA. The IVAs in the state `code_created` should also have a button "confirm transmission". All IVAs should also have a button "Invalidate" that would reset its state to `unverified`.

The RPC-style endpoints that can be used to move the state of the IVAs and send corresponding notifications are explained in the section "IVA Management" above.

## Flow Diagrams

### Frontend

The following flow diagram visualizes the login flow in the frontend.

![Login flow in the frontend](./images/flow_frontend.png)

### Backend

The following flow diagrams visualize the backend flows for the various routes that are handled by the Auth Adapter.

Note that per the ExtAuth protocol, a response with a status code of "200 OK" means that the route is considered valid by the API gateway and forwarded to the corresponding micro service. Any other status code in the response causes the response to be directly passed back to the client.

![Auth flow in the backend](./images/flow_backend.png)

## UI Wireframes

### User Login

UI wireframe for the pages and dialogs related to the user login flow:

![Login flow](./images/wireframe_login.png)

### User Profile

UI wireframe for the profile pages and dialogs related to IVA management by the user:

![IVA management on the profile page](./images/wireframe_profile.png)

### IVA Browser

UI wireframe for the IVA browser and dialog used for IVA management by the data stewards:

![IVA Browser](./images/wireframe_ivas.png)

### Access Request Details

UI wireframe for the access request detail form used by the data stewards:

![Access Request details form](./images/wireframe_access.png)

## Human Resource/Time Estimation

Number of sprints required: 5

Number of developers required: 2
