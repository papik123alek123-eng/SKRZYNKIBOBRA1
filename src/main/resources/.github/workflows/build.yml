package pl.skrzynki;

import org.bukkit.*;
import org.bukkit.block.Block;
import org.bukkit.block.BlockFace;
import org.bukkit.block.data.Directional;
import org.bukkit.command.Command;
import org.bukkit.command.CommandSender;
import org.bukkit.configuration.file.YamlConfiguration;
import org.bukkit.entity.Display;
import org.bukkit.entity.Entity;
import org.bukkit.entity.Firework;
import org.bukkit.entity.ItemDisplay;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.block.Action;
import org.bukkit.event.block.BlockBreakEvent;
import org.bukkit.event.block.BlockExplodeEvent;
import org.bukkit.event.block.BlockPlaceEvent;
import org.bukkit.event.entity.EntityExplodeEvent;
import org.bukkit.event.inventory.InventoryCloseEvent;
import org.bukkit.event.player.PlayerInteractEvent;
import org.bukkit.inventory.EquipmentSlot;
import org.bukkit.inventory.Inventory;
import org.bukkit.inventory.InventoryHolder;
import org.bukkit.inventory.ItemStack;
import org.bukkit.inventory.meta.FireworkMeta;
import org.bukkit.inventory.meta.ItemMeta;
import org.bukkit.persistence.PersistentDataType;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.scheduler.BukkitRunnable;
import org.jetbrains.annotations.NotNull;

import java.io.File;
import java.io.IOException;
import java.util.*;
import java.util.stream.Collectors;

public class Skrzynki extends JavaPlugin implements Listener {

    private NamespacedKey crateKey;
    private File dataFile;
    private YamlConfiguration data;
    private final Set<Block> busy = new HashSet<>();
    private final Set<Entity> displays = new HashSet<>();
    private final Random random = new Random();

    /** Holder rozpoznajacy okno edycji dropu. */
    private static class EditHolder implements InventoryHolder {
        final String crate;
        Inventory inv;
        EditHolder(String crate) { this.crate = crate; }
        @Override public @NotNull Inventory getInventory() { return inv; }
    }

    @Override
    public void onEnable() {
        crateKey = new NamespacedKey(this, "crate");
        getDataFolder().mkdirs();
        dataFile = new File(getDataFolder(), "skrzynki.yml");
        data = YamlConfiguration.loadConfiguration(dataFile);
        Bukkit.getPluginManager().registerEvents(this, this);
        getLogger().info("Skrzynki wlaczone. Zaladowano skrzynek: " + crateNames().size());
    }

    @Override
    public void onDisable() {
        for (Entity e : displays) e.remove();
        displays.clear();
        for (Block b : busy) b.setType(Material.AIR);
        busy.clear();
    }

    // ------------------------------------------------------------ dane

    private void save() {
        try { data.save(dataFile); } catch (IOException ex) { getLogger().severe("Nie mozna zapisac: " + ex.getMessage()); }
    }

    private Set<String> crateNames() {
        var sec = data.getConfigurationSection("crates");
        return sec == null ? Set.of() : sec.getKeys(false);
    }

    private boolean exists(String name) { return data.contains("crates." + name); }

    private List<ItemStack> loadItems(String name) {
        List<ItemStack> out = new ArrayList<>();
        for (Object o : data.getList("crates." + name + ".items", List.of())) {
            if (o instanceof ItemStack is) out.add(is);
        }
        return out;
    }

    private ItemStack crateItem(String name) {
        ItemStack it = new ItemStack(Material.CHEST);
        ItemMeta m = it.getItemMeta();
        m.setDisplayName("§6§lSkrzynka §e" + name);
        m.setLore(List.of("§7Kliknij PPM na ziemie,", "§7aby postawic i otworzyc skrzynke!"));
        m.getPersistentDataContainer().set(crateKey, PersistentDataType.STRING, name);
        m.setEnchantmentGlintOverride(true);
        it.setItemMeta(m);
        return it;
    }

    private String crateOf(ItemStack it) {
        if (it == null || !it.hasItemMeta()) return null;
        return it.getItemMeta().getPersistentDataContainer().get(crateKey, PersistentDataType.STRING);
    }

    private void give(Player p, ItemStack it) {
        p.getInventory().addItem(it).values().forEach(rest -> p.getWorld().dropItemNaturally(p.getLocation(), rest));
    }

    // ------------------------------------------------------------ komendy

    @Override
    public boolean onCommand(@NotNull CommandSender s, @NotNull Command cmd, @NotNull String label, @NotNull String[] a) {
        String c = cmd.getName().toLowerCase();
        switch (c) {
            case "skrzynkastworz" -> {
                if (!(s instanceof Player p)) { s.sendMessage("Tylko dla graczy."); return true; }
                if (a.length < 1) { s.sendMessage("§cUzycie: /skrzynkastworz <nazwa>"); return true; }
                String name = a[0].toLowerCase();
                if (exists(name)) { s.sendMessage("§cSkrzynka §e" + name + " §cjuz istnieje."); return true; }
                data.set("crates." + name + ".items", new ArrayList<ItemStack>());
                save();
                give(p, crateItem(name));
                s.sendMessage("§aStworzono skrzynke §e" + name + "§a. Ustaw drop: §e/skrzynkaedytuj " + name);
            }
            case "skrzynkaedytuj" -> {
                if (!(s instanceof Player p)) { s.sendMessage("Tylko dla graczy."); return true; }
                if (a.length < 1 || !exists(a[0].toLowerCase())) { s.sendMessage("§cUzycie: /skrzynkaedytuj <nazwa> (taka skrzynka musi istniec)"); return true; }
                String name = a[0].toLowerCase();
                EditHolder h = new EditHolder(name);
                h.inv = Bukkit.createInventory(h, 54, "Drop: " + name);
                loadItems(name).forEach(h.inv::addItem);
                p.openInventory(h.inv);
                p.sendMessage("§7Wloz przedmioty, ktore maja wypadac. Zamknij okno, aby zapisac.");
            }
            case "skrzynkadaj" -> {
                if (a.length < 2) { s.sendMessage("§cUzycie: /skrzynkadaj <gracz> <nazwa> [ilosc]"); return true; }
                Player t = Bukkit.getPlayerExact(a[0]);
                String name = a[1].toLowerCase();
                if (t == null) { s.sendMessage("§cGracz offline."); return true; }
                if (!exists(name)) { s.sendMessage("§cNie ma takiej skrzynki."); return true; }
                int n = 1;
                try { if (a.length > 2) n = Math.max(1, Math.min(64, Integer.parseInt(a[2]))); } catch (NumberFormatException ignored) {}
                ItemStack it = crateItem(name);
                it.setAmount(n);
                give(t, it);
                s.sendMessage("§aDano §e" + n + "x §askrzynke §e" + name + " §agraczowi " + t.getName());
            }
            case "skrzynkausun" -> {
                if (a.length < 1 || !exists(a[0].toLowerCase())) { s.sendMessage("§cUzycie: /skrzynkausun <nazwa>"); return true; }
                data.set("crates." + a[0].toLowerCase(), null);
                save();
                s.sendMessage("§aUsunieto skrzynke §e" + a[0].toLowerCase());
            }
        }
        return true;
    }

    @Override
    public List<String> onTabComplete(@NotNull CommandSender s, @NotNull Command cmd, @NotNull String alias, @NotNull String[] a) {
        String c = cmd.getName().toLowerCase();
        String last = a[a.length - 1].toLowerCase();
        List<String> opts = new ArrayList<>();
        if ((c.equals("skrzynkaedytuj") || c.equals("skrzynkausun")) && a.length == 1) opts.addAll(crateNames());
        else if (c.equals("skrzynkadaj")) {
            if (a.length == 1) Bukkit.getOnlinePlayers().forEach(p -> opts.add(p.getName()));
            else if (a.length == 2) opts.addAll(crateNames());
        }
        return opts.stream().filter(x -> x.toLowerCase().startsWith(last)).collect(Collectors.toList());
    }

    // ------------------------------------------------------------ eventy

    @EventHandler
    public void onClose(InventoryCloseEvent e) {
        if (!(e.getInventory().getHolder() instanceof EditHolder h)) return;
        List<ItemStack> items = new ArrayList<>();
        for (ItemStack it : e.getInventory().getContents()) if (it != null && !it.getType().isAir()) items.add(it);
        data.set("crates." + h.crate + ".items", items);
        save();
        e.getPlayer().sendMessage("§aZapisano drop skrzynki §e" + h.crate + "§a (" + items.size() + " przedmiotow).");
    }

    @EventHandler
    public void onPlace(BlockPlaceEvent e) {
        if (crateOf(e.getItemInHand()) != null) e.setCancelled(true);
    }

    @EventHandler
    public void onBreak(BlockBreakEvent e) {
        if (busy.contains(e.getBlock())) e.setCancelled(true);
    }

    @EventHandler
    public void onExplode(EntityExplodeEvent e) { e.blockList().removeIf(busy::contains); }

    @EventHandler
    public void onBlockExplode(BlockExplodeEvent e) { e.blockList().removeIf(busy::contains); }

    @EventHandler
    public void onInteract(PlayerInteractEvent e) {
        if (e.getHand() != EquipmentSlot.HAND) return;
        if (e.getClickedBlock() != null && busy.contains(e.getClickedBlock())) { e.setCancelled(true); return; }
        if (e.getAction() != Action.RIGHT_CLICK_BLOCK) return;
        ItemStack inHand = e.getItem();
        String name = crateOf(inHand);
        if (name == null) return;
        e.setCancelled(true);

        Player p = e.getPlayer();
        if (!exists(name)) { p.sendMessage("§cTa skrzynka juz nie istnieje."); return; }
        List<ItemStack> items = loadItems(name);
        if (items.isEmpty()) { p.sendMessage("§cTa skrzynka nie ma ustawionego dropu!"); return; }

        Block target = e.getClickedBlock().getRelative(e.getBlockFace());
        if (!target.getType().isAir() || busy.contains(target)) { p.sendMessage("§cTu nie da sie postawic skrzynki."); return; }

        if (p.getGameMode() != GameMode.CREATIVE) inHand.setAmount(inHand.getAmount() - 1);
        animate(p, target, items, name);
    }

    // ------------------------------------------------------------ animacja

    private void animate(Player p, Block block, List<ItemStack> items, String name) {
        busy.add(block);
        block.setType(Material.CHEST);
        if (block.getBlockData() instanceof Directional d) {
            BlockFace f = p.getFacing().getOppositeFace();
            if (d.getFaces().contains(f)) { d.setFacing(f); block.setBlockData(d); }
        }

        World w = block.getWorld();
        Location base = block.getLocation().add(0.5, 0, 0.5);
        ItemStack reward = items.get(random.nextInt(items.size())).clone();

        ItemDisplay disp = w.spawn(base.clone().add(0, 1.0, 0), ItemDisplay.class, d -> {
            d.setItemStack(items.get(0));
            d.setBillboard(Display.Billboard.FIXED);
            d.setTransformation(new org.bukkit.util.Transformation(
                    new org.joml.Vector3f(0, 0, 0), new org.joml.Quaternionf(),
                    new org.joml.Vector3f(1.2f, 1.2f, 1.2f), new org.joml.Quaternionf()));
            d.setInterpolationDuration(1);
            d.setTeleportDuration(1);
        });
        displays.add(disp);

        new BukkitRunnable() {
            int t = 0;
            final int OPEN = 15, ROLL = 75, END = 105;

            @Override
            public void run() {
                t++;

                if (t == 1) {
                    w.playSound(base, Sound.BLOCK_CHEST_OPEN, 1f, 1f);
                    w.playSound(base, Sound.BLOCK_BEACON_ACTIVATE, 0.6f, 1.6f);
                    for (Player v : w.getNearbyPlayers(base, 64)) v.sendBlockAction(block.getLocation(), Material.CHEST, 1, 1);
                }

                // przedmiot wznosi sie nad skrzynke i sie kreci
                double progress = Math.min(1.0, t / (double) ROLL);
                double eased = 1 - Math.pow(1 - progress, 3);
                double y = 1.0 + eased * 1.8 + Math.sin(t / 5.0) * 0.08;
                Location dl = base.clone().add(0, y, 0);
                dl.setYaw((t * 14f) % 360);
                disp.teleport(dl);

                // spirala czastek wokol przedmiotu
                double ang = t * 0.5;
                double radius = 0.9 - 0.4 * progress;
                for (int i = 0; i < 2; i++) {
                    double a = ang + i * Math.PI;
                    Location pl = base.clone().add(Math.cos(a) * radius, 0.3 + eased * 2.2, Math.sin(a) * radius);
                    w.spawnParticle(Particle.END_ROD, pl, 1, 0, 0, 0, 0);
                    w.spawnParticle(Particle.ENCHANT, dl, 3, 0.4, 0.3, 0.4, 0.5);
                }
                if (t % 4 == 0) w.spawnParticle(Particle.FIREWORK, base.clone().add(0, 1.2, 0), 4, 0.3, 0.1, 0.3, 0.05);

                // "ruletka" - zmiana wyswietlanego przedmiotu, zwalnia pod koniec
                if (t > OPEN && t < ROLL) {
                    int interval = t < 45 ? 3 : t < 60 ? 5 : 8;
                    if (t % interval == 0) {
                        disp.setItemStack(items.get(random.nextInt(items.size())));
                        w.playSound(dl, Sound.BLOCK_NOTE_BLOCK_CHIME, 0.7f, 0.8f + (float) progress);
                    }
                }

                // wynik
                if (t == ROLL) {
                    disp.setItemStack(reward);
                    w.playSound(dl, Sound.ENTITY_PLAYER_LEVELUP, 1f, 1f);
                    w.playSound(dl, Sound.ENTITY_FIREWORK_ROCKET_TWINKLE, 1f, 1f);
                    w.spawnParticle(Particle.TOTEM_OF_UNDYING, dl, 60, 0.4, 0.4, 0.4, 0.4);
                    Firework fw = w.spawn(dl, Firework.class);
                    FireworkMeta fm = fw.getFireworkMeta();
                    fm.addEffect(FireworkEffect.builder().with(FireworkEffect.Type.BALL_LARGE)
                            .withColor(Color.YELLOW, Color.ORANGE).withFade(Color.WHITE).withFlicker().withTrail().build());
                    fw.setFireworkMeta(fm);
                    fw.detonate();
                }

                if (t > ROLL && t % 3 == 0) w.spawnParticle(Particle.HAPPY_VILLAGER, dl, 3, 0.4, 0.4, 0.4, 0);

                // koniec: zamkniecie skrzynki i wydanie nagrody
                if (t >= END) {
                    cancel();
                    disp.remove();
                    displays.remove(disp);
                    for (Player v : w.getNearbyPlayers(base, 64)) v.sendBlockAction(block.getLocation(), Material.CHEST, 1, 0);
                    w.playSound(base, Sound.BLOCK_CHEST_CLOSE, 1f, 1f);
                    w.spawnParticle(Particle.CLOUD, base.clone().add(0, 0.6, 0), 25, 0.3, 0.3, 0.3, 0.05);
                    block.setType(Material.AIR);
                    busy.remove(block);

                    if (p.isOnline()) {
                        give(p, reward);
                        p.sendMessage("§6Skrzynka §e" + name + "§6: wylosowales §a" + reward.getAmount() + "x "
                                + (reward.getItemMeta() != null && reward.getItemMeta().hasDisplayName()
                                ? reward.getItemMeta().getDisplayName()
                                : reward.getType().name().toLowerCase().replace('_', ' ')) + "§6!");
                    } else {
                        w.dropItemNaturally(base.clone().add(0, 1, 0), reward);
                    }
                }
            }
        }.runTaskTimer(this, 0L, 1L);
    }
}
