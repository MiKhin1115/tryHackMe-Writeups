# After Hours — Forensics Write-Up

## Room Access

- Download the given ZIP/7z attachment.
- If using AttackBox, navigate to:

```bash
cd /root/Rooms/hacker-holidays-2026/after-hours
```

## Objectives

1. Parse the provided system artifacts for hidden custom configuration data.
2. Locate the malicious class and extract its embedded payload.
3. Decode the payload and submit the recovered flag.

## 1. Extract the Challenge Files

Extract the archive using the provided password:

```bash
7z x after-hours.7z
```

Password:

```text
Aft3rH0ursAtt4chm3ntP4ss
```

Navigate to:

```text
/after-hours/after-hours-forensics-hh/challenge_attachments
```

You should find five files:

```text
INDEX.BTR
MAPPING1.MAP
MAPPING2.MAP
MAPPING3.MAP
OBJECTS.DATA
```

## 2. Search `OBJECTS.DATA`

The `OBJECTS.DATA` file contains a large amount of Windows event and system information, making it difficult to read directly.

Extract printable ASCII strings:

```bash
strings -a OBJECTS.DATA > strings-ascii.txt
```

Search for PowerShell-related content:

```bash
grep -i powershell *.txt
```

This reveals Base64-encoded data.

One interesting PowerShell expression is:

```text
$file=([[WmiClass] 'ROOT\cimv2\:Win32_HardwareTelemetry').Properties['ConfigData'].Value;])
```

This indicates that the `Win32_HardwareTelemetry` WMI class contains custom configuration data.

Search for the class:

```bash
grep -C 3 'Win32_HardwareTelemetry' *.txt
```

This produces several Base64-encoded strings.

## 3. Extract the Embedded Payload

Among the Base64 strings, identify the one beginning with:

```text
7PZ
```

This appears to be compressed 7-Zip-related data.

Copy the complete Base64 string and decode it using CyberChef:

1. **From Base64**
2. **Raw Inflate**

Save the resulting decoded data as a payload file.

## 4. Analyze the Payload

Navigate to the tools directory:

```text
/after-hours/after-hours-forensics-hh/tools
```

Read the instructions:

```bash
cat instructions.txt
```

Follow the instructions provided to run the analysis tool.

Load the saved payload file into the tool.

In the left panel, locate the payload and select:

```text
After Hours → Program
```

This reveals additional encoded strings.

## 5. Decode the Final String

Copy the encoded string and decode it using CyberChef.

The recovered flag is:

```text
THM{P4tch_op3ned_th3_BacKd00r}
```

## Flag

**`THM{P4tch_op3ned_th3_BacKd00r}`**
