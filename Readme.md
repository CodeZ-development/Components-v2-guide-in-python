# 🧩 discord.py Components V2 — The Complete Guide

> Learn how to build modern, layout-based Discord bot UIs with **Components V2** in `discord.py` 2.6+.

-----

## 📑 Table of Contents

1. [What Is Components V2?](#what-is-components-v2)
1. [Setup & Requirements](#setup--requirements)
1. [How CV2 Works](#how-cv2-works)
1. [Class Reference](#class-reference)
- [LayoutView](#layoutview)
- [ui.TextDisplay](#uitextdisplay)
- [ui.Separator](#uiseparator)
- [ui.Section & ui.Thumbnail](#uisection--uithumbnail)
- [ui.MediaGallery](#uimediagallery)
- [ui.ActionRow, ui.Button & Selects](#uiactionrow-uibutton--selects)
1. [Sending & Updating CV2 Messages](#sending--updating-cv2-messages)
1. [Worked Examples](#worked-examples)
- [Example 1: Server Stats Card](#example-1-server-stats-card)
- [Example 2: Ticket Panel With Buttons](#example-2-ticket-panel-with-buttons)
- [Example 3: Paginated Leaderboard](#example-3-paginated-leaderboard)
- [Example 4: Settings Menu With a Select](#example-4-settings-menu-with-a-select)
1. [Gotchas & Best Practices](#gotchas--best-practices)
1. [Limitations Cheat Sheet](#limitations-cheat-sheet)

-----

## What Is Components V2?

Components V2 (CV2) is Discord’s layout-first message system. Instead of one rigid embed object, you compose a message out of small blocks — text, separators, images, and interactive rows — arranged inside a view.

|                    |Classic Embeds                   |Components V2                          |
|--------------------|---------------------------------|---------------------------------------|
|Structure           |One rigid embed object           |Composable blocks you stack freely     |
|Buttons/selects     |Bolted on below the embed        |Live inside the layout itself          |
|Images              |`image` / `thumbnail` fields only|Dedicated gallery + thumbnail items    |
|Text                |Title/description/fields         |Any number of free-form markdown blocks|
|Side-by-side content|Not possible                     |`ui.Section` puts text next to an image|

In `discord.py`, CV2 is built around a single new class: **`discord.ui.LayoutView`**. It’s a subclass of `View`, so everything you already know about views (timeouts, `interaction_check`, persistent views) still applies — you’re just filling it with layout items instead of dropping a row of buttons under an embed.

-----

## Setup & Requirements

```bash
pip install -U discord.py
```

CV2 needs **discord.py 2.6 or newer**. Check your version:

```python
import discord
print(discord.__version__)
```

Minimum intents for most bots using CV2:

```python
import discord

intents = discord.Intents.default()
intents.message_content = True

client = discord.Client(intents=intents)
# or, for commands.Bot:
# bot = commands.Bot(command_prefix="!", intents=intents)
```

Common imports you’ll reach for:

```python
import discord
from discord import ui
from discord.ui import LayoutView, TextDisplay, Separator, Section, Thumbnail, MediaGallery, MediaGalleryItem, ActionRow, Button, Select
```

-----

## How CV2 Works

Unlike `discord.js`, which needs an explicit `MessageFlags.IsComponentsV2` flag, **discord.py handles the flag for you automatically** the moment you send a `LayoutView`. You never set it by hand.

```python
class HelloView(ui.LayoutView):
    text = ui.TextDisplay("Hey there 👋")

await channel.send(view=HelloView())
```

**Shape of a CV2 view:**

```
class MyView(ui.LayoutView):       ← the view itself is the top-level container
    text   = ui.TextDisplay(...)   ← markdown text blocks
    sep    = ui.Separator(...)     ← dividers / spacing
    section = ui.Section(...)      ← text (left) + Thumbnail (right)
    gallery = ui.MediaGallery(...) ← up to 10 images
    row    = ui.ActionRow(...)     ← Buttons OR one Select
```

Items are declared as **class attributes**, and `discord.py` renders them top to bottom in declaration order. You can also build a view dynamically and call `.add_item(...)` in `__init__` if the layout depends on runtime data.

You can never attach `embed=` / `embeds=` to a message sent alongside a `LayoutView`. Pick one system per message.

-----

## Class Reference

### LayoutView

The outer container — and your view class itself.

```python
class MyView(ui.LayoutView):
    def __init__(self):
        super().__init__(timeout=180)
        # dynamically add items if needed
        # self.add_item(ui.TextDisplay("Built at runtime"))
```

Static (declarative) style is most common for simple layouts:

```python
class SuccessView(ui.LayoutView):
    text = ui.TextDisplay("## ✅ Done\nYour changes were saved.")
```

> `discord.py` doesn’t expose a per-container accent color setter the way `discord.js` does with `setAccentColor` on every container — color is set via `ui.Container`, covered below.

If you want the colored left-border / grouping look, wrap items in a `discord.ui.Container`:

```python
class StatusView(ui.LayoutView):
    container = ui.Container(
        ui.TextDisplay("## ✅ Saved"),
        accent_colour=discord.Color.green(),
    )
```

-----

### ui.TextDisplay

Free-form markdown text. Every line of text in a CV2 layout goes through this.

```python
ui.TextDisplay("# Heading 1")
ui.TextDisplay("## Heading 2")
ui.TextDisplay("**bold**, *italic*, __underline__")
ui.TextDisplay("> a quote")
ui.TextDisplay("-# tiny subtext")
```

Interpolate variables the same way you always have:

```python
ui.TextDisplay(f"**Balance:** `{coins:,}` coins")
ui.TextDisplay(f"**Next payout:** <t:{next_payout}:R>")
```

-----

### ui.Separator

A horizontal rule or blank gap between blocks.

```python
ui.Separator()                                              # visible line, small gap
ui.Separator(visible=False)                                 # invisible spacer only
ui.Separator(spacing=discord.SeparatorSpacing.large)
```

-----

### ui.Section & ui.Thumbnail

`ui.Section` places one or more `TextDisplay` items next to a small image — great for profile cards.

```python
section = ui.Section(
    ui.TextDisplay(f"# {member.display_name}"),
    ui.TextDisplay(f"Joined <t:{joined_ts}:R>"),
    accessory=ui.Thumbnail(
        media=member.display_avatar.url,
        description=f"{member.name}'s avatar",
    ),
)
```

> `ui.Thumbnail` only works as the `accessory=` of a `ui.Section` — it can’t stand on its own.

-----

### ui.MediaGallery

Shows up to **10 images** in one block.

```python
gallery = ui.MediaGallery(
    MediaGalleryItem(media="https://example.com/banner.png", description="Server banner"),
)
```

For local files, use `attachment://` and attach the file to the same send call:

```python
file = discord.File("chart.png", filename="chart.png")

gallery = ui.MediaGallery(
    MediaGalleryItem(media="attachment://chart.png"),
)

class ChartView(ui.LayoutView):
    g = gallery

await channel.send(view=ChartView(), file=file)
```

-----

### ui.ActionRow, ui.Button & Selects

Rows hold the interactive stuff — **up to 5 buttons, or exactly 1 select menu, never both.**

```python
class ApprovalRow(ui.ActionRow):
    @discord.ui.button(label="Approve", style=discord.ButtonStyle.success)
    async def approve(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message("Approved!", ephemeral=True)

    @discord.ui.button(label="Reject", style=discord.ButtonStyle.danger)
    async def reject(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message("Rejected.", ephemeral=True)
```

Button styles at a glance:

|Style                  |Color       |Typical use                       |
|-----------------------|------------|----------------------------------|
|`ButtonStyle.primary`  |Blue        |Main call to action               |
|`ButtonStyle.secondary`|Grey        |Neutral / “back”                  |
|`ButtonStyle.success`  |Green       |Confirm                           |
|`ButtonStyle.danger`   |Red         |Destructive                       |
|`ButtonStyle.link`     |Grey + arrow|External URL (`url=`, no callback)|

Select menus inside an `ActionRow`:

```python
class RoleRow(ui.ActionRow):
    @discord.ui.select(
        placeholder="Pick a role...",
        options=[
            discord.SelectOption(label="Gamer", value="gamer", emoji="🎮"),
            discord.SelectOption(label="Artist", value="artist", emoji="🎨"),
        ],
    )
    async def pick_role(self, interaction: discord.Interaction, select: discord.ui.Select):
        await interaction.response.send_message(f"You picked {select.values[0]}", ephemeral=True)
```

-----

## Sending & Updating CV2 Messages

**Sending:**

```python
await channel.send(view=MyView())
```

**Replying ephemerally from a slash command:**

```python
await interaction.response.send_message(view=MyView(), ephemeral=True)
```

**Updating in place after a button/select click:**

```python
await interaction.response.edit_message(view=new_view)
```

-----

## Worked Examples

### Example 1: Server Stats Card

A snapshot card using a `Section` for the icon + a `Separator` to break up stats.

```python
import discord
from discord import app_commands
from discord.ext import commands
from discord import ui


class ServerStatsView(ui.LayoutView):
    def __init__(self, guild: discord.Guild, humans: int, bots: int):
        super().__init__()
        created_ts = int(guild.created_at.timestamp())
        icon_url = guild.icon.url if guild.icon else "https://discord.com/assets/icon.png"

        section = ui.Section(
            ui.TextDisplay(f"# {guild.name}"),
            ui.TextDisplay(f"-# Created <t:{created_ts}:R>"),
            accessory=ui.Thumbnail(media=icon_url, description="Server icon"),
        )

        self.container = ui.Container(
            section,
            ui.Separator(),
            ui.TextDisplay(f"**Members:** {guild.member_count}"),
            ui.TextDisplay(f"**Humans:** {humans} • **Bots:** {bots}"),
            ui.TextDisplay(f"**Roles:** {len(guild.roles)}"),
            ui.TextDisplay(f"**Boost Tier:** {guild.premium_tier or 'None'}"),
            accent_colour=discord.Color.blurple(),
        )
        self.add_item(self.container)


class StatsCog(commands.Cog):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.command(name="serverstats")
    async def serverstats(self, ctx: commands.Context):
        guild = ctx.guild
        members = guild.members
        bots = sum(1 for m in members if m.bot)
        humans = len(members) - bots

        await ctx.reply(view=ServerStatsView(guild, humans, bots))


async def setup(bot: commands.Bot):
    await bot.add_cog(StatsCog(bot))
```

-----

### Example 2: Ticket Panel With Buttons

A support-ticket opener that swaps the layout once a ticket is created.

```python
import discord
from discord.ext import commands
from discord import ui


class TicketPanelView(ui.LayoutView):
    def __init__(self):
        super().__init__(timeout=None)
        row = ui.ActionRow()

        button = discord.ui.Button(label="Open Ticket", style=discord.ButtonStyle.success, emoji="🎫", custom_id="open_ticket")
        button.callback = self.on_open_ticket
        row.add_item(button)

        self.container = ui.Container(
            ui.TextDisplay("## 🎫 Support Tickets"),
            ui.TextDisplay("Need help? Open a private ticket and our team will assist you."),
            ui.Separator(),
            row,
            accent_colour=discord.Color.blurple(),
        )
        self.add_item(self.container)

    async def on_open_ticket(self, interaction: discord.Interaction):
        guild = interaction.guild
        overwrites = {
            guild.default_role: discord.PermissionOverwrite(view_channel=False),
            interaction.user: discord.PermissionOverwrite(view_channel=True, send_messages=True),
        }
        ticket_channel = await guild.create_text_channel(
            name=f"ticket-{interaction.user.name}",
            overwrites=overwrites,
        )

        confirm = ui.LayoutView()
        confirm.add_item(
            ui.Container(
                ui.TextDisplay("## ✅ Ticket Created"),
                ui.TextDisplay(f"Head over to {ticket_channel.mention} — a staff member will be with you shortly."),
                accent_colour=discord.Color.green(),
            )
        )
        await interaction.response.send_message(view=confirm, ephemeral=True)


class TicketsCog(commands.Cog):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.command(name="tickets")
    async def tickets(self, ctx: commands.Context):
        await ctx.reply(view=TicketPanelView())


async def setup(bot: commands.Bot):
    await bot.add_cog(TicketsCog(bot))
```

-----

### Example 3: Paginated Leaderboard

Pagination through buttons, rebuilding the layout per page.

```python
import discord
from discord.ext import commands
from discord import ui

PAGE_SIZE = 5


def build_leaderboard_view(entries: list[dict], page: int, owner_id: int) -> ui.LayoutView:
    total_pages = max(1, -(-len(entries) // PAGE_SIZE))  # ceil division
    start = page * PAGE_SIZE
    slice_ = entries[start:start + PAGE_SIZE]

    lines = [
        ui.TextDisplay(f"**#{start + i + 1}** — {entry['name']} • `{entry['score']} pts`")
        for i, entry in enumerate(slice_)
    ]

    view = ui.LayoutView()
    row = ui.ActionRow()

    prev_btn = discord.ui.Button(label="◀ Prev", style=discord.ButtonStyle.secondary, custom_id="lb_prev", disabled=page == 0)
    next_btn = discord.ui.Button(label="Next ▶", style=discord.ButtonStyle.secondary, custom_id="lb_next", disabled=page >= total_pages - 1)

    async def go_prev(interaction: discord.Interaction):
        if interaction.user.id != owner_id:
            return await interaction.response.send_message("This isn't your leaderboard.", ephemeral=True)
        await interaction.response.edit_message(view=build_leaderboard_view(entries, page - 1, owner_id))

    async def go_next(interaction: discord.Interaction):
        if interaction.user.id != owner_id:
            return await interaction.response.send_message("This isn't your leaderboard.", ephemeral=True)
        await interaction.response.edit_message(view=build_leaderboard_view(entries, page + 1, owner_id))

    prev_btn.callback = go_prev
    next_btn.callback = go_next
    row.add_item(prev_btn)
    row.add_item(next_btn)

    view.add_item(
        ui.Container(
            ui.TextDisplay("## 🏆 Leaderboard"),
            ui.Separator(),
            *lines,
            ui.Separator(),
            ui.TextDisplay(f"-# Page {page + 1} of {total_pages}"),
            row,
            accent_colour=discord.Color.yellow(),
        )
    )
    return view


class LeaderboardCog(commands.Cog):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.command(name="leaderboard")
    async def leaderboard(self, ctx: commands.Context):
        entries = [{"name": "Alice", "score": 980}, {"name": "Bob", "score": 870},
                   {"name": "Cara", "score": 760}, {"name": "Dan", "score": 690},
                   {"name": "Eve", "score": 600}, {"name": "Finn", "score": 540}]

        await ctx.reply(view=build_leaderboard_view(entries, 0, ctx.author.id))


async def setup(bot: commands.Bot):
    await bot.add_cog(LeaderboardCog(bot))
```

-----

### Example 4: Settings Menu With a Select

A toggleable settings panel driven by a dropdown.

```python
import discord
from discord.ext import commands
from discord import ui

SETTINGS = {
    "welcome_messages": "Welcome Messages",
    "auto_role": "Auto Role",
    "level_ups": "Level-Up Alerts",
}


def build_settings_view(state: dict, owner_id: int) -> ui.LayoutView:
    view = ui.LayoutView()
    row = ui.ActionRow()

    select = discord.ui.Select(
        placeholder="Toggle a setting...",
        options=[
            discord.SelectOption(
                label=label,
                value=key,
                description="Currently ON" if state[key] else "Currently OFF",
                emoji="🟢" if state[key] else "🔴",
            )
            for key, label in SETTINGS.items()
        ],
    )

    async def on_select(interaction: discord.Interaction):
        if interaction.user.id != owner_id:
            return await interaction.response.send_message("This isn't your settings panel.", ephemeral=True)
        key = select.values[0]
        state[key] = not state[key]
        await interaction.response.edit_message(view=build_settings_view(state, owner_id))

    select.callback = on_select
    row.add_item(select)

    lines = [
        ui.TextDisplay(f"{'🟢' if state[key] else '🔴'} **{label}**")
        for key, label in SETTINGS.items()
    ]

    view.add_item(
        ui.Container(
            ui.TextDisplay("## ⚙️ Server Settings"),
            ui.Separator(),
            *lines,
            ui.Separator(),
            row,
            accent_colour=discord.Color.blurple(),
        )
    )
    return view


class SettingsCog(commands.Cog):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.command(name="settings")
    async def settings(self, ctx: commands.Context):
        state = {"welcome_messages": True, "auto_role": False, "level_ups": True}
        await ctx.reply(view=build_settings_view(state, ctx.author.id))


async def setup(bot: commands.Bot):
    await bot.add_cog(SettingsCog(bot))
```

-----

## Gotchas & Best Practices

- **No manual flag needed.** `discord.py` sets the CV2 flag for you whenever you pass a `LayoutView` to `send`/`edit_message`/`response.send_message`. Just don’t mix it with `embed=`.
- **No mixing with embeds.** A message is either classic embeds or CV2 — never both in the same call.
- **Use `ui.Container` for grouping + color.** Plain `LayoutView` items without a `Container` wrapper still render, but you lose the accent-color border and visual grouping.
- **Factor out builder functions.** Write `build_x_view(...)` helper functions so the same layout logic powers the initial send and every later `edit_message`.
- **Check `interaction.user.id`** before honoring button/select callbacks on shared panels — `discord.py` doesn’t auto-restrict interactions to the original invoker.
- **Disable expired components.** On a view’s `on_timeout`, edit the original message to a disabled-button state so dead interactions don’t linger.
- **Use `attachment://`** for local files in `MediaGalleryItem`/`Thumbnail`, and pass the matching `discord.File` in the same send call.
- **Discord timestamps still work inside `TextDisplay`** — `<t:TIMESTAMP:R>`, `:F>`, `:D>`, `:t>` all render normally.

-----

## Limitations Cheat Sheet

|Limitation               |Detail                                                                      |
|-------------------------|----------------------------------------------------------------------------|
|No nested containers     |A `ui.Container` can’t contain another `ui.Container`                       |
|Thumbnail is Section-only|`ui.Thumbnail` only works as a `Section`’s `accessory=`                     |
|ActionRow capacity       |5 buttons **or** 1 select — never mixed                                     |
|Gallery cap              |10 images max per `ui.MediaGallery`                                         |
|View type matters        |Only `ui.LayoutView` renders CV2 layouts — a regular `discord.ui.View` won’t|
|No embeds alongside CV2  |Pick one system per message                                                 |
|Python version           |CV2 needs `discord.py` 2.6+; older versions don’t have these classes        |

-----

> Made by **void** & **Codez Dev** — [discord.gg/codez](https://discord.gg/codez)
