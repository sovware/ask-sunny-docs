# Ask Sunny Administrator Guide

## 1. Prepare the site

Before installation:

1. Confirm the site uses PHP 7.4–8.4 and WordPress 6.5–7.0.x.
2. Install and activate Directorist 8.8 or newer.
3. Obtain the Ask Sunny backend URL, provisioning key, and OpenAI API key from the responsible service operators.
4. Confirm the WordPress server can make outbound HTTPS requests to the backend.

## 2. Install Ask Sunny

1. In WordPress administration, open **Plugins > Add New Plugin > Upload Plugin**.
2. Select the official Ask Sunny ZIP and choose **Install Now**.
3. Activate **Ask Sunny**.
4. If activation is refused, resolve the PHP, WordPress, or Directorist version shown in the administrator notice before continuing.

Activation preserves WordPress and Directorist content and initializes the plugin's versioned settings state.

## 3. Configure server-side credentials

Define the provisioning key in server configuration, not in a WordPress content field. For example, in a secure environment-aware configuration loaded by WordPress:

```php
define( 'ASK_SUNNY_PROVISIONING_KEY', getenv( 'ASK_SUNNY_PROVISIONING_KEY' ) );
```

Optionally lock the backend URL in server configuration:

```php
define( 'ASK_SUNNY_API_BASE_URL', 'https://sunny-api.example.com' );
```

When `ASK_SUNNY_API_BASE_URL` is defined, it takes precedence over the dashboard setting. Production URLs must use HTTPS. Only loopback addresses such as `http://localhost:<port>` or `http://127.0.0.1:<port>` may use HTTP for local development.

Do not place either constant in a theme template, frontend JavaScript, a public repository containing secrets, or a support ticket. Follow the hosting provider's secure secret-management process.

## 4. Connect Ask Sunny

After activation, open **Ask Sunny** in the WordPress administration menu. Until provisioning succeeds, the dashboard shows **Connect Ask Sunny**.

1. Enter the **Server URL** unless it is locked by `ASK_SUNNY_API_BASE_URL`.
2. Select **Open AI** as the provider.
3. Select **GPT 5.4 Mini** as the model.
4. Enter the **AI Provider API Key**.
5. Choose the connection action and wait for **Ask Sunny connected** or **Connection saved**.

The browser submits the provider key to WordPress over the protected administrator REST request; WordPress forwards it during backend provisioning. WordPress subsequently displays only masked provider metadata and the non-secret installation-key prefix.

### Test or update the connection

Once connected, open the **Connection** tab.

- Choose **Test Connection** to run backend diagnostics. A failed test disconnects the current local connection, so resolve server reachability and credentials before testing again.
- Use the provider edit action to submit a replacement OpenAI key or model selection. The key field is intentionally empty rather than revealing the saved secret.
- Choose **Disconnect** only when the site should stop using its current installation credential. Disconnecting requires the full connection process again.

## 5. Configure content sources

Open **Ask Sunny > Settings > Data Sources**.

### Automatic indexing

Enable **Enable Automatic Indexing** to synchronize content when eligible listings, reviews, or enabled WordPress posts change. If it is disabled, source-wide and background reindex operations will not run.

### Additional WordPress sources

Under **Enable Other Data Sources**, select the public post types Ask Sunny should use, such as Posts or Pages, then choose **Save Changes**.

Directorist **Listings** and **Listing Reviews** are discovered automatically and appear as permanent Data Sources tabs. Optional WordPress post types appear as separate tabs only after they are enabled and saved.

Only enable content that is suitable for public AI answers. Ask Sunny deliberately excludes revisions, autosaves, attachments, scripts, private content, and unapproved metadata, but administrators remain responsible for the public content they publish.

## 6. Build and maintain the index

Open the root **Data Sources** tab in Ask Sunny.

### Initial indexing

For each visible source tab:

1. Select the source, such as **Listings**, **Listing Reviews**, or an enabled WordPress post type.
2. Review the item and status counts.
3. Optionally filter by search text, index status, post status, directory type, category, or location.
4. Choose **Index All**.
5. Keep the page open while the dashboard reports **Indexing**, or return later and inspect the updated counts.
6. Review any rows marked **Failed** and retry them after resolving the underlying connection or content problem.

Indexing uses 25-record checkpoints and may continue through WordPress scheduled processing. Large sources do not need to complete in one browser request.

### Item actions

| Action | Result |
|---|---|
| **Add to Index** | Queues and processes an eligible item for the backend index. |
| **In Queue** | The item is awaiting or undergoing processing; the action is temporarily unavailable. |
| **Retry** | Attempts an item again after a failed indexing job. |
| **Delete Index** | Removes that item's backend record and local index marker without deleting its WordPress content. |

### Source action

**Delete All Index** removes indexed backend records for the active source and clears the corresponding local index state. It does not delete the original posts, listings, or reviews, and it does not disable the source. If automatic indexing remains enabled, later content changes or another **Index All** action can index the content again.

## 7. Create the full chat page

1. Open **Pages > Add New Page**.
2. Give the page a clear title such as “Ask Sunny.”
3. Add a Shortcode block containing:

   ```text
   [ask_sunny_chat_app]
   ```

4. In the page template settings, optionally select **Ask Sunny SPA** for a standalone page containing only the page content and required WordPress assets.
5. Publish the page.
6. Return to **Ask Sunny > Settings > Chat**.
7. Under **Select Chat Page**, search for and choose the published page.
8. Choose **Save Changes**.

The selected page is the destination used by the compact chat form. If no chat page is selected, the compact form falls back to the site home URL, which normally will not host the full chat app.

Avoid adding `[ask_sunny_chat_app]` more than once to the same page. Each instance is separately initialized and would show a separate conversation interface.

## 8. Add the compact chat form

Add this shortcode to any page, post, or shortcode-compatible widget area where visitors should begin a question:

```text
[ask_sunny_chat_form]
```

When a visitor submits a question, the form opens the selected chat page and automatically sends that question. A visitor with an existing conversation may instead use **Continue previous chat**.

Under **Settings > Chat**, configure:

- **Chat Form Submit Button** for the compact form's action label.
- **Chat Form Placeholder** for the input placeholder used by both chat experiences.
- **Chat Quick Starters** for the starter heading and prompt buttons. Selecting a starter fills the compact input; the visitor can edit it before submitting.

Up to 20 unique, non-empty starter prompts are accepted. Keep starters short, specific, and answerable from enabled content, for example:

- “What family events are happening this weekend?”
- “Find indoor activities for young children.”
- “Show family-friendly restaurants near me.”

Choose **Save Changes** after editing chat settings.

## 9. Customize the appearance

Open **Ask Sunny > Settings > Chat Appearance**. Use valid six-digit hexadecimal colors such as `#F5342C`.

### General

- **Primary Button** controls the compact Ask button and full-chat send button.
- **Secondary Button** controls conversation start and continuation actions.
- **Chat Input** controls the compact input and full-chat composer background and border.

### Branding

- Select a **Brand Logo** from the WordPress Media Library.
- Set the brand title and subtitle text.
- Choose Figtree, Dancing Script, or Playfair Display for each brand line.
- Select title and subtitle colors.

Add meaningful alternative text to a custom brand-logo attachment in the Media Library. The assistant avatar is decorative in the conversation and does not need to repeat the assistant's name.

### Chat app

- Select the assistant avatar.
- Use a solid color or gradient for user messages and the chat body.
- Customize the user-message text color.
- Edit the typing and idle status hints.
- Configure container border width, color, radius, and optional shadow.

Gradient angles must be from 0 through 360 degrees. Border width supports 0–10 pixels, border radius 0–80 pixels, and shadow opacity 0–100 percent.

### Chat form

Set the quick-starter title color and chip background/text colors. Check contrast on both desktop and mobile before publishing.

Choose **Save Changes**, then reload a frontend page containing each shortcode to confirm the result. Clear page or asset caches if the old appearance remains visible.

## 10. Perform a launch smoke test

Test in a private browser window and while signed in:

1. Confirm the compact form loads on its intended page.
2. Submit a starter and a typed question.
3. Confirm the selected full chat page opens and sends the question.
4. Verify the answer and recommendation links are relevant to indexed public content.
5. Ask a follow-up question and reload the page to check conversation restoration.
6. Choose **Start new conversation** and confirm the previous conversation is cleared from the interface.
7. Test an expected empty or unavailable result and confirm the interface fails safely.
8. Check keyboard operation, visible focus, and layout at a narrow mobile width and 200% browser zoom.
9. Open **Connection** and run **Test Connection** only after confirming the backend and credentials are available.
10. Inspect Data Sources for failed or unexpectedly unindexed content.

For backend upgrades, follow the [deployment flow](../shared/SETUP_AND_OPERATIONS.md#deployment-flow). For rollback and recovery, follow [Backup and Recovery](../shared/SETUP_AND_OPERATIONS.md#backup-and-recovery). Validate releases against the [Production Release Contract](../server/PRODUCTION_RELEASE_CONTRACT.md).

## 11. Deactivation and uninstall

Deactivation stops scheduled Ask Sunny work and clears the reindex lock, while preserving settings, queue/index state, and content metadata for reactivation.

Uninstall also preserves Ask Sunny data by default. To deliberately remove plugin-owned local data, the server-side `ask_sunny_delete_data_on_uninstall` option must be the boolean value `true` before uninstall. This opt-in deletion does not silently delete backend content, so coordinate backend deletion separately when required.

Review [Privacy and retention](../server/CONVERSATION_CONTEXT_CONTRACT.md#7-deletion-anonymization-and-retention) before uninstalling or responding to a data-subject request.
