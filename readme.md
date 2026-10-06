# How to Write a Plugin

Plugins add new commands to the bot. A plugin is one small JavaScript file. You write it, upload it as a GitHub Gist, and install it with one command. No restart is needed.

---

## 1. The smallest plugin

```js
import { Module } from "../lib/plugins.js";

Module({
  command: "hello",
  package: "general",
  description: "Say hello",
})(async (message, match) => {
  await message.reply(`Hello ${message.pushName || "there"}!`);
});
```

Send `.hello` in a chat and the bot answers. That is the whole structure:

1. Import `Module` from `../lib/plugins.js` (keep this path exactly as written).
2. Call `Module({ ...options })` with the details of your command.
3. Pass it an `async` function. The bot runs this function every time someone uses the command.

The function receives two things:

| Argument | What it is |
|---|---|
| `message` | The incoming message. It has info about the chat and methods for replying (see section 3). |
| `match` | The text typed after the command. For `.hello world`, `match` is `"world"`. It is an empty string when nothing follows. |

---

## 2. Options for `Module({ ... })`

| Option | Required | Meaning |
|---|---|---|
| `command` | yes* | The command name, one lowercase word, no prefix. `"hello"` runs with `.hello`. |
| `aliases` | no | Extra names for the same command, like `["hi", "hey"]`. |
| `package` | no | Menu group: `general`, `tools`, `downloader`, `audio`, `media`, `group`, `search`, `info`, `owner`. |
| `description` | no | One short line shown in the menu. |
| `fromMe` | no | Set `true` so only the bot owner can use the command. |
| `on` | no | Use `"text"` for a plugin that runs on every message instead of a command (section 6). |

\* Use either `command` or `on`.

Rules for names:
- Use lowercase letters and numbers only, with no spaces.
- Do not reuse the name of a built-in command. If two plugins share a name, the newest one takes the name and the old one is switched off until you remove the new one.

---

## 3. What you can do with `message`

### Information

| Field | Meaning |
|---|---|
| `message.from` | Chat ID (group or private). |
| `message.sender` | ID of the person who sent the command. |
| `message.pushName` | Their WhatsApp display name. |
| `message.body` | Full text of the message. |
| `message.isGroup` | `true` in a group chat. |
| `message.isFromMe` / `message.isOwner` | `true` if the sender is the bot owner. |
| `message.mentions` | List of people mentioned in the message. |
| `message.quoted` | The message being replied to, or empty. It has `type`, `text`, `sender`, and `download()`. |
| `message.type` | Type of the message, like `imageMessage`. |

### Actions

| Call | What it does |
|---|---|
| `await message.reply(payload)` | Reply to the user's message (shows as a quoted reply). |
| `await message.send(payload)` | Send a message to the same chat. |
| `await message.react("✅")` | React to the user's message with an emoji. |
| `await message.download()` | Download the media in the user's own message. |
| `await message.quoted.download()` | Download the media in the message they replied to. |
| `await message.loadGroupInfo()` | Load group details (admins, members). Group chats only. |

### What `payload` can be

```js
await message.reply("plain text");
await message.send({ image: buffer, caption: "A caption" });
await message.send({ video: buffer, caption: "A video" });
await message.send({ audio: buffer, ptt: true });      // ptt: true = voice note
await message.send({ sticker: buffer });
await message.send("Hi @user", { mentions: [message.sender] });
```

`image`, `video`, and `audio` accept a `Buffer` or `{ url: "https://..." }`.

---

## 4. Examples

### A. Command with an argument

```js
import { Module } from "../lib/plugins.js";

Module({
  command: "say",
  package: "general",
  description: "Repeat your text",
})(async (message, match) => {
  if (!match) return message.reply("Usage: .say <text>");
  await message.reply(match);
});
```

### B. Work on an image the user replied to

```js
import { Module } from "../lib/plugins.js";

Module({
  command: "sendback",
  package: "tools",
  description: "Send back the image you reply to",
})(async (message) => {
  if (!message.quoted || !/imageMessage/.test(message.quoted.type)) {
    return message.reply("Reply to an image.");
  }
  const buffer = await message.quoted.download();
  await message.send({ image: buffer, caption: "Here it is" });
});
```

### C. Fetch data from the web

```js
import axios from "axios";
import { Module } from "../lib/plugins.js";

Module({
  command: "fact",
  package: "general",
  description: "Random fact",
})(async (message) => {
  try {
    await message.react("⏳");
    const { data } = await axios.get("https://uselessfacts.jsph.pl/api/v2/facts/random", {
      timeout: 15000,
    });
    await message.reply(data.text);
    await message.react("✅");
  } catch (err) {
    await message.react("❌");
    await message.reply("Could not get a fact. Try again later.");
  }
});
```

### D. Group admins only

```js
import { Module } from "../lib/plugins.js";

Module({
  command: "admincheck",
  package: "group",
  description: "Check whether you are an admin",
})(async (message) => {
  if (!message.isGroup) return message.reply("This works in groups only.");
  await message.loadGroupInfo();
  if (!message.isAdmin) return message.reply("Only group admins can use this.");
  await message.reply("You are an admin.");
});
```

---

## 5. Writing good plugins

- **Always use `try/catch`** around anything that can fail (web requests, downloads). Send the user a short friendly error instead of failing silently.
- **Set a timeout** on web requests, like `{ timeout: 15000 }`.
- **Check the input first.** If the user forgot something, show a usage line such as `Usage: .say <text>`.
- **Use `react`** (`⏳`, `✅`, `❌`) so users know the command is working.
- **Keep it small.** One file, one job.
- **Use only packages the bot already has**, such as `axios`, `cheerio`, and `fs-extra`. If a package is missing, the installer tells you its name.
- **Never put passwords, API keys, or tokens in the file.** A gist can be read by anyone with the link.

---

## 6. Plugins that run on every message

Set `on: "text"` instead of `command`. The function then receives only `message` and runs for every message in every chat, so keep it fast and always check the text first.

```js
import { Module } from "../lib/plugins.js";

Module({
  on: "text",
  description: "Reply when someone says good morning",
})(async (message) => {
  if (!message.body) return;
  if (/^good morning$/i.test(message.body.trim())) {
    await message.reply("Good morning! ☀️");
  }
});
```

---

## 7. Install your plugin

1. Go to <https://gist.github.com> and create a new gist.
2. Paste your plugin code and save it. Name the file anything ending in `.js`.
3. Copy the gist link, for example `https://gist.github.com/yourname/abc123def456...`
4. Send this to the bot from the owner number:

```
.plugin https://gist.github.com/yourname/abc123def456...
```

The bot downloads the code, checks it, and activates it right away.

| Command | What it does |
|---|---|
| `.plugin <gist link>` | Install a plugin. Run it again with the same link to update it after editing the gist. |
| `.plugins` | List installed plugins. |
| `.remove <name>` | Remove a plugin by its command name. |

Only the bot owner or developers can install plugins. Only GitHub gist links are accepted. Installed plugins come back automatically after a restart.

---

## 8. Safety check

The installer refuses code that does any of these, to protect the bot and its owner:

- Runs shell commands (`child_process`, `exec`, `spawn`)
- Stops the bot (`process.exit`, `process.kill`)
- Deletes files (`unlinkSync`, `rmSync`, `fs.rm`)
- Runs dynamic code (`eval`, `new Function`)
- Reads secrets or sessions (`process.env`, `config.env`, `creds.json`, `sessions/`)

You will see a warning if your plugin matches one of these. A plugin should not need any of them. Only the bot owner can bypass the warning, and should do so only for code they fully trust.

---

## 9. Quick checklist

- [ ] First line imports `Module` from `"../lib/plugins.js"`
- [ ] `command` is one lowercase word and is not already used
- [ ] The function is `async` and uses `await` for every `message.reply/send`
- [ ] Web calls and downloads are inside `try/catch`
- [ ] No secrets, no shell commands, no file deletion
- [ ] Tested with the usage line, with empty input, and with normal input
