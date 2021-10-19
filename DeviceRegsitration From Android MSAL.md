# How to trigger device registration from android app via MSAL library.

In order to achieve SSO with work account between both 1st and 3rd party apps, [Workplace join](https://apac01.safelinks.protection.outlook.com/?url=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fwindows-server%2Fidentity%2Fad-fs%2Foperations%2Fjoin-to-workplace-from-any-device-for-sso-and-seamless-second-factor-authentication-across-company-applications&data=04%7C01%7Cpramkum%40microsoft.com%7C9c91a379bc9640e699fd08d98ccdbf74%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637695637643014957%7CUnknown%7CTWFpbGZsb3d8eyJWIjoiMC4wLjAwMDAiLCJQIjoiV2luMzIiLCJBTiI6Ik1haWwiLCJXVCI6Mn0%3D%7C1000&sdata=HLlyCLrBtlLAKqyhVt%2B1BZV30BPIQl%2FKamnBh%2F2x1z0%3D&reserved=0) is required. That means, device must be registered with Azure AD.

The simplest way to register the device in android device is through either Authenticator app or Company portal app. However, there might a scenario where you want to trigger the device registration process from your android app using the MSAL library. You could follow the below instructions to achieve it. 

Declare a variable of type [AcquireTokenParameters](https://github.com/AzureAD/microsoft-authentication-library-for-android/blob/dev/msal/src/main/java/com/microsoft/identity/client/AcquireTokenParameters.java) class 

	private AcquireTokenParameters mParameters;

Create an object of [ClaimsRequest](https://github.com/AzureAD/microsoft-authentication-library-for-android/blob/bc3d3012d6c0f311cbfec9c0bb08e00eabacac3f/msal/src/main/java/com/microsoft/identity/client/claims/ClaimsRequest.java#L38) & [RequestedClaimAdditionaInformation](https://github.com/AzureAD/microsoft-authentication-library-for-android/blob/bc3d3012d6c0f311cbfec9c0bb08e00eabacac3f/msal/src/main/java/com/microsoft/identity/client/claims/RequestedClaimAdditionalInformation.java#L35) classes and use the below code to setup the ClaimsRequest object to request for “deviceid” claim. This object is used as a parameter later acquiring the token. 

	ClaimsRequest deviceIdClaimsRequest = new ClaimsRequest();
	RequestedClaimAdditionalInformation deviceIdAdditionalInfo =
						    new RequestedClaimAdditionalInformation();

	deviceIdAdditionalInfo.setEssential(true);
	deviceIdClaimsRequest.requestClaimInAccessToken("deviceid", deviceIdAdditionalInfo);

Now build the AcquireTokenParameters object using the builder pattern as below. Plea

	mParameters = new AcquireTokenParameters.Builder()
		.startAuthorizationFromActivity(getActivity())
		.fromAuthority(mAccount.getAuthority())
		.forAccount(mAccount)
		.withClaims(deviceIdClaimsRequest)
		.withScopes(Arrays.asList(getScopes()))
		.withCallback(getAuthInteractiveCallback())
		.build();

Now pass the AcquireTokenParameters object to [AcquireToken](https://github.com/AzureAD/microsoft-authentication-library-for-android/blob/7d912a44870301f0d8aa6a7627e747c4a1825879/msal/src/main/java/com/microsoft/identity/client/SingleAccountPublicClientApplication.java#L581) method of [SingleAccountPublicClientApplication](https://github.com/AzureAD/microsoft-authentication-library-for-android/blob/7d912a44870301f0d8aa6a7627e747c4a1825879/msal/src/main/java/com/microsoft/identity/client/SingleAccountPublicClientApplication.java#L581) class. 

	     mSingleAccountApp.acquireToken(mParameters);

If the device is not registered in the Azure AD, then it should launch below popup through which user should be able to register the device. If the device is already registered, you will not see the below popup. 

 
![image](https://user-images.githubusercontent.com/62542910/137879022-2f762177-f8e7-4980-910b-a97d47a7aa10.png)

## Note : 

1.	Currently, signIn method of [SingleAccountPublicClientApplication]( https://github.com/AzureAD/microsoft-authentication-library-for-android/blob/7d912a44870301f0d8aa6a7627e747c4a1825879/msal/src/main/java/com/microsoft/identity/client/SingleAccountPublicClientApplication.java#L581) class doesn’t expose any overload methods to pass AcquireTokenParameters. It has to be done while requesting the token.
2.	You could implement the similar approach in [AcquireTokenSilent]( https://github.com/AzureAD/microsoft-authentication-library-for-android/blob/7d912a44870301f0d8aa6a7627e747c4a1825879/msal/src/main/java/com/microsoft/identity/client/SingleAccountPublicClientApplication.java#L665) by passing the object of [AcquireTokenSilentParameters]( https://github.com/AzureAD/microsoft-authentication-library-for-android/blob/7d912a44870301f0d8aa6a7627e747c4a1825879/msal/src/main/java/com/microsoft/identity/client/AcquireTokenSilentParameters.java#L25).If the device is not registered, then it throws an MSALUIRequiredException with error code AADSTS50187.
3.	If you are using the [MultipleAccountPublicClientApplication]( https://github.com/AzureAD/microsoft-authentication-library-for-android/blob/7d912a44870301f0d8aa6a7627e747c4a1825879/msal/src/main/java/com/microsoft/identity/client/MultipleAccountPublicClientApplication.java), then this flow cannot be triggered. 

