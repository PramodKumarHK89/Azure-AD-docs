# Embedding extenstion attribute as a claim in the token. 

## Objectives of this chapter

Take a scenario where user has a custom unique attribute value which is not part of the regular claim attributes in Azure AD, and yet applications which is integrated with Azure Ad has some business logic written around it. This article describes the step-step procedure which need to be followed to get a custom attribute as part of a claim within id and access token in Azure Ad integrated applications. If you follow the below instructions, the custom attribute can be made available to all the applications within a tenant. 

## Scenario

For the sake of better understanding, consider a requirement where org has bunch of applications which has some UI customization based on whether user is left-handed or right-handed. 
- Since there is no such readymade attribute in Azure AD against the user object, we have to create an extension attribute to store each individual Handedness information. 
- We also want to make sure that, every application that is integrated with Azure Ad gets the Handedness information of the user as part of the claim in the token. 

## Implementation

## Step 1 : Create a dummy app registration against which directory extension attribute can be created.

Extension attributes need to be associated with an App object in Azure AD. That is the reason why, we have to create an app registration which can act like a container for the extension properties. Below are the steps to create app registration 
- Sign in to your Azure Account through the [Azure portal](https://portal.azure.com/).
- Select Azure Active Directory.
- Select App registrations.
- Select New registration.
- Name the application. 

  ![image](https://user-images.githubusercontent.com/62542910/160067195-0ed8a7c1-111a-4b6e-822e-ed2f4e8eeb16.png)

  ### Please refer the instructions [Register an application](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app#register- an-application) for detailed information on the process.

## Step 2 : Create directory extension attribute linked to the app regsitration object.
- Open [Graph Explorer](https://developer.microsoft.com/). Alternatively, you could use custom app or powershell to make below GRAPH API calls.However, the instructions in this article is contextual to graph explorer. 
- Run the POST call to below graph API 
  ```
  https://graph.microsoft.com/v1.0/applications/<app's ObjectId>/extensionProperties  
  ```
  with the request body 
  
  ```
  {
    "name": "someExtension",
    "dataType": "string",
    "targetObjects": [
     "User"
    ]
  }
  ```
- The app object ID canfor the previous step can be found in the app registration blade. Below is the screenshot for reference.
  
    ![image](https://user-images.githubusercontent.com/62542910/160069852-ae7ce23d-268f-4953-8709-a37dd2bf09af.png)

-  Below is the screenshot from graph explorer while invoking the POST call for reference. As you can see,the response of POST call has the extension attribute `extension_28abe44bbf2d462bb7ca32e16902b5f1_HandednessExt`. Please do note that, GUID embedded within the extension attribute is updated as app ID even though we used the "app object Id" while making the Graph Http request. 

    ![image](https://user-images.githubusercontent.com/62542910/160070920-349fd3e7-74d4-4410-8701-23f9c4505de3.png)


## Step 3 : Update the user information in the extension property. 
- For the demostration purpose, I'll be taking two users here. James & David. James is left hand and David is right Hand. 
- Open [Graph Explorer](https://developer.microsoft.com/). Alternatively, you could use custom app or powershell to make below GRAPH API calls.However, the instructions in this article is contextual to graph explorer. 
- Run the PATCH call to below graph API to update James extension attribute value to `LeftHand`
  ```
  https://graph.microsoft.com/v1.0/users/<userid | username>
  ```
  
  with the request body 
  
  ```
  {
    "extension_28abe44bbf2d462bb7ca32e16902b5f1_HandednessExt": "LeftHand"
  }
  ```
- Below is the screenshot from graph explorer for reference.
  
    ![image](https://user-images.githubusercontent.com/62542910/160072406-8531bd6e-f35d-47d2-bfaf-fc7c2d9c76fa.png)

- Repeat the same PATCH call to graph API to update David extension attribute value to `RightHand`. Below is the screenshot from graph explorer for reference.
    
    ![image](https://user-images.githubusercontent.com/62542910/160072811-4b9f3008-28ce-4479-be3d-f10287f782c4.png)

- Validate whether the user extension attribute value is updated correctly against the user object by sending GET command as below
  
  ```
  https://graph.microsoft.com/v1.0/users/<userid | username>?$select=extension_28abe44bbf2d462bb7ca32e16902b5f1_HandednessExt
  ```
  Below is the screenshot for reference for David. 
  
  ![image](https://user-images.githubusercontent.com/62542910/160073398-7ac27e18-3faa-4957-8379-7881e3c80976.png)

  
## Step 4 : Create a claim mapping policy

To make the extension attribute be available as a claim to all the applications in the tenant, a [claims mapping policy](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-claims-mapping) must be used to map the extension attribute to the claim. Follow the below step to perform this operation via powershell. 

-	Open the powershell and run below commands. You may have to punch in the admin creds when prompted.
  ```
  New-AzureADPolicy -Definition @(‘{"ClaimsMappingPolicy": {"Version": 1,"IncludeBasicClaimSet": "false","ClaimsSchema": [{"Source": "User","ExtensionID": "extension_28abe44bbf2d462bb7ca32e16902b5f1_HandednessExt","JWTClaimType":"UserHandedness"},]}}’) -DisplayName UserHandednessclaimsMappingPolicy -Type "ClaimsMappingPolicy" 
  ```
-	Refer the below screenshot for reference

    ![image](https://user-images.githubusercontent.com/62542910/160075764-d35d6eee-3d22-47b0-9fa2-a5c70b91a825.png)

# The above steps(Step 1 through to step 4) is one time effort to setup the extension cliam attributes. Further steps in the article is needed for each application which needs this extension attribute as claim in the token. 

## Step 5 : Map the claim mapping policy to service principals of the applications where claim is needed as part of the token.

- For the sake of better understanding, let's assume that, we have 3 apps - AppA, AppB & AppC in which we need to get the claim as part of the token. 

- Run the below command to map the claim mapping policy to service principal objects of AppA & AppB (Do not add it to AppC. Reasoning behind this will be clarified later in the article). 

  ```
  Add-AzureADServicePrincipalPolicy -Id SP_Object_ID  -RefObjectId Policy_Object_ID
  ```
- Refer the below screenshots where the mapping is done for the two applications(SP), AppA & AppB. These two applications are assumed to be present in the tenant.If not, you follow the create app registration step to create these two apps. 
  
    ![image](https://user-images.githubusercontent.com/62542910/160083688-b0ebb3b3-f6d5-4af1-ab04-ef86bb99407a.png)
    
- Service Principal ID can be obtained from Enterprise blade in Azure Ad portal.

  ![image](https://user-images.githubusercontent.com/62542910/160083848-cf64ae32-3dee-431a-9f41-30fc3f9c432d.png)
  ![image](https://user-images.githubusercontent.com/62542910/160083871-c76c6ef2-1594-4668-b582-3c419f0c32d6.png)

## Step 6 : Edit the manifest of the application to accept the claim mapping in the token
- If you try to get the token from AppA or AppB either through postman or sample code, you will get below exception. 
  
  ![image](https://user-images.githubusercontent.com/62542910/160085298-05c8ee82-834b-47a9-9b92-296ece630078.png)
- To resolve the above exception, go to the manifest of App A and App B and then change the value of `acceptMappedClaims` to `true`from `null`. Refer the below screenshot for App A. 
  
  ![image](https://user-images.githubusercontent.com/62542910/160085729-38877f42-f9a4-4dbe-b175-8dccd588fc03.png)

- Refer the [link](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-claims-mapping#update-the-application-manifest) for more details on the manifest change.

## Step 7 : Test the application to check if the claim is part of the id token. 
- There are two methods to test whether we canget claim as part of id token. You could use either of this as per your convenience. 
  1) Postman 
  2) Sample code. 

  ### 1)PostMan

   -   For the simplicity purpose, let's initiate [ROPC flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth-ropc) to complete our test. Please do note that, Microsoft recommends you do not use the ROPC flow. In most scenarios, more secure alternatives are available and recommended. This flow requires a very high degree of trust in the application, and carries risks which are not present in other flows. We are using this flow for testing purpose. 
   -   By default, ROPC flow is not enabled in the app registration. You have to explicitly enable it in the Authentication blade as shown in the below screenshot
      
     ![image](https://user-images.githubusercontent.com/62542910/167793493-cd9cea02-f559-48f5-a036-4454c6934bbf.png)
   
   - Please refer the below screenshot where ROPC request has been sent from postman to retrieve id token. `openid` scope is the parameter that triggers the Azure AD to send id_token as response .
    
    ![image](https://user-images.githubusercontent.com/62542910/167794713-a9b63624-104a-4e5e-9114-9a08747cd6c3.png)

  - Copy the id token in https://jwt.ms/ and verify that claim is recieved in the token. Please refer the below screenshot.
  
    ![image](https://user-images.githubusercontent.com/62542910/160094301-028e6dc1-69ae-4ed0-a7de-2f646e062949.png)
    
  ### 2) Sample Code
   -   If you do not have the sample code for the app, then perhaps you can make use of the quick start blade of app registration blade. 
   -   Visit the quick start blade and select the platform in which you need sample application. For example, in the below screenshot. Select ASP.NET core template for this example.
   
     ![image](https://user-images.githubusercontent.com/62542910/160089351-73e561e6-2c97-4af1-9a35-dd821482e17e.png)

   - Follow the instructions in the Azure portal to setup the sample project. And then download the zip file. 
   
      ![image](https://user-images.githubusercontent.com/62542910/160089623-374098c5-6bc0-45fb-95c4-23c3228e123c.png)
    
   - Extract the zip file and open it in VS and run the app. 
   - When the application is launched, it will prompt for the user creds. Punch in the creds of either James or David. In the below screenshot, you could see        the James account being used.

      ![image](https://user-images.githubusercontent.com/62542910/160090087-67e7721d-2843-4839-a7e9-91e2611db683.png)

   - After successful authetication, trap the network trace via F12 developer tools option and grab the id token. Please refer the below screenshot for       reference. You could also different mechanism to inspect the id token recieved. 

    ![image](https://user-images.githubusercontent.com/62542910/160090972-de421448-333a-4082-8dff-302de190501e.png)
  
  - Copy the id token in https://jwt.ms/ and verify that claim is recieved in the token. Please refer the below screenshot.
  
    ![image](https://user-images.githubusercontent.com/62542910/160094301-028e6dc1-69ae-4ed0-a7de-2f646e062949.png)
 

- Verify the claim being sent in App B as well for the user david by following the same steps as above  

## Step 8 : Test the application to check if the claim is part of the access token. 

- Let us try to get an access token from client app(AppC) which can be used to access the API(Either AppA or AppB)
- To get access token which is intended for resource AppA from AppC, first AppA has to expose an API. Refer below screenshot on how to do it.
  
  ![image](https://user-images.githubusercontent.com/62542910/167796335-1a644504-7784-41ae-a347-320c5f7a7da0.png)
- Now visit AppC app registration in the API permission blade to add the scope for App A. AppA API will be visible under My API section. 
  
  ![image](https://user-images.githubusercontent.com/62542910/167796677-bb0948bd-2df7-4b81-9331-6bf3f8f8bf81.png)
 
- There are two methods to test whether we can get claim as part of access token. You could use either of this as per your convenience.
  1)Postman 
  2)Sample code.  

  ### 1)PostMan

   -   For the simplicity purpose, let's initiate [ROPC flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth-ropc) to complete our test. Please do note that, Microsoft recommends you do not use the ROPC flow. In most scenarios, more secure alternatives are available and recommended. This flow requires a very high degree of trust in the application, and carries risks which are not present in other flows. We are using this flow for testing purpose. 
   -   By default, ROPC flow is not enabled in the app registration. You have to explicitly enable it in the Authentication blade as shown in the below screenshot
      
   ![image](https://user-images.githubusercontent.com/62542910/167793493-cd9cea02-f559-48f5-a036-4454c6934bbf.png)
   
   - Please refer the below screenshot where ROPC request has been sent from postman to retrieve access token. `https://pramkumlab.onmicrosoft.com/user_access` scope sends the access token which is intended for App A.
    
   ![image](https://user-images.githubusercontent.com/62542910/167797620-62862ff1-1c96-47b6-a838-0fe86422cfe5.png)


  - Copy the access token in https://jwt.ms/ and verify that claim is recieved in the token. Please refer the below screenshot. 
  
    ![image](https://user-images.githubusercontent.com/62542910/167798298-41a22cf7-e39c-4817-b6c0-cd87904313e4.png)
    
  ### 2) Sample Code
   -   If you do not have the sample code for the app, then please refer our sample code at https://github.com/Azure-Samples/ms-identity-javascript-angular-tutorial/tree/main/3-Authorization-II/1-call-api
   -   Follow the step-step instructions from the readme of the sample code. Trap the network trace via F12 developer tools option and grab the access token which will be sent as bearer token while invoking the TodoList method. Copy the access token in https://jwt.ms/  to verify the custom claim in the token. 

## References 
- https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-schema-extensions
- https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-claims-mapping
- https://winsmarts.com/using-token-configuration-to-include-arbitrary-claims-in-id-token-or-access-token-or-samltoken-26f75ee13bf0

