
*Applies to: System Center Configuration Manager (Current Branch)*

Co-managed clients are able to communicate with Configuration Manager site using AAD authentication. 

## Client authentication workflow during client installation

When Ccmsetup action starts, client will check if there is an azure user session logged on the computer. Client will use this session to get AAD token:

### AAD Token Request
1. Client uses AAD parameters to request AAD token:

    *Retrieved AAD token for AAD device '82XXXXad-f062-XXXX-b062-dbXXXXac04d5' (CCMSetup.log)*

> [!Note] 
   >If WPJ certificate is not found, we do not try to request AAD token (client is not properly AAD joined)
   >AAD Device token usage available from 1806. You need to update teant settings on upgraded environments. AAD Device token is prefered
   >Traffic with invalid token will be blocked upfront of the CMG, preventing unnecessary communications to the internal roles

2.	CCM Token Request (Using AAD User token)

    2.1 CMG Gets request 
    
      *Getting CCM Token from https://myCMG.cloudapp.net/CCM_Proxy_ServerAuth/9/CCM_STS (CCMSetup.log)*
    
    2.2 CMG moves request to CMG CPhttps://myMP.Mylab.com/CCM_STS
    
      *(IIS log, %CMGHttpHandler.log)*
    
    2.3 CMG CP transforms CMG client request to MP client request 
    
      *MessageID: b90c8940-b368-4f4a-b22f-a8eaae90acb2 RequestURI: https://myMP.Mylab.com/CCM_STS EndpointName: CCM_STS ResponseHeader: HTTP/1.1 200 Ok  (SMS_CLOUD_PROXYCONNECTOR.log)*
   
    2.4 MP runs request against Database to verify user token (CCM_STS.log)
   
      *Validated AAD token. TokenType: Device TenantId: cXXXXXX2-XXXX-4df9-XXXX-cddXXXXXXe0a UserId: 00000000-0000-0000-0000-000000000000 DeviceId: 82XXXXad-f062-XXXX-b062-dbXXXXac04d5 OnPrem_UserSid:  OnPrem_DeviceSid:*  
 
 
## Client authentication workflow during client registration
 

### Communication Validation
-CMG validates client token (via CMG, CP and HTTPS MP database request)

-Client verifies CMG service certificate (or management certificate)

> [!Note]
   > When using PKI for CMG Service certificate, client requires Root CA of the certificate on local store
   > When using third party certificate for CMG service certificate, clients automatically validates certificate on internet

### Common issues
-Root CA not present on client root stores when PKI certs is used for CMG service certificate
-CRL check enabled (use the /NoCRLcheck option in command line or publish CRL on internet*)
-WPJ cert not found ( client Azure regsitered, not AAD joined)

*Using /NoCRLCheck is only good for ccmsetup bootstrap for this case. For the clients to fully functional, admins need to disable CRL check on that site’s client communication page. Otherwise after security settings refreshed by location service, the clients will not talk to the server anymore.*

