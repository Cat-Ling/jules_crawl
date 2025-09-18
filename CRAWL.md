# Web Crawling & API Discovery Best Practices

This document outlines a set of best practices and lessons learned for successfully crawling websites and discovering their internal APIs for the purpose user described. It includes guidance on dealing with difficult, sandboxed environments.

## 1. Environment & Tooling Quirks

When working in a sandboxed or containerized environment, you may encounter non-standard behaviors. The following issues and workarounds have proven effective.

- **File System Volatility:**
  - **Problem:** Newly created directories and files (e.g., `node_modules` after `npm install`, or `build` after `npm run build`) may not persist between separate `run_in_bash_session` calls. A command might succeed, but its artifacts will be gone by the next command.
  - **Solution:** Chain commands together using `&&`. This ensures all operations occur within the same shell instance, preserving the file system state for the duration of the command chain.
    - **Example:** `cd /app/client && npm install && npm run build && ls -l build`

- **Working Directory Instability:**
  - **Problem:** The shell session can lose its current working directory, leading to "No such file or directory" errors on subsequent commands.
  - **Solution:**
    1.  **Be Explicit:** Start command chains with `cd /app` (or another known root) to guarantee a consistent starting point.
    2.  **Use Absolute Paths:** When possible, use absolute paths in test code (the code that's for internal tests) to eliminate ambiguity.

- **`npm` and Build Tools:**
  - **Problem:** A successful `npm install` may still result in "command not found" for binaries like `react-scripts`. This is a direct symptom of the file system volatility issue where the `node_modules/.bin` directory vanishes.
  - **Solution:** Always chain `npm install` and `npm run build` in the same command. Do not run them in separate steps.

- **Blocking Processes:**
  - **Problem:** Running a development server with `npm start` or `yarn dev` will sometimes block the session, preventing further commands.
  - **Solution:** While you can background the process with `&`, a more stable approach for verification is to use utilities like tmux (eg. 'tmux new-session -d "<your command>"') or nohup.  (install these if necessary). This avoids long-running, stateful development servers.

## 2. API Discovery Strategy

- **Scraping is for Discovery, Not Just Data:** The primary goal of scraping a modern web app is to discover its internal APIs, backend networking, not just to parse data from HTML. The most valuable data is in the network requests.
- **Focus on XHR/Fetch Requests:** Use browser developer tools or a library like Playwright to intercept and log all `XHR/fetch` requests. This is the most reliable way to find the API endpoints, request methods, and payload structures.
- **Simulate Real User Behavior:**
  - **Scrolling:** Many sites use infinite scroll to load more content. Your script must simulate scrolling to the bottom of the page to trigger these API calls.
  - **Clicking:** If there are "load more" buttons, your script must be ableto find and click them.
- **Persistent playwright:**
  - **Sites requiring persistent browsing:** Some sites might need you to run a persistent playwright session and navigate around the site with it. In this case you will take screenshots every few seconds, analyze it, then decide where you need to click on the page to continue navigation the website.
  - **Usefulness:** This is especially useful for passing Cloudflare or other robot challenges (like Google Captcha). As well as mitigating the need to create a new session of playwright for every url.
  - **Allows logging in:** The biggest advantage of having a persistent playwright session is that nagivation to login page and then logging in with user provided credentials is possible. And in case there's an OTP or TOTP, it's as easy as asking user for it and then filling it in. Logging in allows more things to be available for use in the way user has requested.

  
## 3. Emulating a Real Browser (Headers & Cookies)

A server will often block requests that don't appear to be from a legitimate browser. Replicate the request headers sent by your browser as closely as possible.

- **User-Agent:** Always set a standard `User-Agent` string.
- **`X-Requested-With`:** For many modern web apps, API requests are identified by the `X-Requested-With: XMLHttpRequest` header.
- **App-Specific Headers:** Look for other custom headers like `X-App-Version`.
- **CSRF Tokens (`X-CSRFToken`) or JWT:** These are essential for `POST` or other state-changing requests. They are almost always dynamic. A hardcoded token will fail. The correct approach is to first make a request to the main page, parse the HTML to find the token, and then use it in subsequent API requests. Some sites might need getting a JWT token which expires quickly.
- **Cookies:** For any site that requires a login or serves personalized content, the `Cookie` header is **critical**. A missing or invalid cookie is the most common cause of `403 Forbidden` errors.
  - **Using Environment Variables:** A robust method is to load sensitive cookies from an environment variable (e.g., `PINTEREST_COOKIE`). This keeps secrets out of source code.

## 4. Debugging

- **Check the Response Body First:** If you expect JSON but get a parsing error, the server is likely returning HTML (e.g., a login page). **Always log the raw response body** to see what you're actually getting.
- **Check the Status Code:** A `403 Forbidden` error indicates an authentication/authorization failure (missing/invalid cookie or CSRF token).
- **Debug via Response:** If file logging is unreliable, modify your server code to write the raw response body from the upstream API directly into its own HTTP response. This makes the error visible in your `curl` output.
