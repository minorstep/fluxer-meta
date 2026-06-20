# Threads Fluxer-native Design Packet

This packet addresses issue #5 with a focused design proposal for desktop, mobile, API, permissions, moderation and instance-admin handling.

It is intentionally anchored in the public Fluxer clients instead of inventing a parallel visual system:

- Desktop reference: `fluxerapp/fluxer` marketing screenshot and web client structure.
- Mobile reference: `fluxerapp/flutter_client`, especially `GuildSidebar`, channel rows, bottom sheets, drawers, theme tokens and Phosphor-style icon language.
- Feedback addressed from prior review: keep the main Fluxer design, use a two-control expiry selector, and avoid mockups that introduce unrelated product decisions.

## Included Artefacts

- [Desktop flow board](desktop-thread-flow.svg)
- [Mobile flow board](mobile-thread-flow.svg)

## Design Principles

Threads should feel like a thin extension of channels, not a separate product area.

- Reuse existing channel row density, text hierarchy, hover surfaces, active row treatment and 4px corner language.
- Use Fluxer's primary accent only to identify thread-specific surfaces: inline thread cards, active thread rows and focus states.
- Preserve the existing message hover action strip. The thread action is inserted between reply and forward, as requested.
- On mobile, use the current app's drawer and bottom-sheet patterns. Avoid a desktop modal squeezed into a phone.
- Keep closed and archived states visible in thread lists without making them look like normal live channels.

## Source-Aligned Tokens

The boards use Fluxer-like tokens, not arbitrary colours.

| Token | Usage |
| --- | --- |
| `backgroundPrimary` | Message canvas and desktop chat body |
| `backgroundSecondary` | Channel sidebar, mobile drawer and settings panels |
| `backgroundTertiary` | Server rail, floating panels and raised sheets |
| `backgroundModifierSelected` | Selected channel/thread row |
| `backgroundModifierHover` | Hovered message and menu rows |
| `brandPrimary` | Thread border, active thread affordance and primary confirm action |
| `textPrimary` | Channel names, thread names and core message text |
| `textChatMuted` | timestamps, secondary metadata and closed/archive state |

## Desktop Flow

### Start Or Join From A Message

1. Message hover keeps the existing Fluxer action strip.
2. Insert the thread icon between reply and forward.
3. Tooltip reads `Start new thread` unless the message already owns a thread, where it reads `Join thread`.
4. Right-click and hamburger menus expose the same action label.
5. `/thread` in the composer opens the same create modal without a parent preview.

### Create Thread Modal

The expiry selector is deliberately two controls, matching maintainer feedback:

- Numeric field: default `7`.
- Unit menu: default `days`, with `hours`, `days`, `weeks` as allowed units.

The modal includes:

- Thread name.
- Expiry controls.
- Parent message preview, omitted for `/thread`.
- Cancel and create actions.
- Inline permission failure copy if permissions change before confirm.

### Inline Thread Card

After creation, the parent message receives an indented thread card:

- A thin connector line from the parent message stack.
- 1px `brandPrimary` border.
- Thread icon, thread name, last sender avatar/name, latest snippet and relative timestamp.
- Closed and archived states keep the card visible but muted.

### Sidebar Behaviour

- Joined threads appear under the parent channel, indented by 18px.
- Previewed threads appear in the same position but are marked `Preview` and disappear when the user navigates away.
- Thread rows use the same channel row hit area, active background and unread indicator logic as normal channels.
- Multiple joined threads are ordered by creation time under their parent channel.

### Topbar Thread List

- Replace the notification settings button with the thread list icon.
- Place it between pinned messages and favourites.
- The dropdown mirrors pinned-message density and ordering: latest activity first, no live resort while open.
- Right-click actions depend on permission and state: join, leave, open, close, archive, unarchive, delete.

## Mobile Flow

### Message Actions

Mobile should use a long-press bottom sheet or message action sheet, not a desktop hover metaphor.

- Sheet title uses parent message author and timestamp.
- `Start new thread` or `Join thread` appears near reply and forward actions.
- Users without permission do not see the thread action.

### Create Thread Sheet

The create flow is a single bottom sheet:

- Thread name field.
- Expiry row with two controls: number and unit.
- Parent message preview.
- Create and cancel actions.
- Inline permission failure copy.

This aligns with Flutter mobile's sheet-heavy interaction model and keeps the flow reachable on small devices.

### Drawer And Thread Rows

- Joined threads appear in the channel drawer under the parent text channel.
- Previewed threads appear temporarily with a `Preview` micro-label.
- Thread rows are indented, connected to the parent channel and use the thread icon instead of the hash icon.
- Closed threads can still be previewed and reopened by message send.
- Archived threads stay readable where permitted but cannot be reopened except by users with manage-channel permissions.

### Thread List Sheet

The topbar thread icon opens a full-height mobile sheet:

- Search field is optional for later, not required for MVP.
- Open, closed and archived threads are grouped with state chips.
- Rows show title, last activity, last sender avatar and latest snippet.
- Long press opens role-aware actions.

## API And Data Shape

Treat threads as sub-channels with explicit parent relationships.

```text
GET /channels/{communityId}/{channelId}/{threadId}/{messageId}
```

Recommended persisted fields:

- `channels.parent_channel_id`
- `channels.thread_id`
- `channels.thread_name`
- `channels.thread_creator_user_id`
- `channels.thread_creator_username`
- `channels.thread_state`: `open`, `closed`, `archived`, `deleted`
- `messages.thread_id`
- `messages.thread_name_snapshot`

Thread creation time should use the thread snowflake, not the parent channel timestamp.

## Permission Rules

- Threads inherit parent channel permissions and notification settings.
- Joined users can override a single thread without leaving it: `Inherit channel`, `All messages`, `Mentions only` or `Mute thread`.
- Muting a thread stops non-mention notifications for that thread only. Parent channel notifications and direct mentions still behave normally.
- Start-thread controls are hidden when the user lacks permission.
- If permissions change while the modal or sheet is open, confirm fails with inline copy.
- Users join by creating, sending a message or explicitly joining.
- Moderators and admins are not automatically joined.
- Bot parity is required: bots can create, send, receive and listen for thread state events.

## Thread Notifications

The notification control stays thread-scoped, not channel-scoped. This covers high-volume social threads such as birthday wishes without forcing the user to leave the thread or mute the whole parent channel.

- Desktop: right-click a joined thread row or open the thread list row menu, then choose `Thread notifications`.
- Mobile: use the bell action in the thread header or the long-press row action from the thread list sheet.
- Rows show a small `Muted` or `Mentions` status only when the thread differs from the parent channel default.
- Leaving a thread removes the row and its explicit notification override.

## Moderation And Admin

### Community Level

Users with manage-channel permissions can:

- close
- open
- archive
- unarchive
- delete

Any user who can view the parent channel can browse the thread list. Action rows are permission-filtered.

### Instance Level

Admin surfaces should not treat thread IDs as ambiguous channel IDs.

- Message Tools accepts a channel ID or thread ID.
- The label changes between `Browse Channel` and `Browse Thread`.
- The resolved line shows `Channel / Thread / Thread started by / Community`.
- Communities page nests threads under parent channels with a smaller, indented row.
- Categories are text headers rather than full boxes, preserving the requested visual order.

## Acceptance Map

| Requirement | Packet coverage |
| --- | --- |
| Desktop start/join entry points | Desktop board, create modal and menu notes |
| `/thread` composer command | Desktop flow notes |
| Two-control expiry selector | Desktop and mobile boards, spec copy |
| Inline thread card | Desktop and mobile boards |
| Joined vs preview thread rows | Desktop and mobile boards |
| Mobile designs | Mobile board and mobile flow section |
| Closed/archive/delete states | Thread list sheets, sidebar states and moderation rules |
| API/database shape | API and data shape section |
| Bot events | Permission and API rules |
| Admin Message Tools | Desktop board admin strip and admin section |
| Communities page thread nesting | Desktop board admin strip and admin section |
| Rollout prompt | Desktop board and notes |

## Non-Goals

- This packet does not introduce a new Fluxer theme.
- This packet does not propose forums, long-lived topic hubs or a multi-step wizard.
- This packet does not prescribe backend implementation details beyond the fields and events required to support the UX.
- This packet does not require mobile desktop parity where the platform pattern differs.
