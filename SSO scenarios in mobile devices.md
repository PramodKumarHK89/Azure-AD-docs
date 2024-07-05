****Needs update and not the latest one ***
# Scenario 1: 
## Jane wants to build a custom mobile app which authenticates the user with Azure AD. Below are the requirements from the management.

### Requirement:
- Application must share SSO with apps like Outlook, teams etc. on the device. 
- Admin must be able to apply CA policies such as app is used only on the Azure ad compliant devices. 
- Organization has Intune license and admin should be able to consume the features offered by Intune app protection policies. 

### Workflow:
Jane can integrate the Intune SDK in their custom app which internally uses the MSAL. Now custom app can mandate the broker on the device to successfully login to the user.  Here is how the workflow will look like.  
- When the user first installs the custom app and launch it in their device, it checks if broker is already installed on the device. It is either company portal or authenticator app in case of android(Whichever installed first will act as an active broker) and authenticator app in case of IOS. 
- If the broker is not installed, app can redirect the user to install it from the respective stores. This is a decision that app can make. 
- After installation, during the first login attempt, broker will register the device with Intune and Azure AD. This information is used for applying CA policies and features of Intune on subsequent login attempts of the apps running on the device. 
  - Once the broker is installed, SSO achieved across first party apps + apps which use broker and your custom app. 
  - Require device to be compliant and require app protection policy under the CA policy works as expected. Note: Require approved client app is applicable only for first party apps. 
  - Intune admin can apply the app protection policies as expected on the custom app. 
- Suppose if broker is uninstalled on the device or app can’t use it for some reason, then it can decide whether to redirect the user back to respective stores to install the broker or it can simply continue with a fallback mechanism to use system browser to authenticate with the azure ad. 
  - In this case, if other first party apps follow the same mechanism, the SSO across apps achieved as system browser can share the cookie/jar details. 
  - With no broker state, if app continues with fallback mechanism, the device to be compliant CA policy still will work as system browser can access the device ID information from the installed certificate(During the first broker authetication)
  - However, with no broker,  require app protection policy under the CA will not work as expected since the authetication request goes via system browser and not through the broker. 
  - With no broker, custom app doesn’t run under the governance of app protection policy. 

# Scenario 2:
## Jane wants to build a custom mobile app which authenticates the user with Azure AD. Below are the requirements from the management.

### Requirement:
- Application must share SSO with apps like Outlook, teams etc. on the device. 
- Admin must be able to apply CA policies such as app is used only on the Azure ad compliant devices. 
- Organization has NO Intune license.

### Workflow:
Jane can integrate the MSAL in their custom app which can be configured to use the go with brokered auth. This is how the workflow look like. 
- When the user first installs the custom app and launch it in their device, it checks if broker is already installed on the device. It is either company portal or authenticator app in case of android and authenticator app in case of IOS. 
- If the broker is present on the machine, custom app will use the broker to complete the authetication request with azure ad. 
  - SSO achieved across first party apps + other apps which use broker and your custom app. 
  - Require device to be compliant under the CA policy works as expected. Note: Require approved client app is applicable only for first party apps and require app protection policy will not work since there is no integration with Intune SDK. 
  - Since no Intune integration, there is no question of app protection policies. 
- Suppose if broker is not present, app can use the fallback mechanism to use system browser to authenticate with the azure ad.
  - In this case, if other first party apps follow the same mechanism, the SSO across the custom app and other apps still achieved as system browser can share the cookie/jar details across apps. 
  - With no broker state, if app continues with fallback mechanism, the device to be compliant CA policy still will work as system browser can access the device ID information from the installed certificate(During the first broker authetication)
	- Neither require app protection policy under the CA nor the ap protection policies in Intune are applicable. 

# Scenario 3:
## Jane wants to build a custom mobile app which authenticates the user with Azure AD. Below are the requirements from the management.

### Requirement:
•	Application must share SSO with apps like Outlook, teams etc. on the device. 
•	Admin must be able to apply CA policies such as app is used only on the Azure ad compliant devices. 
•	Organization has NO Intune license.
•	Devices will NOT have broker installed. 

### Workflow:
Jane can integrate the MSAL in their custom app which can be configured to use system browser as devices are not supposed to have broker. This is how the workflow look like. 
- App uses system browser to authenticate with azure ad. 
  - SSO achieved across apps which use system browser to authenticate and your custom app. 
  - Require device to be compliant under the CA policy works as expected. Note: Require approved client app is applicable only for first party apps and require app protection policy will not work since there is no integration with Intune SDK. 
  - Since no Intune integration, there is no question of app protection policies. 
