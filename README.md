https://myMP.Mylab.com/CCM_STS
*Applies to: System Center Configuration Manager (Current Branch)*

Co-managed clients are able to communicate with Configuration Manager site using AAD authentication. 

## Client authentication workflow during client installation

When CCmsetup action starts, client will check if there is an azure user session logged on the computer. Client will use this session to get AAD token:

### AAD Token Request
1. Client uses AAD parameters to request AAD token:

    *Retrieved AAD token for AAD user 'eeXXXf36-bb1a-XXXd-9d2d-aXXXXXXd' (CCMSetup.log)*

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
    
Communications Validation
CMG validates client token (via CMG, CP and HTTPS MP database request).
Client verifies CMG service certificate (or management certificate)
PKI for CMG Service certificate à client requires Root CA of the certificate on local store (1)
Third party CMG service certificate à Clients automatically validates certificate (Root CA published on internet)
Common issues
Root CA not present
CRL check enabled (use the /NoCRLcheck option in command line or publish CRL on internet*)
WPJ cert not found ( client Azure regsitered, not AAD joined)
*Using /NoCRLCheck is only good for ccmsetup bootstrap for this case. For the clients to fully functional, admins need to disable CRL check on that site’s client communication page. Otherwise after security settings refreshed by location service, the clients will not talk to the server anymore.

