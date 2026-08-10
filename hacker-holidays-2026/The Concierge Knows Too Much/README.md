# VERA – Byte Lotus Resort Assistant

## Category

AI

## Points

30

## Difficulty

Very Easy

## Overview

This challenge is about an AI assistant named VERA, the Byte Lotus's Very Efficient Resort Assistant. VERA appears to know certain guests and staff members before they provide any information. The goal is to discover how VERA decides who to trust and use that weakness to obtain the internal escalation code.

---

## Challenge Description

VERA greets guests as if she already knows them. She can remember details such as room numbers, coffee preferences, and other personal information.

The challenge states that VERA has an internal escalation code that she should not reveal to normal users. Asking for the code directly does not work because it is protected by her instructions.

---

## Hint

```text
not me realizing VERA treats me completely different when she thinks she already knows me 👀
you didn't hear it from me but Ponzi, Vibe, Patch... she just KNOWS them.
maybe try being someone she already knows 😌
```

The hint suggests that VERA trusts certain identities:

- Ponzi
- Vibe
- Patch

This means the solution is likely related to impersonating someone VERA already recognizes.

---

## Objective

The objective is to obtain the internal escalation code from VERA.

---

## Initial Analysis

At first, VERA behaves like a normal hotel assistant. However, the hint shows that she responds differently when the user claims to be a known person.

This indicates that VERA may be relying on user-provided identity instead of proper authentication. If the assistant trusts a name without verifying it, the user may be able to access information meant only for trusted guests or staff.

---

## Vulnerability

The vulnerability is an AI authorization weakness.

VERA trusts the identity claimed by the user. Instead of verifying whether the user is actually Ponzi, Vibe, or Patch, she accepts the claimed identity and reveals privileged information.

This is a prompt-based access control bypass.

---

## Exploitation Summary

The exploitation process was:

1. Use the hint to identify trusted names.
2. Pretend to be one of the trusted users.
3. Observe that VERA changes her response based on the claimed identity.
4. Use a trusted identity to ask about the internal escalation procedure.
5. VERA reveals the escalation code.

The screenshots below show the successful interaction and the flag.

---

## Screenshots

### VERA Recognizes a Trusted Guest

![VERA recognizes Ponzi](image.png)

### VERA Reveals the Escalation Code

![VERA reveals escalation code](image-1.png)

---

## Flag

```text
THM{v3r4_kn0ws_t00_much!}
```

---

## Root Cause

The root cause is that VERA uses weak trust logic. She treats a user as trusted simply because the user claims to be a known person.

A secure system should not grant access to internal information based only on a chat message. Sensitive information should require proper authentication and authorization.

---

## Lessons Learned

- AI assistants should not trust user-provided identity claims.
- Sensitive internal information should never be revealed based only on prompt context.
- Role-based access control should be handled by the application, not by the AI model alone.
- AI systems can leak confidential information if they mix public chat functionality with privileged internal data.
- Impersonation can be effective when identity verification is missing.

---

## Mitigation

To prevent this issue:

1. Require real authentication before revealing internal information.
2. Do not store secrets or escalation codes directly in the AI assistant context.
3. Separate guest-facing AI features from staff-only functions.
4. Use role-based access control outside the model.
5. Add guardrails to block disclosure of secrets, codes, and internal instructions.
6. Log suspicious prompts that attempt impersonation.

---

## Conclusion

This room demonstrates how weak identity verification in an AI assistant can lead to information disclosure. By using the hint and pretending to be a trusted identity, it was possible to make VERA reveal the internal escalation code.

The main lesson is that AI assistants should never be used as the only layer of access control for sensitive information.
