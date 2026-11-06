# Wireshark Filters I Used

Quick list for next time.

## General

```
http
ftp
sip
rtp
ip.addr == 192.168.10.66
tcp.port == 5000
tcp.flags.syn == 1
```

## HTTP

```
http.request.method == "POST"
http.request.uri contains "download"
```

Form fields: click the packet, then expand HTML Form URL Encoded.

## FTP

```
ftp.request.command == "USER"
ftp.request.command == "PASS"
ftp.response.code == 530
ftp.response.code >= 500
ftp.response.code == 230
```

530 = login incorrect, 230 = login successful.

## SIP / VoIP

```
sip.Method == "INVITE"
```

- Caller: SIP > Message Header > From
- Codec: RTP > Payload type
- Play the call: Telephony > VoIP Calls > Play Streams > Play

## Menus

- Statistics > Protocol Hierarchy
- Statistics > Conversations (TCP tab)
- Right-click packet > Follow > TCP Stream
