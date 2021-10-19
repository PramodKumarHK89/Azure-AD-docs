**How to trigger device registration from android app via MSAL library. **

In order to achieve SSO with work account between both 1st and 3rd party apps, Workplace join is required. That means, device must be registered with Azure AD.

The simplest way to register the device in android device is through either Authenticator app or Company portal app. However, there might a scenario where you want to trigger the device registration process from your android app using the MSAL library. You could follow the below instructions to achieve it. 

Declare a variable of type AcquireTokenParameters class 

	private AcquireTokenParameters mParameters;

Create an object of ClaimsRequest & RequestedClaimAdditionaInformation classes and use the below code to setup the ClaimsRequest object to request for “deviceid” claim. This object is used as a parameter later acquiring the token. 

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

Now pass the AcquireTokenParameters object to AcquireToken method of SingleAccountPublicClientApplication class. 

	     mSingleAccountApp.acquireToken(mParameters);

If the device is not registered in the Azure AD, then it should launch below popup through which user should be able to register the device. If the device is already registered, you will not see the below popup. 

 
![image](https://user-images.githubusercontent.com/62542910/137879022-2f762177-f8e7-4980-910b-a97d47a7aa10.png)


