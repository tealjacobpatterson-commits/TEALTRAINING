# LinkedIn auto-post — setup

When configured, clicking **Share on LinkedIn** on the confirmation page posts the
attendee's ticket (text **and** image) straight to *their* LinkedIn feed via the
official LinkedIn API. Until you add credentials, the button falls back to the
old behaviour (opens LinkedIn's composer with pre-filled text + downloads the
ticket image to attach).

The whole thing runs on `teal-server.py` (Python 3, no dependencies to install).

---

## 1. Create a LinkedIn app

1. Go to <https://www.linkedin.com/developers/apps> → **Create app**.
2. Associate it with your Teal Products company page.
3. On the **Products** tab, request/add:
   - **Sign In with LinkedIn using OpenID Connect**
   - **Share on LinkedIn**
   (Both are self-serve — no lengthy review.)
4. On the **Auth** tab:
   - Copy the **Client ID** and **Client Secret**.
   - Under **Authorized redirect URLs for your app**, add exactly:
     ```
     http://localhost:7788/oauth/linkedin/callback
     ```
     (For a public/live site, add that domain's `https://…/oauth/linkedin/callback` too.)

## 2. Add your credentials

Copy the example and fill it in:

```bash
cp teal-partners-day/linkedin_config.example.json teal-partners-day/linkedin_config.json
```

```json
{
  "client_id": "YOUR_CLIENT_ID",
  "client_secret": "YOUR_CLIENT_SECRET",
  "redirect_uri": "http://localhost:7788/oauth/linkedin/callback",
  "api_version": "202409"
}
```

> Keep `linkedin_config.json` private — it holds your secret. (Env vars
> `LINKEDIN_CLIENT_ID` / `LINKEDIN_CLIENT_SECRET` / `LINKEDIN_REDIRECT_URI` work too
> and take precedence.)

## 3. Run the server

```bash
python3 teal-server.py 7788
```

Open <http://localhost:7788/teal-partners-day/>. On start-up it prints whether
auto-post is **ENABLED**. The `redirect_uri` must match the LinkedIn app exactly
**and** the port you run on — so use `7788` locally.

## How it works

1. Attendee submits the RSVP → confirmation page renders their ticket.
2. They click **Share on LinkedIn** → the ticket is rendered to a PNG and the
   browser is redirected to LinkedIn to authorise the Teal app (once per person).
3. LinkedIn redirects back to `/oauth/linkedin/callback`; the server uploads the
   image and publishes the post on their behalf, then shows `shared.html`.

## Notes / gotchas

- **Each attendee authorises once.** LinkedIn only lets an app post to a member's
  feed after that member grants permission (`w_member_social`). There is no way to
  post to someone's profile without their sign-in — that's LinkedIn policy.
- **`api_version`** (`LinkedIn-Version`) is rotated by LinkedIn roughly monthly. If
  posting starts failing with a 426/"version" error, bump it to a current value
  from the LinkedIn API changelog.
- The edit copy of the post lives in `teal-server.py` as `SHARE_TEXT`. Avoid the
  characters `( ) [ ] { } < > @ | * _ ~` in it — the Posts API rejects them
  unless escaped. Hashtags (`#`) and apostrophes are fine.
- Sessions (the pending ticket image) are held in memory, so posting must finish
  in the same server run the attendee started in. Fine for a single-process
  server; if you move to multiple workers, back it with a shared store.
