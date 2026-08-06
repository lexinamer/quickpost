# QuickPost

A single-page web app for posting to a WordPress site from your phone: title, date, body text, and a multi-photo picker. No build step, no backend — it's one static `index.html` that talks directly to the WordPress REST API.

## How it works

- **Settings** (gear icon) stores your WordPress site URL, username, and an [Application Password](https://make.wordpress.org/core/2020/11/05/application-passwords-integration-guide/) in `localStorage`. Nothing is hardcoded or sent anywhere except your own site.
- **Publish** uploads each selected photo to `/wp-json/wp/v2/media` (Basic Auth with the Application Password), collects the returned media IDs, then creates the post via `/wp-json/wp/v2/posts` with the body content plus a `[gallery ids="..."]` shortcode (Meow Gallery) for the uploaded photos.

## WordPress setup

1. **Create a dedicated account.** Use an Author or Editor role for this app — not your Admin login.
2. **Generate an Application Password**: WordPress admin → Users → Profile → Application Passwords. Enter this (with your username and site URL) in the app's Settings screen.
3. **Enable CORS** for the REST API so a request from your Cloudflare Pages domain is allowed. WordPress doesn't send `Access-Control-Allow-Origin` for cross-origin requests by default. Add this to a must-use plugin (`wp-content/mu-plugins/quickpost-cors.php`) or your theme's `functions.php`, replacing the origin with your exact deployed URL:

   ```php
   <?php
   add_action('init', function () {
       $allowed_origin = 'https://your-app.pages.dev'; // no trailing slash; use your custom domain if you set one up
       $origin = $_SERVER['HTTP_ORIGIN'] ?? '';

       if ($origin === $allowed_origin) {
           header('Access-Control-Allow-Origin: ' . $allowed_origin);
           header('Access-Control-Allow-Methods: GET, POST, OPTIONS');
           header('Access-Control-Allow-Headers: Authorization, Content-Type, Content-Disposition');
           header('Access-Control-Allow-Credentials: true');
       }

       if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
           status_header(200);
           exit();
       }
   });
   ```

   Lock this to your app's exact URL — not a wildcard (`*`) — since it's paired with Basic Auth credentials.

4. Revoke the Application Password from your WordPress profile any time you stop using the app.

## Deploy to Cloudflare Pages

This is a static site with no build step, so Pages needs almost no configuration.

1. Push this repo to GitHub (see below).
2. In the Cloudflare dashboard: **Workers & Pages → Create → Pages → Connect to Git**, and select this repo.
3. Build settings:
   - Framework preset: `None`
   - Build command: *(leave empty)*
   - Build output directory: `/`
4. Deploy. Cloudflare will give you a `*.pages.dev` URL — use that (or a custom domain you attach later) as the `allowed_origin` in the CORS snippet above.

## Local use

Open `index.html` directly in a browser, or serve the folder with any static file server. No dependencies, no install step.

## Security notes

- Credentials live only in `localStorage` on your device — never committed to this repo, never sent anywhere but your own WordPress site.
- Application Passwords can be scoped to a low-privilege account and revoked independently of your main login.
- Because this app is a public static site, anyone with your Pages URL can load the form — but without your saved credentials (which live only in your browser's `localStorage`) they can't publish anything.
