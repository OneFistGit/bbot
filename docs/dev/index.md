# BBOT Developer Reference

BBOT exposes a convenient API that allows you to create, start, and stop scans using Python code.

Documented in this section are commonly-used classes and functions within BBOT, along with usage examples.

## Discord Bot Example

Below is a simple Discord bot designed to run BBOT scans.

```python
import asyncio
import discord
from discord.ext import commands

from bbot.scanner import Scanner
from bbot.modules import module_loader
from bbot.modules.output.discord import Discord


# make list of BBOT modules to enable for the scan
bbot_modules = ["excavate", "speculate", "aggregate"]
for module_name, preloaded in module_loader.preloaded().items():
    flags = preloaded["flags"]
    if "subdomain-enum" in flags and "passive" in flags and "slow" not in flags:
        bbot_modules.append(module_name)


class BBOTDiscordBot(commands.Cog):
    """
    A simple Discord bot capable of running a BBOT scan.

    To set up:
        1. Go to Discord Developer Portal (https://discord.com/developers)
        2. Create a new application
        3. Create an invite link for the bot, visit the link to invite it to your server
            - Your Application --> OAuth2 --> URL Generator
                - For Scopes, select "bot""
                - For Bot Permissions, select:
                    - Read Messages/View Channels
                    - Send Messages
        4. Turn on "Message Content Intent"
            - Your Application --> Bot --> Privileged Gateway Intents --> Message Content Intent
        5. Copy your Discord Bot Token and put it at the top this file
            - Your Application --> Bot --> Reset Token
        6. Run this script

    To scan evilcorp.com, you would type:

        /scan evilcorp.com

    Results will be output to the same channel.
    """
    def __init__(self):
        self.current_scan = None

    @commands.command(name="scan", description="Scan a target with BBOT.")
    async def scan(self, ctx, target: str):
        if self.current_scan is not None:
            self.current_scan.stop()
        await ctx.send(f"Starting scan against {target}.")

        # creates scan instance
        self.current_scan = Scanner(target, modules=bbot_modules)
        discord_module = Discord(self.current_scan)

        seen = set()
        num_events = 0
        # start scan and iterate through results
        async for event in self.current_scan.async_start():
            if hash(event) in seen:
                continue
            seen.add(hash(event))
            await ctx.send(discord_module.format_message(event))
            num_events += 1

        await ctx.send(f"Finished scan against {target}. {num_events:,} results.")
        self.current_scan = None


if __name__ == "__main__":
    intents = discord.Intents.default()
    intents.message_content = True
    bot = commands.Bot(command_prefix="/", intents=intents)

    @bot.event
    async def on_ready():
        print(f"We have logged in as {bot.user}")
        await bot.add_cog(BBOTDiscordBot())

    bot.run("DISCORD_BOT_TOKEN_HERE")
```

[Next Up: Scanner -->](scanner.md){ .md-button .md-button--primary }
