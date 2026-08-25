# Mod "Nuke Gun" — Bản cập nhật cho Minecraft/Forge 26.2

## ⚠️ Đọc trước khi làm

Minecraft 26.2 dùng hệ đánh version mới (26 = năm 2026, .2 = đợt phát hành thứ 2), thay cho kiểu "1.20.1" cũ. Đây là thông tin phát sinh sau thời điểm dữ liệu huấn luyện của mình, nên:

- Phần **kiến trúc mod** (Gradle, DeferredRegister, RegistryObject, mods.toml, cách đăng ký item/entity) mình đã tra cứu và xác nhận **vẫn dùng được** cho 26.2 — đây là phần khung sườn ổn định qua nhiều bản.
- Phần **API sâu bên trong Minecraft** (tên chính xác một số class entity/particle, chữ ký hàm nổ...) mình **không thể đảm bảo chính xác tuyệt đối 100%**, vì Minecraft đổi nội dung mỗi quý và mình không có cách tự build để kiểm tra thật.

**Quan trọng nhất:** nếu lúc chạy `./gradlew build` bị lỗi đỏ, **chụp/copy nguyên văn dòng lỗi gửi cho mình** — lỗi biên dịch cho biết chính xác tên hàm/class thật trong bản 26.2, mình sẽ sửa đúng 100% dựa trên đó, nhanh hơn nhiều so với việc mình đoán trước.

---

## PHẦN 1: Chuẩn bị

1. Tải Forge MDK cho **26.2** tại `files.minecraftforge.net` (đúng như ảnh bạn gửi) — bấm **Mdk** ở khối "Download Latest" hoặc "Download Recommended".
2. Cài **JDK 21 trở lên** (không phải JDK 17 như bản cũ). Một số nguồn cho biết 26.2 yêu cầu Java 25 — nếu build báo lỗi liên quan "class file version" hoặc "release version", hãy kiểm tra đúng file `build.gradle` trong MDK vừa tải (dòng có chữ `JavaVersion` hoặc `sourceCompatibility`) để biết chính xác bản Java cần dùng, và cài đúng bản đó.
3. Giải nén MDK ra, mở bằng Codespaces/IDE như hướng dẫn trước.

---

## PHẦN 2: Cấu trúc thư mục (giữ nguyên như trước)

```
src/main/java/com/nukegun/
├── NukeGunMod.java
├── ClientSetup.java
├── item/
│   ├── ModItems.java
│   └── NukeGunItem.java
├── entity/
│   ├── ModEntities.java
│   └── NukeProjectile.java
└── client/
    └── NukeProjectileRenderer.java

src/main/resources/
├── META-INF/mods.toml
├── assets/nukegun/
│   ├── models/item/nuke_gun.json
│   ├── textures/item/nuke_gun.png
│   ├── textures/entity/nuke_bomb.png
│   └── lang/en_us.json
```

> **Lưu ý mới:** MDK bản mới có thể đã dùng file `gradle.properties` để lưu sẵn `mod_id`, `mod_version`... và tự thay thế `${mod_id}` vào `mods.toml` khi build (thay vì bạn tự sửa tay). Sau khi giải nén, mở file `gradle.properties` trước — nếu thấy dòng `mod_id = examplemod`, sửa thành `mod_id = nukegun` ở đó, và kiểm tra xem `mods.toml` có đang dùng `${mod_id}` hay ghi cứng chữ `examplemod` để sửa cho đúng.

---

## PHẦN 3: Code Java (giữ nguyên kiến trúc, đã xác minh phần khung Forge)

### 3.1 `NukeGunMod.java`

```java
package com.nukegun;

import com.nukegun.entity.ModEntities;
import com.nukegun.item.ModItems;
import net.minecraft.world.item.CreativeModeTabs;
import net.minecraftforge.event.BuildCreativeModeTabContentsEvent;
import net.minecraftforge.eventbus.api.IEventBus;
import net.minecraftforge.fml.common.Mod;
import net.minecraftforge.fml.javafmlmod.FMLJavaModLoadingContext;

@Mod(NukeGunMod.MOD_ID)
public class NukeGunMod {
    public static final String MOD_ID = "nukegun";

    public NukeGunMod() {
        IEventBus modEventBus = FMLJavaModLoadingContext.get().getModEventBus();

        ModItems.ITEMS.register(modEventBus);
        ModEntities.ENTITIES.register(modEventBus);

        modEventBus.addListener(this::addCreative);
    }

    private void addCreative(BuildCreativeModeTabContentsEvent event) {
        if (event.getTabKey() == CreativeModeTabs.COMBAT) {
            event.accept(ModItems.NUKE_GUN);
        }
    }
}
```

> Nếu build báo lỗi không tìm thấy `BuildCreativeModeTabContentsEvent` hoặc `CreativeModeTabs.COMBAT`, đây là điểm dễ đổi tên nhất giữa các bản Minecraft — gửi lỗi cho mình để sửa tên chính xác.

### 3.2 `item/ModItems.java`

```java
package com.nukegun.item;

import com.nukegun.NukeGunMod;
import net.minecraft.world.item.Item;
import net.minecraftforge.registries.DeferredRegister;
import net.minecraftforge.registries.ForgeRegistries;
import net.minecraftforge.registries.RegistryObject;

public class ModItems {
    public static final DeferredRegister<Item> ITEMS =
            DeferredRegister.create(ForgeRegistries.ITEMS, NukeGunMod.MOD_ID);

    public static final RegistryObject<Item> NUKE_GUN = ITEMS.register("nuke_gun",
            () -> new NukeGunItem(new Item.Properties().stacksTo(1).durability(50)));
}
```

### 3.3 `item/NukeGunItem.java`

```java
package com.nukegun.item;

import com.nukegun.entity.ModEntities;
import com.nukegun.entity.NukeProjectile;
import net.minecraft.sounds.SoundEvents;
import net.minecraft.sounds.SoundSource;
import net.minecraft.world.InteractionHand;
import net.minecraft.world.InteractionResultHolder;
import net.minecraft.world.entity.player.Player;
import net.minecraft.world.item.Item;
import net.minecraft.world.item.ItemStack;
import net.minecraft.world.item.UseAnim;
import net.minecraft.world.level.Level;

public class NukeGunItem extends Item {

    public NukeGunItem(Properties properties) {
        super(properties);
    }

    @Override
    public InteractionResultHolder<ItemStack> use(Level level, Player player, InteractionHand hand) {
        ItemStack stack = player.getItemInHand(hand);

        if (!level.isClientSide) {
            NukeProjectile projectile = new NukeProjectile(ModEntities.NUKE_PROJECTILE.get(), level);
            projectile.setPos(
                    player.getX() - (player.getLookAngle().x * 0.5),
                    player.getEyeY() - 0.2,
                    player.getZ() - (player.getLookAngle().z * 0.5)
            );
            projectile.setOwner(player);

            float power = 1.6f;
            float inaccuracy = 0.5f;
            projectile.shootFromRotation(player, player.getXRot() - 10f, player.getYRot(), 0.0f, power, inaccuracy);

            level.addFreshEntity(projectile);

            level.playSound(null, player.getX(), player.getY(), player.getZ(),
                    SoundEvents.GENERIC_EXPLODE.get(), SoundSource.PLAYERS, 0.5f, 1.6f);

            if (!player.getAbilities().instabuild) {
                stack.hurtAndBreak(1, player, (p) -> p.broadcastBreakEvent(hand));
            }
        }

        player.getCooldowns().addCooldown(this, 30);
        return InteractionResultHolder.sidedSuccess(stack, level.isClientSide());
    }

    @Override
    public UseAnim getUseAnimation(ItemStack stack) {
        return UseAnim.NONE;
    }
}
```

### 3.4 `entity/ModEntities.java`

```java
package com.nukegun.entity;

import com.nukegun.NukeGunMod;
import net.minecraft.world.entity.EntityType;
import net.minecraft.world.entity.MobCategory;
import net.minecraftforge.registries.DeferredRegister;
import net.minecraftforge.registries.ForgeRegistries;
import net.minecraftforge.registries.RegistryObject;

public class ModEntities {
    public static final DeferredRegister<EntityType<?>> ENTITIES =
            DeferredRegister.create(ForgeRegistries.ENTITY_TYPES, NukeGunMod.MOD_ID);

    public static final RegistryObject<EntityType<NukeProjectile>> NUKE_PROJECTILE =
            ENTITIES.register("nuke_projectile", () -> EntityType.Builder.<NukeProjectile>of(NukeProjectile::new, MobCategory.MISC)
                    .sized(0.5f, 0.5f)
                    .clientTrackingRange(64)
                    .updateInterval(10)
                    .build("nuke_projectile"));
}
```

### 3.5 `entity/NukeProjectile.java`

```java
package com.nukegun.entity;

import net.minecraft.core.particles.ParticleTypes;
import net.minecraft.server.level.ServerLevel;
import net.minecraft.sounds.SoundEvents;
import net.minecraft.sounds.SoundSource;
import net.minecraft.world.effect.MobEffectInstance;
import net.minecraft.world.effect.MobEffects;
import net.minecraft.world.entity.EntityType;
import net.minecraft.world.entity.LivingEntity;
import net.minecraft.world.entity.projectile.ThrowableItemProjectile;
import net.minecraft.world.item.Item;
import net.minecraft.world.item.Items;
import net.minecraft.world.level.Level;
import net.minecraft.world.phys.EntityHitResult;
import net.minecraft.world.phys.HitResult;

import java.util.List;

public class NukeProjectile extends ThrowableItemProjectile {

    private int fuseTicks = 0;
    private static final int MAX_FUSE = 20 * 2;

    public NukeProjectile(EntityType<? extends NukeProjectile> type, Level level) {
        super(type, level);
    }

    public NukeProjectile(EntityType<? extends NukeProjectile> type, Level level, LivingEntity owner) {
        super(type, owner, level);
    }

    @Override
    protected Item getDefaultItem() {
        return Items.FIRE_CHARGE;
    }

    @Override
    protected float getGravity() {
        return 0.05f;
    }

    @Override
    public void tick() {
        super.tick();
        fuseTicks++;

        if (this.level().isClientSide) {
            this.level().addParticle(ParticleTypes.SMOKE,
                    this.getX(), this.getY(), this.getZ(), 0, 0, 0);
            this.level().addParticle(ParticleTypes.FLAME,
                    this.getX(), this.getY(), this.getZ(),
                    (this.random.nextDouble() - 0.5) * 0.05, 0.02, (this.random.nextDouble() - 0.5) * 0.05);
            if (this.tickCount % 2 == 0) {
                this.level().addParticle(ParticleTypes.LARGE_SMOKE,
                        this.getX(), this.getY(), this.getZ(), 0, 0.01, 0);
            }
        }

        if (fuseTicks >= MAX_FUSE && !this.level().isClientSide) {
            explode();
        }
    }

    @Override
    protected void onHitEntity(EntityHitResult result) {
        super.onHitEntity(result);
        if (!this.level().isClientSide) {
            explode();
        }
    }

    @Override
    protected void onHit(HitResult result) {
        super.onHit(result);
        if (!this.level().isClientSide && result.getType() != HitResult.Type.MISS) {
            explode();
        }
    }

    private void explode() {
        if (this.isRemoved()) return;

        Level level = this.level();
        if (level instanceof ServerLevel serverLevel) {

            double x = this.getX();
            double y = this.getY();
            double z = this.getZ();

            level.explode(this, x, y, z, 9.0f, true, Level.ExplosionInteraction.TNT);

            spawnMushroomCloud(serverLevel, x, y, z);

            level.playSound(null, x, y, z, SoundEvents.GENERIC_EXPLODE.get(), SoundSource.HOSTILE, 4.0f, 0.6f);
            level.playSound(null, x, y, z, SoundEvents.LIGHTNING_BOLT_THUNDER.get(), SoundSource.HOSTILE, 3.0f, 0.5f);

            double radius = 20.0;
            List<LivingEntity> victims = level.getEntitiesOfClass(LivingEntity.class,
                    this.getBoundingBox().inflate(radius));
            for (LivingEntity victim : victims) {
                double dist = victim.position().distanceTo(this.position());
                if (dist <= radius) {
                    float damage = (float) (60.0 * (1.0 - (dist / radius)));
                    victim.hurt(this.damageSources().explosion(this, this.getOwner()), damage);
                    victim.addEffect(new MobEffectInstance(MobEffects.WITHER, 20 * 8, 1));
                    victim.addEffect(new MobEffectInstance(MobEffects.CONFUSION, 20 * 5, 0));
                }
            }

            serverLevel.getEntitiesOfClass(LivingEntity.class, this.getBoundingBox().inflate(12))
                    .forEach(e -> e.setSecondsOnFire(6));
        }

        this.discard();
    }

    private void spawnMushroomCloud(ServerLevel level, double x, double y, double z) {
        level.sendParticles(ParticleTypes.EXPLOSION_EMITTER, x, y, z, 1, 0, 0, 0, 0);
        level.sendParticles(ParticleTypes.FLASH, x, y, z, 3, 1, 1, 1, 0);

        for (int i = 0; i < 40; i++) {
            double height = i * 0.5;
            double spread = 0.3 + (height * 0.02);
            level.sendParticles(ParticleTypes.LARGE_SMOKE,
                    x, y + height, z, 6, spread, 0.2, spread, 0.05);
            level.sendParticles(ParticleTypes.CAMPFIRE_SIGNAL_SMOKE,
                    x, y + height, z, 2, spread * 0.5, 0.1, spread * 0.5, 0.02);
        }

        double capHeight = 20;
        for (int i = 0; i < 80; i++) {
            double angle = (i / 80.0) * Math.PI * 2;
            double capRadius = 4 + level.random.nextDouble() * 3;
            double px = x + Math.cos(angle) * capRadius;
            double pz = z + Math.sin(angle) * capRadius;
            level.sendParticles(ParticleTypes.EXPLOSION,
                    px, y + capHeight, pz, 1, 0.3, 0.3, 0.3, 0.01);
            level.sendParticles(ParticleTypes.LARGE_SMOKE,
                    px, y + capHeight, pz, 1, 0.5, 0.3, 0.5, 0.02);
        }

        for (int i = 0; i < 60; i++) {
            double angle = (i / 60.0) * Math.PI * 2;
            double ringRadius = 10;
            double px = x + Math.cos(angle) * ringRadius;
            double pz = z + Math.sin(angle) * ringRadius;
            level.sendParticles(ParticleTypes.CLOUD, px, y + 0.5, pz, 1, 0.1, 0.1, 0.1, 0.1);
            level.sendParticles(ParticleTypes.ASH, px, y + 0.2, pz, 2, 0.2, 0.1, 0.2, 0.02);
        }
    }
}
```

> Đây là phần **rủi ro nhất** trong toàn bộ mod, vì đụng trực tiếp vào nội bộ Minecraft (particle, entity nổ). Nếu lỗi ở file này, khả năng cao chỉ là đổi tên hàm/enum (ví dụ `Level.ExplosionInteraction.TNT` có thể đổi tên, hoặc `level.explode(...)` đổi số lượng tham số) — gửi lỗi, mình sửa nhanh.

### 3.6 `client/NukeProjectileRenderer.java`

```java
package com.nukegun.client;

import com.nukegun.NukeGunMod;
import com.nukegun.entity.NukeProjectile;
import net.minecraft.client.renderer.entity.EntityRendererProvider;
import net.minecraft.client.renderer.entity.ThrownItemRenderer;
import net.minecraft.resources.ResourceLocation;

public class NukeProjectileRenderer extends ThrownItemRenderer<NukeProjectile> {

    public NukeProjectileRenderer(EntityRendererProvider.Context context) {
        super(context, 1.0f, true);
    }

    @Override
    public ResourceLocation getTextureLocation(NukeProjectile entity) {
        return new ResourceLocation(NukeGunMod.MOD_ID, "textures/entity/nuke_bomb.png");
    }
}
```

> Nếu lỗi ở `new ResourceLocation(...)`, các bản Minecraft gần đây (từ khoảng cuối 1.20.x) đôi khi đổi cách tạo `ResourceLocation` sang dạng `ResourceLocation.fromNamespaceAndPath(...)`. Thử cách này nếu constructor cũ báo lỗi "deprecated" hoặc không tồn tại:
> ```java
> return ResourceLocation.fromNamespaceAndPath(NukeGunMod.MOD_ID, "textures/entity/nuke_bomb.png");
> ```

### 3.7 `ClientSetup.java`

```java
package com.nukegun;

import com.nukegun.client.NukeProjectileRenderer;
import com.nukegun.entity.ModEntities;
import net.minecraftforge.api.distmarker.Dist;
import net.minecraftforge.client.event.EntityRenderersEvent;
import net.minecraftforge.eventbus.api.SubscribeEvent;
import net.minecraftforge.fml.common.Mod;

@Mod.EventBusSubscriber(modid = NukeGunMod.MOD_ID, bus = Mod.EventBusSubscriber.Bus.MOD, value = Dist.CLIENT)
public class ClientSetup {

    @SubscribeEvent
    public static void registerRenderers(EntityRenderersEvent.RegisterRenderers event) {
        event.registerEntityRenderer(ModEntities.NUKE_PROJECTILE.get(), NukeProjectileRenderer::new);
    }
}
```

---

## PHẦN 4: Resource files (giữ nguyên)

### `models/item/nuke_gun.json`
```json
{
  "parent": "item/generated",
  "textures": {
    "layer0": "nukegun:item/nuke_gun"
  }
}
```

### `lang/en_us.json`
```json
{
  "item.nukegun.nuke_gun": "Nuke Launcher"
}
```

### Texture
Dùng lại đúng 2 file ảnh mình đã gửi bạn ở tin nhắn trước (`nuke_gun.png`, `nuke_bomb.png`) — texture PNG không liên quan gì đến version code nên vẫn dùng được nguyên.

### `mods.toml` / `gradle.properties`
Kiểm tra kỹ theo lưu ý ở Phần 2 — bản 26.2 nhiều khả năng đã chuyển sang cơ chế template `${mod_id}`, nên chỗ cần sửa có thể là `gradle.properties` thay vì sửa tay trong `mods.toml`.

---

## PHẦN 5: Build

Giống hệt quy trình cũ:
```bash
./gradlew build
```
File `.jar` ra ở `build/libs/`.

---

## Tóm tắt việc cần làm khi build lỗi

1. Copy nguyên văn dòng lỗi đỏ (thường có dạng `cannot find symbol` hoặc `method does not exist`)
2. Gửi cho mình kèm tên file đang lỗi
3. Mình sẽ chỉ sửa đúng phần đó dựa trên chữ ký thật mà compiler báo — không cần đoán lại từ đầu
