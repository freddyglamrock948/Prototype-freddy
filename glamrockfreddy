import os, random, json, logging, asyncio
from pathlib import Path
from collections import defaultdict, deque
import discord
from aiohttp import web
from discord import app_commands
from discord.ext import commands

TOKEN = os.getenv('DISCORD_BOT_TOKEN')
# Owner IDs are comma-separated. Both configured accounts receive owner notifications.
DEFAULT_OWNER_IDS = '1411436991005851832,1552785126033858650'
OWNER_IDS = {
    int(value.strip())
    for value in os.getenv('DISCORD_OWNER_IDS', DEFAULT_OWNER_IDS).split(',')
    if value.strip().isdigit()
}
OWNER_ID = min(OWNER_IDS) if OWNER_IDS else 0  # Legacy display/default owner ID
ADMIN_CODE = os.getenv('FREDDY_ADMIN_CODE', '')
WEBSITE_API_SECRET = os.getenv('FREDDY_WEB_API_SECRET', '')
WEBSITE_GUILD_ID = int(os.getenv('FREDDY_WEBSITE_GUILD_ID', '0') or 0)
WEBSITE_CHANNEL_ID = int(os.getenv('FREDDY_WEBSITE_CHANNEL_ID', '0') or 0)
WEBSITE_VOICE_CHANNEL_ID = int(os.getenv('FREDDY_WEBSITE_VOICE_CHANNEL_ID', '0') or 0)
WEBSITE_API_PORT = int(os.getenv('FREDDY_WEB_API_PORT', '8081') or 8081)

# Freddy's family lore. These IDs are bot accounts, not website settings.
GREGORY_BOT_ID = 1560782109180952707
CYN_BOT_ID = 1549065500313784491
FAMILY_BOT_IDS = {GREGORY_BOT_ID, CYN_BOT_ID}
website_runner = None
BASE_DIR = Path(__file__).resolve().parent
VC_GREETING_FILE = BASE_DIR / 'glamrock_freddy.mp3'
LOCKDOWN_AUDIO_FILE = BASE_DIR / 'glamrock-freddy.mp3'
MUSIC_FILES = {
    'happy': BASE_DIR / 'happy_true_ending.mp3',
    'rockstar': BASE_DIR / '04 Rockstar Row.mp3',
}
DATA_FILE = BASE_DIR / 'freddy_data.json'
logging.basicConfig(level=logging.INFO)
log = logging.getLogger('glamrock_freddy')
if not TOKEN:
    raise RuntimeError('Missing DISCORD_BOT_TOKEN. Add it to Replit Secrets / Environment Variables.')

intents = discord.Intents.default()
intents.members = True
intents.voice_states = True
intents.message_content = True  # Enable Message Content Intent in Discord Developer Portal.
bot = commands.Bot(command_prefix='!', intents=intents)
queues = defaultdict(deque)
verified_admins = set()
roleplay_enabled = True
music_volume = defaultdict(lambda: 0.6)

GREETINGS = ['Hello, Superstar! Freddy is here to help!', 'You are doing great! Keep going, Superstar!', 'Stay safe out there. I am watching over you!']
JOKES = ['Why did the animatronic bring a guitar? To make some byte-sized music!', 'What is Freddy\'s favorite kind of party? A rock-and-roll party!', 'Why did I go to the daycare? I wanted to brighten everyone\'s day!']
ROASTS = ['You have the stealth skills of a marching band, Superstar.', 'Even the STAFF bots saw that coming.', 'That plan needs a little more superstar energy.', 'Monty called. He said your aim needs practice.']
FRIENDLY = ['Of course, Superstar! I am always here to help.', 'You can count on me. I will keep you safe.', 'That sounds like a great idea! Let us do it together.', 'I believe in you, Superstar. You have got this!']

TRACK_NAMES = {'rockstar': 'Rockstar Row', 'happy': 'Happy (True) Ending'}

def load_data():
    try:
        return json.loads(DATA_FILE.read_text(encoding='utf-8'))
    except (OSError, json.JSONDecodeError):
        return {'friendship': {}, 'roleplay': {}, 'settings': {'welcomeMessage': '🐻🎸 Welcome to the Pizzaplex, {user}! I’m Glamrock Freddy. Stay safe, Superstar!', 'welcomeChannel': '', 'welcomeImage': '', 'goodbyeMessage': 'Goodbye, {user}! We hope to see you again at the Pizzaplex.', 'goodbyeChannel': '', 'goodbyeImage': ''}}

data = load_data()
def save_data():
    DATA_FILE.write_text(json.dumps(data, indent=2), encoding='utf-8')

DEFAULT_WEBSITE_SETTINGS = {
    'welcomeMessage': '🐻🎸 Welcome to the Pizzaplex, {user}! I’m Glamrock Freddy. Stay safe, Superstar!',
    'welcomeChannel': '',
    'welcomeImage': '',
    'goodbyeMessage': 'Goodbye, {user}! We hope to see you again at the Pizzaplex.',
    'goodbyeChannel': '',
    'goodbyeImage': '',
}
def website_settings():
    if not isinstance(data.get('settings'), dict):
        data['settings'] = {}
    for key, value in DEFAULT_WEBSITE_SETTINGS.items():
        data['settings'].setdefault(key, value)
    return data['settings']

def render_member_message(template, member):
    return str(template).replace('{user}', member.mention).replace('{server}', member.guild.name)

def make_embed(title, description, color=discord.Color.orange()):
    return discord.Embed(title=title, description=description, color=color)
def is_owner(interaction):
    return interaction.user.id in OWNER_IDS
def is_admin(interaction):
    return is_owner(interaction) or interaction.user.id in verified_admins

async def notify_owner(message):
    # Send the same private alert to every configured owner account.
    for owner_id in OWNER_IDS:
        try:
            owner = await bot.fetch_user(owner_id)
            await owner.send(message)
        except (discord.HTTPException, discord.Forbidden):
            log.warning('Could not DM configured owner ID %s.', owner_id)

async def play_next(guild_id):
    guild = bot.get_guild(guild_id)
    if not guild:
        return
    voice = guild.voice_client
    if not voice or not queues[guild_id]:
        return
    key = queues[guild_id].popleft()
    path = MUSIC_FILES[key]
    if not path.is_file():
        log.error('Music file missing: %s', path)
        await play_next(guild_id)
        return
    try:
        source = discord.PCMVolumeTransformer(discord.FFmpegPCMAudio(str(path)), volume=music_volume[guild_id])
        loop = bot.loop
        def after_play(error):
            if error:
                log.error('Audio playback error: %s', error)
            asyncio.run_coroutine_threadsafe(play_next(guild_id), loop)
        voice.play(source, after=after_play)
    except (discord.ClientException, OSError):
        log.exception('Could not start the next track.')

@bot.event
async def on_ready():
    await start_website_bridge()
    try:
        synced = await bot.tree.sync()
        log.info('Synced %s slash commands.', len(synced))
    except discord.HTTPException:
        log.exception('Failed to sync slash commands.')
    await bot.change_presence(activity=discord.Game(name='Protecting Superstars 🎸'))
    log.info('Logged in as %s', bot.user)

@bot.event
async def on_member_join(member):
    settings = website_settings()
    channel_id = str(settings.get('welcomeChannel', '') or '').strip()
    channel = member.guild.get_channel(int(channel_id)) if channel_id.isdigit() else member.guild.system_channel
    if not isinstance(channel, discord.TextChannel) or not channel.permissions_for(member.guild.me).send_messages:
        return
    message = render_member_message(settings.get('welcomeMessage', DEFAULT_WEBSITE_SETTINGS['welcomeMessage']), member)
    embed = discord.Embed(description=message, color=discord.Color.orange())
    image_url = str(settings.get('welcomeImage', '') or '').strip()
    if image_url.startswith(('https://', 'http://')):
        embed.set_image(url=image_url)
    try:
        await channel.send(embed=embed)
    except discord.HTTPException:
        log.exception('Could not send configured welcome message.')

@bot.event
async def on_member_remove(member):
    settings = website_settings()
    channel_id = str(settings.get('goodbyeChannel', '') or settings.get('welcomeChannel', '') or '').strip()
    channel = member.guild.get_channel(int(channel_id)) if channel_id.isdigit() else member.guild.system_channel
    if not isinstance(channel, discord.TextChannel) or not channel.permissions_for(member.guild.me).send_messages:
        return
    message = render_member_message(settings.get('goodbyeMessage', DEFAULT_WEBSITE_SETTINGS['goodbyeMessage']), member)
    embed = discord.Embed(description=message, color=discord.Color.dark_grey())
    image_url = str(settings.get('goodbyeImage', '') or '').strip()
    if image_url.startswith(('https://', 'http://')):
        embed.set_image(url=image_url)
    try:
        await channel.send(embed=embed)
    except discord.HTTPException:
        log.exception('Could not send configured goodbye message.')

@bot.event
async def on_message(message):
    if message.author.bot:
        return
    # Mention chat is enabled as a lightweight, local Freddy character response (no external AI service).
    if bot.user and bot.user in message.mentions:
        text = message.content.replace(f'<@{bot.user.id}>', '').replace(f'<@!{bot.user.id}>', '').strip()
        response = random.choice(FRIENDLY)
        if any(w in text.lower() for w in ('cyn', 'daughter', 'my daughter')):
            response = 'Cyn is my daughter, Superstar. She may be unusual, but I care about her and want her to be safe.'
        elif any(w in text.lower() for w in ('gregory', 'son', 'my son')):
            response = 'Gregory is my son, Superstar. I will always look out for him and help keep him safe.'
        elif any(w in text.lower() for w in ('my children', 'my kids', 'family')):
            response = 'Gregory and Cyn are my children. Family means looking after one another, Superstar.'
        elif any(w in text.lower() for w in ('scared', 'afraid', 'help', 'danger')):
            response = 'Do not worry, Superstar. Stay close to me. I will protect you!'
        elif any(w in text.lower() for w in ('where',)):
            response = 'I am here, Superstar. Let us check the security cameras and find a safe route.'
        elif any(w in text.lower() for w in ('joke', 'funny')):
            response = random.choice(JOKES)
        await message.reply(response, mention_author=False)
    await bot.process_commands(message)

async def play_vc_greeting(voice_client):
    # Play the supplied greeting audio once when Freddy enters a voice channel.
    if not voice_client or voice_client.is_playing() or not VC_GREETING_FILE.is_file():
        return
    try:
        source = discord.FFmpegPCMAudio(str(VC_GREETING_FILE))
        voice_client.play(source)
    except (discord.ClientException, OSError):
        log.exception('Could not play Freddy voice-channel greeting.')

@bot.event
async def on_voice_state_update(member, before, after):
    if member.bot or not member.guild:
        return
    voice = member.guild.voice_client
    if after.channel is not None:
        if voice is None:
            try:
                voice = await after.channel.connect()
                await play_vc_greeting(voice)
            except (discord.ClientException, discord.HTTPException, discord.Forbidden, OSError):
                log.exception('Could not join voice channel or play greeting.')
            return
        if voice.channel != after.channel:
            humans_here = [m for m in voice.channel.members if not m.bot]
            if not humans_here:
                try:
                    await voice.move_to(after.channel)
                    await play_vc_greeting(voice)
                except (discord.ClientException, discord.HTTPException, discord.Forbidden, OSError):
                    log.exception('Could not move voice channels or play greeting.')
        return
    if before.channel is not None and voice is not None and voice.channel == before.channel:
        humans_here = [m for m in before.channel.members if not m.bot]
        if not humans_here:
            try: await voice.disconnect()
            except (discord.ClientException, discord.HTTPException): log.exception('Could not disconnect.')

freddy_group = app_commands.Group(name='freddy', description='Glamrock Freddy commands.')

@freddy_group.command(name='hello', description='Say hello to Glamrock Freddy.')
async def freddy_hello(interaction: discord.Interaction):
    await interaction.response.send_message(random.choice(GREETINGS), embed=make_embed('🐻 Glamrock Freddy', 'Your superstar protector is here!'))

@freddy_group.command(name='lockdown', description='Activate lockdown and play Freddy’s security announcement.')
async def freddy_lockdown(interaction: discord.Interaction):
    voice_state = getattr(interaction.user, 'voice', None)
    if voice_state is None or voice_state.channel is None:
        await interaction.response.send_message(
            '🐻 Superstar, join a voice channel first so I can deliver the lockdown announcement.',
            ephemeral=True
        )
        return

    if not LOCKDOWN_AUDIO_FILE.is_file():
        await interaction.response.send_message(
            'The lockdown announcement audio file is missing from the bot installation.',
            ephemeral=True
        )
        return

    channel = voice_state.channel
    voice_client = interaction.guild.voice_client if interaction.guild else None
    try:
        if voice_client and voice_client.is_connected():
            if voice_client.channel != channel:
                await voice_client.move_to(channel)
        else:
            voice_client = await channel.connect()

        if voice_client.is_playing():
            voice_client.stop()

        voice_client.play(discord.FFmpegPCMAudio(str(LOCKDOWN_AUDIO_FILE)))
        await interaction.response.send_message(
            '🚨 **LOCKDOWN ACTIVATED.** Freddy is playing the security announcement in your voice channel.',
            ephemeral=True
        )
    except (discord.ClientException, discord.HTTPException, discord.Forbidden, OSError):
        log.exception('Unable to play Freddy’s lockdown announcement.')
        if not interaction.response.is_done():
            await interaction.response.send_message(
                'I could not play the announcement. Please check my Connect/Speak permissions and FFmpeg installation.',
                ephemeral=True
            )

bot.tree.add_command(freddy_group)

@bot.tree.command(name='freddy_chat', description='Chat with Freddy in character.')
@app_commands.describe(message='What would you like to say to Freddy?')
async def freddy_chat(interaction, message: str):
    text = message.lower()
    if any(w in text for w in ('cyn', 'daughter', 'my daughter')):
        answer = 'Cyn is my daughter, Superstar. I care about her and want her to be safe.'
    elif any(w in text for w in ('gregory', 'son', 'my son')):
        answer = 'Gregory is my son, Superstar. I will always look out for him and help keep him safe.'
    elif any(w in text for w in ('my children', 'my kids', 'family')):
        answer = 'Gregory and Cyn are my children. Family means looking after one another, Superstar.'
    elif any(w in text for w in ('scared', 'afraid', 'help', 'danger')):
        answer = 'Do not worry, Superstar. Stay close to me. I will protect you!'
    elif any(w in text for w in ('joke', 'funny')):
        answer = random.choice(JOKES)
    elif any(w in text for w in ('hug', 'love', 'friend')):
        answer = 'A big bear hug for you, Superstar! You are my friend.'
    else:
        answer = random.choice(FRIENDLY)
    await interaction.response.send_message(f'🐻 **Freddy:** {answer}')

@bot.tree.command(name='hug', description='Get a friendly Freddy hug.')
async def hug(interaction):
    await interaction.response.send_message(f'🫂 {interaction.user.mention}, come here, Superstar! Freddy gives you a big bear hug!')
    key = str(interaction.user.id)
    data['friendship'][key] = data['friendship'].get(key, 0) + 1
    save_data()

@bot.tree.command(name='friendship', description='Check your friendship points with Freddy.')
async def friendship(interaction):
    points = data['friendship'].get(str(interaction.user.id), 0)
    level = 'New Superstar' if points < 5 else 'Pizzaplex Pal' if points < 15 else 'Freddy\'s Best Friend'
    await interaction.response.send_message(f'💛 {interaction.user.mention}, you have **{points} friendship points** with Freddy! Rank: **{level}**.')

@bot.tree.command(name='protect', description='Freddy promises to protect someone.')
@app_commands.describe(member='The superstar Freddy should protect')
async def protect(interaction, member: discord.Member = None):
    target = member or interaction.user
    await interaction.response.send_message(f'🛡️ Don\'t worry, {target.mention}. I will protect you, Superstar!')

@bot.tree.command(name='joke', description='Ask Freddy for a joke.')
async def joke(interaction):
    await interaction.response.send_message(f'😄 {random.choice(JOKES)}')

@bot.tree.command(name='sing', description='Ask Freddy to sing.')
async def sing(interaction):
    await interaction.response.send_message('🎤 Freddy\'s imaginary concert starts now. Everybody clap, Superstar!')

@bot.tree.command(name='roast', description='Get a playful roast from Freddy.')
@app_commands.describe(member='Who Freddy should roast (optional)')
async def roast(interaction, member: discord.Member = None):
    target = member or interaction.user
    await interaction.response.send_message(f'🔥 {target.mention}, {random.choice(ROASTS)}')

@bot.tree.command(name='roar', description='Hear Freddy\'s dramatic text roar.')
async def roar(interaction):
    await interaction.response.send_message(embed=make_embed('🐻🎸 FREDDY ROAR!', 'ROOOOOAAARRR!!! WELCOME TO THE PIZZAPLEX, SUPERSTAR!'))

@bot.tree.command(name='roleplay', description='Start a Security Breach roleplay scene.')
@app_commands.choices(scene=[app_commands.Choice(name='Protect Gregory', value='gregory'), app_commands.Choice(name='Security alert', value='alert'), app_commands.Choice(name='Escape the Pizzaplex', value='escape')])
async def roleplay(interaction, scene: app_commands.Choice[str]):
    scenes = {
        'gregory': 'Gregory, I have located you! Stay behind me. We need to get you somewhere safe before the others find us.',
        'alert': '⚠️ SECURITY ALERT! Superstar, please remain calm. I am detecting movement nearby. Follow me!',
        'escape': 'The doors are almost open, Superstar! Keep moving. We can make it out of the Pizzaplex together!'
    }
    await interaction.response.send_message(embed=make_embed('🎮 Security Breach Roleplay', scenes[scene.value]))

@bot.tree.command(name='solver', description='Activate Freddy’s Absolute Solver powers.')
async def solver(interaction):
    embed = discord.Embed(
        title='🔺 ABSOLUTE SOLVER: ACTIVATED',
        description=(
            '🐻 Superstar… something is different.\n\n'
            '**[ NULL ] [ REALITY ] [ CONTROL ]**\n'
            'The Solver is online. Freddy’s systems are glitching—but he is still here to protect you.'
        ),
        color=discord.Color.from_rgb(255, 65, 35),
    )
    embed.set_footer(text='Glamrock Freddy • Absolute Solver')
    await interaction.response.send_message(embed=embed)


@bot.tree.command(name='solver_action', description='Choose an Absolute Solver action for Freddy.')
@app_commands.choices(action=[
    app_commands.Choice(name='Reality Glitch', value='glitch'),
    app_commands.Choice(name='Teleport', value='teleport'),
    app_commands.Choice(name='Solver Shield', value='shield'),
    app_commands.Choice(name='System Override', value='override'),
    app_commands.Choice(name='Solver Jumpscare', value='scare'),
])
async def solver_action(interaction, action: app_commands.Choice[str]):
    actions = {
        'glitch': ('🟥 REALITY GLITCH', 'The room flickers. Freddy’s eyes flash red, orange, and blue as reality briefly distorts around him.'),
        'teleport': ('🌀 SOLVER TELEPORT', 'A burst of Solver symbols—and Freddy instantly appears beside you. “I’m right here, Superstar!”'),
        'shield': ('🛡️ SOLVER SHIELD', 'A glowing Solver barrier surrounds Freddy and his friends. “Nobody gets past this shield!”'),
        'override': ('⚠️ SYSTEM OVERRIDE', 'Freddy’s systems display: **OVERRIDE ACCEPTED**. The Solver takes control of the nearby security interface.'),
        'scare': ('🔺 SOLVER JUMPSCARE', '**BOO!!!** Freddy’s eyes flash with Solver symbols. “SURPRISE, SUPERSTAR!”'),
    }
    title, description = actions[action.value]
    embed = discord.Embed(title=title, description=description, color=discord.Color.from_rgb(255, 65, 35))
    embed.set_footer(text='Glamrock Freddy • Absolute Solver')
    await interaction.response.send_message(embed=embed)


@bot.tree.command(name='solver_status', description='Check Freddy’s Absolute Solver status.')
async def solver_status(interaction):
    embed = discord.Embed(
        title='🔺 Absolute Solver Status',
        description='**ONLINE**\nPower: **Stable**\nControl: **Freddy**\nProtection: **Enabled**\n\nFreddy’s Solver powers are ready, Superstar.',
        color=discord.Color.from_rgb(255, 65, 35),
    )
    await interaction.response.send_message(embed=embed, ephemeral=True)


@bot.tree.command(name='jumpscare', description='Trigger a Freddy security-alert jumpscare.')
async def jumpscare(interaction):
    embed = discord.Embed(title='⚠️ SECURITY ALERT ⚠️', description='# BOO!!!\n\n**WAAAAAAAHHHHH!!!**\n\nSURPRISE, SUPERSTAR! 🐻', color=discord.Color.red())
    embed.set_footer(text='Glamrock Freddy • Jumpscare')
    await interaction.response.send_message(embed=embed)

@bot.tree.command(name='play', description='Play or queue a Glamrock Freddy soundtrack.')
@app_commands.describe(track='Choose a soundtrack')
@app_commands.choices(track=[app_commands.Choice(name='Rockstar Row', value='rockstar'), app_commands.Choice(name='Happy (True) Ending', value='happy')])
async def play(interaction, track: app_commands.Choice[str]):
    if not interaction.guild:
        return await interaction.response.send_message('Use this in a server.', ephemeral=True)
    member = interaction.guild.get_member(interaction.user.id)
    if not member or not member.voice or not member.voice.channel:
        return await interaction.response.send_message('🎸 Join a voice channel first, Superstar!', ephemeral=True)
    path = MUSIC_FILES[track.value]
    if not path.is_file():
        return await interaction.response.send_message(f'I cannot find `{path.name}` beside `freddy.py`.', ephemeral=True)
    await interaction.response.defer()
    voice = interaction.guild.voice_client
    try:
        if voice is None: voice = await member.voice.channel.connect()
        elif voice.channel != member.voice.channel: await voice.move_to(member.voice.channel)
        gid = interaction.guild.id
        if voice.is_playing() or voice.is_paused():
            queues[gid].append(track.value)
            await interaction.followup.send(f'🎵 Added **{TRACK_NAMES[track.value]}** to Freddy\'s queue.')
        else:
            queues[gid].appendleft(track.value)
            await play_next(gid)
            await interaction.followup.send(f'🐻🎸 Freddy is playing **{TRACK_NAMES[track.value]}**, Superstar!')
    except (discord.ClientException, discord.HTTPException, discord.Forbidden, OSError) as exc:
        log.exception('Music playback failed')
        await interaction.followup.send(f'Playback failed ({type(exc).__name__}). Check FFmpeg and Connect/Speak permissions.')

@bot.tree.command(name='pause', description='Pause Freddy\'s music.')
async def pause(interaction):
    voice = interaction.guild.voice_client if interaction.guild else None
    if voice and voice.is_playing():
        voice.pause(); await interaction.response.send_message('⏸️ Music paused, Superstar!')
    else: await interaction.response.send_message('Nothing is playing.', ephemeral=True)

@bot.tree.command(name='resume', description='Resume Freddy\'s music.')
async def resume(interaction):
    voice = interaction.guild.voice_client if interaction.guild else None
    if voice and voice.is_paused():
        voice.resume(); await interaction.response.send_message('▶️ Music resumed!')
    else: await interaction.response.send_message('Music is not paused.', ephemeral=True)

@bot.tree.command(name='skip', description='Skip to the next queued song.')
async def skip(interaction):
    voice = interaction.guild.voice_client if interaction.guild else None
    if voice and (voice.is_playing() or voice.is_paused()):
        voice.stop(); await interaction.response.send_message('⏭️ Skipping to the next track!')
    else: await interaction.response.send_message('Nothing is playing.', ephemeral=True)

@bot.tree.command(name='queue', description='Show Freddy\'s music queue.')
async def queue_command(interaction):
    q = list(queues[interaction.guild.id]) if interaction.guild else []
    if not q: text = 'The queue is empty.'
    else: text = '\n'.join(f'{i}. {TRACK_NAMES[k]}' for i, k in enumerate(q, 1))
    await interaction.response.send_message(embed=make_embed('🎶 Freddy\'s Queue', text), ephemeral=True)

@bot.tree.command(name='volume', description='Set Freddy\'s music volume (0-100).')
@app_commands.describe(percent='Volume percent from 0 to 100')
async def volume(interaction, percent: app_commands.Range[int, 0, 100]):
    gid = interaction.guild.id if interaction.guild else 0
    music_volume[gid] = percent / 100
    voice = interaction.guild.voice_client if interaction.guild else None
    if voice and voice.source and isinstance(voice.source, discord.PCMVolumeTransformer):
        voice.source.volume = percent / 100
    await interaction.response.send_message(f'🔊 Freddy\'s volume is now **{percent}%**.')

@bot.tree.command(name='stop', description='Stop Freddy\'s current music and clear the queue.')
async def stop(interaction):
    voice = interaction.guild.voice_client if interaction.guild else None
    if voice:
        queues[interaction.guild.id].clear()
        if voice.is_playing() or voice.is_paused(): voice.stop()
        await interaction.response.send_message('🐻🎸 Music stopped and queue cleared, Superstar!')
    else: await interaction.response.send_message('Freddy is not in a voice channel.', ephemeral=True)

class FreddyAdminView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=120)
    async def interaction_check(self, interaction):
        if not is_owner(interaction):
            await interaction.response.send_message('Owner-only control panel.', ephemeral=True)
            return False
        return True
    @discord.ui.button(label='Stop music in this server', style=discord.ButtonStyle.danger)
    async def stop_music(self, interaction, button):
        voice = interaction.guild.voice_client if interaction.guild else None
        if voice:
            queues[interaction.guild.id].clear()
            if voice.is_playing() or voice.is_paused(): voice.stop()
        await interaction.response.send_message('Admin action: music stopped.', ephemeral=True)
    @discord.ui.button(label='Show bot status', style=discord.ButtonStyle.primary)
    async def bot_status(self, interaction, button):
        await interaction.response.send_message(f'Online as {bot.user}. Guilds: {len(bot.guilds)}. Owner IDs: {', '.join(str(owner_id) for owner_id in sorted(OWNER_IDS))}.', ephemeral=True)

@bot.tree.command(name='freddy_admin', description='Open the owner-only Freddy admin panel.')
async def freddy_admin(interaction):
    if not is_owner(interaction):
        await interaction.response.send_message('⛔ This panel is restricted to the bot owner.', ephemeral=True)
        await notify_owner(f'⚠️ Unauthorized /freddy_admin attempt by {interaction.user} ({interaction.user.id}) in {interaction.guild}.')
        return
    await interaction.response.send_message('🔐 **Freddy Admin Panel** — owner verified.', view=FreddyAdminView(), ephemeral=True)

@bot.tree.command(name='admin_verify', description='Verify with the configured admin security code.')
@app_commands.describe(code='The private security code from your host secrets')
async def admin_verify(interaction, code: str):
    if not ADMIN_CODE:
        return await interaction.response.send_message('Admin code is not configured. Set FREDDY_ADMIN_CODE in host Secrets.', ephemeral=True)
    if code == ADMIN_CODE and (is_owner(interaction) or interaction.user.guild_permissions.manage_guild):
        verified_admins.add(interaction.user.id)
        await interaction.response.send_message('🔐 Admin verification successful. Access lasts until the bot restarts.', ephemeral=True)
        await notify_owner(f'🔐 Admin verification succeeded for {interaction.user} ({interaction.user.id}).')
    else:
        await interaction.response.send_message('❌ Invalid code or insufficient permissions.', ephemeral=True)
        await notify_owner(f'⚠️ Failed admin verification by {interaction.user} ({interaction.user.id}).')

@bot.tree.command(name='status', description='Check Freddy\'s status.')
async def status(interaction):
    await interaction.response.send_message(f'✅ Freddy is online and protecting **{interaction.guild.name if interaction.guild else "the server"}**!')

@bot.tree.command(name='help', description='Show Freddy\'s commands.')
async def help_command(interaction):
    text = ('**Character:** `/freddy` `/freddy_chat` `/hug` `/friendship` `/protect` `/joke` `/sing` `/roast` `/roar`\n'
            '**Roleplay/scares:** `/roleplay` `/jumpscare`\n'
            '**Absolute Solver:** `/solver` `/solver_action` `/solver_status`\n'
            '**Music:** `/play` `/pause` `/resume` `/skip` `/queue` `/volume` `/stop`\n'
            '**Admin:** `/freddy_admin` `/admin_verify`\n'
            '**Utility:** `/status` `/help`\n\nMention Freddy to get an in-character reply. Friendship points are saved locally.')
    await interaction.response.send_message(embed=make_embed('🎸 Freddy\'s Command List', text), ephemeral=True)


# Website-to-bot bridge. Set a long random FREDDY_WEB_API_SECRET and configure
# the website's FREDDY_BOT_API_URL / FREDDY_WEB_API_SECRET to match.
def website_auth(request):
    supplied = request.headers.get("X-Freddy-API-Key", "")
    return bool(WEBSITE_API_SECRET) and supplied == WEBSITE_API_SECRET

async def website_member(request):
    if not WEBSITE_GUILD_ID:
        raise web.HTTPServiceUnavailable(text="FREDDY_WEBSITE_GUILD_ID is not configured.")
    guild = bot.get_guild(WEBSITE_GUILD_ID)
    if not guild:
        raise web.HTTPServiceUnavailable(text="Configured website guild is not available to the bot.")
    user_id = int(request.get("user_id", 0))
    member = guild.get_member(user_id)
    if member is None:
        try:
            member = await guild.fetch_member(user_id)
        except (discord.NotFound, discord.Forbidden, discord.HTTPException):
            raise web.HTTPForbidden(text="Your Discord account is not a member of the configured server.")
    return guild, member

async def bridge_chat(request):
    if not website_auth(request):
        raise web.HTTPUnauthorized(text="Invalid website API key.")
    payload = await request.json()
    guild, member = await website_member(payload)
    message = str(payload.get("message", "")).strip()[:500]
    if not message:
        raise web.HTTPBadRequest(text="Please enter a message.")
    low = message.lower()
    if any(w in low for w in ("scared", "afraid", "help", "danger")):
        answer = "Do not worry, Superstar. Stay close to me. I will protect you!"
    elif any(w in low for w in ("gregory", "where")):
        answer = "Gregory? I am here, Superstar. Let us check the security cameras and find a safe route."
    elif any(w in low for w in ("joke", "funny")):
        answer = random.choice(JOKES)
    elif any(w in low for w in ("hug", "love", "friend")):
        answer = "A big bear hug for you, Superstar! You are my friend."
    else:
        answer = random.choice(FRIENDLY)
    if WEBSITE_CHANNEL_ID:
        channel = guild.get_channel(WEBSITE_CHANNEL_ID)
        if channel and hasattr(channel, "send"):
            try:
                await channel.send(f"💬 **Website chat — {member.display_name}:** {message}\n🐻 **Freddy:** {answer}")
            except discord.HTTPException:
                log.exception("Could not post website chat to Discord.")
    return web.json_response({"reply": answer})

async def bridge_action(request):
    if not website_auth(request):
        raise web.HTTPUnauthorized(text="Invalid website API key.")
    payload = await request.json()
    guild, member = await website_member(payload)
    action = str(payload.get("action", "")).lower()
    actions = {
        "reality glitch": ("🟥 REALITY GLITCH", "The room flickers. Freddy’s eyes flash red, orange, and blue as reality briefly distorts."),
        "teleport": ("🌀 SOLVER TELEPORT", "A burst of Solver symbols—and Freddy instantly appears beside you. “I’m right here, Superstar!”"),
        "solver shield": ("🛡️ SOLVER SHIELD", "A glowing Solver barrier surrounds Freddy and his friends. “Nobody gets past this shield!”"),
        "system override": ("⚠️ SYSTEM OVERRIDE", "Freddy’s systems display: **OVERRIDE ACCEPTED**. The Solver takes control of the nearby security interface."),
        "solver jumpscare": ("🔺 SOLVER JUMPSCARE", "**BOO!!!** Freddy’s eyes flash with Solver symbols. “SURPRISE, SUPERSTAR!”"),
        "jumpscare": ("⚠️ SECURITY ALERT ⚠️", "**BOO!!!** WAAAAAHHH! SURPRISE, SUPERSTAR! 🐻"),
        "joke": ("😄 Freddy’s Joke", random.choice(JOKES)),
        "roar": ("🐻🎸 FREDDY ROAR!", "ROOOOOAAARRR!!! WELCOME TO THE PIZZAPLEX, SUPERSTAR!"),
    }
    if action not in actions:
        raise web.HTTPBadRequest(text="Unknown action.")
    title, description = actions[action]
    if WEBSITE_CHANNEL_ID:
        channel = guild.get_channel(WEBSITE_CHANNEL_ID)
        if channel and hasattr(channel, "send"):
            try:
                await channel.send(embed=make_embed(title, f"Requested by {member.mention}\n\n{description}", discord.Color.from_rgb(255, 65, 35)))
            except discord.HTTPException:
                log.exception("Could not post website action to Discord.")
    return web.json_response({"message": f"{title}\n{description}"})

async def bridge_music(request):
    if not website_auth(request):
        raise web.HTTPUnauthorized(text="Invalid website API key.")
    payload = await request.json()
    guild, member = await website_member(payload)
    action = str(payload.get("action", "")).lower()
    voice = guild.voice_client
    if action == "join":
        channel_id = WEBSITE_VOICE_CHANNEL_ID
        if not channel_id:
            raise web.HTTPBadRequest(text="Set FREDDY_WEBSITE_VOICE_CHANNEL_ID to the voice channel Freddy should join.")
        channel = guild.get_channel(channel_id)
        if not isinstance(channel, discord.VoiceChannel):
            raise web.HTTPBadRequest(text="Configured voice channel was not found.")
        if voice is None:
            await channel.connect()
        elif voice.channel != channel:
            await voice.move_to(channel)
        result = "Freddy joined the configured voice channel."
    elif action == "leave":
        if voice:
            queues[guild.id].clear()
            await voice.disconnect()
        result = "Freddy left the voice channel."
    elif action == "pause":
        if voice and voice.is_playing():
            voice.pause()
            result = "Music paused."
        else:
            result = "Freddy is not currently playing music."
    elif action == "resume":
        if voice and voice.is_paused():
            voice.resume()
            result = "Music resumed."
        else:
            result = "Freddy's music is not paused."
    elif action == "skip":
        if voice and (voice.is_playing() or voice.is_paused()):
            voice.stop()
            result = "Skipped the current track."
        else:
            result = "Nothing is playing."
    elif action == "stop":
        queues[guild.id].clear()
        if voice and (voice.is_playing() or voice.is_paused()):
            voice.stop()
        result = "Music stopped and queue cleared."
    elif action in ("play rockstar", "play happy"):
        key = "rockstar" if action == "play rockstar" else "happy"
        path = MUSIC_FILES[key]
        if not path.is_file():
            raise web.HTTPBadRequest(text=f"Music file missing: {path.name}")
        if voice is None:
            channel_id = WEBSITE_VOICE_CHANNEL_ID
            channel = guild.get_channel(channel_id) if channel_id else None
            if not isinstance(channel, discord.VoiceChannel):
                raise web.HTTPBadRequest(text="Configure the website voice channel first, or have Freddy join a voice channel.")
            voice = await channel.connect()
        if voice.is_playing() or voice.is_paused():
            queues[guild.id].append(key)
            result = f"Queued {TRACK_NAMES[key]}."
        else:
            queues[guild.id].appendleft(key)
            await play_next(guild.id)
            result = f"Freddy is playing {TRACK_NAMES[key]}."
    else:
        raise web.HTTPBadRequest(text="Unknown music control.")
    if WEBSITE_CHANNEL_ID:
        channel = guild.get_channel(WEBSITE_CHANNEL_ID)
        if channel and hasattr(channel, "send"):
            try:
                await channel.send(f"🎛️ Website music control by {member.mention}: {result}")
            except discord.HTTPException:
                pass
    return web.json_response({"message": result})

async def bridge_server(request):
    if not website_auth(request):
        raise web.HTTPUnauthorized(text="Invalid website API key.")
    payload = await request.json()
    guild, member = await website_member(payload)
    action = str(payload.get("action", "")).lower()
    if action == "status":
        result = f"Freddy is online. Voice connected: {'yes' if guild.voice_client else 'no'}."
    elif action == "commands":
        result = "Try /freddy_chat, /solver, /solver_action, /jumpscare, /play, /pause, /resume, /skip, /queue, /volume, /stop."
    else:
        raise web.HTTPBadRequest(text="Unknown server action.")
    return web.json_response({"message": result})

async def bridge_settings(request):
    if not website_auth(request):
        raise web.HTTPUnauthorized(text="Invalid website API key.")
    payload = await request.json()
    guild, member = await website_member(payload)
    settings = website_settings()
    action = str(payload.get('action', 'get')).lower()
    if action == 'get':
        return web.json_response({'settings': settings, 'message': 'Welcome and goodbye settings loaded.'})
    if action != 'save':
        raise web.HTTPBadRequest(text='Unknown settings action.')
    incoming = payload.get('settings') or {}
    updated = dict(settings)
    for key in ('welcomeMessage', 'goodbyeMessage'):
        value = str(incoming.get(key, settings[key])).strip()
        if not value or len(value) > 1800:
            raise web.HTTPBadRequest(text=f'{key} must be between 1 and 1800 characters.')
        updated[key] = value
    for key in ('welcomeChannel', 'goodbyeChannel'):
        value = str(incoming.get(key, settings[key])).strip()
        if value and not value.isdigit():
            raise web.HTTPBadRequest(text=f'{key} must be a Discord channel ID or blank.')
        if value and guild.get_channel(int(value)) is None:
            raise web.HTTPBadRequest(text=f'{key} channel was not found in Freddy’s configured server.')
        updated[key] = value
    for key in ('welcomeImage', 'goodbyeImage'):
        value = str(incoming.get(key, settings[key])).strip()
        if value and (not value.startswith('https://') or len(value) > 2000):
            raise web.HTTPBadRequest(text=f'{key} must be a valid HTTPS image URL or blank.')
        updated[key] = value
    data['settings'] = updated
    save_data()
    return web.json_response({'settings': updated, 'message': f'Settings saved by {member.display_name}.'})

async def start_website_bridge():
    global website_runner
    if website_runner is not None or not WEBSITE_API_SECRET:
        return
    app = web.Application(client_max_size=64 * 1024)
    app.router.add_post("/api/chat", bridge_chat)
    app.router.add_post("/api/action", bridge_action)
    app.router.add_post("/api/music", bridge_music)
    app.router.add_post("/api/server", bridge_server)
    app.router.add_get("/health", lambda request: web.json_response({"ok": True, "bot": str(bot.user)}))
    website_runner = web.AppRunner(app)
    await website_runner.setup()
    site = web.TCPSite(website_runner, host="0.0.0.0", port=WEBSITE_API_PORT)
    await site.start()
    log.info("Glamrock Freddy website API bridge listening on port %s", WEBSITE_API_PORT)


bot.run(TOKEN)
