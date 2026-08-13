# Room Access — Ponzi

**Target:**

```text
http://MACHINE_IP:3000
```

## Objectives

- Create a guest account and explore Ponzi's daily reward mechanism.
- Work out exactly what's standing between you and Whale Vault status.
- Find a way past it and retrieve the flag from the vault.

## Category

- **Web Exploitation**
- **Business Logic**
- **Burp Suite**

---

## 1. Information Gathering

I first accessed the room through the browser.

The application presented a login form, so I registered a guest account and logged in.

After logging in, I found a **Claim Reward** button.

When I clicked it, the application awarded PONZI points and displayed a timer indicating that I had to wait **24 hours** before claiming another reward.

The page also contained the following message:

```text
Reach 150 PONZI to unlock the Whale Vault and claim your exclusive reward.
```

Therefore, the objective was to reach at least:

```text
150 PONZI
```

without waiting for the normal 24-hour cooldown.

Each successful reward claim gives:

```text
50 PONZI
```

So, three successful claims are required:

```text
50 + 50 + 50 = 150 PONZI
```

---

## 2. Identifying the Race Condition

The daily reward mechanism should normally prevent repeated claims during the 24-hour cooldown.

However, this functionality can be vulnerable to a **race condition** if multiple requests are processed at approximately the same time.

The idea is to send several reward-claim requests concurrently before the application can update the account's reward state.

Normally:

```text
Request 1 → claim → cooldown applied
Request 2 → rejected
Request 3 → rejected
```

The goal is instead:

```text
Request 1 ─┐
Request 2 ─┼─→ processed concurrently → multiple rewards
Request 3 ─┘
```

---

## 3. Preparing Burp Suite

I created another guest account because the reward had already been claimed on my test account.

I then:

1. Logged into the new account.
2. Opened **Burp Suite**.
3. Enabled **Proxy → Intercept**.
4. Returned to the application.
5. Clicked the **Claim Reward** button.

Burp intercepted the reward request.

---

## 4. Sending the Request to Repeater

I sent the intercepted request to **Repeater**.

In Burp Repeater, I created multiple copies of the request.

One successful request gives:

```text
50 PONZI
```

To reach the required 150 PONZI:

```text
50 PONZI × 3 requests = 150 PONZI
```

---

## 5. Sending the Requests in Parallel

I grouped the Repeater tabs together.

Then I used Burp Suite's option to send the request group **in parallel**.

The important part is that the requests should reach the application as close together as possible.

I sent three reward-claim requests concurrently.

Conceptually:

```text
              ┌── Request 1 → +50 PONZI
Client ───────┼── Request 2 → +50 PONZI
              └── Request 3 → +50 PONZI
                         ↓
                    150 PONZI
```

This bypasses the intended 24-hour waiting period because the application processes the simultaneous requests before the reward state is properly enforced.

---

## 6. Unlocking the Whale Vault

After sending the requests in parallel, I closed Burp's interception and refreshed the web page.

The account had reached:

```text
150 PONZI
```

The **Whale Vault** was now unlocked.

I clicked:

```text
Open Vault
```

and retrieved the flag.

---

## 7. Flag

```text
THM{t0w3l_0n_th3_sunb3d_d0ubl3_sp3nt}
```

---

## Attack Flow

```text
Create guest account
        │
        ▼
Login
        │
        ▼
Claim daily reward
        │
        ▼
Discover 24-hour cooldown
        │
        ▼
Need 150 PONZI
        │
        ▼
One claim = 50 PONZI
        │
        ▼
Intercept claim request with Burp Suite
        │
        ▼
Send request to Repeater
        │
        ▼
Duplicate request 3 times
        │
        ▼
Send requests in parallel
        │
        ▼
Race condition
        │
        ▼
150 PONZI
        │
        ▼
Whale Vault unlocked
        │
        ▼
Retrieve flag
```

## Key Takeaway

The vulnerability is a **race condition in the reward-claim business logic**.

The application attempts to enforce a 24-hour cooldown, but concurrent requests can be processed before the account's reward state is consistently updated. This allows the same daily reward to be claimed multiple times and makes it possible to reach the Whale Vault threshold without waiting 24 hours.

## Flag

```text
THM{t0w3l_0n_the_sunb3d_d0ubl3_sp3nt}
```
