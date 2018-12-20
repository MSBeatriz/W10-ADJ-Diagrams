
*Applies to: System Center Configuration Manager (Current Branch)*

Co-managed clients are able to communicate with Configuration Manager site using AAD authentication. 

## Client authentication workflow during client installation

On this example, we are installing Configmgr client via intune on an AD join only Windows 10 client
When Ccmsetup action starts, client will check if there is an azure user session logged on the computer. Client will use this session to get AAD token.

1. Client uses AAD parameters to request AAD token:

    *Retrieved AAD token for AAD device '82XXXXad-f062-XXXX-b062-dbXXXXac04d5' (CCMSetup.log)*

>[!TIPS] 
   >- If WPJ certificate is not found, we do not try to request AAD token (client is not properly AAD joined)
   >- AAD Device token usage available from 1806. You need to update teant settings on upgraded environments. AAD Device token is prefered
   >- Traffic with invalid token will be blocked upfront of the CMG, preventing unnecessary communications to the internal roles

2.	CCM Token Request (Using AAD User token) via CCM_STS channel

-   CMG Gets request 
    
      *Getting CCM Token from https://myCMG.cloudapp.net/CCM_Proxy_ServerAuth/9/CCM_STS (CCMSetup.log)*
    
-   CMG moves request to CMG CPhttps://myMP.Mylab.com/CCM_STS
    
      *(IIS log, %CMGHttpHandler.log)*
    
-   CMG CP transforms CMG client request to MP client request 
    
      *MessageID: b90c8940-b368-4f4a-b22f-a8eaae90acb2 RequestURI: https://myMP.Mylab.com/CCM_STS EndpointName: CCM_STS ResponseHeader: HTTP/1.1 200 Ok  (SMS_CLOUD_PROXYCONNECTOR.log)*
   
-   MP runs request against Database to verify user token (CCM_STS.log)
   
      *Validated AAD token. TokenType: Device TenantId: cXXXXXX2-XXXX-4df9-XXXX-cddXXXXXXe0a UserId: 00000000-0000-0000-0000-000000000000 DeviceId: 82XXXXad-f062-XXXX-b062-dbXXXXac04d5 OnPrem_UserSid:  OnPrem_DeviceSid:*  
 
 ![Data flow diagram for Configuration Manager Client installation using AAD Auth] (/AADInstallWF.png)
 
## Client authentication workflow during client registration

On this example we are checking registration on a Windows 10 Hybrid Client. After installing or when renewal registration task is needed, client will check if there is an availalbe AAD token to use for registration.

1. Client runs registration request agains Management Point:

   *[RegTask] - Executing registration task synchronously.*
   *[RegTask] - Client registration is pending. Server assigned ClientID is GUID: (ClientIDManagerStartup.log)*

> [!Note]
>Checking client certificates task always runs, but if AAD token is obtained we register using AAD Auth

2. AAD Token request (after installation) 

   *Getting AAD token for logged on user. Authority: https://login.microsoftonline.com/7A5AD1D5-XXXX-4428-XXXX-E4C9D9846E7F ClientId: 6a415267-XXXX-4d49-XXXX-4b283d300cf4 ResourceId: https://ConfigMgrService UserSID: S-1-5-21-XXXXXXXXXX-1260630808-XXXXXXXXXX-25583
Attempting to obtain AAD token. WebAccountProviderId='https://login.windows.net', Authority='https://login.microsoftonline.com/7A5AD1D5-XXXX-4428-XXXX-E4C9D9846E7F', ClientID='6a415267-XXXX-4d49-XXXX-4b283d300cf4', ResourceId='https://ConfigMgrService', SessionId='2'
Successfully obtained AAD token with WAM (ADALOperationProvider.log)*

3. CCM Token request via CCM_STS channel

-   Getting CCM Token from STS server 

     *'Getting CCM Token from STS server 'https://myCMG.cloudapp.net/CCM_Proxy_ServerAuth/9’ (CcmMessagin.log)
Getting CCM Token from https://myCMG.cloudapp.net/CCM_Proxy_ServerAuth/9/CCM_STS*

-   Proxy Connector transfer request from CMG to MP

      *Request - MessageID: 68d2628a-4f50-4357-af56-1f0da3a75ae3 RequestURI:https://myCMG.cloudapp.net/CCM_Proxy_ServerAuth/9/CCM_STS
Response - MessageID: 68d2628a-4f50-4357-af56-1f0da3a75ae3 ResponseHeader: HTTP/1.1 200 OK (CMGService.log)
(SMS_CLOUD_PROXYCONNECTOR.log)*

-   Token Validation and reply to client:

      *Validated AAD token. TenantId: 239cf739-XXXX-4632-XXXX-9bXXXXXXXX08 UserId: 09a435fb-1bdd-4d6f-b49f-bc41c696171e DeviceId: 675b4fdc-893d-4a56-9abe-1303e07a4e2c OnPrem_UserSid: S-1-5-21-398374713-1260630808-142979382-25583 OnPrem_DeviceSid:
Return token to client, token type: User, hierarchyId: e8b5d451-XXXX-462f-XXXX-b5b9cd580a56, userId: 96c95f8d-2748-465b-8b58-5feca75dbfda, deviceId: (CCM_STS.log)*

4. Once we get the CCM token back, we confirm registration

   *Getting CCM Token from STS server ' https://myMP.Mylab.com'
Getting CCM Token from https://myMP.Mylab.com/CCM_STS
[RegTask] - Client is already registered. Exiting. (ClientIDManagerStartup.log)*
 


### Communication Validation
-CMG validates client token (via CMG, CP and HTTPS MP database request)

-Client verifies CMG service certificate (or management certificate)

>[!TIPS] 
   >- When using PKI for CMG Service certificate, client requires Root CA of the certificate on local store
   >- When using third party certificate for CMG service certificate, clients automatically validates certificate on internet


### Common issues
-Root CA not present on client root stores when PKI certs is used for CMG service certificate
-CRL check enabled (use the /NoCRLcheck option in command line or publish CRL on internet*)
-WPJ cert not found ( client Azure regsitered, not AAD joined)


> [!NOTE] 
>*Using /NoCRLCheck is only good for ccmsetup bootstrap. For the clients to be fully functional, admins need to disable CRL check on the site’s properties communication page. Otherwise after security settings refreshed by location service, the clients will not talk to the server anymore.*

