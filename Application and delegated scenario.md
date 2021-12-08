# Take a scenario where there are 4 entities and each of them have separate app registration in Azure AD. Client 1, Client 2, APIM.  Below diagram is a rough sketch of a conceptual scenario. 

  ![image](https://user-images.githubusercontent.com/62542910/145208812-11af6447-0981-40fb-97e0-979acbf3f6a7.png)



## Client Layer :
- WebApp(Client 1) where user authenticates against Azure AD and request for an access token for the Web App. Webapp makes the call to Graph API and APIs exposed in the APIM. Since it needs an access token to call these two resources, web app requests for a new access tokens from azure AD via on behalf of flow. 
- Another way of consuming the API exposed in the APIM is from Client 2. This application is assumed to have user & role management built into the system and depending on the access level of the user, app calls the API in the APIM with the help of subscription key(Kind of a secret value that APIM offers to connect to the API’s exposed). Any requests that come though this channel and hits the API, are app request(Since there is no user identifiable information. 

## API layer :
- Under the hoods of APIM, there are function apps which hosts the actual API implementation. Function App APIs are secured via EasyAuth and APIM connects to function app via managed identity(APIM AAD App token represents this flow) to execute the API in the function app. 
- Since there could be two sources of incoming requests(Client 1 & client 2), with user and app context, respectively. APIM discards the user contexts here, and then via managed identity executes the function app APIs. 

## Cons of current model.
- User context is lost while executing the actual function app. So, that may not be ideal especially API method is trying to access other resources like Graph and pass the access token. App token sometimes could become hazardous depending on how sensitive the API call is. 
- The entire responsibility of user authorization is handled by client 2 interface. More prone to fishing attacks, since APIM exposes a channel which doesn’t validate the user context. 
- Managing the secure string is cumbersome process between APIM layer and client 2. Additionally, no user context in the APIM layer, if it gets exposed, app level token is generated and could become costly. 

## Improvements that can be made to the current model.

-	There is no need to get an access token for web app, instead user can authenticate the user via open id connect. Once user authenticates, it can acquire the token for graph and APIM separately. So, concept of user getting the access token for the web app and then webapp internally getting the access token for Graph and APIM via OBO flow can be avoided. 
-	Replace the method of accessing the APIM from client 2 with access token using an interactive auth flow. Hence, there is a consistency in Client 1 and Client 2 which sends the access token in the bearer header to APIM. APIM then validates the token and requests Azure AD for a new token on behalf of user to call the function app API. 
-	Advantage of this approach is obvious – User context is available from API call origin till the end. Thus, more granular level of role-based access can be implemented. 
-	More secure as the token issued to the user context and suppose if function app/APIM layer making graph or other calls, it limits the permission based on the user access level versus the app level which has to be managed more cautiously. 

# This is how the modified architecture would look like. 

![image](https://user-images.githubusercontent.com/62542910/145209837-a9431c37-a29b-42cb-aea7-5fd8627d6338.png)

