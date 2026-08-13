# Ask Sunny Troubleshooting

Start with the narrowest relevant check. Record the UTC time, plugin version, safe error code, and correlation ID when available. Never record credentials, raw visitor messages, private content, raw request bodies, or database dumps in a support ticket.

## Ask Sunny will not activate or load

1. Read the administrator compatibility notice.
2. Confirm PHP is 7.4–8.4 and WordPress is 6.5–7.0.x.
3. Confirm Directorist 8.8 or newer is active.
4. Confirm the official release ZIP—not a source checkout—was installed.
5. Check the [supported requirements](README.md#requirements) before changing the environment.

The plugin intentionally stops booting when its supported runtime dependencies are unavailable.

## The Connect screen says provisioning is not configured

The WordPress server cannot read `ASK_SUNNY_PROVISIONING_KEY`.

1. Confirm the constant or environment variable exists in the PHP/WordPress runtime, not only in an interactive shell.
2. Confirm the environment variable name is exactly `ASK_SUNNY_PROVISIONING_KEY`.
3. Restart PHP or the relevant application service if the hosting platform requires it after changing environment variables.
4. Do not paste the key into the dashboard.

## The backend URL is rejected

- Use a complete production HTTPS URL, such as `https://sunny-api.example.com`.
- Do not include an unsupported scheme.
- Plain HTTP is accepted only for `localhost`, IPv6 loopback, or the `127.0.0.0/8` loopback range.
- If `ASK_SUNNY_API_BASE_URL` is defined, update the server constant rather than expecting the dashboard value to take precedence.

## Connection or provisioning fails

1. Confirm the backend URL is reachable from the WordPress server. Browser reachability alone is not sufficient.
2. Confirm the provisioning key and OpenAI key are current and belong to the expected environment.
3. Check for outbound-request blocks, DNS failures, TLS certificate errors, a reverse-proxy timeout, or maintenance at the backend.
4. Retry only after correcting the cause. A rejected or malformed provisioning response preserves an existing working installation credential.
5. If **Test Connection** fails, note that the current dashboard disconnects the local installation afterward; complete the connection flow again once the cause is fixed.

Share only the safe error code, correlation ID, UTC timestamp, plugin version, and dependency versions with support.

## Chat form or chat app does not appear

1. Confirm the page contains exactly one intended shortcode:
   - `[ask_sunny_chat_form]` for the compact form.
   - `[ask_sunny_chat_app]` for the full chat interface.
2. Confirm Ask Sunny is connected.
3. Confirm the page is published and the shortcode is not inside escaped code or a block that does not process shortcodes.
4. Check that the widget JavaScript, CSS, and bundled fonts return HTTP 200.
5. Clear page, object, CDN, and asset-minification caches.
6. Temporarily test with a supported default theme and without JavaScript combination or delay features.

The shortcodes intentionally render no markup when the stored `enabled` setting is false. If the site was upgraded from a version or integration that disabled it, inspect the saved Ask Sunny settings through an authorized administrator workflow.

## Compact questions open the wrong page

1. Publish a page containing `[ask_sunny_chat_app]`.
2. Open **Ask Sunny > Settings > Chat**.
3. Select that page under **Select Chat Page** and save.
4. Clear frontend caches and resubmit the compact form.

Without a selected chat page, the form submits to the site home URL.

## Chat requests fail or time out

1. Test backend reachability and authentication from the **Connection** tab, keeping in mind that a failed test disconnects locally.
2. Confirm the installation has not been disconnected or revoked on the backend.
3. Confirm the OpenAI provider key configured through the backend remains valid.
4. For signed-in visitors, refresh the page to replace an expired WordPress REST nonce.
5. For anonymous visitors, wait briefly if the rate limit was reached, then retry.
6. Check the browser network response for a safe Ask Sunny error code; do not copy request payloads containing visitor text.

Messages are limited to 2,000 characters. Empty and oversized messages are not submitted by the frontend.

## A conversation does not restore

Conversation continuity depends on browser storage and the same visitor identity.

1. Confirm local storage is allowed for the site and has not been cleared by privacy software.
2. Confirm the visitor is using the same browser profile and is either consistently signed in or consistently anonymous.
3. Confirm the backend still has the conversation and the installation remains connected.
4. If restoration fails, Ask Sunny clears the unusable local conversation identifier and starts a new interface.

Using **Start new conversation** deliberately requests conversation deletion, clears local continuity, and starts again. The compact form deliberately starts a new conversation when it receives a new `ask_sunny_prompt` question.

## Content is missing from answers

1. Open **Ask Sunny > Settings > Data Sources** and confirm the relevant WordPress post type is enabled. Directorist sources should be discovered automatically.
2. Save source settings, then return to the root **Data Sources** tab.
3. Select the correct source and search for the item.
4. Confirm the content is public and eligible. Drafts, private posts, revisions, autosaves, attachments, and unapproved reviews are not intended for retrieval.
5. Choose **Add to Index**, **Retry**, or **Index All** as appropriate.
6. Wait for **In Queue** to clear, then confirm **Indexed At** has a value.
7. Ask a specific question whose wording matches the public content and inspect the returned recommendations.

For a review, make sure its parent listing is also public and indexable. Ask Sunny schedules a missing parent listing before indexing the review.

## Indexing is disabled, stuck, or failing

1. Enable **Enable Automatic Indexing** under **Settings > Data Sources** and save.
2. Confirm the site can run WordPress cron or has a real cron job invoking scheduled WordPress events.
3. Confirm the backend is reachable and the installation credential is valid.
4. Filter the active source to **Failed** and use **Retry** after fixing the cause.
5. Run **Index All** again. Jobs are designed to deduplicate and resume safely.
6. Do not edit the queue options or `_ask_sunny_*` metadata manually.

Indexing processes 25 records per checkpoint and retries transient failures up to five times. Large sources may continue after the first administrator request completes.

## An item shows the wrong indexing status

1. Refresh the Data Sources page after scheduled processing has run.
2. Verify you are viewing the correct source and filters.
3. Check whether the WordPress item changed status, was trashed, or became ineligible.
4. Use **Add to Index** or **Retry** for an eligible item.
5. Use **Delete Index** only when the backend record should be removed.

Deleting an index record does not delete WordPress content. Likewise, disabling a source or automatic indexing does not automatically delete its existing backend records.

## Delete Index or Delete All Index was used accidentally

The original WordPress, Directorist listing, or review content remains intact.

1. Confirm automatic indexing is enabled.
2. Select the affected source.
3. Use **Add to Index** for one item or **Index All** for the source.
4. Wait for processing and confirm the indexed counts and timestamps.

## Appearance changes do not show

1. Confirm **Save Changes** completed successfully.
2. Reload a page containing the relevant shortcode.
3. Clear WordPress page/object caches, the CDN, and browser cache.
4. Confirm the page loads the current `widget.css` and `widget.js` files.
5. Check that submitted colors are valid six-digit hexadecimal values.
6. If a selected logo or avatar is absent, confirm the Media Library attachment still exists and is publicly readable.

## Layout or accessibility problems

1. Test at 390×844, 768×1024, and 1440×900 viewports and at 200% zoom.
2. Check keyboard navigation, visible focus, Enter/Shift+Enter behavior, and screen-reader status announcements.
3. Review custom color contrast, border width, radius, gradients, and shadow values.
4. Temporarily restore prototype images/colors and test with a supported default theme.
5. Disable frontend CSS/JavaScript optimization to identify an ordering or minification conflict.

## Dates or times are incorrect

1. Set a named timezone under **Settings > General > Timezone** in WordPress.
2. Confirm source dates are valid WordPress dates rather than manually embedded display strings.
3. Reindex the affected item.
4. Repeat the question with an explicit date or time reference.

## Safe support checklist

Provide:

- Ask Sunny plugin version.
- WordPress, PHP, and Directorist versions.
- UTC timestamp of the failure.
- Safe error code and correlation ID.
- The affected source label and public record ID when appropriate.
- Whether the failure affects connection, indexing, or chat.

Do not provide:

- Provisioning, installation, or OpenAI credentials.
- Raw visitor questions or conversation transcripts.
- Private source content, email addresses, IP addresses, cookies, or session identifiers.
- Raw request/response bodies or database exports.

For additional operational guidance, see the [backend troubleshooting guide](../shared/SETUP_AND_OPERATIONS.md#troubleshooting), [privacy and retention contract](../server/CONVERSATION_CONTEXT_CONTRACT.md#7-deletion-anonymization-and-retention), [deployment flow](../shared/SETUP_AND_OPERATIONS.md#deployment-flow), and [backup and recovery guide](../shared/SETUP_AND_OPERATIONS.md#backup-and-recovery).
