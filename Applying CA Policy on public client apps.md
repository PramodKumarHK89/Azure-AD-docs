# Digging deeper into the enforcement of Entra ID Conditional Access Policy feature on a native mobile application.

## Abstract : 

Accessing enterprise apps from mobile devices has become the new norm. Many organizations now allow employees to access corporate apps through their personal mobile devices. This boosts productivity and provides flexibility for employees to do their jobs seamlessly. However, at the same time, it's crucial to protect these apps to prevent any security breaches and protect organization data.

Organizations commonly use the Intune MDM suite to protect devices as a whole. Devices managed through Intune MDM solutions are termed managed devices, while those not managed by any solution are called unmanaged devices. Most organizations prefer managed devices, but unmanaged devices scenario also exist where usage MDM management on a personal device is not desired for various reasons. Nevertheless, in both cases, it's essential to protect corporate apps and enforce security measures if you are letting the users/apps to access Corp data.

To protect apps running on both managed and unmanaged mobile devices, app protection policies[MAM] are typically used in combination with the Entra ID Conditional Access Policy feature. For example, for managed devices, you may apply a policy that allows sign-ins only from compliant devices. For unmanaged devices, you may apply a policy that ensures app protection policies are applied during the sign-in and block if it is not in place. 

These are a couple of commonly used examples, but there are multiple other scenarios. You can explore more on conditional access policy feature at https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-policies

This article sheds light on how an admin can enhance security of native mobile apps through the Conditional Access Policy feature of Entra ID. 

## Technical Components Involved:

- Entra ID Conditional Access Policy Feature: Ensures that access to corporate resources is controlled based on specific conditions.
- MSAL Library: Used to implement the authentication and authorization of the native app.
- Intune SDK: Used to protect the native app with Intune app protection policy (optional).


## Example Scenario 1 : Blocking Sign-In for Outlook Client from a Non-Compliant Devices

Accessing email clients or using Teams to manage emails and chats respectively from a mobile device is a very common requirement in every organization. While it boosts productivity to allow employees to access these apps from their mobile devices, it is crucial to add security measures to protect organizational data.

Let us elaborate on the Outlook client scenario and see how we can apply conditional access policy on the apps. The Entra ID Conditional Access Policy feature facilitates this scenario and the link to the guide is here https://learn.microsoft.com/en-us/entra/identity/conditional-access/howto-conditional-access-policy-compliant-device-admin	

Although covering step-by-step implementation of this feature is beyond the scope of this article, it is important to highlight a key aspect when defining the Conditional Access Policy. You do not have the option to choose Outlook as a client under the Target Resources option. Instead, you will protect the resource, Office Exchange Online. This means you are protecting the resource, not the Outlook app as a client. Therefore, whether you use Outlook Lite, the full Outlook client app, or even access it from a browser, the policy still applies.

CA policies are primarily designed to secure target resources. In other words, the policy is not set directly on a client (public/native) application but is applied when a client calls a service/resource. For example, a policy set on the SharePoint service applies to all clients calling SharePoint, and a policy set on Exchange applies to attempts to access email using the Outlook client. This is why client (public/native) applications are not available for selection in the Cloud Apps picker, and the Conditional Access option is not available in the application settings for these applications registered in your tenant.

For more details, refer to Cloud apps, actions, and authentication context in Conditional Access policy - Microsoft Entra ID | Microsoft Learn.

Many admins/developers often misinterpret the above design as a limitation of Conditional Access Policy, complaining that it cannot apply the CA policy to a public client. In reality, there is no need to enforce the Conditional Access Policy on the client apps as client apps will always need some service in backend to provide the corp data which can be protected through the conditional access policy. If you want to protect specifically the client apps, then app protection policy is the right choice.

## Example Scenario 2: Blocking Sign-In for **CustomClient** App from Non-Compliant Devices

et us assume that an organization has a mobile app called **CustomClient**, which is an HR-related app for their employees, and this app relies on the HR service **CustomService** that exposes corporate data through an API. As an IT admin, you want to ensure that this app is accessed only through compliant devices. In other words, although the app is available through Google Play or the Apple App Store, it must be accessed from an MDM-enrolled or managed device.

Let us dig a little deeper into the implementation of **CustomClient** and **CustomService**. As mentioned earlier, **CustomService** exposes all the data through an API and is protected by Entra ID. In simple terms, **CustomService** has it’s own app registration, and each of the APIs exposed requires some kind of permission in the token, either through scp or role claim. These permissions are exposed through the "Expose an API" blade of the app registration.

On the other hand, **CustomClient** has a separate app registration, and it has added the necessary permissions to invoke the **CustomService** through the "API Permissions" blade by adding and granting the permissions exposed by **CustomService**. From the code implementation perspective, during the sign-in activity of the **CustomClient** app, it also requests a token for **CustomService** to invoke the API and populate data.

Now, comparing this requirement to the Example Scenario 1 with Outlook Client, the trivial solution here is to apply the CA policy to **CustomService** while defining the CA policy(Target resource here is **CustomService**). This ensures that sign-in fails in the **CustomClient** app if it is accessed from a non-compliant device, since the **CustomClient** app requests a token for **CustomService** during the sign-in flow, which enforces the Conditional Access Policy to be applied on the app.

## Example Scenario 3: Blocking Sign-In for **CustomClient** App Without **CustomService**

This scenario is a variation of Example Scenario 2, where the **CustomClient** app does not have a dependency on **CustomService**, or **CustomService** isn’t protected by Entra ID. As you can imagine, you cannot define the Conditional Access Policy since the **CustomClient** app will not be available while defining the policy under the target resource, as it is a public client app registration. So, the question is, how do you enforce the Conditional Access Policy?

A workaround here is to introduce a new dependency on a dummy service and configure the app to request a token for this service. Here’s how you can implement it:
1.	Create API App Registration: Register a dummy API in Entra AD and expose a scope. This dummy service will act as the target resource for the Conditional Access Policy.
2.	Add/Grant Permissions: In the **CustomClient** app registration, add permissions to the exposed scope of the dummy API. Even though there is no actual API implementation

From code perspective, the **CustomClient** app will request a token for this dummy service during the sign-in, which ensures that the Conditional Access Policy is evaluated against the target resource **CustomService** and hence policy will be enforced. This approach leverages the underlying requirement of Conditional Access Policies to be tied to a resource, thereby indirectly protecting the **CustomClient** app.


## Anti-pattern Alert: Conditional Access Policies in case of All Cloud Apps

Applying Conditional Access Policies to all cloud apps may seem comprehensive, but it will still not enforce the conditional access policy on the client apps which lacks dependencies on protected resources or services under Entra ID. In such scenarios, there is a common anti-pattern arises since cli ent apps unnecessarily request high-privileged scopes for the graph resources , as scope requests to graph like User.Read are excluded during policy evaluation. You can find more details here <LIMK>.

This approach is not recommended. A better choice, as illustrated in Example Scenario 3, involves using a dummy service. This approach ensures that tokens issued to the dummy service cannot be exploited by malicious actors or the app itself since there is no actual API implementation and hence the token is useless.

