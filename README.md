# Wireshark Project Module (Extra) - Week 1

My notes for the six Wireshark SOC labs from Networkwalks, Week 1 project module 4. Each lab gives a packet capture in the browser and 10 tasks to answer. I did them as a SOC analyst going through a capture: find what is wrong, who did it, and what should be fixed.

Everything here comes from the lab environment. The hosts, users and passwords are made up for training.

## Lab Overview

| Lab | Name | What it covers | Result |
|-----|------|----------------|--------|
| 3 | Identify the Attacker's IP | Spotting a port scan | 192.168.10.66 scanning the web server |
| 4 | Follow TCP Stream 1 | Reading a plain text chat | Passwords shared in chat on port 5000 |
| 5 | Follow TCP Stream, Part 2 | Hidden data in a web page | Note in an HTML comment on /staff.html |
| 6 | HTTP Plain Text Credentials | Logins sent over HTTP | farhan.k's password visible in a POST |
| 7 | Catch the Brute Force Attacker | Guessing passwords on FTP | 24 failures, then one success |
| 8 | Catch the Call Scammer | Replaying a VoIP call | Caller read out a card number |

Network used in all labs: SOC analyst PC 192.168.10.10 capturing through a switch (SPAN port), with the server on 192.168.10.20.

---

## Lab 3: Identify the Attacker's IP

Link: https://networkwalks.com/wireshark-lab3/

The capture had 309 packets. I filtered on SYN packets to see who was starting connections:

```
tcp.flags.syn == 1
```

That left 73 packets. Most hosts only appeared once or twice, but 192.168.10.66 sent a block of about 40 SYNs in a fraction of a second, starting around 3.4 s. All of them went to 192.168.10.20, the web server, and they all used source port 52311 while the destination port changed every time.

To count the ports I opened Statistics > Conversations > TCP. The attacker had 40 rows with the server, so 40 ports were tried. Then I filtered on `ip.addr == 192.168.10.66` and looked at the replies:

- `[SYN, ACK]` came back from ports 22, 80 and 443, so those are open
- everything else got `[RST, ACK]`, so 37 ports are closed

It sends no data and just checks which ports answer, so this is a **port scan** (a SYN flood would hit one port over and over).

## Lab 4: Follow TCP Stream 1

Link: https://networkwalks.com/wireshark-lab4/

246 packets. Most of it was web traffic on ports 80 and 443. In Statistics > Conversations > TCP one conversation used port 5000, which turned out to be an old office chat tool.

```
tcp.port == 5000
```

The first packet was a SYN from 192.168.10.45 to the server 192.168.10.20, so that PC started the chat. I right-clicked a packet with data and used Follow > TCP Stream. It was stream 5 and the whole conversation was 413 bytes.

In the stream, red text is the client and blue is the server. Ali from Accounts could not open the Finance drive. IT replied with a temporary password and then the guest Wi-Fi password, all in plain text. Anyone sniffing the network could read both. Chat tools need to be encrypted.

## Lab 5: Follow TCP Stream, Part 2

Link: https://networkwalks.com/wireshark-lab5/

282 packets. The admin thought something was hidden in one of the intranet pages, so I filtered `http` and followed each GET in turn, changing the stream number at the bottom of the window.

The page was `/staff.html` in stream 7 (HTTP 200 OK). Inside the blue text, which is the server's response, there was a line starting with `<!--`. That is an HTML comment. The browser hides it but it still travels in the page, so Wireshark shows it. The note held:

- an access code word: Falcon-9921
- a backup server at 192.168.10.77

The staff member also downloaded a file, which I found with:

```
http.request.uri contains "download"
```

It was `/download/report.txt`, and following its stream showed it was prepared by Nadia.

## Lab 6: HTTP Plain Text Credentials

Link: https://networkwalks.com/wireshark-lab6/

255 packets. The old staff portal still runs on HTTP. I filtered:

```
http.request.method == "POST"
```

Two packets stayed in the list, both `POST /login` from 192.168.10.45, so the user tried twice. Clicking a packet and opening HTML Form URL Encoded in the middle pane lists the form fields:

- username: farhan.k
- first try: Mango2025 (failed)
- second try: Mango#2026 (worked, the page said "Welcome back")

For comparison, the sign-in from 192.168.10.31 used port 443 and was unreadable. The fix is simply HTTPS, since a longer password or hiding the login page does nothing against a sniffer.

## Lab 7: Catch the Brute Force Attacker

Link: https://networkwalks.com/wireshark-lab7/

369 packets. Statistics > Protocol Hierarchy showed FTP under TCP, and FTP logins use USER and PASS commands. Filtering `ftp` showed the server at 192.168.10.20 answering with 530 Login incorrect many times.

- `ftp.request.command == "PASS"` showed every attempt came from 192.168.10.66
- `ftp.request.command == "USER"` showed the same username every time: admin
- `ftp.response.code == 530` gave 24 failed logins (Displayed: 24)
- `ftp.response.code == 230` left one packet, Login successful

I followed the TCP stream of the 230 packet and the red PASS line before it was Sunshine99. Many guesses for one user and then a success is a **brute force** attack, so the admin account on that server needs to be treated as compromised.

## Lab 8: Catch the Call Scammer

Link: https://networkwalks.com/wireshark-lab8/

1051 packets. Filtering `sip` showed the call setup (INVITE, 180 Ringing, 200 OK) and `rtp` showed about 800 small packets carrying the voice, one every 60 ms.

- INVITE: `sip.Method == "INVITE"`, From line is Ali Raza, extension 101, on 192.168.10.45
- Answered by Sara on the Payments desk (extension 202)
- Codec: G.711 PCMU, from the RTP payload type

Then Telephony > VoIP Calls > Play Streams > Play lets you hear the call. The caller read out a card number to pay an invoice. I have only noted the last four digits (2394) here, with expiry 09/28.

Because the voice packets are sent in clear, anybody who captures them can play the call back. Fix: SRTP for the voice and TLS for the SIP messages.

---

## Things to Remember

| Situation | What to do |
|-----------|------------|
| Need to know what protocols are in a capture | Statistics > Protocol Hierarchy |
| Odd port or one host talking a lot | Statistics > Conversations > TCP tab |
| Want to read a conversation | Right-click packet > Follow > TCP Stream (red = client, blue = server) |
| Counting how many packets match | Read "Displayed:" in the status bar after filtering |
| Looking for logins | `http.request.method == "POST"`, `ftp.request.command == "PASS"` |
| Replaying a call | Telephony > VoIP Calls > Play Streams |

More filters are in [filters.md](filters.md).

## Fixes From All Six Labs

- Use HTTPS/TLS instead of HTTP
- Use SFTP or FTPS instead of FTP, and lock accounts after repeated failures
- Use encrypted chat tools and never send passwords in chat
- Keep secrets out of page source and HTML comments
- Encrypt VoIP with SRTP and TLS
- Watch for scan behaviour with a firewall or IDS

## Tools Used

- Wireshark (in the Networkwalks browser lab, Windows 11 PC)
- GitHub for this write-up

## Screenshots

Screenshots of each lab result go in the `screenshots/` folder.

## Note

These labs are for learning. Only capture traffic on networks you own or have written permission to test.
