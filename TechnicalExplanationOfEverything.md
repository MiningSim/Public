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

Anyway, also inside the Database class is all the functions for interacting with the Database. Getting stats, setting stats, finding item prices, etc can all be done using the Database class.

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

### Mines
Of course, the most important part of a Mining Simulator, mines! Mines are defined by block location, and in some cases only by block type. Every mine block is stored in a table called miningsim_mines.

For the actual mine drops and information, it's all stored in a different table called miningsim_mineinfo. I also created a constructor for mine info.

```
public MineInfo(String name, int exp, int gold, int levelreq, int rebirthreq, ItemStack item) {
    this.name = name;
    this.exp = exp;
    this.gold = gold;
    this.levelreq = levelreq;
    this.rebirthreq = rebirthreq;
    this.item = item;
}
```

The MineInfo constructor is used whenever interacting with mines. For example, when you mine a block, the plugin will define a new MineInfo object based on the info in the database. Then it pulls what it needs from the MineInfo object to check if you meet requirements and give you the drops/rewards for that mine.

Mine rewards can be multiplied using certain tools, so it also will get the item the player is holding and check for a custom NBT tag that defines the gold/exp boost, and uses that to adjust how much you get. Here is the whole thing put together.

```
String mine = this.plugin.getDB().checkMineBlock(event.getBlock());
if(mine != null){
    MineInfo mineInfo = this.plugin.getDB().getMineInfo(mine);
    if(mineInfo == null){
        player.sendMessage(ChatColor.RED + "This mine has not been configured.");
    } else {
        PlayerStats stats = this.plugin.getDB().findPlayerStatsByUUID(player.getUniqueId().toString());
        if(stats.getRebirth() >= mineInfo.getRebirthreq() && stats.getLevel() >= mineInfo.getLevelreq()){
            NBTItem nbtItem = null;
            boolean hasItem = false;
            if(player.getItemInHand().getType() != Material.AIR){
                nbtItem = new NBTItem(player.getItemInHand());
                hasItem = true;
            }
            int expBoost = 1;
            if(hasItem && nbtItem.hasKey("expboost")){
                expBoost = nbtItem.getInteger("expboost") + 1;
            }
            int exp = mineInfo.getExp() * expBoost;
            stats.setExp(stats.getExp() + exp);
            int goldBoost = 1;
            if(hasItem && nbtItem.hasKey("goldboost")){
                goldBoost = nbtItem.getInteger("goldboost") + 1;
            }
            int gold = mineInfo.getGold() * goldBoost;
            stats.setGold(stats.getGold() + gold);
            stats.setBlocksMined(stats.getBlocksMined() + 1);
            player.playSound(player.getLocation(), Sound.ORB_PICKUP, 1 , 1);
            ItemStack item = mineInfo.getItem();
            NBTItem nbtRewardItem = new NBTItem(item);
            ItemStack almostDoneItem = nbtRewardItem.getItem();
            if(hasItem && nbtItem.hasKey("goldboost") && nbtItem.getInteger("goldboost") > 0){
                nbtRewardItem.setInteger("goldmultiplier", goldBoost);
                almostDoneItem = nbtRewardItem.getItem();
                ItemMeta meta = almostDoneItem.getItemMeta();
                List<String> lore = meta.getLore();
                lore.add(ChatColor.GOLD + "(" + goldBoost + "x Gold)");
                meta.setLore(lore);
                almostDoneItem.setItemMeta(meta);
            }
            player.getInventory().addItem(almostDoneItem);
            player.updateInventory();
            if(mineInfo.getGold() > 0 && mineInfo.getExp() > 0){
                ActionBarAPI.sendActionBar(player, ChatColor.AQUA + "+" + String.format("%,d", exp) + "✦    " + ChatColor.GOLD + "+" + String.format("%,d", gold) + "$");
            } else {
                if(mineInfo.getGold() > 0){
                    ActionBarAPI.sendActionBar(player,ChatColor.GOLD + "+" + String.format("%,d", gold) + "$");
                }
                if(mineInfo.getExp() > 0){
                    ActionBarAPI.sendActionBar(player,ChatColor.AQUA + "+" + String.format("%,d", exp) + "✦");
                }
            }
            this.plugin.getDB().updatePlayerStats(stats);
        } else {
            String reason = "";
            if(stats.getLevel() < mineInfo.getLevelreq()){
                reason = ChatColor.WHITE + "Level Requirement: " + ChatColor.LIGHT_PURPLE + String.format("%,d", mineInfo.getLevelreq()) + "❂";
            }
            if(stats.getRebirth() < mineInfo.getRebirthreq()){
                reason = ChatColor.WHITE + "Rebirth Requirement: " + ChatColor.RED + String.format("%,d", mineInfo.getRebirthreq()) + "✯";
            }
            player.sendTitle(ChatColor.RED + "You can't mine here!", reason);
        }
    }
    event.setCancelled(true);
}
```

As you can see, theres quite a bit of stuff in there. This code doesn't apply to crops and wood though, because those need to be regenerated differently. When a crop is broken, for example, rather than replacing it instantly, the wheat needs to regenerate randomly, to appear natural. For that, I used a BukkitRunnable. Here is that code.

```
block.setType(Material.AIR);
player.playSound(player.getLocation(), Sound.ORB_PICKUP, 1 , (float) 0.5);
int randomNum = ThreadLocalRandom.current().nextInt(200, 600 + 1);
new BukkitRunnable() {
    @Override
    public void run() {
        // What you want to schedule goes here
        block.setTypeId(59);
        block.setData((byte) 7);
    }
}.runTaskLater(this.plugin, randomNum);
```

### Item Selling
In order to sell items, we need a place to store the values for those items. I bet you can't guess where we'll do that. Thats right, a table in the database! This one is called miningsim_itemsell. It stores the name of the item, and the value of the item. I didn't make it check for lore because it's unlikely 2 items will have the same name but different values. Well, actually that's exactly what happened when I added multipliers but I decided to go for a custom NBT tag on that too. NBTAPI is quite useful for stuff like this.

```
Player p = (Player) sender;

int sold = 0;
int gold = 0;
for(int i = 0; i < p.getInventory().getSize(); i++) {
    try{
        if(p.getInventory().getItem(i) != null && p.getInventory().getItem(i).hasItemMeta() && p.getInventory().getItem(i).getItemMeta().hasDisplayName()){
            int price = getDB().getItemPrice(p.getInventory().getItem(i));
            NBTItem nbt = new NBTItem(p.getInventory().getItem(i));
            if(nbt.hasKey("goldmultiplier")){
                price = price * nbt.getInteger("goldmultiplier");
            }
            int stackSize = p.getInventory().getItem(i).getAmount();
            if(price > 0){
                int newprice = stackSize * price;
                gold = newprice + gold;
                sold = stackSize + sold;
                p.getInventory().setItem(i, null);
            }
        }

    }catch(SQLException ex){
        ex.printStackTrace();
        return true;
    }
}
if(sold > 0) {
    p.sendMessage(ChatColor.GREEN + "Sold " + String.format("%,d", sold) + " items for " + ChatColor.GOLD + String.format("%,d", gold) + "$");
    p.playSound(p.getLocation(), Sound.LEVEL_UP, 1, 2);
} else {
    p.sendMessage(ChatColor.RED + "You have nothing to sell.");
    p.playSound(p.getLocation(), Sound.VILLAGER_NO, 1, 1);
}
try{
    PlayerStats stats = getDB().findPlayerStatsByUUID(p.getUniqueId().toString());
    stats.setGold(stats.getGold() + gold);
    getDB().updatePlayerStats(stats);
}catch(SQLException ex){
    ex.printStackTrace();
}
return true;
```
As you can see, there are 2 variables initialized immediately when the command is received. The sold variable is incremented every time an ItemStack with a value is found in the player's inventory, by the amount in that ItemStack. The gold variable is similar, but increments by the value of the item multiplied by the amount of the item. This tells me at the end how much gold to give the player.

You can see the NBT tag I used in there too, which is added to the item when its broken with a tool that has a multiplier tag, which you can scroll up and see too.

### Item Purchasing
Because our gold is not based on any economy plugin, I had to manually create an item purchasing system. This is done through a command run by console, because all the shops are in GUIs, which I can make run console commands. The purchaseable items are all stored in a, you guessed it, table in the database. This one is called miningsim_itempurchase. Inside miningsim_itempurchase is a name key for the item, so we can use the command and a normal name on it, the item's metadata, item enchantments, the price, and exp/gold boosts.

There are enough variables involved in this I made a constructor for it too, which is here.

```
public PurchaseInfo(String name, String itemname, int price, String[] lore, int type, byte data, Enchantment[] enchantments, int[] enchantmentLevels, int expboost, int goldboost) {
    this.name = name;
    this.itemname = itemname;
    this.price = price;
    this.lore = lore;
    this.type = type;
    this.data = data;
    this.enchantments = enchantments;
    this.enchantmentLevels = enchantmentLevels;
    this.expboost = expboost;
    this.goldboost = goldboost;
}
```
And here's the code for the purchase command.

```
if(args.length != 2){
    sender.sendMessage(ChatColor.RED + "You must specify the player and the item.");
    return true;
}
String item = args[1];
OfflinePlayer op = Bukkit.getOfflinePlayer(args[0]);
Player p = op.getPlayer();
if(p == null){
    sender.sendMessage(ChatColor.RED + "That player is offline.");
    return true;
}
try{
    PurchaseInfo info = getDB().getPurchaseInfo(item);
    PlayerStats stats = getDB().findPlayerStatsByUUID(p.getUniqueId().toString());
    int gold = stats.getGold();
    if(gold < info.getPrice()){
        p.sendMessage(ChatColor.RED + "You need " + ChatColor.GOLD + String.format("%,d", info.getPrice()) + "$" + ChatColor.RED + " to purchase this item.");
        p.playSound(p.getLocation(), Sound.VILLAGER_NO, 1, 1);
        return true;
    }
    stats.setGold(gold - info.getPrice());
    getDB().updatePlayerStats(stats);
    ItemStack itemStack = new ItemStack(info.getType(), 1);
    itemStack.setData(new MaterialData(info.getType(), info.getData()));
    ItemMeta itemMeta = itemStack.getItemMeta();
    itemMeta.setDisplayName(info.getItemname());
    itemMeta.setLore(Arrays.asList(info.getLore()));
    itemMeta.addItemFlags(ItemFlag.HIDE_UNBREAKABLE);
    itemMeta.addItemFlags(ItemFlag.HIDE_ATTRIBUTES);
    itemMeta.addItemFlags(ItemFlag.HIDE_ENCHANTS);
    itemMeta.spigot().setUnbreakable(true);
    itemStack.setItemMeta(itemMeta);
    Enchantment[] enchantments = info.getEnchantments();
    int[] enchantmentLevels = info.getEnchantmentLevels();
    for(int i = 0; i < enchantments.length; i++){
        itemStack.addUnsafeEnchantment(enchantments[i], enchantmentLevels[i]);
    }
    NBTItem nbt = new NBTItem(itemStack);
    nbt.setInteger("expboost", info.getExpboost());
    nbt.setInteger("goldboost", info.getGoldboost());
    p.getInventory().addItem(nbt.getItem());
    p.playSound(p.getLocation(), Sound.ORB_PICKUP, 1, 1);
    p.sendMessage(ChatColor.GREEN + "You purchased " + info.getItemname() + ChatColor.GREEN + " for " + ChatColor.GOLD + String.format("%,d", info.getPrice()) + "$");
    return true;
}catch(SQLException ex){
    ex.printStackTrace();
    return true;
}
```
The enchantments are stored in a kinda weird way, since they need to go into the database in string form. I chose to do 2 seperate columns, one for enchantment names and one for enchantment levels. In the constructor, I store them as Enchantment objects in an array, so they're easy to use when giving players items. In order to convert the strings into Enchantments, this code is used in the Database class

```
public PurchaseInfo getPurchaseInfo(String name) throws SQLException{
    PreparedStatement statement = getConnection().prepareStatement("SELECT * FROM miningsim_itempurchase WHERE name = ?");
    statement.setString(1, name);
    ResultSet results = statement.executeQuery();

    if(results.next()) {

        String itemname = results.getString("itemname").replace("&", "§");
        int price = results.getInt("price");
        String[] itemlore = results.getString("itemlore").replace("&", "§").split("<br>");
        int itemtype = results.getInt("itemtype");
        byte itemdata = (byte) results.getInt("itemdata");
        String[] enchantmentstrings = results.getString("enchantments").split(";");
        Enchantment[] enchantments = new Enchantment[enchantmentstrings.length];
        for(int i = 0; i < enchantmentstrings.length; i++){
            String enchantmentstring = enchantmentstrings[i];
            enchantments[i] = Enchantments.getByName(enchantmentstring);
        }
        String[] numberStrs = results.getString("enchantmentlevels").split(";");
        int[] enchantmentLevels = new int[numberStrs.length];
        for(int i = 0;i < numberStrs.length;i++)
        {
            enchantmentLevels[i] = Integer.parseInt(numberStrs[i]);
        }
        int expBoost = results.getInt("expboost");
        int goldBoost = results.getInt("goldboost");

        return new PurchaseInfo(name, itemname, price, itemlore, itemtype, itemdata, enchantments, enchantmentLevels, expBoost, goldBoost);
    }

    statement.close();

    return null;
}
```
## Menus
Our menus are handled by a different plugin, called CommandPanels. CommandPanels is an extremely powerful plugin, and has insane amounts of customization. Anything you could ever want to do in a menu, you can do it with this plugin. I find this extremely useful, because manually making GUIs is very tedious.

Here's the main menu config.

```
panels:
  menu:
    perm: default
    rows: 4
    title: '&8Menu'
    empty: STAINED_GLASS_PANE
    emptyID: 7
    commands:
    - menu
    - m
    item:
      '10':
        material: CHEST
        stack: 1
        name: '&eStorage'
        commands:
        - open= storage
        - console= playsound gui.button.press %cp-player-name% %cp-player-x% %cp-player-y% %cp-player-z% 1 1
      '11':
        material: WORKBENCH
        stack: 1
        name: '&9Craft'
        commands:
        - cpc
        - sudo= craft
      '12':
        material: GOLD_INGOT
        stack: 1
        name: '&6Sell'
        lore:
        - '&eValue: &6%miningsim_value%$'
        commands:
        - cpc
        - sudo= sell
      '13':
        material: cps= self
        stack: 1
        name: '&aMy Profile'
        lore: 
        - '&3Exp: &b%miningsim_exp%✦'
        - '&5Level: &d%miningsim_level%❂'
        - '&eGold: &6%miningsim_gold%$'
        - '&4Rebirth: &c%miningsim_rebirth%✯'
        - '&7Blocks Mined: &f%miningsim_blocksMined%'
        commands:
        - open= myprofile
        - console= playsound gui.button.press %cp-player-name% %cp-player-x% %cp-player-y% %cp-player-z% 1 1
      '14':
        material: ENDER_PEARL
        stack: 1
        name: '&5Warp Menu'
        commands:
        - open= warp1
        - console= playsound gui.button.press %cp-player-name% %cp-player-x% %cp-player-y% %cp-player-z% 1 1
      '15':
        material: DIAMOND
        stack: 1
        name: '&dLevel Up'
        lore:
        - '&3Required EXP: &b1,000✦'
        commands:
        - sudo= levelup
        - cpc
      '16':
        material: REDSTONE
        stack: 1
        name: '&cRebirth'
        lore:
        - '&5Required Level: &d10,000❂'
        commands:
        - sudo= rebirth
        - cpc
      '22':
        material: CAULDRON_ITEM
        stack: 1
        name: '&4Trash Can'
        commands:
        - open= trashcan
        - console= playsound gui.button.press %cp-player-name% %cp-player-x% %cp-player-y% %cp-player-z% 1 1
      '31':
        material: BARRIER
        stack: 1
        name: '&cClose'
        commands:
        - cpc
        - console= playsound gui.button.press %cp-player-name% %cp-player-x% %cp-player-y% %cp-player-z% 1 1
      
    open-with-item:
      material: NETHER_STAR
      name: '&bMenu'
      lore:
      - '&7(Right Click)'
      stationary: 8
```
I love being able to quickly edit the names and lores of items on the fly, and just run a quick reload command to see it in-game. Anything custom I need to do can simply be done by MSPlugin, and get triggered by this menu. Having a menu makes the game a lot more polished feeling. Custom menus tend to not be something used by servers with low effort or quality, so having one, especially one this integrated into gameplay, feels a lot better.

## Crafting

In order to make the game feel completely different from normal Minecraft, which is one of the goals of this project, crafting had to be completely redone. I found a great plugin for custom recipes, called ProRecipes. Unfortunately, this plugin does not disable vanilla recipes itself, and it's quite difficult to do so myself. So, I used MSPlugin to handle that too.

```
Iterator<Recipe> recipes = getServer().recipeIterator();
Recipe recipe;
while (recipes.hasNext()) {
    recipe = recipes.next();
    if (recipe != null && !recipe.getResult().hasItemMeta()) recipes.remove();
}
```
When the plugin is loaded, it deletes all the recipes where the result doesn't have an item meta, because every one of our custom crafting recipes will have an item meta. Shockingly small and simple code for fixing and issue I spent about an hour on.

## Ending
Well, I hope you learned something or another from reading this. I didn't.
