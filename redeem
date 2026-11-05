# Glamrock Freddy Ultimate Update

Includes Freddy character chat (mention `@Freddy` or use `/freddy_chat`), music controls and queue, friendship points, Security Breach roleplay, scary jumpscare embed, and an owner-only admin panel.

## Replit setup
1. Upload/extract the ZIP into your Replit project.
2. In Replit Secrets, set:
   - `DISCORD_BOT_TOKEN` = your Discord bot token (never share it)
   - `DISCORD_OWNER_ID` = `1411436991005851832`
   - `FREDDY_ADMIN_CODE` = a long, private random code (do not send it in chat)
3. Install dependencies: `pip install -r requirements.txt`
4. Install FFmpeg in the runtime and enable **Message Content Intent** in the Discord Developer Portal for mention-based chat.
5. Run `freddy.py`.


## Commands
- `/freddy`, `/freddy_chat`, `/hug`, `/friendship`, `/protect`, `/joke`, `/sing`, `/roast`, `/roar`
- `/roleplay`, `/jumpscare`
- `/solver`, `/solver_action`, `/solver_status` (Absolute Solver powers)
- `/play`, `/pause`, `/resume`, `/skip`, `/queue`, `/volume`, `/stop`
- `/freddy_admin`, `/admin_verify`, `/status`, `/help`

## Music files
- `04 Rockstar Row.mp3` is included.
- `happy_true_ending.mp3` is an optional local audio file. Add an authorized copy beside `freddy.py` to enable that selection.

## Notes
- Freddy chat is a lightweight local character-response system, not a connected external AI model.
- Friendship data is stored in `freddy_data.json`.
- Admin verification resets when the bot restarts.
- The admin panel is owner-ID restricted. The security code is configured only in host Secrets.
- Discord voice playback requires FFmpeg, PyNaCl, and the bot's Connect/Speak permissions.


## Multiple owner accounts
Both owner accounts are configured by default:
- Main: `1411436991005851832`
- Alt: `1552785126033858650`

Both accounts can access `/freddy_admin` and receive private security notifications.
To change the owners, edit `DISCORD_OWNER_IDS` in your host environment/Secrets as a comma-separated list. Do not put your Discord bot token or private admin code in chat.


### Voice-channel greeting audio
The bot includes `glamrock_freddy.mp3`. When Freddy joins a voice channel after a member, he plays this clip once. Ensure FFmpeg is available in the hosting environment for Discord voice audio playback.


### Freddy family lore
Freddy's bot-side character responses describe Gregory (bot ID `1560782109180952707`) as his son and Cyn (bot ID `1549065500313784491`) as his daughter. This family lore is configured only in the bot code and is not exposed as a website control.


### Lockdown announcement
Use `/freddy lockdown` while you are in a voice channel. Freddy joins that channel and plays `glamrock-freddy.mp3`. The original greeting command is now `/freddy hello`. FFmpeg and the bot's Connect/Speak voice permissions are required.
