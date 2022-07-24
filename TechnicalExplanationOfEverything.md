# Mining Simulator Technical Explanation

## Introduction
If you are interested in coding, perhaps this will be useful to you. I don't know. I don't care. I'm making this because I'm bored.

## Machine/Server Information
Mining Simulator is running on a Google Cloud VPS (e2-standard-4) with 4 vCores and 16 GB RAM, using Debian 11 for the OS. The server is based in Iowa, since the majority of our players are in the United States.

For our game server software, we use Waterfall for the proxy and Paper 1.8.8 for the backend servers. The server is based on 1.8.8 to allow compatibility with as many players as possible. The proxy has 2 GB RAM dedicated to it, and the game server has 8 GB RAM dedicated to it. We use screen to keep those servers running without needing a constant SSH connection.

## Plugins
Our primary plugin is MSPlugin, which is custom. MSPlugin handles all stats, mines, shops, item selling, and is the backbone the game's functionality.

This is a list of every other plugin we use:
- PlaceholderAPI (handles placeholders so we can use our stats in leaderboards, the scoreboard, menus, etc.)
- LuckPerms (handles all permissions and prefixes)
- EssentialsX (handles spawning, chat, and several smaller features)
- CommandPanels (what we use for our menus, as it's far easier than creating them manually)
- TAB (handles tablist and player nametags, it's way cooler than it sounds)
- HolographicDisplays (for mine signs and leaderboards)
- ViaVersion (allows players on any version from 1.8.8 - 1.19.1 to connect)
- Scoreboard-revision (handles scoreboard, obviously)
- ProRecipes (handles custom crafting recipes)
- Citizens2 (lets us make NPCs)
- WorldGuard (prevents block breaking, crop trampling, etc.)
- FastAsyncWorldEdit (for builders)
- Multiverse (allows for multiple worlds on one backend server)
- ItemEdit (for admins to create new items)
- VoidGen (makes the world a void)
- CommandNPC (allows our Citizens NPCs to have commands on click)
- SkinsRestorer (allows skin changing, important to nick system)
- MyCommand (lets us make commands without having to code the whole thing in the plugin, useful for some less complicated commands)
- ActionBarAPI (dev API for displaying actionbars)
- NBTAPI (dev API for custom NBT tags)
- CommandToItem (currently unused, but theoretically would allow us to put commands on items)
- SwagChat (stupid plugin for funny filters)

Now here's the proxy plugins:
- AdvancedBan (handles bans, mutes, etc.)
- BungeeChat (handles some aspects of chat, but most features are disabled)
- Buycraft (for purchaseable ranks, we gotta make money somehow man)
- LuckPerms (for syncing perms/prefixes across servers)
- PremiumVanish (staff vanish plugin)
- ServerListPlus (for the MOTD int the server list, makes the server pop a bit more)

Most of these plugins only play a small part in the game, each of them making the experience just a little better at a time, even if you never would interact with the plugin as a player.
As a server admin, you need to make sure that the experience on the player end is smooth and polished, no matter how messy it has to get on the admin side.

## MSPlugin
This section will be about the functions of MSPlugin, from relatively small things like deleting crafting recipes, to major gameplay functions like mining.

### Database
We use a MySQL 8.0 database to store player stats, mine information, shop information, to even player display names.

The table with the most vital information is miningsim_stats, which stores the player's exp, gold, level, rebirth, and blocks mined statistics. When the plugin enables during server start, we initialize this database like this:

```
Statement statement = getConnection().createStatement();
String sql = "CREATE TABLE IF NOT EXISTS miningsim_stats(uuid varchar(36) primary key, exp int, level int, gold int, rebirth int, blocksMined int)";
statement.execute(sql);
statement.close();
```

We also initialize 6 other tables, like this:

```
Statement statement2 = getConnection().createStatement();
String sql2 = "CREATE TABLE IF NOT EXISTS miningsim_uuid(uuid varchar(36) primary key, username varchar(16), displayname varchar(200))";
statement2.execute(sql2);
statement2.close();

Statement statement3 = getConnection().createStatement();
String sql3 = "CREATE TABLE IF NOT EXISTS miningsim_mines(block varchar(200) primary key, mine varchar(200))";
statement3.execute(sql3);
statement3.close();

Statement statement4 = getConnection().createStatement();
String sql4 = "CREATE TABLE IF NOT EXISTS miningsim_minemaking(uuid varchar(36) primary key, mine varchar(200))";
statement4.execute(sql4);
statement4.close();

Statement statement5 = getConnection().createStatement();
String sql5 = "CREATE TABLE IF NOT EXISTS miningsim_mineinfo(name varchar(200) primary key, exp int, gold int, levelreq int, rebirthreq int, itemname varchar(200), itemlore varchar(200), itemtype int, itemdata int)";
statement5.execute(sql5);
statement5.close(); // dude if your actually reading this you need to get some hoes

Statement statement6 = getConnection().createStatement();
String sql6 = "CREATE TABLE IF NOT EXISTS miningsim_itemsell(name varchar(200) primary key, price int)";
statement6.execute(sql6);
statement6.close();

Statement statement7 = getConnection().createStatement();
String sql7 = "CREATE TABLE IF NOT EXISTS miningsim_itempurchase(name varchar(200) primary key, itemname varchar(200), price int, itemlore varchar(200), itemtype int, itemdata int, enchantments varchar(200), enchantmentlevels varchar(200), expboost int, goldboost int)";
statement7.execute(sql7);
statement7.close();
```

All this is inside the Database class, and called when the plugin enables. If you are unfamiliar with Java, you probably shouldn't be reading this, but classes can have public functions inside the which can be called to do basically anything, bascically just to keep everything organized. Theoretically you could do everything inside one file, but it would be extremely cluttered and impossible to navigate.

Anyway, also inside the Database class is all the functions for interacting with the Database. Getting stats, setting stats, finding item prices, etc can all be done using the Database class. You'll find me mentioning and showing functions from this class a lot.

### Stats
Because a player's stats are made up of so many things, I've created a PlayerStats constructor to more easily interact with them. It would probably be easier to show you the constructor than type it out, so here you go...

```
public PlayerStats(String uuid, int exp, int level, int gold, int rebirth, int blocksMined) {
    this.uuid = uuid;
    this.exp = exp;
    this.level = level;
    this.gold = gold;
    this.rebirth = rebirth;
    this.blocksMined = blocksMined;
}
```

This constructor is used every time we interact with player stats.

But of course, this is just the boring backend stuff I'm explaining. You want to know how these stats actually get displayed to the player. We use PlaceholderAPI for that, so we can use the stats in as many places as possible. Here's an example of one of the placeholders.

```
if(params.equalsIgnoreCase("exp")) {
  try{
      PlayerStats stats = this.plugin.getDB().findPlayerStatsByUUID(player.getUniqueId().toString());

      if(stats == null){

          stats = new PlayerStats(player.getUniqueId().toString(), 0, 0, 0, 0, 0);

          this.plugin.getDB().createPlayerStats(stats);

          return "0";
      } else {
          return String.format("%,d", stats.getExp());
      }
  }catch(SQLException ex){
      ex.printStackTrace();
      return "fuck";
  }
}
```
I'm not gonna go into detail about every line of this code, but basically when the placeholder is requested it searches for that player's stats. If the player isn't in the database, it adds them to it and returns 0. If the player is in the database, it grabs their Exp and formats it to return. Oh yeah, and when it throws and error it returns a nice little message for the player.

Anyway, this is what makes basically everything that displays your stats in-game possible, such as this little hologram.

![The Cool Thing](https://cdn.unchld.me/img/uz10j.png)

## Unfinished
This page is unfinished. Return at a later time when unchilled feels like finishing it.
