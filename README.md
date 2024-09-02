# Broken Access Control - Testing Workflow

My go-to checklist when testing for BAC. Not meant to be exhaustive, just
the stuff that actually finds bugs consistently.

## Setup

- Burp Suite running, scope set
- At least 2 accounts ready (admin + low-priv, ideally a third with no auth)
- Map out all the roles the app has - check docs, settings pages, invite flows

## 1. Role mapping

First thing I do is map out all the roles and what each one can access.
I log into each account and just click through everything, noting which
endpoints show up in Burp for each role.

- Save the sitemap per role
- Diff them - anything that only shows up for admin is your target list
- Pay attention to the HTTP methods too, not just the paths

## 2. Horizontal access (IDOR)

This is where most of the bugs are.

- Grab object IDs from your own account (user id, order id, file id, etc)
- Swap them with IDs from the other account
- Check both read and write - sometimes you can't view but you CAN edit
- Don't just test in the URL, check request body params and headers too
- Try UUIDs vs sequential IDs - sequential is way easier to enumerate
- Check if the API returns more data than the frontend shows

## 3. Vertical privilege escalation

- Take a request that only admin can make
- Replay it with a low-priv user's session token
- Try changing role parameters in requests (role=admin, isAdmin=true, etc)
- Check if admin endpoints are just hidden in the UI but still functional
- Test the registration/invite flow - can you assign yourself a higher role?

## 4. Authentication gaps

- Hit every endpoint with no auth token at all
- Try expired tokens, see if they still work
- Check if logging out actually invalidates the session server-side
- Look for endpoints that rely on obscurity instead of auth checks

## 5. CORS and cross-origin

- Send requests with Origin headers from random domains
- Check if credentials are allowed with wildcard origins
- Look for reflected origins in Access-Control-Allow-Origin

## 6. API-specific checks

- If there's a GraphQL endpoint, try introspection and query fields you shouldn't see
- REST APIs - try different HTTP methods (PUT/DELETE on things you should only GET)
- Check if the mobile API has weaker controls than the web one
- Look for older API versions (v1 vs v2) that might not be patched

## 7. Business logic stuff

- Multi-step flows: skip steps, go backwards, replay steps
- State transitions: can you cancel something already completed?
- Rate limits: are they per-account or per-session? Can you bypass by rotating?

## 8. Things people miss

- Exported data (CSV/PDF downloads) sometimes skips access checks
- Webhook/notification endpoints often lack auth
- File uploads - can you access other users' uploaded files by guessing the path?
- Admin functionality hidden behind JS toggles but accessible via direct requests
- Password reset flows that leak tokens or let you reset other users

## Quick wins

- Always check `/admin`, `/debug`, `/internal`, `/graphql`, `/api-docs`
- Try adding `.json` to endpoints that return HTML
- Test what happens when you send a valid admin cookie with a low-priv JWT
- Check if deleting your cookie vs sending an invalid one gives different errors
  (different errors = different code paths = potential bypass)
