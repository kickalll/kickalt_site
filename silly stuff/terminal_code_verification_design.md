# Terminal Code Verification System

Design Specification

------------------------------------------------------------------------

# 1. Objective

Replace the current client-side terminal code verification system with a
server-side verification architecture so that:

-   valid codes are not visible in browser source
-   code responses cannot be extracted through inspect tools
-   the website and code logic can be updated independently
-   the system is scalable for adding new codes

The website will act only as a **terminal interface**, while a
**Cloudflare Worker** performs all code validation and response
generation.

------------------------------------------------------------------------

# 2. Current Problem

The current implementation stores valid codes directly inside the
website JavaScript.

Example structure:

``` javascript
const TERMINAL_CODES = {
  code1: {...},
  code2: {...}
}
```

This causes several security issues.

## 2.1 Full code list exposed

Users can open DevTools → Sources and see all valid codes.

## 2.2 Internal links exposed

Hidden URLs or secret pages are visible in source.

## 2.3 Reward logic exposed

Point values and actions are readable.

## 2.4 Code scraping possible

Bots can automatically extract all valid codes.

------------------------------------------------------------------------

# 3. Target Architecture

All verification logic moves off the website into a serverless API.

System structure:

User\
↓\
Terminal UI (Neocities)\
↓\
POST request\
↓\
Cloudflare Worker\
↓\
Code lookup + response logic\
↓\
JSON response\
↓\
Terminal executes result

The browser never contains the valid code list.

------------------------------------------------------------------------

# 4. Component Overview

## 4.1 Terminal Website

Responsibilities:

-   capture user code input
-   send request to worker
-   receive response
-   execute action

The terminal becomes a **dumb client**.

It does not know:

-   valid codes
-   response logic
-   hidden URLs
-   point rewards

## 4.2 Cloudflare Worker

Responsibilities:

-   receive code submissions
-   verify code validity
-   determine action
-   return response data

The Worker is the **source of truth**.

------------------------------------------------------------------------

# 5. API Design

## Endpoint

POST /verify

## Request Format

``` json
{
  "code": "user entered string"
}
```

## Response Format (valid popup)

``` json
{
  "status": "valid",
  "action": "popup",
  "url": "https://example.com/page",
  "title": "page name"
}
```

## Response Format (points reward)

``` json
{
  "status": "valid",
  "action": "points",
  "amount": 1000000
}
```

## Response Format (invalid)

``` json
{
  "status": "invalid"
}
```

------------------------------------------------------------------------

# 6. Worker Implementation

Worker contains the code database.

Example structure:

``` javascript
const codes = {
   codeName: responseObject
}
```

Example:

``` javascript
const codes = {

  onionclub: {
    status: "valid",
    action: "popup",
    url: "https://example.com/onionclub",
    title: "onion cafe"
  },

  oniondark: {
    status: "valid",
    action: "popup",
    url: "https://example.com/oniondark",
    title: "onion dark"
  },

  fishitime: {
    status: "valid",
    action: "points",
    amount: 1000000
  }

};
```

Worker logic:

1.  receive request
2.  parse JSON body
3.  normalize code
4.  check dictionary
5.  return response object

------------------------------------------------------------------------

# 7. Worker Script

``` javascript
export default {
  async fetch(request) {

    if (request.method !== "POST") {
      return new Response("Method Not Allowed", { status: 405 });
    }

    const body = await request.json();
    const code = (body.code || "").toLowerCase();

    const codes = {

      onionclub: {
        status: "valid",
        action: "popup",
        url: "https://example.com/onionclub",
        title: "onion cafe"
      },

      oniondark: {
        status: "valid",
        action: "popup",
        url: "https://example.com/oniondark",
        title: "onion dark"
      },

      fishitime: {
        status: "valid",
        action: "points",
        amount: 1000000
      }

    };

    if (!codes[code]) {
      return Response.json({ status: "invalid" });
    }

    return Response.json(codes[code]);

  }
}
```

------------------------------------------------------------------------

# 8. Website Integration

The terminal no longer checks a local object.

Instead it sends the code to the Worker.

Worker URL:

``` javascript
const TERMINAL_API = "https://yourworker.workers.dev/verify";
```

Code submission:

``` javascript
async function checkCode(code) {

  const res = await fetch(TERMINAL_API, {
    method: "POST",
    headers: {
      "Content-Type": "application/json"
    },
    body: JSON.stringify({ code })
  });

  return await res.json();

}
```

Terminal logic:

``` javascript
async function handleSubmit() {

  const raw = inputEl.value.trim().toLowerCase();

  if (!raw) return;

  const result = await checkCode(raw);

  if (result.status !== "valid") {
    outputEl.textContent = "invalid code";
    return;
  }

  outputEl.textContent = "accepted";

  if (result.action === "popup") {
    openEntityPopup(result.url, result.title);
  }

  if (result.action === "points") {
    fishPoints += result.amount;
    renderFishPoints();
  }

}
```

------------------------------------------------------------------------

# 9. Updating Codes

To add new codes, edit the Worker file only.

Example:

``` javascript
const codes = {

  onionclub: {...},

  newsecretcode: {
    status: "valid",
    action: "popup",
    url: "https://example.com/secret",
    title: "secret page"
  }

}
```

Redeploy the Worker.

Website requires no change.

------------------------------------------------------------------------

# 10. Security Model

This system prevents:

-   source inspection attacks
-   static scraping
-   URL discovery from the client

Hidden data exists only inside the Worker.

------------------------------------------------------------------------

# 11. Remaining Limitations

Users can still:

-   brute force codes
-   observe API responses

Mitigation options:

## Rate limiting

Track requests per IP in the Worker.

## Hashing codes

Store hashes instead of plaintext codes.

## Cloudflare KV

Move code storage into a KV database.

------------------------------------------------------------------------

# 12. Future Scalable Architecture

User\
↓\
Terminal\
↓\
Worker API\
↓\
Cloudflare KV database\
↓\
Code entries

Benefits:

-   thousands of codes
-   live updates
-   no worker redeploy needed

------------------------------------------------------------------------

# 13. Final System Properties

  Property          Result
  ----------------- -------------
  Code visibility   hidden
  Hidden URLs       protected
  Client logic      minimal
  Server logic      centralized
  Code updates      worker only
  Website updates   independent

------------------------------------------------------------------------

# 14. Outcome

The terminal becomes a **secure client interface** while all code
verification and hidden data remain inside the serverless worker
environment.
