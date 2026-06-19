# Flutter Skill for Claude Code

A comprehensive, **version-accurate Flutter & Dart development skill** for [Claude Code](https://claude.com/claude-code), packaged as an installable plugin via a Claude Code plugin marketplace.

It gives Claude a `SKILL.md` router plus **19 on-demand reference files** covering the modern Flutter stack (Dart 3, Material 3) — so Claude pulls the right, current API patterns instead of guessing. APIs are checked against pinned package versions (e.g. `flutter_bloc` 9, `firebase_auth` 6, `in_app_purchase` 3, `flutter_map` 8, `geolocator` 14, `flutter_local_notifications` 22).

## Install (via the marketplace)

In Claude Code:

```text
/plugin marketplace add Arcturus91/claude-flutter-skill
/plugin install flutter@flutter-skill
```

That's it. The skill auto-activates whenever you work on Flutter/Dart code, and is available in **every repo** on your machine. Update later with:

```text
/plugin marketplace update flutter-skill
/plugin update flutter@flutter-skill
```

### Alternative: install as a personal skill (no plugin)

Copy the skill straight into your user-level skills directory:

```bash
git clone https://github.com/Arcturus91/claude-flutter-skill
cp -R claude-flutter-skill/skills/flutter ~/.claude/skills/flutter
```

## What's inside

It's a **reference skill**: `SKILL.md` routes to the right topic; Claude loads only the reference file(s) it needs.

| Area | File |
|---|---|
| Dart language essentials | `references/dart-essentials.md` |
| Async / streams / isolates / lifecycle | `references/async-streams-isolates.md` |
| Widgets & layout | `references/widgets-and-layout.md` |
| State management (BLoC / Cubit) | `references/state-management.md` |
| Navigation & routing (flow_builder, go_router) | `references/navigation-and-routing.md` |
| Forms & input (formz) | `references/forms-and-input.md` |
| Theming & Material 3 | `references/theming-material3.md` |
| Networking & REST (http) | `references/networking-rest.md` |
| WebSockets & realtime | `references/websockets-realtime.md` |
| Firebase core & auth | `references/firebase-core-auth.md` |
| FCM & local notifications | `references/firebase-messaging-fcm.md` |
| Maps & location (flutter_map) | `references/maps-and-location.md` |
| Media, files & assets | `references/media-files-assets.md` |
| In-app purchases | `references/in-app-purchase.md` |
| Animations | `references/animations.md` |
| Testing | `references/testing.md` |
| Performance & DevTools | `references/performance-and-devtools.md` |
| Tooling, build & deploy | `references/tooling-build-deploy.md` |
| Architecture & conventions | `references/architecture-conventions.md` |

Each file follows the same shape: **Overview → Quick reference → Real-world usage → Common mistakes → See also** (with cited official doc URLs), and leans on tables and runnable Dart over prose.

## How it works

Claude Code reads `SKILL.md`'s description to decide when the skill is relevant, then opens the specific reference file(s) for the task at hand. Because it's a router + on-demand references, it stays token-light until needed.

## Opinionated, but transferable

The examples reflect a proven production setup — Cubit/BLoC + `equatable` + `formz`, a single static `ApiService` with JWT injection, `flow_builder` auth-gated routing, a singleton `web_socket_channel` service with reference counting and backoff, Firebase auth + FCM with a notification event bus, `flutter_map` (OpenStreetMap), and Material 3 theming. Adapt the patterns to your own app; the APIs and structure are standard Flutter.

## Contributing

Issues and PRs welcome. Keep code examples version-accurate and add the relevant official doc URL under **See also**.

## License

[MIT](LICENSE) © Arturo Barrantes Vasquez
