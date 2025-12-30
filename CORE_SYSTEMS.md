# Core Systems Implementation Guide

## 1. PLAYER CONTROLLER SYSTEM

### PlayerController.cs - Core Hero Control Logic

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerController : MonoBehaviour
{
    [Header("Movement")]
    [SerializeField] private float moveSpeed = 10f;
    [SerializeField] private float laneWidth = 2f;
    [SerializeField] private float laneSwitchDuration = 0.15f;

    [Header("Combat")]
    [SerializeField] private Hero currentHero;
    [SerializeField] private Transform firePoint;
    [SerializeField] private GameObject bulletPrefab;
    [SerializeField] private float fireRate = 2f;

    [Header("FX")]
    [SerializeField] private ParticleSystem hitParticles;
    [SerializeField] private float screenShakeMagnitude = 5f;
    [SerializeField] private float screenShakeDuration = 0.1f;

    private int currentLane = 1; // 0, 1, 2 (left, center, right)
    private float fireTimer = 0f;
    private Vector3 targetPosition;
    private CanvasGroup healthBar;
    private int comboCounter = 0;
    private float comboResetTimer = 0f;
    private const float COMBO_RESET_TIME = 5f;

    private void Start()
    {
        targetPosition = transform.position;
        healthBar = GetComponentInChildren<CanvasGroup>();
    }

    private void Update()
    {
        // Auto-run forward (speed increases over time)
        float speedModifier = 1f + (Time.timeSinceLevelLoad * 0.05f);
        transform.position += Vector3.forward * moveSpeed * speedModifier * Time.deltaTime;

        // Move to target lane
        MoveTowardLane();

        // Auto-fire
        AutoFire();

        // Update combo
        UpdateCombo();

        // Handle input
        HandleInput();
    }

    private void HandleInput()
    {
        if (Input.GetKeyDown(KeyCode.A) || Input.GetKeyDown(KeyCode.LeftArrow))
            SwitchLane(-1);

        if (Input.GetKeyDown(KeyCode.D) || Input.GetKeyDown(KeyCode.RightArrow))
            SwitchLane(1);

        if (Input.GetKeyDown(KeyCode.Space))
            UseSpecialAbility();

        if (Input.GetKeyDown(KeyCode.E))
            UseGrenade();
    }

    private void SwitchLane(int direction)
    {
        int newLane = Mathf.Clamp(currentLane + direction, 0, 2);
        if (newLane != currentLane)
        {
            currentLane = newLane;
            targetPosition = new Vector3(
                (currentLane - 1) * laneWidth,
                0,
                transform.position.z
            );
        }
    }

    private void MoveTowardLane()
    {
        Vector3 currentPos = transform.position;
        Vector3 desiredPos = new Vector3(
            targetPosition.x,
            currentPos.y,
            currentPos.z
        );

        transform.position = Vector3.Lerp(
            currentPos,
            desiredPos,
            Time.deltaTime / laneSwitchDuration
        );
    }

    private void AutoFire()
    {
        fireTimer -= Time.deltaTime;

        if (fireTimer <= 0 && HasEnemiesInLane())
        {
            fireTimer = 1f / fireRate;
            Fire();
        }
    }

    private void Fire()
    {
        GameObject bulletGO = ObjectPool.Instance.GetBullet();
        Bullet bullet = bulletGO.GetComponent<Bullet>();

        if (bullet != null)
        {
            bullet.Initialize(
                firePoint.position,
                Vector3.forward,
                currentHero.GetDamage(),
                currentHero.GetCritChance()
            );
        }

        // Audio
        AudioManager.Instance.PlaySFX("gunfire");
    }

    private void UseSpecialAbility()
    {
        if (currentHero != null && currentHero.CanUseAbility())
        {
            currentHero.UseAbility(this);
            AudioManager.Instance.PlaySFX("ability_cast");
        }
    }

    private void UseGrenade()
    {
        if (currentHero.GetGrenadeCount() > 0)
        {
            GameObject grenadeGO = Instantiate(
                Resources.Load<GameObject>("Prefabs/Grenade"),
                transform.position,
                Quaternion.identity
            );

            Grenade grenade = grenadeGO.GetComponent<Grenade>();
            grenade.Explode(transform.position + Vector3.forward * 5f, 100f);

            currentHero.ConsumeGrenade();
            AudioManager.Instance.PlaySFX("grenade_throw");
        }
    }

    private void UpdateCombo()
    {
        comboResetTimer -= Time.deltaTime;
        if (comboResetTimer <= 0)
        {
            comboCounter = 0;
        }
    }

    private bool HasEnemiesInLane()
    {
        // Raycast check for enemies in current lane
        RaycastHit[] hits = Physics.RaycastAll(
            transform.position,
            Vector3.forward,
            50f
        );

        foreach (RaycastHit hit in hits)
        {
            Enemy enemy = hit.transform.GetComponent<Enemy>();
            if (enemy != null &&
                Mathf.Abs(hit.transform.position.x - transform.position.x) < laneWidth * 0.5f)
            {
                return true;
            }
        }

        return false;
    }

    public void TakeDamage(int damage)
    {
        currentHero.TakeDamage(damage);

        // Screen shake effect
        CameraShake();

        if (currentHero.GetHealth() <= 0)
        {
            Die();
        }
    }

    public void OnEnemyKilled(Enemy enemy)
    {
        comboCounter++;
        comboResetTimer = COMBO_RESET_TIME;

        float scoreMultiplier = 1f;
        if (comboCounter >= 5) scoreMultiplier = 2f;
        if (comboCounter >= 10) scoreMultiplier = 3f;

        GameManager.Instance.AddScore((int)(enemy.GetKillReward() * scoreMultiplier));
        GameManager.Instance.AddCoins((int)(enemy.GetCoinReward() * scoreMultiplier));
    }

    private void CameraShake()
    {
        StartCoroutine(ShakeCoroutine(screenShakeMagnitude, screenShakeDuration));
    }

    private System.Collections.IEnumerator ShakeCoroutine(float magnitude, float duration)
    {
        Vector3 originalPos = Camera.main.transform.localPosition;
        float elapsed = 0f;

        while (elapsed < duration)
        {
            float x = Random.Range(-magnitude, magnitude);
            float y = Random.Range(-magnitude, magnitude);

            Camera.main.transform.localPosition = originalPos + new Vector3(x, y, 0);
            elapsed += Time.deltaTime;
            yield return null;
        }

        Camera.main.transform.localPosition = originalPos;
    }

    private void Die()
    {
        GameManager.Instance.EndRun(comboCounter, GameManager.Instance.GetDistance());
        gameObject.SetActive(false);
    }
}
```

---

## 2. ENEMY MANAGER SYSTEM

### Enemy.cs - Individual Enemy Logic

```csharp
public class Enemy : MonoBehaviour
{
    [SerializeField] private EnemyType type;
    [SerializeField] private int health;
    [SerializeField] private int damage;
    [SerializeField] private float speed;
    [SerializeField] private int killReward = 10;
    [SerializeField] private int coinReward = 5;

    private int currentHealth;
    private SpriteRenderer spriteRenderer;
    private Color originalColor;

    private void Start()
    {
        currentHealth = health;
        spriteRenderer = GetComponent<SpriteRenderer>();
        originalColor = spriteRenderer.color;
    }

    private void Update()
    {
        // Move forward
        transform.position += Vector3.back * speed * Time.deltaTime;

        // Check if off-screen
        if (transform.position.z < -50f)
        {
            ReturnToPool();
        }
    }

    public void TakeDamage(int damageAmount, bool isCritical = false)
    {
        currentHealth -= damageAmount;

        // Hit feedback
        StartCoroutine(FlashDamage());

        if (isCritical)
        {
            // Critical hit effect
            AudioManager.Instance.PlaySFX("crit_hit");
            VFXManager.Instance.SpawnCriticalEffect(transform.position);
        }
        else
        {
            AudioManager.Instance.PlaySFX("enemy_hit");
        }

        if (currentHealth <= 0)
        {
            Die();
        }
    }

    private System.Collections.IEnumerator FlashDamage()
    {
        spriteRenderer.color = Color.white;
        yield return new WaitForSeconds(0.05f);
        spriteRenderer.color = originalColor;
    }

    private void Die()
    {
        // Spawn loot
        SpawnLoot();

        // Notify player
        PlayerController player = FindObjectOfType<PlayerController>();
        player.OnEnemyKilled(this);

        // VFX
        VFXManager.Instance.SpawnDeathEffect(transform.position);
        AudioManager.Instance.PlaySFX("enemy_death");

        ReturnToPool();
    }

    private void SpawnLoot()
    {
        // Small chance for power-up
        if (Random.value < 0.05f)
        {
            PowerUpType[] powerUps = {
                PowerUpType.Shield,
                PowerUpType.RapidFire,
                PowerUpType.DamageBoost
            };

            PowerUpType randomPowerUp = powerUps[Random.Range(0, powerUps.Length)];
            SpawnPowerUp(randomPowerUp);
        }

        // Coin spawn
        for (int i = 0; i < coinReward; i++)
        {
            Vector3 randomOffset = Random.insideUnitSphere * 2f;
            randomOffset.z = 0;

            GameObject coinGO = ObjectPool.Instance.GetCoin();
            coinGO.transform.position = transform.position + randomOffset;
        }
    }

    private void ReturnToPool()
    {
        currentHealth = health;
        spriteRenderer.color = originalColor;
        gameObject.SetActive(false);
    }

    public int GetKillReward() => killReward;
    public int GetCoinReward() => coinReward;
    public int GetHealth() => currentHealth;
}
```

### EnemyManager.cs - Spawning & Wave Control

```csharp
public class EnemyManager : MonoBehaviour
{
    [SerializeField] private GameObject[] enemyPrefabs;
    [SerializeField] private float spawnInterval = 1f;
    [SerializeField] private int enemiesPerWave = 5;

    private float spawnTimer = 0f;
    private int currentWave = 0;
    private int enemiesSpawned = 0;
    private float difficultyMultiplier = 1f;

    private void Update()
    {
        spawnTimer -= Time.deltaTime;

        if (spawnTimer <= 0 && enemiesSpawned < enemiesPerWave)
        {
            SpawnEnemy();
            spawnTimer = spawnInterval;
        }

        // Increase difficulty every 30 seconds
        difficultyMultiplier = 1f + (Time.timeSinceLevelLoad / 30f) * 0.05f;
    }

    private void SpawnEnemy()
    {
        GameObject enemyPrefab = enemyPrefabs[Random.Range(0, enemyPrefabs.Length)];
        GameObject enemyGO = Instantiate(
            enemyPrefab,
            GetRandomSpawnPosition(),
            Quaternion.identity
        );

        Enemy enemy = enemyGO.GetComponent<Enemy>();
        enemy.ApplyDifficultyScaling(difficultyMultiplier);

        enemiesSpawned++;
    }

    private Vector3 GetRandomSpawnPosition()
    {
        int lane = Random.Range(0, 3);
        float laneX = (lane - 1) * 2f;

        return new Vector3(laneX, 0, 50f);
    }

    public void NextWave()
    {
        currentWave++;
        enemiesSpawned = 0;
        enemiesPerWave += 2;
    }
}
```

---

## 3. PROGRESSION SYSTEM

### HeroSystem.cs - Hero Management

```csharp
public class HeroSystem : MonoBehaviour
{
    [System.Serializable]
    public class HeroData
    {
        public string name;
        public int level;
        public int stars;
        public int experience;
        public int health;
        public int damage;
        public float fireRate;
        public float critChance;
        public int[] equippedGear = new int[4]; // weapon, armor, accessory, special
    }

    private Dictionary<int, HeroData> ownedHeroes = new Dictionary<int, HeroData>();
    private HeroData currentHero;

    public void UnlockHero(int heroId, HeroData data)
    {
        if (!ownedHeroes.ContainsKey(heroId))
        {
            ownedHeroes[heroId] = data;
            DataManager.Instance.SaveHeroes();
        }
    }

    public void SelectHero(int heroId)
    {
        if (ownedHeroes.ContainsKey(heroId))
        {
            currentHero = ownedHeroes[heroId];
        }
    }

    public void UpgradeHero(int heroId, UpgradeType type)
    {
        if (ownedHeroes.ContainsKey(heroId))
        {
            HeroData hero = ownedHeroes[heroId];

            switch (type)
            {
                case UpgradeType.Level:
                    if (hero.level < 100)
                    {
                        int costPerLevel = 5000 * (hero.level + 1);
                        if (GameManager.Instance.SpendCoins(costPerLevel))
                        {
                            hero.level++;
                            hero.health += 50;
                            hero.damage += 10;
                        }
                    }
                    break;

                case UpgradeType.Star:
                    // Requires duplicate shards
                    if (hero.stars < 6)
                    {
                        int shardsCost = 120 + (20 * hero.stars);
                        if (DataManager.Instance.SpendHeroShards(heroId, shardsCost))
                        {
                            hero.stars++;
                            UnlockPassiveAbility(heroId, hero.stars);
                        }
                    }
                    break;
            }

            DataManager.Instance.SaveHeroes();
        }
    }

    public void GainExperience(int amount)
    {
        currentHero.experience += amount;

        int levelUpThreshold = 1000 * currentHero.level;
        if (currentHero.experience >= levelUpThreshold)
        {
            LevelUpHero();
        }
    }

    private void LevelUpHero()
    {
        currentHero.level++;
        currentHero.experience = 0;
        currentHero.health += 50;
        currentHero.damage += 10;
    }

    public HeroData GetCurrentHero() => currentHero;
}
```

---

## 4. MONETIZATION MANAGER

### MonetizationManager.cs - IAP & Shop Integration

```csharp
public class MonetizationManager : MonoBehaviour
{
    public enum Currency { Coins, Gems }

    [System.Serializable]
    public class ShopItem
    {
        public string itemId;
        public string displayName;
        public int price;
        public Currency priceType;
        public ItemType itemType;
        public int quantity;
    }

    private Dictionary<string, ShopItem> shopItems = new Dictionary<string, ShopItem>();
    private int playerGems = 0;
    private int playerCoins = 0;

    private void Start()
    {
        LoadShop();
        LoadCurrencies();
    }

    private void LoadShop()
    {
        // Define IAP packages
        shopItems["starter_pack"] = new ShopItem
        {
            itemId = "starter_pack",
            displayName = "Starter Pack",
            price = 99, // $0.99
            priceType = Currency.Gems,
            quantity = 200
        };

        shopItems["legendary_hero"] = new ShopItem
        {
            itemId = "legendary_hero_phoenix",
            displayName = "Phoenix Hero",
            price = 1499, // $14.99
            priceType = Currency.Gems,
            quantity = 1
        };
    }

    public void PurchaseItem(string itemId)
    {
        if (shopItems.ContainsKey(itemId))
        {
            ShopItem item = shopItems[itemId];

            // For demo, simulate purchase
            if (item.priceType == Currency.Gems && playerGems >= item.price)
            {
                playerGems -= item.price;
                GrantReward(item);
            }
            else if (item.priceType == Currency.Coins && playerCoins >= item.price)
            {
                playerCoins -= item.price;
                GrantReward(item);
            }
        }
    }

    private void GrantReward(ShopItem item)
    {
        switch (item.itemType)
        {
            case ItemType.Gems:
                playerGems += item.quantity;
                break;
            case ItemType.Coins:
                playerCoins += item.quantity;
                break;
            case ItemType.HeroShard:
                DataManager.Instance.AddHeroShard(item.itemId, item.quantity);
                break;
        }

        DataManager.Instance.SaveCurrencies(playerGems, playerCoins);
    }

    public bool SpendGems(int amount)
    {
        if (playerGems >= amount)
        {
            playerGems -= amount;
            DataManager.Instance.SaveCurrencies(playerGems, playerCoins);
            return true;
        }
        return false;
    }

    public bool SpendCoins(int amount)
    {
        if (playerCoins >= amount)
        {
            playerCoins -= amount;
            DataManager.Instance.SaveCurrencies(playerGems, playerCoins);
            return true;
        }
        return false;
    }

    public void AddGems(int amount) => playerGems += amount;
    public void AddCoins(int amount) => playerCoins += amount;

    public int GetGems() => playerGems;
    public int GetCoins() => playerCoins;
}
```

---

## 5. DATA PERSISTENCE

### DataManager.cs - Cloud Save Integration

```csharp
public class DataManager : MonoBehaviour
{
    [System.Serializable]
    public class GameSaveData
    {
        public int playerLevel;
        public int totalCoins;
        public int totalGems;
        public int highScore;
        public Dictionary<int, HeroData> heroes;
        public Dictionary<int, WeaponData> weapons;
        public int[] levelStars = new int[100]; // 3-star rating per level
    }

    private GameSaveData currentSaveData;
    private string cloudSaveKey = "DEADZONE_RUNNER_SAVE";

    public void SaveGameData()
    {
        string json = JsonUtility.ToJson(currentSaveData);
        PlayerPrefs.SetString(cloudSaveKey, json);
        PlayerPrefs.Save();

        // TODO: Upload to PlayFab for cloud sync
    }

    public void LoadGameData()
    {
        string json = PlayerPrefs.GetString(cloudSaveKey, "");

        if (!string.IsNullOrEmpty(json))
        {
            currentSaveData = JsonUtility.FromJson<GameSaveData>(json);
        }
        else
        {
            CreateNewSave();
        }
    }

    private void CreateNewSave()
    {
        currentSaveData = new GameSaveData
        {
            playerLevel = 1,
            totalCoins = 0,
            totalGems = 0,
            highScore = 0,
            heroes = new Dictionary<int, HeroData>(),
            weapons = new Dictionary<int, WeaponData>()
        };

        SaveGameData();
    }

    public GameSaveData GetSaveData() => currentSaveData;
}
```

---

## 6. PERFORMANCE OPTIMIZATION

### ObjectPool.cs - Memory Management

```csharp
public class ObjectPool : MonoBehaviour
{
    public static ObjectPool Instance { get; private set; }

    [SerializeField] private GameObject bulletPrefab;
    [SerializeField] private GameObject enemyPrefab;
    [SerializeField] private int poolSize = 100;

    private Queue<GameObject> bulletPool = new Queue<GameObject>();
    private Queue<GameObject> enemyPool = new Queue<GameObject>();

    private void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
        }
        else
        {
            Destroy(gameObject);
        }
    }

    private void Start()
    {
        // Pre-allocate bullets
        for (int i = 0; i < poolSize; i++)
        {
            GameObject bullet = Instantiate(bulletPrefab);
            bullet.SetActive(false);
            bulletPool.Enqueue(bullet);
        }
    }

    public GameObject GetBullet()
    {
        if (bulletPool.Count > 0)
        {
            GameObject bullet = bulletPool.Dequeue();
            bullet.SetActive(true);
            return bullet;
        }
        else
        {
            // Instantiate new if pool is empty
            return Instantiate(bulletPrefab);
        }
    }

    public void ReturnBullet(GameObject bullet)
    {
        bullet.SetActive(false);
        bulletPool.Enqueue(bullet);
    }
}
```

---

## Integration Checklist

- [ ] PlayerController handles lane switching (150ms smooth)
- [ ] - [ ] Auto-fire works in current lane only
- [ ] - [ ] Screen shake & particle effects on hit
- [ ] - [ ] Combo system tracks kill streaks
- [ ] - [ ] Critical hits show special effects
- [ ] - [ ] Enemy spawning with difficulty scaling
- [ ] - [ ] Hero progression system saving
- [ ] - [ ] Monetization shop functional
- [ ] - [ ] Data persistence to cloud
- [ ] - [ ] Object pooling reduces memory

- [ ] ---

- [ ] ## Next Steps

- [ ] 1. Implement UI layer (MainMenuUI, GameplayHUD, ShopUI)
- [ ] 2. Add audio system integration (Wwise/FMOD)
- [ ] 3. Implement analytics (Firebase/PlayFab)
- [ ] 4. Create visual effects package (VFXManager)
- [ ] 5. Set up Android/iOS build configuration
