# Integrating Entra ID for Enhanced Security in Three-Tier Architecture

## Abstract : 

As we all know, three-tier architecture is a well-established design that organizes applications into three logical and physical computing tiers:
- The UI Layer (Client App),
- The API layer where data is processed, and
- The data tier where application data is stored and managed (DB).
Today, security is at the centre of everything. This article sheds light on the different approaches developers use to integrate the Entra ID Authentication & Authorization product into a three-tier architecture to secure the design, making it reliable and robust. The content presumes that all components are running on the Azure cloud and covers various three-tier architecture designs.


## Technical Components Involved:

- Entra ID products
  - MSAL
  - Managed Identity 

## Example Scenario 1: A typical client app, API service, and a database:

UI Layer (Client App):
•	Regardless of the platform type (web, mobile, or desktop), the app can use the MSAL library to authenticate the user by requesting an ID token to identify the user.
•	The prerequisite is that you need one app registration in Entra ID for the client app and must configure the redirect URI depending on the client app's platform type.
•	As part of the sign-in process, the user is redirected to the Entra ID sign-in page. Once the user completes the authentication process (including MFA if configured), the client app allows the user to log in to the app. (Transaction 1 in the diagram).
•	The app then needs to invoke the main service API to retrieve or update data for its functionality.
•	To secure this transaction, there will be another app registration for the main service API, which exposes the APIs. The client app requests an access token for that API from Entra ID on behalf of the user. This process can also be combined with the sign-in request through hybrid flow, where the initial sign-in request asks for both the ID token (for the client app to identify the user) and the access token (which can be used as authorization bearer while invoking the main service API).
•	The client app invokes the main service API by attaching the access token in the authorization bearer header (Transaction 2 in the diagram).
API Layer (Main Service):
•	The main service API should accept the HTTPS call (Transaction 2) from the client app and validate the authorization by decoding the access token received through the authorization bearer header.
•	As mentioned earlier, the main service API will have its own app registration in Entra ID. It must download the keys from Entra ID, cache them for 24 hours (Transaction 3), and validate the signature, audience, issuer, and app-specific roles/claims to allow only authorized users and app requests.
•	If the above validation fails, Transaction 2 will be denied with a 401 (Unauthorized) status.
•	After successful validation of the user and app, the main API service is ready to perform CRUD operations on the database. There are two approaches here:
•	Variation 1: OBO Flow 
o	The main API service should request an access token for the data layer in exchange for the access token it received through Transaction 1. This process is known as the OBO (On-Behalf-Of) flow (Transaction 4 in the diagram).
o	For practical implementation, let's assume the data layer is Azure SQL. The main API service requests a token for the Azure SQL delegated permission through the OBO flow and then attaches it to the authorization bearer header while invoking Azure SQL methods (Transaction 5 in the diagram).
o	The advantage of this approach is that the user context persisted throughout the entire transaction from the client app to the data layer.
•	Variation 2: Managed Identity Flow 
o	Managed identity is set up between the main service API and the data layer. For example, if the API is an Azure function and the data layer is Azure SQL, the system-assigned identity is enabled for the function app, and Azure SQL RBAC is configured to establish trust with the system-assigned identity (Transaction 7 in the diagram).
o	In this approach, the user context is lost in the transaction, and the data layer solely trusts that the main API service is allowing only authorized users to invoke the operation as intended.
o	In this approach, it is important to ensure that the database layer is accessed exclusively through the main API service and not from any other services. Failure to do so, may lead to improper handling of authorization violating the least privilege principle. 
Data Layer (DB):
•	In the case of Variation 1, the database layer downloads the keys from Entra ID, caches them for 24 hours (Transaction 6), and validates the signature, audience, issuer, and app-specific roles/claims to allow only authorized users and app requests.
•	In the case of Variation 2, the database layer relies on the Azure managed identity implementation and validates the RBAC configured between the database layer and the main API service (no validation on the user).(Transaction 7)

## Example Scenario 2: Blocking Sign-In for **CustomClient** App from Non-Compliant Devices

Let us assume that an organization has a mobile app called **CustomClient**, which is an HR-related app for their employees, and this app relies on the HR service **CustomService** that exposes corporate data through an API. As an IT admin, you want to ensure that this app is accessed only through compliant devices. In other words, although the app is available through Google Play or the Apple App Store, it must be accessed from an MDM-enrolled or managed device.

Looking at the implementation of **CustomClient** and **CustomService**. As mentioned earlier, **CustomService** exposes all the data through an API and is protected by Entra ID. In simple terms, **CustomService** has it’s own app registration, and each of the APIs exposed requires some kind of permission in the token, either through scp or role claim. These permissions are exposed through the "Expose an API" blade of the app registration.

On the other hand, **CustomClient** has a separate app registration, and it has added the necessary permissions to invoke the **CustomService** through the "API Permissions" blade by adding and granting the permissions exposed by **CustomService**. From the code implementation perspective, during the sign-in activity of the **CustomClient** app, it also requests a token for **CustomService** to invoke the API and populate data.

Now, comparing this requirement to the Example Scenario 1 with Outlook Client, the trivial solution here is to apply the CA policy to **CustomService** while defining the CA policy(Target resource here is **CustomService**). This ensures that sign-in fails in the **CustomClient** app if it is accessed from a non-compliant device, since the **CustomClient** app requests a token for **CustomService** during the sign-in flow, which enforces the Conditional Access Policy to be applied on the app.

On the contrary, let us assume an admin has applied a Conditional Access policy to the target resource **CustomService**, which is accessed by multiple client applications on web, desktop, and mobile platforms, including our example **CustomClient** app. By now, you would have guessed the behavior: the CA policy applies to all client apps requesting a token for CustomService.

Now, let's consider a scenario where you would like to exclude the mobile app CustomClient from the Conditional Access policy. Can this be done? The answer is NO, since the mobile app will not be listed under the selected apps options in the CA policy definition.

## Example Scenario 3: Blocking Sign-In for **CustomClient** App Without **CustomService** from Non-Compliant Devices

This scenario is a variation of Example Scenario 2, where the **CustomClient** app does not have a dependency on **CustomService**, or **CustomService** isn’t protected by Entra ID. As you can imagine, you cannot define the Conditional Access Policy since the **CustomClient** app will not be available while defining the policy under the target resource, as it is a public client app registration. So, the question is, how do you enforce the Conditional Access Policy?

A workaround here is to introduce a new dependency on a dummy service and configure the app to request a token for this service. Here’s how you can implement it:
- Create API App Registration: Register a dummy API in Entra AD and expose a scope. This dummy service will act as the target resource for the Conditional Access Policy.
- Add/Grant Permissions: In the **CustomClient** app registration, add permissions to the exposed scope of the dummy API. Even though there is no actual API implementation

From code perspective, the **CustomClient** app will request a token for this dummy service during the sign-in, which ensures that the Conditional Access Policy is evaluated against the target resource **CustomService** and hence policy will be enforced. This approach leverages the underlying requirement of Conditional Access Policies to be tied to a resource, thereby indirectly protecting the **CustomClient** app.


## Anti-pattern Alert: Conditional Access Policies in case of All Cloud Apps

Applying Conditional Access Policies to all cloud apps may seem comprehensive, but it will not enforce the conditional access policy on client apps that lack dependencies on protected resources or services under Entra ID. In such scenarios, a common anti-pattern arises where client apps are unnecessarily made to request high-privileged scopes to graph resources to bring conditional access policy evaluation into the picture. This is done because low-scope requests to graph, like User.Read, are excluded during policy evaluation. You can find more detailss [here](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-cloud-apps#all-cloud-apps) .

This approach is not recommended. A better choice, as illustrated in Example Scenario 3, involves using a dummy service. This approach ensures that tokens issued to the dummy service cannot be exploited by malicious actors or the app itself since there is no actual API implementation and hence the token is useless.

## Conclusion

- Conditional Access Policy is not set directly on a client (public/native) application.
- Conditional Access Policy is designed to protect the service, meaning it is applied when a client calls a service.
- If there is no dependency on a service in the client app, introduce a service dependency for the app and apply the policy to that service.
- If for some reason the mobile app cannot have any dependencies on services protected under Entra ID, then you cannot leverage the Conditional Access Policy feature. In such cases, you will have to rely solely on app protection policy features.
