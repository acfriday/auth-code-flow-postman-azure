# Authorization Code Flow from OAuth 2.0 with PKCE and OpenID Connect

The Authorization Code Flow, enhanced with Proof Key for Code Exchange (PKCE) and integrated with OpenID Connect, provides a secure method for authorizing client applications to access server resources on behalf of users. This advanced flow builds upon OAuth 2.0, introducing additional security measures for mobile and public clients, and employs OpenID Connect for user authentication. We will illustrate this flow using Azure Cloud services and Postman.

## Prerequisites

1. **Microsoft Azure** - Ensure you have proper permissions to create resources and register applications for your tenant. The licenses I personally possess are `Office 365 E5` and `Enterprise Mobility + Security E5`.
2. **Postman API Platform** - This is a popular API development tool that can downloaded for free at its website https://www.getpostman.com. I'll be using the `64 bit` desktop version `v10.24.15`. We will utilize Postman to make requests to Azure's Microsoft Graph API.

## **The Components of OAuth 2.0**

<details><summary><b>Read Overview</b></summary>

#### **`Resources`**

* The digital assets or services the user grants access to via OAuth 2.0. Resources are hosted by Resource Servers, which require valid access tokens for data access.

#### **`Resource Owners`**

* Individuals or entities that have the authority to grant access to their resources. In most cases, the resource owner is the end-user.

#### **`Clients`**

* Applications requesting access to resources on behalf of the Resource Owner. Clients are authenticated by the Authorization Server and authorized by the Resource Owner to access specified resources.

#### **`Authorization Server`**

* The server that issues access tokens to clients after successfully authenticating the Resource Owner and obtaining authorization. It plays a critical role in the OAuth 2.0 security framework, ensuring that access to resources is granted only to clients with proper authorization from the Resource Owners.
* **Authorization Endpoint** `/auth` initiates the flow. Clients request this endpoint with parameters like `response_type=code`, `client_id`, `redirect_uri`, `scope`, `state`, and `code_challenge`.
* **Token Endpoint** `/token` exchanges the `authorization code` for tokens. The request includes `grant_type=authorization_code`, `code`, `redirect_uri`, `client_id`, and `code_verifier`.
* **Userinfo Endpoint** `/userinfo` when accessed with an access token, returns `claims` about the authenticated user.

#### **`Tokens`** - Strings representing the granted permissions

* **Access Token**: Enables access to the user's data via the Authorization: `Bearer <token>` header in API requests.
* **Refresh Token**: Used to renew an access token via the token endpoint with `grant_type=refresh_token`, without the user's interaction.
* **ID tokens**: Issued by the authorization server to the client application. Clients use ID tokens when signing in users and to get basic information about them.

#### **`Grants`**

* **Authorization Code Grant**: Involves redirecting the user to the authorization endpoint, obtaining an authorization code, and exchanging the code for tokens at the token endpoint.
* **Client Credentials Grant**: Used for server-to-server communication where the application acts on its own behalf. Access is granted based on the authorization of the client, not the end user.
* **Resource Owner Password Credentials Grant**: Allows direct exchange of user credentials for access tokens. Recommended only for trusted clients, as it exposes the user's password.
* **Implicit Grant**: Optimized for clients implemented in a browser using a scripting language. Deprecated in OAuth 2.1 due to security vulnerabilities.

#### **`Scope`** - Defines the level of access the application requests

* Expressed in `space-delimited strings`, such as `scope=openid profile email`, determining which resources the application can access and actions it can perform.

#### **`Proof Key for Code Exchange (PKCE)`** - Enhances security for public clients

* Uses `code_challenge` and `code_challenge_method` during the authorization request, and `code_verifier` in the token exchange process to mitigate interception attacks.

#### **`OpenID Connect (OIDC)`** - An authentication and authorization layer built on top of OAuth 2.0, incompatible with OAuth 1.0

* Utilizes `ID Tokens`, returned along with the `access token`, containing claims about the authentication of the user.

</details>

## Microsoft Azure - Register a Web Application in Entra ID

<details><summary><b>Instructions</b></summary>

1. Sign in to the `Microsoft Entra admin center` as at least a `Cloud Application Administrator`.
2. Browse to `Identity` > `Applications` > `App registrations` and select `New Registration`.

![image](https://github.com/acfriday/todolist-webapp-flask-sqlite3/assets/82184168/86e397ac-3102-4156-a5c9-92e04ab7c499)

3. Enter a `Display Name` for your application.
4. We'll select the default `single-tenant` option.
5. Select Web as our platform with `http(s)://localhost` (excluding the parenthesis as seen below) as our redirect URI.
6. Complete this step by selecting `Register`.

![image](https://github.com/acfriday/todolist-webapp-flask-sqlite3/assets/82184168/b0913646-6c21-4374-a599-d3ddba9b7171)

</details>

## Microsoft Azure - Client ID, Client Secret, API permissions, Endpoints

<details><summary><b>Instructions</b></summary>

1. We'll need to note our app's `Client ID` from the Entra ID `Overview` tab under `App Registrations` for later use.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/8978643a-7391-4dac-81d1-ea33ab743776)


2. Next we'll retrieve our `Client Secret`. Select `Certificates & secrets` > `Client secrets` > `New client secret`.
Click `Add` to save your `Client Secret`. I chose the `default expiry time` after selecting `new client secret`,
we'll need to retrieve this secret again later.

*    `I'll be deleting this Client Secret from my account before posting it publicly here.`
*   `Important! Record the secret's value for later use This secret value
    is never displayed again after you leave this webpage.`

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/9ebe2c0d-e0a2-4afd-9cef-21819850879d)

4. Now let's check the `API permissions` tab and verify that our app has the default access of `User.Read`
for the `Microsoft Graph API` `resource` that we'll be querying.

![image](https://github.com/acfriday/todolist-webapp-flask-sqlite3/assets/82184168/204c2521-11a1-45fa-8cf8-60066ccd316f)

5. Finally we'll need to retrieve the Azure REST Endpoints we'll send out requests towards.
The Authorization Endpoint `OAuth 2.0 authorization endpoint (v2)`
and Token Endpoint `OAuth 2.0 token endpoint (v2)` are what we'll copy from here.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/cd9634f8-132f-45cc-a497-d3bba8c75bf3)


</details>

## Postman - Request OAuth 2.0 Access Token, Integrate PKCE and OpenID Connect

<details><summary><b>Instructions</b></summary>

1. In `Postman`, create a new `Request` and navigate to the `Authorization tab` and select `OAuth 2.0` as the auth `type`.

![image](https://github.com/acfriday/todolist-webapp-flask-sqlite3/assets/82184168/5a7eab8f-f740-45e3-a6e5-e9020891d38a)

2. This is where we'll input data for the values below:

* Token Name: `Any name of your personal choice`
* Grant Type: `Authorization Code (With PKCE)`
* Callback URL: `https://localhost`
* Auth URL: `Entra ID > App Registrations > your app > Overview > Endpoints`
* Access Token URL: `Entra ID > App Registrations > your app > Overview > Endpoints`
* Client ID: `Entra ID > App Registrations > your app > Overview`
* Client Secret: `Entra ID > App Registrations > your app > Certificates & secrets`
* Code Challenge Method: `SHA265` `(to encrypt the randomly generated Code Verifier below)`
* Code Verifier: `1380829049516597410379244643308228514410431459843774421183596`
* Scope: `openid` `profile` `email` `https://graph.microsoft.com/user.read`


  `Note! Remember that our 'Scope' values should be space-delimited.`


![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/94181010-fb2c-4cfd-9b80-fbc9f6d63538)

3. Scroll to the bottom and click `Get New Access Token`.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/ce6fdca9-c21c-4b69-992c-853c307553c3)

</details>

## Postman - Authenticate, Retrieve Access and Identity Token

<details><summary><b>Instructions</b></summary>

1. You should receive a popup from Microsoft after clicking `Get New Access Token`, input your credentials to
authentication with Microsoft with an account from your tenant where this app is registered.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/b028748e-1391-46bc-a7bd-ab645e0ed142)

2. After providing consent and successfully authenticating you should receive the following acknowledgement, click `Proceed` here.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/b030e627-430f-4d7a-865d-760298adba22)

3. The `identity token` below is taken from another `access token` of mine as you can see from the `token name` but the format is the same, after selecting `proceed`, scroll down to the `id_token` and copy its entire value.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/7d251414-ec8f-4548-adab-520dcdb65dda)

4. Decypt this value by pasting it over on https://jwt.ms for inspection. The `authorization server` issues `id tokens` that contain `claims` that carry information about the user. You can use the `id_token` parameter to verify the user's identity with the `client` application performing the access request to begin a session with the user, these shouldn't be used for authorization as `access tokens` are used for authorization.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/2f70d76f-9c9d-4327-aa5a-4b41ebd803a8)

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/8f4fb6e5-815b-4f1c-9cc7-8485a845970a)

5. Returning to `Postman`, scroll back to the top of our `Token Details`, go ahead and `use` the `access token`.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/c57fe7a3-81f4-446f-8386-7afede714b10)

6. Scroll to the top of `Postman` after selecting `Use Token` to verify our `named` `access token` is being used in our upcoming request.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/39c870d4-44a3-45f9-8515-03c6973ef89d)

</details>

# Query and Response
* With us now authenticated, let's use our permissions to call the `http(s)://graph.microsoft.com/v1.0/me/` `Endpoint` which allows the app to read the profile and discover relationships of the currently signed-in Microsoft account.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/e1c8baab-15d4-487a-b816-c6dea6ecbc31)

# Code Verifier Encrypted vs Plain Output

* `Code verifier`: a cryptographically random string used to correlate the `authorization request` to the `token request`.
* `Code challenge`: derived from the `code verifier` sent in the `authorization request` to be verified against later.
* `Code challenge method`: what was used to derive the `code challenge`.

When the `verifier` matches the expected value, the `authorization server` will issue an `access token` appropriately. If there is a problem, then the server responds with an `invalid_grant` error.

![image](https://github.com/acfriday/auth-code-flow-postman-azure/assets/82184168/157eb680-227c-4313-aec8-a900893280a0)

#### `Plain Method`
GET http(s)://login.microsoftonline.com/d8fabb20-154c-4c50-9b38-a71084f7d070/oauth2/v2.0/authorize?response_type=code&client_id=919ae2d5-deca-416e-9a6f-f7e56b072325&scope=openid%20profile%20email%20https%3A%2F%2Fgraph.microsoft.com%2FUser.Read&redirect_uri=https%3A%2F%2Flocalhost&**`code_challenge=1380829049516597410379244643308228514410431459843774421183596&code_challenge_method=plain`**
* If I leave the method on `Plain` and click `Get New Access Token`, our request will send our `code_challege` derived from our `Code verifier` insecurely across the internet.

#### `SHA-256 Method`
GET http(s)://login.microsoftonline.com/d8fabb20-154c-4c50-9b38-a71084f7d070/oauth2/v2.0/authorize?response_type=code&client_id=919ae2d5-deca-416e-9a6f-f7e56b072325&scope=openid%20profile%20email%20https%3A%2F%2Fgraph.microsoft.com%2FUser.Read&redirect_uri=https%3A%2F%2Flocalhost&**`code_challenge=BdayhqpHPROIB57cO6WsOlrkK7BMvdRkn834UrEV1Gc&code_challenge_method=S256`**
* When selecting `Get New Access Token` with the `SHA-256` method and nothing else changed we can verify in our request that our `code_challenge` is encrypted when sent across the internet.
