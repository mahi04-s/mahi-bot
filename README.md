# mahi-bot
import discord
from discord.ext import commands
import youtube_dl
import asyncio

# Suppress noise about console usage from errors
youtube_dl.utils.bug_reports_message = lambda: ''

ytdl_format_options = {
    'format': 'bestaudio/best',
    'outtmpl': '%(extractor)s-%(id)s-%(title)s.%(ext)s',
    'restrictfilenames': True,
    'noplaylist': True,
    'nocheckcertificate': True,
    'ignoreerrors': False,
    'logtostderr': False,
    'quiet': True,
    'no_warnings': True,
    'default_search': 'auto',
    'source_address': '0.0.0.0'  # Bind to ipv4 since ipv6 addresses cause issues sometimes
}

ffmpeg_options = {
    'options': '-vn'
}

ytdl = youtube_dl.YoutubeDL(ytdl_format_options)

class YTDLSource(discord.PCMVolumeTransformer):
    def __init__(self, source, *, data, volume=0.5):
        super().__init__(source, volume)

        self.data = data

        self.title = data.get('title')
        self.url = data.get('url')

    @classmethod
    async def from_url(cls, url, *, loop=None, stream=False):
        loop = loop or asyncio.get_event_loop()
        data = await loop.run_in_executor(None, lambda: ytdl.extract_info(url, download=not stream))

        if 'entries' in data:
            # Take first item from a playlist
            data = data['entries'][0]

        filename = data['url'] if stream else ytdl.prepare_filename(data)
        return cls(discord.FFmpegPCMAudio(filename, **ffmpeg_options), data=data)

intents = discord.Intents.default()
intents.message_content = True
intents.members = True

bot = commands.Bot(command_prefix=commands.when_mentioned_or("!"), description='An advanced music bot with extra features', intents=intents)

# Global Ban List
global_ban_list = set()

@bot.event
async def on_ready():
    print(f'Logged in as {bot.user} (ID: {bot.user.id})')
    print('------')

# Music Commands
@bot.command()
async def join(ctx, *, channel: discord.VoiceChannel):
    """Joins a voice channel"""
    if ctx.voice_client is not None:
        return await ctx.voice_client.move_to(channel)

    await channel.connect()

@bot.command()
async def play(ctx, *, query):
    """Plays a file from the local filesystem"""
    async with ctx.typing():
        player = await YTDLSource.from_url(query, loop=bot.loop, stream=True)
        ctx.voice_client.play(player, after=lambda e: print(f'Player error: {e}') if e else None)

    await ctx.send(f'Now playing: {player.title}')

@bot.command()
async def stream(ctx, *, url):
    """Streams from a URL (same as yt, but doesn't predownload)"""
    async with ctx.typing():
        player = await YTDLSource.from_url(url, loop=bot.loop, stream=True)
        ctx.voice_client.play(player, after=lambda e: print(f'Player error: {e}') if e else None)

    await ctx.send(f'Now streaming: {player.title}')

@bot.command()
async def volume(ctx, volume: int):
    """Changes the player's volume"""
    if ctx.voice_client is None:
        return await ctx.send("Not connected to a voice channel.")

    ctx.voice_client.source.volume = volume / 100
    await ctx.send(f"Changed volume to {volume}%")

@bot.command()
async def stop(ctx):
    """Stops and disconnects the bot from voice"""
    await ctx.voice_client.disconnect()

@play.before_invoke
@stream.before_invoke
async def ensure_voice(ctx):
    if ctx.voice_client is None:
        if ctx.author.voice:
            await ctx.author.voice.channel.connect()
        else:
            await ctx.send("You are not connected to a voice channel.")
            raise commands.CommandError("Author not connected to a voice channel.")
    elif ctx.voice_client.is_playing():
        ctx.voice_client.stop()

# Anti-Spam and Raid Protection
@bot.event
async def on_message(message):
    if message.author.bot:
        return

    # Check for spam (simple example, consider using a more sophisticated method)
    if message.content.lower().count("spam") > 5:
        await message.channel.send(f"{message.author.mention}, please do not spam.")
        await message.delete()
        return

    await bot.process_commands(message)

# Global Ban Command
@bot.command()
async def gban(ctx, user: discord.User):
    """Globally bans a user from the server"""
    if ctx.author.guild_permissions.ban_members:
        global_ban_list.add(user.id)
        await ctx.send(f"{user.name} has been globally banned.")
    else:
        await ctx.send("You do not have permission to use this command.")

@bot.event
async def on_member_join(member):
    """Checks if a joining member is in the global ban list"""
    if member.id in global_ban_list:
        await member.ban(reason="Globally banned user")

# DM Protection
@bot.event
async def on_message(message):
    if isinstance(message.channel, discord.DMChannel) and not message.author.bot:
        await message.author.send("Direct messages are protected. If you have an issue, please contact the server moderators.")
        return

    await bot.process_commands(message)

bot.run('7849066112:AAGO-2iemtbetNxw9imf0dfS0-kx4kmHBwY
')
