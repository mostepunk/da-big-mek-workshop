# Da Big Mek Workshop 🟢⚙️💥

> **WAAAGH!** Образовательный проект для изучения Domain-Driven Design на примере орочьей мастерской из вселенной Warhammer 40,000

## 🎯 Что делает этот проект

**Da Big Mek Workshop** — это система управления орочьей мастерской, где Большой Мек создаёт оружие и боевые машины из награбленного хлама.

Орки собирают стрелялы (shootas), тяжёлое вооружение, грузовики и боевые платформы из чего попало: имперской техники, эльдарских артефактов, ржавого железа. При этом действуют абсурдные, но строгие законы орочьей физики:
- **Красное быстрее** → красные вещи получают бонус к скорости
- **Больше дакки = лучше** → чем больше огневой мощи, тем лучше
- **Если орки верят, что работает → работает** → WAAAGH энергия делает невозможное возможным
- **Некачественные детали взрываются** → использование хлама повышает шанс катастрофического отказа

## 🔧 Какие задачи решает

### Образовательные задачи

1. **Показать эволюцию архитектуры** от простого CRUD к Domain-Driven Design
2. **Продемонстрировать недостатки** анемичной модели и "толстых" сервисов
3. **Обучить практическому применению** DDD паттернов и концепций
4. **Объяснить когда и почему** нужен переход на DDD

### Технические задачи

Система управляет:
- **Инвентарём мастерской** — награбленные детали разного качества и происхождения
- **Создание оружия** — сборка с проверкой совместимости деталей и навыков мека
- **Тестирование изделий** — стрельба с учётом качества, WAAAGH энергии и случайных отказов
- **Постройкой техники** — многоэтапные процессы сборки сложных машин
- **Репутацией мека** — влияние на эффективность создаваемого оружия
- **Статистикой** — успешность создания, взрывы, рейтинги оружия

## 📚 Чему научит проект

### Концепции DDD

- **Ubiquitous Language** — единый язык бизнеса и разработки (dakka, WAAAGH, shoota, mek)
- **Entities vs Value Objects** — когда идентичность важна, а когда нет
- **Aggregates** — как защитить бизнес-инварианты и управлять консистентностью
- **Domain Events** — как моделировать изменения состояния системы
- **Specifications** — как выразить сложные бизнес-правила в коде
- **Domain Services** — когда логика не принадлежит одному aggregate
- **Bounded Contexts** — как разделить большую систему на независимые части

### Архитектурные паттерны

- **Rich Domain Model** vs Anemic Domain Model
- **CQRS** (Command Query Responsibility Segregation)
- **Event Sourcing** — восстановление состояния из событий
- **Saga Pattern** — управление распределёнными транзакциями
- **Repository Pattern** — абстракция доступа к данным
- **Anti-Corruption Layer** — интеграция с внешними системами

### Практические навыки

- Рефакторинг от процедурного к объектно-ориентированному коду
- Выделение бизнес-логики из инфраструктурных слоёв
- Написание выразительных тестов на domain логику
- Работа с асинхронными процессами и eventual consistency
- Проектирование API между bounded contexts

---

## 🚀 План эволюции проекта

### Stage 0: Anemic Model — "Почувствуй боль"

**Цель:** Показать проблемы традиционного подхода с сервисным слоем

#### Требования (простые):

1. **Создать мастерскую** с именем мека
2. **Добавить лут** в инвентарь (название, тип, количество)
3. **Собрать простое стреляло** из имеющихся деталей
4. **Выстрелить** из стреляла и узнать попал или нет

#### Архитектура:

```
views.py → services.py → models.py (Django ORM)
```

#### Реализация:

```python
# models.py - Anemic Domain Model
class Workshop(models.Model):
    mek_name = models.CharField(max_length=100)
    reputation = models.IntegerField(default=0)

class Scrap(models.Model):
    workshop = models.ForeignKey(Workshop)
    name = models.CharField(max_length=100)
    scrap_type = models.CharField(max_length=50)
    quantity = models.IntegerField()

class Weapon(models.Model):
    workshop = models.ForeignKey(Workshop)
    name = models.CharField(max_length=100)
    dakka = models.IntegerField()
    parts = models.JSONField()

# services.py - Fat Services
class WeaponService:
    def build_weapon(self, workshop_id, weapon_data):
        workshop = Workshop.objects.get(id=workshop_id)
        parts = Scrap.objects.filter(workshop=workshop)
        
        # Вся бизнес-логика здесь
        if parts.count() < 3:
            raise ValidationError("Not enough parts")
        
        dakka = 0
        for part in parts[:3]:
            dakka += 5
            part.quantity -= 1
            part.save()
        
        weapon = Weapon.objects.create(
            workshop=workshop,
            name=weapon_data['name'],
            dakka=dakka,
            parts=[p.id for p in parts[:3]]
        )
        
        workshop.reputation += 1
        workshop.save()
        
        return weapon
    
    def fire_weapon(self, weapon_id):
        weapon = Weapon.objects.get(id=weapon_id)
        
        # Логика стрельбы в сервисе
        hit_chance = weapon.dakka / 100
        if random.random() < hit_chance:
            return {"result": "HIT"}
        return {"result": "MISS"}
```

#### Проблемы которые появятся:

❌ **Логика размазана** — валидация в views, бизнес-логика в services, данные в models  
❌ **Дублирование** — проверка "достаточно ли деталей" будет в нескольких местах  
❌ **Невозможно тестировать** бизнес-логику без БД  
❌ **Нарушение инвариантов** — можно создать weapon с отрицательным dakka  
❌ **Неясная семантика** — что значит `parts = models.JSONField()`?  
❌ **Сложно расширять** — добавление "красной краски" потребует изменений в 5 местах

---

### Stage 1: Tactical DDD — "Rich Domain Model"

**Цель:** Вынести бизнес-логику в domain слой, ввести Value Objects и Entities

#### Новые требования:

5. **Качество деталей** влияет на шанс взрыва (SHINY, NORMAL, RUSTY, EXPLODY)
6. **Происхождение лута** влияет на характеристики (IMPERIAL, ELDAR, ORK, JUNK)
7. **Покрасить оружие в красный цвет** для увеличения дакки (+5 bonus)
8. **Учитывать состояние оружия** (NEW, USED, DAMAGED, EXPLODED)
9. **Проверять совместимость деталей** (имперское + эльдарское = взрыв)

#### Архитектура:

```
┌─────────────────────────────────────────────┐
│           Presentation Layer                │
│         (views.py, serializers.py)          │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│          Application Layer                  │
│    (command handlers, query handlers)       │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│            Domain Layer                     │
│  ┌──────────────────────────────────────┐  │
│  │ Entities: Weapon, Mek                │  │
│  │ Value Objects: Scrap, DakkaRating    │  │
│  │ Aggregates: MekWorkshop              │  │
│  │ Domain Events: WeaponBuiltEvent      │  │
│  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│        Infrastructure Layer                 │
│    (ORM models, repositories, DB)           │
└─────────────────────────────────────────────┘
```

#### Ключевые изменения:

**Value Objects** (иммутабельные, без идентичности):

```python
@dataclass(frozen=True)
class Scrap:
    source: ScrapSource      # Enum: IMPERIAL, ELDAR, ORK, JUNK
    scrap_type: ScrapType    # Enum: ENGINE, ARMOR, SHOOTY_BITZ...
    quality: Quality         # Enum: SHINY, NORMAL, RUSTY, EXPLODY
    quantity: int
    
    def is_compatible_with(self, other: 'Scrap') -> bool:
        """Бизнес-правило: некоторые детали несовместимы"""
        if self.source == ScrapSource.IMPERIAL and other.source == ScrapSource.ELDAR:
            return False  # Имперская и эльдарская технология не совместимы
        return True
    
    def __post_init__(self):
        if self.quantity <= 0:
            raise ValueError("Quantity must be positive")

@dataclass(frozen=True)
class DakkaRating:
    base_dakka: int
    bonus_dakka: int
    
    @property
    def total(self) -> int:
        return self.base_dakka + self.bonus_dakka
    
    def with_bonus(self, bonus: int) -> 'DakkaRating':
        """Возвращает новый объект с дополнительным бонусом"""
        return DakkaRating(
            base_dakka=self.base_dakka,
            bonus_dakka=self.bonus_dakka + bonus
        )
```

**Entity** (с идентичностью):

```python
class Weapon(Entity):
    def __init__(
        self,
        weapon_id: WeaponId,
        name: str,
        parts: List[Scrap],
        dakka: DakkaRating,
        color: Color = Color.UNPAINTED
    ):
        self._id = weapon_id
        self._name = name
        self._parts = parts
        self._dakka = dakka
        self._color = color
        self._condition = Condition.NEW
        self._malfunctions = 0
        self._events: List[DomainEvent] = []
    
    def paint_red(self) -> None:
        """Покрасить в красный - увеличивает дакку"""
        if self._color == Color.RED:
            raise AlreadyRedError("ITZ ALREADY RED YA GIT!")
        
        self._color = Color.RED
        self._dakka = self._dakka.with_bonus(5)  # Иммутабельное изменение
        
        self._add_event(WeaponPaintedRedEvent(
            weapon_id=self._id,
            new_dakka=self._dakka.total
        ))
    
    def fire(self, ork_count: int) -> FiringResult:
        """Выстрел с учётом веры орков и состояния оружия"""
        self._ensure_can_fire()
        
        # WAAAGH энергия: чем больше орков верят, тем лучше работает
        belief_modifier = min(ork_count // 10, 10)
        
        # Проверка на взрыв
        if self._check_explosion():
            self._condition = Condition.EXPLODED
            self._add_event(WeaponExplodedEvent(self._id))
            return FiringResult.EXPLODED
        
        # Расчёт попадания
        effective_dakka = self._dakka.total + belief_modifier
        hit_chance = min(effective_dakka / 100, 0.95)
        
        if random.random() < hit_chance:
            self._malfunctions = 0
            self._add_event(WeaponFiredSuccessfullyEvent(self._id))
            return FiringResult.HIT
        else:
            self._malfunctions += 1
            self._condition = Condition.USED
            return FiringResult.MISS
    
    def _ensure_can_fire(self) -> None:
        """Инвариант: нельзя стрелять из взорванного оружия"""
        if self._condition == Condition.EXPLODED:
            raise CannotFireExplodedWeaponError()
    
    def _check_explosion(self) -> bool:
        """Шанс взрыва зависит от качества деталей"""
        base_chance = 0.05
        
        for part in self._parts:
            if part.quality == Quality.EXPLODY:
                base_chance += 0.2
            elif part.quality == Quality.RUSTY:
                base_chance += 0.1
        
        base_chance += self._malfunctions * 0.05
        
        return random.random() < base_chance
```

**Aggregate Root** (контролирует консистентность):

```python
class MekWorkshop(AggregateRoot):
    """Мастерская - корень агрегата"""
    
    def __init__(self, workshop_id: WorkshopId, mek_name: str):
        self._id = workshop_id
        self._mek_name = mek_name
        self._inventory: List[Scrap] = []
        self._weapons: Dict[WeaponId, Weapon] = {}
        self._reputation = 0
    
    def loot_scrap(self, scrap: Scrap) -> None:
        """Добавить лут - всегда валидная операция"""
        self._inventory.append(scrap)
        self._add_event(ScrapLootedEvent(self._id, scrap))
    
    def build_shoota(self, parts_needed: List[ScrapType]) -> Weapon:
        """Собрать стреляло - защита инвариантов"""
        
        # Инвариант 1: достаточно деталей
        available = self._find_parts(parts_needed)
        if len(available) < len(parts_needed):
            raise NotEnuffBitsError(
                f"Need {len(parts_needed)} parts, have {len(available)}"
            )
        
        # Инвариант 2: детали совместимы
        for i, part1 in enumerate(available):
            for part2 in available[i+1:]:
                if not part1.is_compatible_with(part2):
                    raise IncompatiblePartsError(
                        f"{part1.source} and {part2.source} don't work together!"
                    )
        
        # Создание оружия
        dakka = self._calculate_dakka(available)
        weapon = Weapon(
            weapon_id=WeaponId.generate(),
            name="Shoota",
            parts=available,
            dakka=dakka
        )
        
        # Удаление использованных деталей
        for part in available:
            self._inventory.remove(part)
        
        self._weapons[weapon.id] = weapon
        self._reputation += 1
        
        self._add_event(WeaponBuiltEvent(self._id, weapon.id))
        
        return weapon
    
    def _calculate_dakka(self, parts: List[Scrap]) -> DakkaRating:
        """Бизнес-логика расчёта огневой мощи"""
        base = 10
        bonus = 0
        
        for part in parts:
            if part.quality == Quality.SHINY:
                bonus += 2
            elif part.source == ScrapSource.IMPERIAL:
                bonus += 3  # Имперское надёжнее
        
        return DakkaRating(base_dakka=base, bonus_dakka=bonus)
```

#### Что мы получили:

✅ **Бизнес-логика в одном месте** — все правила в domain слое  
✅ **Защищённые инварианты** — невозможно создать невалидное состояние  
✅ **Тестируемость** — domain логика не зависит от БД  
✅ **Выразительность** — код читается как бизнес-требования  
✅ **Расширяемость** — новые правила добавляются в одном месте

---

### Stage 2: Specifications & Domain Services

**Цель:** Ввести сложные бизнес-правила и логику, затрагивающую несколько aggregates

#### Новые требования:

10. **Навыки мека** влияют на что он может строить (уровни: GROT, NORMAL, BIG_MEK)
11. **Сложные blueprints** для разных типов оружия (shoota, big shoota, mega blasta)
12. **Проверка "can build"** с учётом навыков, деталей и совместимости
13. **WAAAGH энергия** вычисляется от количества и ранга орков поблизости
14. **Репутация мека** влияет на качество создаваемого оружия

#### Новые паттерны:

**Specification Pattern:**

```python
class CanBuildWeaponSpec(Specification[MekWorkshop]):
    """Спецификация: можно ли построить оружие"""
    
    def __init__(self, blueprint: WeaponBlueprint, mek_skill: MekSkill):
        self._blueprint = blueprint
        self._mek_skill = mek_skill
    
    def is_satisfied_by(self, workshop: MekWorkshop) -> bool:
        # Проверка навыков
        if blueprint.complexity > self._mek_skill.level:
            return False
        
        # Проверка наличия деталей
        required = self._blueprint.required_parts
        if not workshop.has_parts(required):
            return False
        
        # Проверка совместимости
        parts = workshop.get_parts(required)
        compatibility_spec = PartsCompatibilitySpec(parts)
        if not compatibility_spec.is_satisfied_by(workshop):
            return False
        
        return True
    
    def why_not_satisfied(self, workshop: MekWorkshop) -> str:
        """Объяснение почему нельзя"""
        if self._blueprint.complexity > self._mek_skill.level:
            return f"Mek skill {self._mek_skill} too low for {self._blueprint.name}"
        
        if not workshop.has_parts(self._blueprint.required_parts):
            return "Not enough parts in inventory"
        
        return "Parts are incompatible"

class RedVehicleGoFasterSpec(Specification[Vehicle]):
    """Спецификация: красные машины едут быстрее"""
    
    def is_satisfied_by(self, vehicle: Vehicle) -> bool:
        return vehicle.color == Color.RED
    
    def apply_bonus(self, vehicle: Vehicle) -> int:
        """Вернуть бонус к скорости"""
        if self.is_satisfied_by(vehicle):
            return 20  # +20% скорости
        return 0
```

**Domain Service:**

```python
class WaaghEnergyService:
    """Domain Service: вычисление WAAAGH энергии
    
    Логика не принадлежит ни Workshop, ни Weapon, ни Ork
    → выносим в Domain Service
    """
    
    def calculate_belief_power(
        self,
        workshop: MekWorkshop,
        weapon: Weapon,
        nearby_orks: List[Ork]
    ) -> int:
        """Чем больше орков верят, тем лучше работает"""
        
        # Базовая вера от количества орков
        base = len(nearby_orks)
        
        # Бонус от репутации мека
        reputation_bonus = workshop.reputation // 10
        
        # Ранг орков влияет сильнее
        rank_bonus = sum(
            self._rank_multiplier(ork.rank) 
            for ork in nearby_orks
        )
        
        # Качество оружия тоже важно
        weapon_quality_bonus = self._weapon_quality_bonus(weapon)
        
        total = base + reputation_bonus + rank_bonus + weapon_quality_bonus
        
        return min(total, 100)  # Максимум 100
    
    def _rank_multiplier(self, rank: OrkRank) -> int:
        return {
            OrkRank.GROT: 0,      # Гроты не считаются
            OrkRank.BOY: 1,       # Обычные орки
            OrkRank.NOB: 3,       # Нобзы дают больше веры
            OrkRank.BOSS: 5,      # Боссы ещё больше
            OrkRank.WARBOSS: 10   # Варбосс = мега бонус
        }[rank]
    
    def _weapon_quality_bonus(self, weapon: Weapon) -> int:
        if weapon.condition == Condition.NEW:
            return 5
        elif weapon.condition == Condition.USED:
            return 2
        return 0

class MekSkillProgressionService:
    """Domain Service: прогресс навыков мека"""
    
    def check_level_up(
        self,
        workshop: MekWorkshop,
        weapons_built: int,
        explosions: int
    ) -> Optional[MekSkill]:
        """Проверить, повысился ли уровень мека"""
        
        success_rate = 1 - (explosions / max(weapons_built, 1))
        current_skill = workshop.mek_skill
        
        # Правила повышения
        if current_skill == MekSkill.GROT and weapons_built >= 10 and success_rate > 0.7:
            return MekSkill.NORMAL
        
        if current_skill == MekSkill.NORMAL and weapons_built >= 50 and success_rate > 0.8:
            return MekSkill.BIG_MEK
        
        return None
```

**Изменения в Aggregate:**

```python
class MekWorkshop(AggregateRoot):
    def __init__(self, workshop_id: WorkshopId, mek_name: str, mek_skill: MekSkill):
        # ... существующие поля
        self._mek_skill = mek_skill
    
    def build_weapon(
        self,
        blueprint: WeaponBlueprint,
        can_build_spec: CanBuildWeaponSpec  # Инъекция спецификации
    ) -> Weapon:
        """Построить оружие с проверкой спецификации"""
        
        # Проверка через спецификацию
        if not can_build_spec.is_satisfied_by(self):
            reason = can_build_spec.why_not_satisfied(self)
            raise CannotBuildWeaponError(reason)
        
        # ... остальная логика создания
```

#### Зачем это нужно:

✅ **Переиспользование бизнес-правил** — спецификации комбинируются через AND/OR  
✅ **Явная бизнес-логика** — `RedVehicleGoFasterSpec` вместо `if color == 'red'`  
✅ **Тестируемость сложных правил** — спецификации тестируются отдельно  
✅ **Domain Services** — для логики между несколькими aggregates

---

### Stage 3: CQRS — Command Query Responsibility Segregation

**Цель:** Разделить модель записи и модель чтения для производительности

#### Новые требования:

15. **Дашборд мастерской** — статистика: созданное оружие, взрывы, success rate
16. **Топ оружия по дакке** — рейтинг самых мощных стрелял
17. **Аналитика по источникам лута** — какой лут чаще используется
18. **История событий** — audit trail всех действий в мастерской
19. **Real-time обновления** — WebSocket для отображения текущего состояния

#### Проблема:

Текущая модель (PostgreSQL с нормализацией) плохо подходит для:
- Сложных аналитических запросов (JOIN по 5 таблицам)
- Real-time дашбордов (постоянные SELECT)
- Исторических данных (нужен audit trail)

#### Решение через CQRS:

```
┌─────────────────────────────────────────────────┐
│               WRITE MODEL (Commands)            │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │  MekWorkshop Aggregate (PostgreSQL)       │ │
│  │  - Нормализованная структура              │ │
│  │  - Защита инвариантов                     │ │
│  │  - Источник истины                        │ │
│  └───────────────────────────────────────────┘ │
│                      │                          │
│                      │ Domain Events            │
│                      ↓                          │
└─────────────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
┌─────────────┐ ┌──────────────┐ ┌─────────────┐
│ READ MODEL  │ │ READ MODEL   │ │ EVENT STORE │
│ (MongoDB)   │ │ (Redis Cache)│ │ (Append-only│
│             │ │              │ │  log)       │
│ - Stats     │ │ - Real-time  │ │             │
│ - Rankings  │ │ - Current    │ │ - Audit     │
│ - Analytics │ │   state      │ │ - Replay    │
└─────────────┘ └──────────────┘ └─────────────┘
```

#### Реализация:

**Commands (Write):**

```python
@dataclass
class BuildWeaponCommand:
    """Команда: построить оружие"""
    workshop_id: WorkshopId
    blueprint: WeaponBlueprint
    parts_types: List[ScrapType]

class BuildWeaponCommandHandler:
    def __init__(
        self,
        workshop_repo: WorkshopRepository,
        event_bus: EventBus
    ):
        self._repo = workshop_repo
        self._event_bus = event_bus
    
    def handle(self, command: BuildWeaponCommand) -> WeaponId:
        # 1. Загрузить aggregate
        workshop = self._repo.get(command.workshop_id)
        
        # 2. Выполнить domain логику
        weapon = workshop.build_weapon(
            command.blueprint,
            command.parts_types
        )
        
        # 3. Сохранить aggregate
        self._repo.save(workshop)
        
        # 4. Опубликовать события (асинхронно!)
        for event in workshop.domain_events:
            self._event_bus.publish(event)
        
        return weapon.id
```

**Queries (Read):**

```python
@dataclass
class GetWorkshopStatsQuery:
    """Запрос: статистика мастерской"""
    workshop_id: WorkshopId

class GetWorkshopStatsQueryHandler:
    def __init__(self, mongo_client: MongoClient):
        self._db = mongo_client['read_models']
    
    def handle(self, query: GetWorkshopStatsQuery) -> WorkshopStats:
        # Читаем из денормализованной MongoDB
        stats_doc = self._db.workshop_stats.find_one({
            'workshop_id': str(query.workshop_id)
        })
        
        return WorkshopStats(
            total_weapons=stats_doc['total_weapons'],
            total_explosions=stats_doc['total_explosions'],
            success_rate=stats_doc['success_rate'],
            top_weapons=stats_doc['top_weapons'],
            most_used_scrap=stats_doc['most_used_scrap']
        )

@dataclass
class GetDakkaLeaderboardQuery:
    """Запрос: топ оружия по дакке"""
    limit: int = 10

class GetDakkaLeaderboardQueryHandler:
    def __init__(self, mongo_client: MongoClient):
        self._db = mongo_client['read_models']
    
    def handle(self, query: GetDakkaLeaderboardQuery) -> List[WeaponRanking]:
        # Простой запрос из предвычисленной коллекции
        results = self._db.dakka_leaderboard.find().sort('dakka', -1).limit(query.limit)
        
        return [
            WeaponRanking(
                weapon_id=doc['weapon_id'],
                name=doc['name'],
                dakka=doc['dakka'],
                mek_name=doc['mek_name']
            )
            for doc in results
        ]
```

**Event Handlers (проекции в Read Model):**

```python
class WeaponBuiltEventHandler:
    """Обновить статистику при создании оружия"""
    
    def __init__(self, mongo_client: MongoClient):
        self._db = mongo_client['read_models']
    
    async def handle(self, event: WeaponBuiltEvent):
        # Обновить счётчик
        self._db.workshop_stats.update_one(
            {'workshop_id': str(event.workshop_id)},
            {
                '$inc': {'total_weapons': 1},
                '$set': {'last_updated': datetime.now()}
            },
            upsert=True
        )
        
        # Добавить в leaderboard
        weapon_details = await self._get_weapon_details(event.weapon_id)
        self._db.dakka_leaderboard.insert_one({
            'weapon_id': str(event.weapon_id),
            'name': weapon_details.name,
            'dakka': weapon_details.dakka,
            'mek_name': weapon_details.mek_name,
            'created_at': event.occurred_at
        })

class WeaponExplodedEventHandler:
    """Обновить статистику при взрыве"""
    
    async def handle(self, event: WeaponExplodedEvent):
        # Инкремент взрывов
        result = self._db.workshop_stats.find_one_and_update(
            {'workshop_id': str(event.workshop_id)},
            {'$inc': {'total_explosions': 1}},
            return_document=ReturnDocument.AFTER
        )
        
        # Пересчитать success rate
        success_rate = 1 - (result['total_explosions'] / result['total_weapons'])
        self._db.workshop_stats.update_one(
            {'workshop_id': str(event.workshop_id)},
            {'$set': {'success_rate': success_rate}}
        )
        
        # Удалить из leaderboard
        self._db.dakka_leaderboard.delete_one({
            'weapon_id': str(event.weapon_id)
        })
```

**Event Store (audit trail):**

```python
class EventStore:
    """Хранение всех событий для audit и replay"""
    
    def append(self, event: DomainEvent) -> None:
        self._db.events.insert_one({
            'event_id': str(uuid.uuid4()),
            'event_type': event.__class__.__name__,
            'aggregate_id': str(event.aggregate_id),
            'payload': event.to_dict(),
            'occurred_at': event.occurred_at,
            'version': event.version
        })
    
    def get_events(
        self,
        aggregate_id: WorkshopId,
        from_version: int = 0
    ) -> List[DomainEvent]:
        """Получить все события для aggregate"""
        docs = self._db.events.find({
            'aggregate_id': str(aggregate_id),
            'version': {'$gte': from_version}
        }).sort('version', 1)
        
        return [self._deserialize(doc) for doc in docs]
    
    def replay_all(self) -> None:
        """Пересоздать Read Models из событий"""
        # Полезно при добавлении новой проекции
        events = self._db.events.find().sort('occurred_at', 1)
        
        for event_doc in events:
            event = self._deserialize(event_doc)
            self._event_bus.publish(event)
```

#### Преимущества CQRS:

✅ **Производительность** — read model оптимизирован под запросы  
✅ **Масштабируемость** — read и write масштабируются независимо  
✅ **Гибкость** — можно добавить новую проекцию без изменения write model  
✅ **Audit trail** — полная история изменений через Event Store  
✅ **Eventual consistency** — read model обновляется асинхронно

---

### Stage 4: Saga Pattern — Сложные процессы

**Цель:** Управление распределёнными транзакциями при создании сложной техники

#### Новые требования:

20. **Построить Wartrukk** (грузовик) — многоэтапный процесс:
    - Собрать шасси из металлолома
    - Установить двигатель (лутнутый имперский или орочий)
    - Добавить броню
    - Покрасить в красный (обязательно!)
    - Тест-драйв (может взорваться)
    
21. **Построить Deff Dread** (боевой дроид) — ещё сложнее:
    - Найти подходящего орка-пилота (добровольца)
    - Построить каркас
    - Установить оружейные системы (2-4 орудия)
    - Подключить системы жизнеобеспечения
    - Тест боя (может убить пилота)
    
22. **Откат при неудаче** — вернуть уцелевшие детали в инвентарь

#### Проблема:

Эти процессы:
- Занимают несколько этапов
- Могут провалиться на любом этапе
- Требуют компенсирующих действий при откате
- Затрагивают несколько aggregates (Workshop, Vehicle, Pilot)

#### Решение через Saga:

```python
class BuildWartrukSaga:
    """Saga: построить грузовик
    
    Координирует многошаговый процесс с возможностью отката
    """
    
    def __init__(
        self,
        command_bus: CommandBus,
        event_bus: EventBus,
        saga_repo: SagaRepository
    ):
        self._commands = command_bus
        self._events = event_bus
        self._repo = saga_repo
    
    async def execute(self, cmd: BuildWartrukCommand) -> WartrukId:
        """Выполнить сагу"""
        
        saga_id = SagaId.generate()
        saga_state = WartrukBuildState()
        
        try:
            # Шаг 1: Собрать шасси
            chassis_result = await self._commands.send(
                BuildChassisCommand(
                    workshop_id=cmd.workshop_id,
                    required_parts=[ScrapType.METAL, ScrapType.WHEELS]
                )
            )
            saga_state.chassis_id = chassis_result.chassis_id
            saga_state.mark_step_complete('chassis')
            await self._repo.save(saga_id, saga_state)
            
            # Шаг 2: Установить двигатель
            engine_result = await self._commands.send(
                InstallEngineCommand(
                    chassis_id=saga_state.chassis_id,
                    engine_type=cmd.engine_type
                )
            )
            saga_state.engine_installed = True
            saga_state.mark_step_complete('engine')
            await self._repo.save(saga_id, saga_state)
            
            # Шаг 3: Добавить броню
            await self._commands.send(
                AddArmorCommand(
                    vehicle_id=saga_state.chassis_id,
                    armor_parts=cmd.armor_parts
                )
            )
            saga_state.mark_step_complete('armor')
            await self._repo.save(saga_id, saga_state)
            
            # Шаг 4: Покрасить в красный (обязательно!)
            await self._commands.send(
                PaintVehicleCommand(
                    vehicle_id=saga_state.chassis_id,
                    color=Color.RED
                )
            )
            saga_state.painted_red = True
            saga_state.mark_step_complete('paint')
            await self._repo.save(saga_id, saga_state)
            
            # Шаг 5: Тест-драйв
            test_result = await self._commands.send(
                TestDriveCommand(
                    vehicle_id=saga_state.chassis_id,
                    driver_id=cmd.driver_id
                )
            )
            
            if test_result.outcome == TestOutcome.EXPLODED:
                # Компенсация: грузовик взорвался
                await self._compensate_explosion(saga_state)
                raise VehicleExplodedDuringTestError()
            
            elif test_result.outcome == TestOutcome.CATASTROPHIC_FAILURE:
                # Полная компенсация: откатить всё
                await self._compensate_all(saga_state)
                raise BuildFailedError()
            
            # Успех!
            saga_state.mark_complete()
            await self._repo.save(saga_id, saga_state)
            
            await self._events.publish(
                WartrukBuiltEvent(
                    wartruk_id=saga_state.chassis_id,
                    workshop_id=cmd.workshop_id
                )
            )
            
            return saga_state.chassis_id
            
        except Exception as e:
            # Откат при любой ошибке
            await self._compensate_all(saga_state)
            saga_state.mark_failed(str(e))
            await self._repo.save(saga_id, saga_state)
            raise
    
    async def _compensate_explosion(self, state: WartrukBuildState):
        """Компенсация при взрыве: вернуть уцелевшие детали"""
        
        # Двигатель обычно выживает
        if state.engine_installed:
            await self._commands.send(
                RecoverEngineCommand(
                    vehicle_id=state.chassis_id,
                    condition=Condition.DAMAGED
                )
            )
        
        # Броня частично уцелела
        if state.steps_completed['armor']:
            await self._commands.send(
                RecoverArmorPartsCommand(
                    vehicle_id=state.chassis_id,
                    recovery_rate=0.5  # 50% деталей
                )
            )
    
    async def _compensate_all(self, state: WartrukBuildState):
        """Полный откат: вернуть все детали"""
        
        # Откат в обратном порядке
        if state.steps_completed['paint']:
            # Краску не вернёшь, но можно списать
            pass
        
        if state.steps_completed['armor']:
            await self._commands.send(
                RemoveArmorCommand(vehicle_id=state.chassis_id)
            )
        
        if state.steps_completed['engine']:
            await self._commands.send(
                RemoveEngineCommand(vehicle_id=state.chassis_id)
            )
        
        if state.steps_completed['chassis']:
            await self._commands.send(
                DisassembleChassisCommand(chassis_id=state.chassis_id)
            )

@dataclass
class WartrukBuildState:
    """Состояние саги"""
    chassis_id: Optional[VehicleId] = None
    engine_installed: bool = False
    painted_red: bool = False
    steps_completed: Dict[str, bool] = field(default_factory=lambda: {
        'chassis': False,
        'engine': False,
        'armor': False,
        'paint': False,
        'test': False
    })
    
    def mark_step_complete(self, step: str):
        self.steps_completed[step] = True
    
    def mark_complete(self):
        self.completed = True
    
    def mark_failed(self, reason: str):
        self.failed = True
        self.failure_reason = reason
```

**Saga для ещё более сложного процесса:**

```python
class BuildDeffDreadSaga:
    """Построить Deff Dread - самый сложный процесс"""
    
    async def execute(self, cmd: BuildDeffDreadCommand):
        saga_id = SagaId.generate()
        state = DeffDreadBuildState()
        
        try:
            # Шаг 1: Найти орка-добровольца (или не очень добровольца)
            pilot_result = await self._commands.send(
                FindPilotCommand(
                    min_rank=OrkRank.BOY,
                    willing_to_die=True  # Важное требование!
                )
            )
            
            if not pilot_result.pilot_found:
                raise NoPilotAvailableError("NO ORK WANTZ TO BE A DREAD!")
            
            state.pilot_id = pilot_result.pilot_id
            
            # Шаг 2: Построить каркас
            frame_result = await self._commands.send(
                BuildDreadFrameCommand(
                    workshop_id=cmd.workshop_id,
                    pilot_size=pilot_result.pilot.size
                )
            )
            state.frame_id = frame_result.frame_id
            
            # Шаг 3: Установить оружейные системы (параллельно!)
            weapon_tasks = [
                self._commands.send(
                    InstallWeaponSystemCommand(
                        frame_id=state.frame_id,
                        weapon_type=weapon_type,
                        mount_point=mount
                    )
                )
                for mount, weapon_type in enumerate(cmd.weapon_types)
            ]
            
            weapon_results = await asyncio.gather(*weapon_tasks)
            state.weapons_installed = [r.weapon_id for r in weapon_results]
            
            # Шаг 4: Подключить жизнеобеспечение
            await self._commands.send(
                InstallLifeSupportCommand(
                    frame_id=state.frame_id,
                    pilot_id=state.pilot_id
                )
            )
            
            # Шаг 5: Интегрировать пилота (точка невозврата!)
            await self._commands.send(
                IntegratePilotCommand(
                    dread_id=state.frame_id,
                    pilot_id=state.pilot_id
                )
            )
            state.pilot_integrated = True
            
            # Шаг 6: Тест в бою
            test_result = await self._commands.send(
                CombatTestCommand(
                    dread_id=state.frame_id,
                    enemy_count=10
                )
            )
            
            if test_result.pilot_survived:
                # Успех!
                await self._events.publish(
                    DeffDreadBuiltEvent(
                        dread_id=state.frame_id,
                        pilot_id=state.pilot_id
                    )
                )
                return state.frame_id
            else:
                # Пилот погиб, но дред уцелел - можно найти нового
                await self._events.publish(
                    PilotKilledInTestEvent(
                        dread_id=state.frame_id,
                        pilot_id=state.pilot_id
                    )
                )
                
                # Retry с новым пилотом?
                if cmd.retry_on_pilot_death:
                    return await self.execute(cmd)  # Рекурсия!
                
                raise PilotKilledError()
                
        except Exception as e:
            # Компенсация очень дорогая - пилот может быть уже интегрирован!
            if state.pilot_integrated:
                # Слишком поздно для отката
                await self._events.publish(
                    DeffDreadBuildFailedAfterIntegrationEvent(...)
                )
            else:
                await self._compensate_all(state)
            
            raise
```

#### Преимущества Saga:

✅ **Управление сложными процессами** — явное описание шагов  
✅ **Компенсация при ошибках** — откат изменений  
✅ **Сохранение состояния** — можно продолжить после сбоя  
✅ **Параллельное выполнение** — шаги выполняются асинхронно где возможно  
✅ **Retry логика** — автоматические повторы

---

### Stage 5: Bounded Contexts — Разделение на контексты

**Цель:** Показать как разбить систему на независимые bounded contexts

#### Новые требования:

23. **Интеграция с "Lootin' Department"** — система управления грабежами
24. **Интеграция с "WAAAGH! HQ"** — центральное управление ордой
25. **Интеграция с "Battle Management"** — система управления боями
26. **Billing для мастерских** — оплата за создание оружия (в зубах)

#### Проблема:

Один монолитный контекст становится слишком большим:
- Мастерская не должна знать как организованы рейды
- Битвы не должны знать как строится оружие
- Биллинг имеет свою модель "оплаты"

#### Bounded Contexts:

```
╔══════════════════════════════════════════════════════╗
║              ORK ECOSYSTEM (Context Map)             ║
╚══════════════════════════════════════════════════════╝

┌────────────────────────────────┐
│   MEK WORKSHOP CONTEXT         │ ← CORE DOMAIN
│   (наш основной контекст)      │
│                                │
│   - MekWorkshop (Aggregate)    │
│   - Weapon (Entity)            │
│   - Building weapons           │
│   - WAAAGH energy calculation  │
│                                │
│   Ubiquitous Language:         │
│   • Shoota, Dakka, Mek         │
│   • Build, Loot, Fire          │
└────────────────────────────────┘
         │           │
         │           │ Published Language
         │           │ (Domain Events)
         ↓           ↓
┌──────────────┐  ┌──────────────────┐
│   LOOTIN'    │  │   WAAAGH! HQ     │
│   CONTEXT    │  │   CONTEXT        │
│              │  │                  │
│ - Raid       │  │ - Warband        │
│ - Target     │  │ - Ork (different │
│ - Plunder    │  │   model!)        │
│              │  │ - Territory      │
│ Upstream ────┼──┤ Conformist       │
│ (Supplier)   │  │ (Customer)       │
└──────────────┘  └──────────────────┘
         │                  │
         │ ACL              │ Events
         ↓                  ↓
┌──────────────────────────────────┐
│      BATTLE CONTEXT              │
│                                  │
│  - Battle (different aggregate!) │
│  - Unit                          │
│  - CombatResult                  │
│                                  │
│  Shared Kernel with Workshop    │
│  (Color enum, DakkaRating)       │
└──────────────────────────────────┘
         │
         │ Separate Ways
         ↓
┌──────────────────────────────────┐
│      BILLING CONTEXT             │
│                                  │
│  - Invoice                       │
│  - Payment (in teef!)            │
│  - MekAccount                    │
│                                  │
│  Completely different model      │
└──────────────────────────────────┘
```

#### Context Map отношения:

**1. Workshop → Lootin' (Customer/Supplier):**

```python
# LOOTIN' CONTEXT (Upstream)
class Raid:
    """Рейд для добычи лута - их модель"""
    raid_id: RaidId
    target: ImperialTarget
    loot_acquired: List[ImperialEquipment]  # Их термины!
    
    def complete_raid(self) -> RaidCompletedEvent:
        return RaidCompletedEvent(
            raid_id=self.raid_id,
            loot=self.loot_acquired
        )

# WORKSHOP CONTEXT (Downstream) - Anti-Corruption Layer
class LootTranslationService:
    """ACL: переводит из терминов Lootin' в наши термины"""
    
    def translate_imperial_loot(
        self,
        imperial_equipment: ImperialEquipment
    ) -> Scrap:
        """Имперское снаряжение → орочий лут"""
        
        # Они называют это "Lasgun Mark IV"
        # Мы называем это "Shooty bitz"
        
        scrap_type_map = {
            "Lasgun": ScrapType.SHOOTY_BITZ,
            "Bolter": ScrapType.BIG_SHOOTY_BITZ,
            "Rhino APC": ScrapType.ENGINE,
            "Power Armor": ScrapType.ARMOR
        }
        
        return Scrap(
            source=ScrapSource.IMPERIAL,
            scrap_type=scrap_type_map.get(
                imperial_equipment.type,
                ScrapType.JUNK
            ),
            quality=self._assess_quality(imperial_equipment),
            quantity=1
        )
    
    def _assess_quality(self, equipment: ImperialEquipment) -> Quality:
        """Оценка качества имперского снаряжения"""
        if equipment.condition == "pristine":
            return Quality.SHINY
        elif equipment.condition == "damaged":
            return Quality.RUSTY
        else:
            return Quality.NORMAL

class RaidCompletedEventHandler:
    """Обработчик события из Lootin' контекста"""
    
    def __init__(
        self,
        translator: LootTranslationService,
        workshop_repo: WorkshopRepository
    ):
        self._translator = translator
        self._repo = workshop_repo
    
    async def handle(self, event: RaidCompletedEvent):
        """Когда рейд завершён - добавить лут в мастерскую"""
        
        # Получить мастерскую орды
        workshop = await self._repo.get_by_warband(event.warband_id)
        
        # Перевести через ACL
        for imperial_item in event.loot:
            scrap = self._translator.translate_imperial_loot(imperial_item)
            workshop.loot_scrap(scrap)
        
        await self._repo.save(workshop)
```

**2. Workshop → WAAAGH! HQ (Published Language):**

```python
# WORKSHOP публикует события на общем языке
@dataclass
class WeaponBuiltEvent(DomainEvent):
    """Событие понятное всем контекстам"""
    weapon_id: WeaponId
    workshop_id: WorkshopId
    weapon_type: str  # "shoota", "big_shoota"
    dakka: int
    occurred_at: datetime

# WAAAGH! HQ подписывается на события
class WaaghHQEventHandler:
    """В контексте HQ - своя модель Weapon!"""
    
    async def handle(self, event: WeaponBuiltEvent):
        # В HQ weapon - это просто строка в инвентаре
        warband = await self._repo.get_warband_by_workshop(
            event.workshop_id
        )
        
        # Их модель намного проще
        warband.add_equipment(
            equipment_type="weapon",
            name=event.weapon_type,
            power_level=event.dakka // 10  # Упрощённая метрика
        )
        
        await self._repo.save(warband)
```

**3. Workshop ↔ Battle (Shared Kernel):**

```python
# SHARED KERNEL - общие Value Objects
# shared/domain/value_objects.py

@dataclass(frozen=True)
class DakkaRating:
    """Используется в обоих контекстах"""
    base_dakka: int
    bonus_dakka: int
    
    @property
    def total(self) -> int:
        return self.base_dakka + self.bonus_dakka

class Color(Enum):
    """Используется в обоих контекстах"""
    RED = "red"
    BLUE = "blue"
    GREEN = "green"
    UNPAINTED = "unpainted"

# WORKSHOP CONTEXT
class Weapon(Entity):
    def __init__(self, ..., dakka: DakkaRating, color: Color):
        self._dakka = dakka  # Shared VO
        self._color = color  # Shared VO

# BATTLE CONTEXT
class BattleWeapon(Entity):
    """Совершенно другой entity, но использует те же VO"""
    def __init__(self, ..., firepower: DakkaRating, paint: Color):
        self._firepower = firepower  # Тот же VO!
        self._paint = paint  # Тот же VO!
```

**4. Workshop ⊥ Billing (Separate Ways):**

```python
# BILLING CONTEXT - полностью независимая модель
class MekAccount:
    """В биллинге мек - это просто счёт"""
    account_id: AccountId
    mek_name: str  # Только имя, без domain логики
    balance_in_teef: int  # Зубы как валюта!
    
    def charge_for_weapon(self, weapon_complexity: int):
        cost = weapon_complexity * 10
        if self.balance_in_teef < cost:
            raise InsufficientTeefError()
        
        self.balance_in_teef -= cost

# Интеграция через HTTP API (Separate Ways)
class BillingClient:
    """REST клиент для биллинга"""
    
    async def charge_for_build(
        self,
        mek_name: str,
        weapon_complexity: int
    ):
        response = await self._http.post(
            "/api/billing/charge",
            json={
                "mek_name": mek_name,
                "complexity": weapon_complexity
            }
        )
        
        if response.status_code != 200:
            raise BillingError()

# В Workshop используем клиент
class BuildWeaponCommandHandler:
    def __init__(
        self,
        workshop_repo: WorkshopRepository,
        billing_client: BillingClient  # Внешняя зависимость
    ):
        self._repo = workshop_repo
        self._billing = billing_client
    
    async def handle(self, cmd: BuildWeaponCommand):
        workshop = await self._repo.get(cmd.workshop_id)
        
        # Сначала проверить оплату
        await self._billing.charge_for_build(
            mek_name=workshop.mek_name,
            weapon_complexity=cmd.blueprint.complexity
        )
        
        # Потом строить
        weapon = workshop.build_weapon(cmd.blueprint)
        await self._repo.save(workshop)
```

#### Структура проекта по контекстам:

```
da-big-mek-workshop/
├── shared-kernel/
│   └── domain/
│       └── value_objects/
│           ├── dakka_rating.py
│           ├── color.py
│           └── primitives.py
│
├── contexts/
│   ├── workshop/              # Наш основной контекст
│   │   ├── domain/
│   │   │   ├── aggregates/
│   │   │   ├── entities/
│   │   │   ├── value_objects/
│   │   │   └── events/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── anti_corruption/   # ACL для интеграций
│   │       └── loot_translator.py
│   │
│   ├── lootin/                # Внешний контекст (для примера)
│   │   └── (их код)
│   │
│   ├── waaagh-hq/             # Внешний контекст
│   │   └── (их код)
│   │
│   ├── battle/                # Shared Kernel с Workshop
│   │   └── (их код)
│   │
│   └── billing/               # Отдельный сервис
│       └── (их код)
│
└── integration/
    ├── event_bus.py           # Для Published Language
    └── http_clients/          # Для Separate Ways
```

#### Зачем Bounded Contexts:

✅ **Независимость** — контексты развиваются отдельно  
✅ **Разные модели** — Weapon в Workshop ≠ Weapon в Battle  
✅ **Упрощение** — каждый контекст решает свою задачу  
✅ **Масштабирование** — можно выделить в микросервисы  
✅ **Команды** — разные команды работают над разными контекстами

---

## 📊 Итоговое сравнение: Сервисный слой vs DDD

| Аспект | Сервисный слой (Stage 0) | DDD (Stage 5) |
|--------|--------------------------|---------------|
| **Бизнес-логика** | Размазана по services.py | Инкапсулирована в aggregates |
| **Валидация** | Дублируется в 5 местах | Защищена инвариантами |
| **Тестирование** | Требует моки БД | Pure domain tests без БД |
| **Понимание** | Нужно читать код | Ubiquitous Language |
| **Изменения** | Затрагивают 10+ файлов | Изменения в одном aggregate |
| **Производительность** | Одна БД для всего | CQRS: write/read разделены |
| **Сложные процессы** | Спагетти в сервисах | Явные Sagas |
| **Масштабирование** | Монолит | Bounded Contexts → микросервисы |
| **Интеграция** | Прямые зависимости | ACL, Published Language |
| **Audit trail** | Нужно логировать вручную | Event Sourcing из коробки |

---

## 🎓 Что вы изучите на каждом этапе

### Stage 0 → 1: Tactical DDD
- Value Objects vs Entities
- Aggregate Roots
- Domain Events
- Rich Domain Model
- Инварианты

### Stage 1 → 2: Specifications & Services
- Specification Pattern
- Domain Services
- Composable business rules
- Dependency Injection

### Stage 2 → 3: CQRS
- Command/Query разделение
- Event-driven архитектура
- Event Sourcing
- Eventual Consistency
- Read Models

### Stage 3 → 4: Sagas
- Distributed transactions
- Компенсирующие действия
- Orchestration vs Choreography
- Saga state management

### Stage 4 → 5: Strategic DDD
- Bounded Contexts
- Context Mapping
- Anti-Corruption Layer
- Published Language
- Microservices architecture

---

## 🚀 Начало работы

```bash
# Клонировать репозиторий
git clone https://github.com/yourusername/da-big-mek-workshop.git

# Переключиться на нужный этап
git checkout stage-0-anemic  # Начать с боли
git checkout stage-1-tactical # Или сразу с DDD
git checkout stage-5-contexts # Или финальная версия

# Установить зависимости
pip install -r requirements.txt

# Запустить тесты
pytest

# Поднять инфраструктуру (для stage 3+)
docker-compose up -d

# Запустить приложение
python manage.py runserver
```

---

## 📖 Дополнительные материалы

- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [Implementing Domain-Driven Design by Vaughn Vernon](https://vaughnvernon.com/)
- [Warhammer 40k Ork Lore](https://warhammer40k.fandom.com/wiki/Orks) для вдохновения 🟢

---

**WAAAGH! Let's build some proppa' DDD!** 🟢⚙️💥
