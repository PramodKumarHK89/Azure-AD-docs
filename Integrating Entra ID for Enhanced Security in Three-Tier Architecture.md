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
![image](https://github.com/user-attachments/assets/744a3e56-a0db-48f6-9297-fc5d168c0b94)
### UI Layer (Client App):
- Regardless of the platform type (web, mobile, or desktop), the app can use the MSAL library to authenticate the user by requesting an ID token to identify the user.
- The prerequisite is that you need one app registration in Entra ID for the client app and must configure the redirect URI depending on the client app's platform type.
- As part of the sign-in process, the user is redirected to the Entra ID sign-in page. Once the user completes the authentication process (including MFA if configured), the client app allows the user to log in to the app. (Transaction 1 in the diagram).
- The app then needs to invoke the main service API to retrieve or update data for its functionality.
- To secure this transaction, there will be another app registration for the main service API, which exposes the APIs. The client app requests an access token for that API from Entra ID on behalf of the user. This process can also be combined with the sign-in request through hybrid flow, where the initial sign-in request asks for both the ID token (for the client app to identify the user) and the access token (which can be used as authorization bearer while invoking the main service API).
- The client app invokes the main service API by attaching the access token in the authorization bearer header (Transaction 2 in the diagram).
### API Layer (Main Service):
- The main service API should accept the HTTPS call (Transaction 2) from the client app and validate the authorization by decoding the access token received through the authorization bearer header.
- As mentioned earlier, the main service API will have its own app registration in Entra ID. It must download the keys from Entra ID, cache them for 24 hours (Transaction 3), and validate the signature, audience, issuer, and app-specific roles/claims to allow only authorized users and app requests.
- If the above validation fails, Transaction 2 will be denied with a 401 (Unauthorized) status.
- After successful validation of the user and app, the main API service is ready to perform CRUD operations on the database. There are two approaches here:
- #### Variation 1: OBO Flow
  - The main API service should request an access token for the data layer in exchange for the access token it received through Transaction 1. This process is known as the OBO (On-Behalf-Of) flow (Transaction 4 in the diagram).
  - For practical implementation, let's assume the data layer is Azure SQL. The main API service requests a token for the Azure SQL delegated permission through the OBO flow and then attaches it to the authorization bearer header while invoking Azure SQL methods (Transaction 5 in the diagram).
  - The advantage of this approach is that the user context persisted throughout the entire transaction from the client app to the data layer.
- #### Variation 2: Managed Identity Flow
  - Managed identity is set up between the main service API and the data layer. For example, if the API is an Azure function and the data layer is Azure SQL, the system-assigned identity is enabled for the function app, and Azure SQL RBAC is configured to establish trust with the system-assigned identity (Transaction 7 in the diagram).
  - In this approach, the user context is lost in the transaction, and the data layer solely trusts that the main API service is allowing only authorized users to invoke the operation as intended.
  - In this approach, it is important to ensure that the database layer is accessed exclusively through the main API service and not from any other services. Failure to do so, may lead to improper handling of authorization violating the least privilege principle.
- ### Data Layer (DB):
  - In the case of Variation 1, the database layer downloads the keys from Entra ID, caches them for 24 hours (Transaction 6), and validates the signature, audience, issuer, and app-specific roles/claims to allow only authorized users and app requests.
  - In the case of Variation 2, the database layer relies on the Azure managed identity implementation and validates the RBAC configured between the database layer and the main API service (no validation on the user).(Transaction 7)
## Example Scenario 2: A client app, multiple microservices, and a database.
This scenario is very similar to Example Scenario 1. However, instead of a single monolithic main service, the architecture involves multiple microservices that are invoked by the client app.
![image](https://github.com/user-attachments/assets/dba3ff8d-037e-45d4-8b60-62c5604f3651)
### UI Layer (Client App)
- Regardless of the platform type (web, mobile, or desktop), the app can use the MSAL library to authenticate the user by requesting an ID token to identify the user.
- The prerequisite is that you need one app registration in Entra ID for the client app and must configure the redirect URI depending on the client app's platform type.
- As part of the sign-in process, the user is redirected to the Entra ID sign-in page. Once the user completes the authentication process (including MFA if configured), the client app allows the user to log in to the app (Transaction 1 in the diagram).
- The app then needs to invoke multiple microservices to retrieve or update data for its functionality.
- To secure this transaction, there will be a separate app registration for each of the microservices, which exposes the APIs. The client app requests an access token for each of the microservices on behalf of the user.
- The client app invokes the microservice API by attaching its associated access token in the authorization bearer header (Transaction 2 in the diagram).
- Even though the client app requests access tokens for different microservices from Entra ID, if consent is set up, SSO should kick in, and token acquisition happens silently in the background, ensuring no compromise on user experience.
### API Layer (Micro services)
- Each micro service should accept the HTTPS call (Transaction 2) from the client app and validate the authorization by decoding the access token received through the authorization bearer header.
- As mentioned earlier, each micro service API will have its own app registration. It must download the keys from Entra ID, cache them for 24 hours (Transaction 3), and validate the signature, audience, issuer, and app-specific roles/claims to allow only authorized users and app requests.
- If the above validation fails, Transaction 2 will be denied with a 401 (Unauthorized) status.
- After successful validation of the user and app, the main API service is ready to perform CRUD operations on the database. There are two approaches here:
- #### Variation 1: OBO Flow
  - Each micro-API service should request an access token for the data layer in exchange for the access token it received through Transaction 1. This process is known as the OBO (On-Behalf-Of) flow (Transaction 4 in the diagram where Micro Service 1 is obtaining an access token via OBO flow).
  - For practical implementation, let's assume the data layer is Azure SQL. The micro service requests a token for the Azure SQL delegated permission through the OBO flow and then attaches it to the authorization bearer header while invoking Azure SQL methods (Transaction 5 in the diagram).
  - The advantage of this approach is that the user context persisted throughout the entire transaction from the client app to the data layer.
- #### Variation 2: Managed Identity Flow
  - A dedicated microservice is configured to communicate with the database layer, and a managed identity is set up between that particular microservice and the data layer. For example, if the microservice is an Azure function and the data layer is Azure SQL, the system-assigned identity is enabled for the function app, and Azure SQL RBAC is configured to establish trust with the system-assigned identity (Transaction 7 in the diagram, where Microservice 4 exclusively communicates with the database layer through managed identity).
  - Any microservice that needs to interact with the database layer will go through the dedicated microservice (Microservice 4). For this invocation, the OBO flow should be used from the invoking microservice to the dedicated microservice (Transaction 8 in the diagram).
  - In this approach, the user context is lost in the transaction, and the data layer solely trusts that the dedicated microservice (Microservice 4) is allowing only authorized users to invoke the operation as intended.
  - It is crucial to ensure that the database layer is accessed exclusively through the dedicated microservice and not from any other services. Failure to do so may lead to improper handling of authorization violating the least privilege principle. 
### Data Layer (DB)
•	Same as example scenario 1
## Example Scenario 3: A client app, an orchestrator service, multiple microservices, and a database.
Similar to the APIM implementation.
![image](https://github.com/user-attachments/assets/0af021b0-4a13-4928-ad96-de71c6e0ecc3)
### UI Layer (Client App)
- Same as example scenario 1
### API Layer (Main Service & Micro services)
- The main service API should accept the HTTPS call (Transaction 2) from the client app and validate the authorization by decoding the access token received through the authorization bearer header.
- As mentioned earlier, the main service API will have its own app registration in Entra ID. It must download the keys from Entra ID, cache them for 24 hours (Transaction 3), and validate the signature, audience, issuer, and app-specific roles/claims to allow only authorized users and app requests.
- If the above validation fails, Transaction 2 will be denied with a 401 (Unauthorized) status.
- After successful validation of the user and app, the main API service is ready to perform rest of the operations. There are couple of approaches here:
- #### Variation 1 : Separate app registration for each of the micro services
  - Each microservice API will have its own app registration. The main orchestration service requests an access token from Entra ID for each of the microservices through the OBO flow (Transaction 4).
  - Once the token is obtained through OBO flow, the main service invokes the respective microservices by attaching the newly obtained access token in the authorization bearer header.
  - Each microservice must download the keys from Entra ID, cache them for 24 hours (Transaction 6), and validate the signature, audience, issuer, and app-specific roles/claims to allow only authorized users and app requests.
- #### Variation 2 : Single API App Registration for main service
  - From an administrative perspective, it might sound easier that, you have a single app registration for the main service and then, depending on the scope claim of the token, the main service decides which user has access to which microservice. While this approach may seem reasonable, it exposes the communication between the main service and the microservices. There are sub-variations to consider:
  - Sub Variation 1: The access token which is received by the main service from client app is passed along with the call to the microservice, and each microservice uses the same app registration to validate the token. If this approach is followed, the microservices can use the access token to perform the OBO flow for further processing with the database layer. (User context is lost) However, the access token is passed along over the wire twice in the authorization header and there token validation process must be repeated.
  - Sub Variation 2: Establish exclusive mutual communication between the main service and the microservices using a mechanism outside of Entra ID and drop the access token in the main service. In this case, each microservice does not need its own app registration, and the user context will be completely lost.
  - For this variation, an alternative is to use an API Management (APIM) solution, which often handles the operations mentioned above and provides additional management and security features.
### Data Layer (DB)
- Same as scenario 2 

## Conclusion
In a complex multi-layer architecture, introducing app-only authorization (such as managed identity or client credential flow) while dropping the user context along the transactions may seem easier and more convenient, and it might also simplify the overall architecture. However, maintaining the user context throughout the transaction provides stronger security and granular level of authorization handling. By investing in separate app registrations and proper validation, you ensure compliance with the least privilege model and secure your application better. 
