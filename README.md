import discord
from discord.ext import commands, tasks
import asyncio
import logging
import os
import signal
import sys
import time
from datetime import datetime
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('moderation_bot.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger('ModerationBot')

# Bot configuration
BOT_TOKEN = os.getenv('DISCORD_BOT_TOKEN')
TARGET_ROLE_ID = int(os.getenv('TARGET_ROLE_ID', '1403752677648629832'))

# Bot setup with necessary intents
intents = discord.Intents.default()
intents.members = True  # Required to monitor member updates
intents.guilds = True   # Required to access guild information

bot = commands.Bot(command_prefix='!', intents=intents)

# 24/7 Operation Variables
class BotStats:
    def __init__(self):
        self.start_time = datetime.utcnow()
        self.reconnect_attempts = 0

bot_stats = BotStats()
MAX_RECONNECT_ATTEMPTS = 10

class ModerationBot:
    def __init__(self, bot):
        self.bot = bot
        self.target_role_id = TARGET_ROLE_ID
        
    async def check_and_ban_user(self, member, guild):
        """Check if user has target role and ban them if they do"""
        try:
            # Check if the bot has permission to ban members
            if not guild.me.guild_permissions.ban_members:
                logger.error(f"Bot lacks ban permissions in guild: {guild.name}")
                return
            
            # Check if user has the target role
            target_role = discord.utils.get(member.roles, id=self.target_role_id)
            
            if target_role:
                # Prevent banning the bot owner or users with higher roles than the bot
                if member.top_role >= guild.me.top_role:
                    logger.warning(f"Cannot ban {member.display_name} - user has higher or equal role to bot")
                    return
                
                # Ban the user
                reason = f"Automatic ban: User has prohibited role ID {self.target_role_id}"
                await member.ban(reason=reason, delete_message_days=0)
                
                logger.info(f"BANNED USER: {member.display_name} ({member.id}) in guild {guild.name} - Reason: {reason}")
                
                # Optional: Send notification to a log channel if configured
                await self.log_ban_action(guild, member, reason)
                
        except discord.Forbidden:
            logger.error(f"Insufficient permissions to ban {member.display_name} in {guild.name}")
        except discord.HTTPException as e:
            logger.error(f"HTTP error while banning {member.display_name}: {e}")
        except Exception as e:
            logger.error(f"Unexpected error while processing {member.display_name}: {e}")
    
    async def log_ban_action(self, guild, member, reason):
        """Log ban action to a designated channel if available"""
        try:
            # Look for a moderation log channel (you can customize this)
            log_channel = discord.utils.get(guild.channels, name='moderation-logs')
            if log_channel and isinstance(log_channel, discord.TextChannel):
                embed = discord.Embed(
                    title="🔨 Automatic Ban",
                    color=discord.Color.red(),
                    timestamp=discord.utils.utcnow()
                )
                embed.add_field(name="User", value=f"{member.mention} ({member.id})", inline=False)
                embed.add_field(name="Reason", value=reason, inline=False)
                embed.set_footer(text="Moderation Bot")
                
                await log_channel.send(embed=embed)
        except Exception as e:
            logger.error(f"Failed to send log message: {e}")

# Initialize moderation system
moderation = ModerationBot(bot)

@tasks.loop(minutes=5)
async def keepalive_task():
    """Keep bot alive and log status every 5 minutes"""
    uptime = datetime.utcnow() - bot_stats.start_time
    logger.info(f"Bot keepalive - Uptime: {uptime} | Guilds: {len(bot.guilds)} | Ping: {round(bot.latency * 1000)}ms")

@keepalive_task.before_loop
async def before_keepalive():
    await bot.wait_until_ready()

@bot.event
async def on_ready():
    """Event triggered when bot is ready"""
    bot_stats.start_time = datetime.utcnow()
    bot_stats.reconnect_attempts = 0
    
    if bot.user:
        logger.info(f"Bot logged in as {bot.user.name} ({bot.user.id})")
    logger.info(f"Monitoring for role ID: {TARGET_ROLE_ID}")
    logger.info(f"Bot is ready and monitoring {len(bot.guilds)} guild(s)")
    logger.info("🟢 24/7 OPERATION MODE ACTIVE")
    
    # Start keepalive task for 24/7 operation
    if not keepalive_task.is_running():
        keepalive_task.start()
        logger.info("Started keepalive task for 24/7 operation")
    
    # Check all existing members in all guilds for the target role
    for guild in bot.guilds:
        logger.info(f"Scanning existing members in guild: {guild.name}")
        try:
            async for member in guild.fetch_members(limit=None):
                await moderation.check_and_ban_user(member, guild)
                # Small delay to avoid rate limiting
                await asyncio.sleep(0.1)
        except Exception as e:
            logger.error(f"Error scanning guild {guild.name}: {e}")

@bot.event
async def on_member_join(member):
    """Event triggered when a new member joins"""
    logger.info(f"New member joined: {member.display_name} in {member.guild.name}")
    await moderation.check_and_ban_user(member, member.guild)

@bot.event
async def on_member_update(before, after):
    """Event triggered when a member's roles are updated"""
    # Check if roles were added
    added_roles = set(after.roles) - set(before.roles)
    
    if added_roles:
        logger.info(f"Member {after.display_name} received new roles in {after.guild.name}")
        await moderation.check_and_ban_user(after, after.guild)

@bot.event
async def on_guild_join(guild):
    """Event triggered when bot joins a new guild"""
    logger.info(f"Bot joined new guild: {guild.name} ({guild.id})")
    
    # Scan all members in the new guild
    try:
        async for member in guild.fetch_members(limit=None):
            await moderation.check_and_ban_user(member, guild)
            await asyncio.sleep(0.1)
    except Exception as e:
        logger.error(f"Error scanning new guild {guild.name}: {e}")

@bot.command(name='status')
@commands.has_permissions(administrator=True)
async def status(ctx):
    """Check bot status and configuration"""
    embed = discord.Embed(
        title="🤖 Moderation Bot Status",
        color=discord.Color.blue()
    )
    embed.add_field(name="Target Role ID", value=TARGET_ROLE_ID, inline=False)
    embed.add_field(name="Guilds Monitored", value=len(bot.guilds), inline=True)
    embed.add_field(name="Bot Permissions", value="✅ Running" if ctx.guild.me.guild_permissions.ban_members else "❌ Missing ban permissions", inline=True)
    
    await ctx.send(embed=embed)

@bot.command(name='check_role')
@commands.has_permissions(administrator=True)
async def check_role(ctx, member: discord.Member = None):
    """Check if a specific member has the target role"""
    if member is None:
        member = ctx.author
    
    target_role = discord.utils.get(member.roles, id=TARGET_ROLE_ID)
    
    embed = discord.Embed(
        title="🔍 Role Check",
        color=discord.Color.orange()
    )
    embed.add_field(name="Member", value=f"{member.mention} ({member.id})", inline=False)
    embed.add_field(name="Has Target Role", value="✅ Yes" if target_role else "❌ No", inline=True)
    
    if target_role:
        embed.add_field(name="Role Name", value=target_role.name, inline=True)
    
    await ctx.send(embed=embed)

@bot.event
async def on_disconnect():
    """Handle disconnection events for 24/7 operation"""
    logger.warning("🔴 Bot disconnected - attempting to reconnect...")
    bot_stats.reconnect_attempts += 1

@bot.event
async def on_resumed():
    """Handle reconnection events"""
    logger.info("🟢 Bot reconnected successfully")
    bot_stats.reconnect_attempts = 0

@bot.event
async def on_command_error(ctx, error):
    """Handle command errors"""
    if isinstance(error, commands.MissingPermissions):
        await ctx.send("❌ You don't have permission to use this command.")
    elif isinstance(error, commands.CommandNotFound):
        pass  # Ignore unknown commands
    else:
        logger.error(f"Command error: {error}")
        await ctx.send("❌ An error occurred while processing the command.")

@bot.command(name='uptime')
@commands.has_permissions(administrator=True)
async def uptime(ctx):
    """Check bot uptime for 24/7 monitoring"""
    uptime = datetime.utcnow() - bot_stats.start_time
    days = uptime.days
    hours, remainder = divmod(uptime.seconds, 3600)
    minutes, seconds = divmod(remainder, 60)
    
    embed = discord.Embed(
        title="⏰ Bot Uptime",
        color=discord.Color.green()
    )
    embed.add_field(name="Uptime", value=f"{days}d {hours}h {minutes}m {seconds}s", inline=False)
    embed.add_field(name="Ping", value=f"{round(bot.latency * 1000)}ms", inline=True)
    embed.add_field(name="Status", value="🟢 24/7 ACTIVE", inline=True)
    embed.add_field(name="Reconnects", value=bot_stats.reconnect_attempts, inline=True)
    
    await ctx.send(embed=embed)

async def shutdown_handler(signum, frame):
    """Handle graceful shutdown"""
    logger.info("Received shutdown signal - stopping bot gracefully...")
    if keepalive_task.is_running():
        keepalive_task.stop()
    await bot.close()
    sys.exit(0)

def main():
    """Main function to run the bot with 24/7 operation support"""
    if not BOT_TOKEN:
        logger.error("DISCORD_BOT_TOKEN not found in environment variables!")
        return
    
    # Setup signal handlers for graceful shutdown
    signal.signal(signal.SIGINT, lambda s, f: asyncio.create_task(shutdown_handler(s, f)))
    signal.signal(signal.SIGTERM, lambda s, f: asyncio.create_task(shutdown_handler(s, f)))
    
    # Enhanced error handling for 24/7 operation
    reconnect_attempts = 0
    while reconnect_attempts < MAX_RECONNECT_ATTEMPTS:
        try:
            logger.info(f"🚀 Starting Discord Moderation Bot (24/7 Mode) - Attempt {reconnect_attempts + 1}")
            bot.run(BOT_TOKEN, reconnect=True)
            break  # If we get here, bot stopped normally
            
        except discord.LoginFailure:
            logger.error("❌ Invalid bot token provided!")
            break
            
        except discord.ConnectionClosed:
            reconnect_attempts += 1
            logger.warning(f"⚠️ Connection closed - Reconnecting... (Attempt {reconnect_attempts}/{MAX_RECONNECT_ATTEMPTS})")
            if reconnect_attempts < MAX_RECONNECT_ATTEMPTS:
                time.sleep(5)  # Wait before reconnecting
            
        except Exception as e:
            reconnect_attempts += 1
            logger.error(f"❌ Bot crashed: {e} - Reconnecting... (Attempt {reconnect_attempts}/{MAX_RECONNECT_ATTEMPTS})")
            if reconnect_attempts < MAX_RECONNECT_ATTEMPTS:
                time.sleep(10)  # Wait longer for unexpected errors
    
    if reconnect_attempts >= MAX_RECONNECT_ATTEMPTS:
        logger.error("💀 Max reconnection attempts reached - Bot shutting down")

if __name__ == "__main__":
    main()
