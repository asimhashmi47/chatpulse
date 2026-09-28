# PulseChat

A small, Microsoft Teams-inspired real-time chat application built as an MVP portfolio project.

> **This is an MVP.** It intentionally covers a small, focused feature set — private 1:1 messaging with
> presence and typing indicators — rather than a full enterprise chat platform.

## Screenshots

_Add a screenshot of the Chat page here, e.g. `![PulseChat](docs/screenshot.png)`._

## Features

- **Authentication** — register, log in, log out (ASP.NET Core Identity)
- **User list** — see every other registered user with a live online/offline indicator
- **Private 1:1 chat** — persisted conversation history, sent in real time
- **Typing indicator** — see when the other person is typing
- **Delete your own messages** — soft-deleted, replaced with `[deleted]` for both participants
- **Responsive UI** — usable on desktop, tablet and mobile

## Tech stack

| Layer | Technology |
|---|---|
| Backend | ASP.NET Core 10, C#, Razor Pages |
| Real-time | SignalR |
| Data access | Entity Framework Core |
| Database | SQLite |
| Auth | ASP.NET Core Identity |
| Styling | Hand-written CSS + Bootstrap primitives |

## Architecture

A lightweight Clean Architecture layout, kept proportional to the size of the project:

```text
PulseChat.sln
src/
  PulseChat.Domain/          Plain entities (ChatMessage) — no dependencies
  PulseChat.Application/     Business rules (ChatService), DTOs, repository interfaces
  PulseChat.Infrastructure/  EF Core DbContext, Identity (ApplicationUser), repository implementation
  PulseChat.Web/             Razor Pages UI, SignalR hub, composition root (Program.cs)
tests/
  PulseChat.Tests/           xUnit tests for Application and Infrastructure
```

`PulseChat.Web` depends on `Application` and `Infrastructure`; `Infrastructure` depends on `Application`
and `Domain`; `Application` depends only on `Domain`. `ChatService` is unit-tested against an in-memory
fake repository, independent of EF Core.

## How SignalR works

`ChatHub` (`src/PulseChat.Web/Hubs/ChatHub.cs`) is the single hub, mapped at `/hubs/chat`. Each connection
is added to a SignalR group named after the connected user's id, so **all of a user's open tabs** receive
events sent to `Clients.Group(userId)`.

- `SendMessage(receiverId, content)` — validates and persists the message via `IChatService`, then pushes
  `ReceiveMessage` to both the sender's and receiver's groups.
- `DeleteMessage(messageId, otherUserId)` — soft-deletes the message (only if the caller owns it) and
  broadcasts `MessageDeleted` to both sides.
- `Typing` / `StopTyping` — forwarded to the other user's group.
- `OnConnectedAsync` / `OnDisconnectedAsync` — a singleton `PresenceTracker` counts live connections per
  user (supporting multiple open tabs) and broadcasts `UserConnected` / `UserDisconnected` on the first
  connect / last disconnect.

The browser side (`wwwroot/js/chat.js`) is a small vanilla-JS client using `@microsoft/signalr`
(vendored locally under `wwwroot/lib`) — no framework required.

## Project structure

```text
src/PulseChat.Domain/Entities/ChatMessage.cs        Domain entity
src/PulseChat.Application/Chat/                     IChatService, ChatService, DTOs, validation rules
src/PulseChat.Infrastructure/Identity/               ApplicationUser (extends IdentityUser)
src/PulseChat.Infrastructure/Persistence/            ApplicationDbContext, ChatMessageRepository, migrations
src/PulseChat.Web/Pages/Chat.cshtml(.cs)             Primary chat screen
src/PulseChat.Web/Pages/Profile.cshtml(.cs)          Edit display name
src/PulseChat.Web/Areas/Identity/Pages/Account/      Custom Login/Register pages (styled, with display name)
src/PulseChat.Web/Hubs/ChatHub.cs, PresenceTracker.cs SignalR hub + in-memory presence tracking
tests/PulseChat.Tests/                               ChatService, repository and validation tests
```

## How to run

Prerequisites: [.NET 10 SDK](https://dotnet.microsoft.com/download).

```bash
dotnet restore
dotnet build
dotnet run --project src/PulseChat.Web
```

The app applies any pending EF Core migrations automatically on startup, so the SQLite database
(`src/PulseChat.Web/app.db`, gitignored) is created the first time you run it — no manual step needed.
Open the printed `http://localhost:.../` URL, register two accounts (in separate browser profiles or
incognito windows, since sessions are cookie-based), and start chatting.

### Database setup (manual, optional)

If you'd rather apply migrations yourself instead of relying on the automatic startup migration:

```bash
dotnet tool install --global dotnet-ef
dotnet ef database update --project src/PulseChat.Infrastructure --startup-project src/PulseChat.Web
```

## Testing

```bash
dotnet test
```

Covers `ChatService` business rules (message validation, self-messaging, delete-own-message rule,
conversation retrieval/ordering) against an in-memory fake repository, `ChatMessageRepository` against
the EF Core InMemory provider, and the registration form's validation rules.

## Future improvements

- Group chats / channels
- Message editing and read receipts
- File and image attachments
- Push notifications for offline users
- Pagination for very long conversation histories

---

Built as a compact, self-contained portfolio project demonstrating ASP.NET Core + Razor Pages + EF Core
+ SQLite + ASP.NET Core Identity + SignalR working together.
