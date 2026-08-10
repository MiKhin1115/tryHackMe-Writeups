# Overheard at Breakfast

## Category

OSINT

## Points

60

## Difficulty

Easy

---

## Overview

Overheard at Breakfast is an OSINT challenge from TryHackMe Hacker Holidays. The challenge provides a screenshot of a conversation between two Byte Lotus guests. The main goal is to carefully read the conversation, extract useful identifying information, and use it to find a hidden online account.

This room focuses on social media investigation, email-based profile discovery, and hashing. The important clue leads to Gravatar, a platform that can associate an email address with a public profile and linked social accounts.

---

## Challenge Description

The briefing explains that a guest saw part of a conversation at breakfast and captured a screenshot. Somewhere in the conversation, there is enough information to track down an account that was not supposed to be found.

The challenge hint says:

```text
the breakfast crowd really said the quiet part out loud this morning 😭
y'all need to actually READ what they said, not just skim it
```

This tells us that the solution depends on carefully reading the conversation instead of randomly searching.

---

## Objective

The objective is to:

1. Analyze the provided conversation screenshot.
2. Extract identifying details.
3. Find the hidden account.
4. Retrieve the flag.

---

## Provided Conversation

![Conversation Screenshot](conversation.png)

---

## Initial Analysis

The screenshot shows a conversation between `Ponzi - Influencer` and `Lambo!`.

At first, the conversation looks normal. Ponzi asks Lambo how things are going at the resort, and Lambo replies that Byte Lotus has been treating them well.

The important part appears later in the conversation when Ponzi asks for Lambo's social media handle.

Lambo says they do not use much social media anymore, but they mention that they used to use a free tool to upload a profile and link other media accounts. Lambo also says the tool started with the letter `G`.

This is the key clue.

Another important detail is that Lambo gives an email address:

```text
lambobytelotushotel@gmail.com
```

So the two important clues are:

```text
Tool starts with G
Email address: lambobytelotushotel@gmail.com
```

---

## Key Clue

The phrase:

```text
free tool that let me upload my profile and link other media accounts
Started with a G
```

points to:

```text
Gravatar
```

Gravatar is commonly used to create a public profile linked to an email address. It can also contain profile information and links to other social media accounts.

The challenge category also includes:

```text
OSINT
Social Media
Hashing
```

This confirms that the email address likely needs to be hashed to locate the Gravatar profile.

---

## Why Gravatar Was Important

Gravatar profiles are commonly connected to email addresses. A Gravatar avatar or profile can be searched using an MD5 hash of the email address.

The email must be:

1. Converted to lowercase.
2. Trimmed of spaces.
3. Hashed using MD5.

The email from the screenshot was:

```text
lambobytelotushotel@gmail.com
```

---

## Hashing the Email

To calculate the MD5 hash, I used the following command:

```bash
echo -n "lambobytelotushotel@gmail.com" | md5sum
```

This produced the hash:

```text
d4a5fc5d3128890778667e24617d7cc0
```

This hash can be used to check the Gravatar profile connected to the email address.

---

## Finding the Gravatar Profile

After generating the MD5 hash, I used it to look up the Gravatar profile.

The profile could be checked using the hash value:

```text
d4a5fc5d3128890778667e24617d7cc0
```

The Gravatar profile revealed more information connected to Lambo's identity. From there, I checked the visible profile details and linked accounts.

The hidden account contained the flag.

---

## Exploitation / Investigation Summary

The solving process was:

1. Open the provided conversation screenshot.
2. Read the full conversation carefully.
3. Notice that Lambo mentions a profile-linking tool starting with `G`.
4. Identify the platform as Gravatar.
5. Extract Lambo's email address from the conversation.
6. Convert the email address to lowercase.
7. Generate the MD5 hash of the email.
8. Use the hash to locate the Gravatar profile.
9. Review the profile and linked social media information.
10. Find the hidden account and retrieve the flag.

---

## Flag

```text
THM{S3creT_Pr0fil3_H4s_b33n_Ident1fi3d}
```

---

## Vulnerability / OSINT Concept

This challenge demonstrates how an email address can expose more information than expected.

Even if a person does not directly share their social media username, their email address may still be linked to public profiles. In this case, the email address was enough to discover a Gravatar profile, which then led to the hidden account.

This is not a technical vulnerability in the traditional sense. It is an OSINT exposure caused by linked online identities.

---

## Root Cause

The root cause was identity reuse.

Lambo reused the same email address with a public profile service. Because Gravatar profiles can be discovered using an email hash, the exposed email address became a way to find additional online information.

The conversation also leaked too much context by mentioning the tool started with `G`, making the investigation easier.

---

## Lessons Learned

- OSINT challenges require careful reading.
- Small details in conversations can reveal important clues.
- Email addresses can be used to discover public profiles.
- Gravatar profiles can be found using an MD5 hash of an email address.
- Linked accounts can expose hidden identities.
- Reusing the same email across services can reduce privacy.

---

## Mitigation

To avoid this type of exposure in real life:

1. Avoid sharing personal email addresses publicly.
2. Use separate emails for public and private accounts.
3. Do not link private social media accounts to public profile services.
4. Review Gravatar and other profile-linking services for exposed information.
5. Avoid reusing usernames, bios, and profile images across accounts.
6. Be careful when screenshots include emails, usernames, or platform hints.

---

## Conclusion

This room was solved by carefully analyzing the conversation and identifying the clue pointing to Gravatar. The email address `lambobytelotushotel@gmail.com` was hashed using MD5, and the resulting hash was used to find the associated Gravatar profile.

The profile led to the hidden account, where the flag was found.

The challenge shows how a simple email address, combined with public profile services and linked accounts, can reveal information that was meant to stay hidden.
