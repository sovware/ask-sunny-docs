# Ask Sunny Administrator Documentation

Ask Sunny is a WordPress plugin that lets visitors ask conversational questions whose answers are grounded in public content from the site. It connects WordPress to the Ask Sunny backend, indexes Directorist listings and reviews, can index other public WordPress content, and provides compact and full-page chat experiences.

This documentation describes Ask Sunny 0.1.0 as implemented in this plugin. It is intended for WordPress administrators who install, configure, publish, and maintain Ask Sunny.

## Documentation map

- [Features](FEATURES.md) explains the plugin's capabilities and important behavior.
- [Administrator guide](ADMIN_GUIDE.md) covers installation, connection, indexing, publishing the chat experience, and appearance settings.
- [Troubleshooting](TROUBLESHOOTING.md) provides checks for common connection, indexing, chat, and caching problems.

The plugin also includes deeper operational references:

- [Installation, upgrade, and rollback](../docs/INSTALLATION.md)
- [Compatibility matrix](../docs/COMPATIBILITY.md)
- [Privacy and retention](../docs/PRIVACY.md)
- [Performance thresholds](../docs/PERFORMANCE.md)

## Requirements

Ask Sunny supports:

- PHP 7.4 through 8.4.
- WordPress 6.5 through 7.0.x.
- Directorist 8.8 or newer.
- A reachable Ask Sunny backend. Production backend URLs must use HTTPS; loopback HTTP URLs are supported for local development.
- A server-side Ask Sunny provisioning key.
- An OpenAI API key for the currently available OpenAI provider and GPT 5.4 Mini model.
- An administrator account with the `manage_options` capability.

Directorist must be active and supported. The plugin will refuse to activate or will not boot its Directorist features when required runtime dependencies are missing or unsupported.

## Quick start

1. Back up the WordPress site and install Directorist.
2. Install and activate the official `ask-sunny-<version>.zip` package.
3. Define `ASK_SUNNY_PROVISIONING_KEY` in server configuration.
4. Open **Ask Sunny** in WordPress administration and connect the plugin to the backend.
5. Open **Settings > Data Sources**, enable automatic indexing, and select any additional public WordPress post types.
6. Return to **Data Sources** and use **Index All** for each source that should be searchable.
7. Create a published chat page containing `[ask_sunny_chat_app]`, then select that page under **Settings > Chat**.
8. Place `[ask_sunny_chat_form]` wherever visitors should be able to begin a question.
9. Test the connection, submit a frontend question, follow up in the same conversation, and verify the returned recommendations.

See the [administrator guide](ADMIN_GUIDE.md) for the complete procedure.

## Important terms

| Term | Meaning |
|---|---|
| Backend URL | The Ask Sunny service URL contacted by WordPress. |
| Provisioning key | A server-side credential used only to create or rotate this site's installation connection. It must not be exposed in WordPress fields or public pages. |
| AI provider API key | The OpenAI key submitted during connection or provider update. WordPress retains only masked provider metadata returned by the backend. |
| Installation key | The installation-scoped credential issued by the backend and stored server-side in WordPress. The dashboard exposes only its non-secret prefix. |
| Data source | A collection of eligible content, such as Directorist listings, listing reviews, posts, pages, or another public post type. |
| Index | The backend search representation used to ground Ask Sunny's answers in site content. |
| Conversation | A multi-turn visitor chat that can be restored using an identity-isolated conversation identifier. |

## Security reminders

- Never paste `ASK_SUNNY_PROVISIONING_KEY` into the WordPress dashboard, browser console, screenshots, or support tickets.
- Never include API keys, visitor messages, private content, raw database rows, or request bodies in support material.
- Use HTTPS for the production WordPress site and backend.
- Treat **Delete Index** and **Delete All Index** as destructive backend operations. They do not delete the original WordPress or Directorist content.

