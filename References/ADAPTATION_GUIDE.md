# 🔧 Руководство по адаптации движка под новый концепт

Это руководство поможет вам адаптировать движок раннера под совершенно новый концепт (например, бегающий человечек вместо ракеты, без пауер-апов, птиц и т.д.).

## 📋 Обзор архитектуры

Движок состоит из следующих модулей:

1. **Player (Игрок)** - управление персонажем
2. **SpawnManager** - управление спавном объектов
3. **GameManager** - общая логика игры
4. **Obstacle/Collectable** - базовые классы для препятствий и собираемых предметов
5. **Background** - фон и параллакс

## 🎯 Шаг 1: Замена игрока (Player)

### 1.1 Переименование и рефакторинг

**Файлы для изменения:**
- `Player/rocket.gd` → переименовать в `player.gd` или `character.gd`
- `Player/rocket.tscn` → переименовать в `player.tscn` или `character.tscn`

**В `Player/rocket.gd` (или новом `player.gd`):**

```gdscript
# ЗАМЕНИТЬ:
@onready var rocket_collision_shape: CollisionShape2D = $CollisionShape2D
var half_of_rocket_width: float

# НА:
@onready var player_collision_shape: CollisionShape2D = $CollisionShape2D
var half_of_player_width: float

# ЗАМЕНИТЬ все упоминания "rocket" на "player" или "character"
```

**В `Managers/game_manager.gd`:**

```gdscript
# ЗАМЕНИТЬ:
signal rocket_speed_changed(new_speed: float)
signal ordered_rocket_to_target(new_target_x_pos: float, side_expected: int)
signal rocket_has_reached_target
var rocket_speed: float = 600
var current_rocket_x_pos: float

# НА:
signal player_speed_changed(new_speed: float)
signal ordered_player_to_target(new_target_x_pos: float, side_expected: int)
signal player_has_reached_target
var player_speed: float = 600
var current_player_x_pos: float
```

**В `Player/rocket.gd`:**

```gdscript
# ЗАМЕНИТЬ:
GameManager.rocket_speed_changed.connect(_on_rocket_speed_changed)
GameManager.ordered_rocket_to_target.connect(_set_target_pos)
GameManager.rocket_has_reached_target.connect(_on_target_reached)
initial_flame_speed = GameManager.rocket_speed
self.position.y += (GameManager.rocket_speed * delta)
GameManager.current_rocket_x_pos = self.position.x
GameManager.emit_signal("rocket_speed_changed", rocket_speed_before_boost * 1.5)

# НА:
GameManager.player_speed_changed.connect(_on_player_speed_changed)
GameManager.ordered_player_to_target.connect(_set_target_pos)
GameManager.player_has_reached_target.connect(_on_target_reached)
initial_speed = GameManager.player_speed
self.position.y += (GameManager.player_speed * delta)
GameManager.current_player_x_pos = self.position.x
GameManager.emit_signal("player_speed_changed", player_speed_before_boost * 1.5)
```

### 1.2 Удаление специфичных для ракеты элементов

**Удалить из `Player/rocket.gd`:**
- `flame_particle_node` (частицы пламени)
- `FLAME_GRADIENTS` (градиенты пламени)
- `SHIELD_TEXTURE` и логику щита (если не нужны пауер-апы)
- `powerup_overlay_node` (если не нужны пауер-апы)
- Все функции связанные с boost и shield

**Упрощенная версия для бегающего человечка:**

```gdscript
extends Area2D

signal player_hurt

const MAX_SPEED: float = 100
const ACCELERATION: float = 70
const FRICTION = 90
const ROTATION_PER_FRAME = 50 # in degrees

@onready var player_collision_shape: CollisionShape2D = $CollisionShape2D
@onready var sprite_node: Sprite2D = $Sprite  # Простой спрайт вместо сложной структуры

var recently_collided_obstacles: Array[Node2D] = []
var is_game_running: bool = false
var is_free_falling: bool = false
var move_vec: Vector2
var half_of_player_width: float

func move(delta: float):
    # Логика движения остается той же
    # ...
```

### 1.3 Обновление скинов

Если у человечка будут разные костюмы/цвета:

**В `Managers/skin_manager.gd`:**
- Оставить логику загрузки текстур
- Обновить пути к новым файлам

**В `Data/skin_data.json`:**
- Обновить список скинов под новый концепт

## 🚫 Шаг 2: Удаление ненужных элементов

### 2.1 Удаление пауер-апов

**Файлы для удаления/отключения:**
- `Collectables/Shield/` - удалить или закомментировать
- `Collectables/Boost/` - удалить или закомментировать
- `Managers/powerup_manager.gd` - удалить или отключить

**В `Managers/spawn_manager.gd`:**

```gdscript
enum {
    # BIRD,        # Закомментировать если не нужны
    # SATELLITE,   # Закомментировать если не нужны
    STAR,          # Оставить если нужны собираемые предметы
    # SHIELD,      # Удалить
    # BOOST        # Удалить
}

@onready var SPAWNABLE_SCENES: Dictionary = {
    # BIRD: preload("res://Enemies/Birds/Bird.tscn"),
    # SATELLITE: preload("res://Enemies/Satellite/satellite.tscn"),
    STAR: preload("res://Collectables/Star/star.tscn"),
    # SHIELD: preload("res://Collectables/Shield/shield.tscn"),
    # BOOST: preload("res://Collectables/Boost/boost.tscn")
}
```

**В `spawner.gd`:**

```gdscript
# Удалить или закомментировать:
# @onready var powerup_timer: Timer = $PowerupTImer
# powerup_timer.timeout.connect(_on_powerup_timer_timeout)
# powerup_timer.stop()
# powerup_timer.start(2)

# Удалить функцию:
# func _on_powerup_timer_timeout() -> void:
#     ...
```

**В `Player/rocket.gd`:**

```gdscript
# Удалить все связанное с пауер-апами:
# PowerupManager.use_powerup.connect(_on_use_powerup)
# PowerupManager.stop_powerup.connect(_on_stop_powerup)
# Все функции activate_shield, apply_boost и т.д.
```

**В `project.godot`:**

```ini
# Закомментировать или удалить:
# PowerupManager="*res://Managers/powerup_manager.gd"
```

### 2.2 Удаление врагов (если не нужны)

**В `Managers/spawn_manager.gd`:**

```gdscript
enum {
    # BIRD,        # Удалить
    # SATELLITE,   # Удалить
    STAR,          # Оставить если нужны собираемые предметы
}

# Удалить из SPAWNABLE_SCENES все враги
# Удалить из active_spawns все враги
# Удалить из difficulty_level_values все настройки врагов
```

**В `spawner.gd`:**

```gdscript
# Удалить:
# @onready var bird_timer: Timer = $BirdTimer
# @onready var satellite_timer: Timer = $SatelliteTimer
# bird_timer.timeout.connect(_on_bird_timer_timeout)
# satellite_timer.timeout.connect(_on_satellite_timer_timeout)
# bird_timer.stop()
# satellite_timer.stop()

# Удалить функции:
# func _on_bird_timer_timeout() -> void: ...
# func _on_satellite_timer_timeout() -> void: ...
```

**В `spawner.tscn`:**
- Удалить ноды `BirdTimer` и `SatelliteTimer` из сцены

### 2.3 Упрощение системы препятствий

Если нужны только статичные препятствия (без движения по горизонтали):

**Создать новый простой класс препятствия:**

```gdscript
# Enemies/SimpleObstacle.gd
extends Area2D

@export var free_fall_multiplier: float = 1

func _ready() -> void:
    GameManager.game_over.connect(_on_game_over)

func free_fall(delta) -> void:
    var moveable_y: float = (GameManager.player_speed * free_fall_multiplier * delta)
    self.position.y += moveable_y

func _physics_process(delta: float) -> void:
    free_fall(delta)

func _on_visible_on_screen_notifier_2d_screen_exited() -> void:
    queue_free()

func _on_hit() -> void:
    $CollisionShape2D.set_deferred("disabled", true)
    # Эффекты при попадании
```

## 🎨 Шаг 3: Адаптация под новый концепт

### 3.1 Новые типы объектов

**Добавить в `Managers/spawn_manager.gd`:**

```gdscript
enum {
    OBSTACLE_1,    # Новый тип препятствия 1
    OBSTACLE_2,    # Новый тип препятствия 2
    COLLECTABLE_1, # Новый тип собираемого предмета
    # ... и т.д.
}

@onready var SPAWNABLE_SCENES: Dictionary = {
    OBSTACLE_1: preload("res://Enemies/NewObstacle1/obstacle1.tscn"),
    OBSTACLE_2: preload("res://Enemies/NewObstacle2/obstacle2.tscn"),
    COLLECTABLE_1: preload("res://Collectables/NewCollectable/collectable1.tscn"),
}
```

### 3.2 Настройка сложности

**В `Managers/spawn_manager.gd`:**

```gdscript
var difficulty_level_values = {
    GameManager.DIFFICULTY_LEVELS.EASY: {
        OBSTACLE_1: {
            "spawning_interval" = {
                "min": 2.0,
                "max": 4.0,
            },
        },
        COLLECTABLE_1: {
            "spawning_interval" = {
                "min": 1.5,
                "max": 3.0,
            },
        }
    },
    # ... для других уровней сложности
}
```

### 3.3 Обновление спавнера

**В `spawner.gd`:**

```gdscript
@onready var obstacle1_timer: Timer = $Obstacle1Timer
@onready var collectable1_timer: Timer = $Collectable1Timer

func _ready() -> void:
    # ...
    obstacle1_timer.timeout.connect(_on_obstacle1_timer_timeout)
    collectable1_timer.timeout.connect(_on_collectable1_timer_timeout)

func _on_start_spawning() -> void:
    obstacle1_timer.start(2.0)
    collectable1_timer.start(1.5)

func _on_obstacle1_timer_timeout() -> void:
    spawn_obstacle(SpawnManager.OBSTACLE_1)
    obstacle1_timer.start(randf_range(2.0, 4.0))

func _on_collectable1_timer_timeout() -> void:
    spawn_obstacle(SpawnManager.COLLECTABLE_1)
    collectable1_timer.start(randf_range(1.5, 3.0))
```

## 🎮 Шаг 4: Адаптация управления

### 4.1 Для бегающего человечка

Логика движения уже подходит, но можно добавить:

**Анимации бега:**

```gdscript
# В Player/player.gd
@onready var animated_sprite: AnimatedSprite2D = $AnimatedSprite2D

func move(delta: float):
    # ... существующая логика движения
    
    # Добавить анимации
    if move_vec.length() > 0:
        animated_sprite.play("run")
    else:
        animated_sprite.play("idle")
    
    # Поворот спрайта в направлении движения
    if move_vec.x > 0:
        animated_sprite.flip_h = false
    elif move_vec.x < 0:
        animated_sprite.flip_h = true
```

### 4.2 Прыжки (опционально)

```gdscript
# Добавить в Player/player.gd
const JUMP_FORCE: float = -300
var velocity: Vector2 = Vector2.ZERO
var is_on_ground: bool = true

func _input(event):
    if event.is_action_pressed("jump") and is_on_ground:
        velocity.y = JUMP_FORCE
        is_on_ground = false

func _physics_process(delta: float):
    # Гравитация
    if not is_on_ground:
        velocity.y += 980 * delta  # гравитация
    
    # Применение скорости
    self.position += velocity * delta
    
    # Проверка на землю (упрощенно)
    if self.position.y >= ground_level:
        self.position.y = ground_level
        velocity.y = 0
        is_on_ground = true
```

## 🎨 Шаг 5: Обновление визуала

### 5.1 Фон

- Заменить `Background/` файлы на новые референсы
- Обновить параллакс под новый стиль

### 5.2 UI

- Обновить иконки и элементы интерфейса
- Убрать элементы связанные с пауер-апами (если удаляете)

## ✅ Чеклист адаптации

- [ ] Переименованы файлы игрока (rocket → player/character)
- [ ] Обновлены все упоминания "rocket" на "player"
- [ ] Удалены/закомментированы пауер-апы
- [ ] Удалены/закомментированы ненужные враги
- [ ] Обновлен SpawnManager под новые типы объектов
- [ ] Обновлен spawner.gd под новые таймеры
- [ ] Созданы новые сцены препятствий/собираемых предметов
- [ ] Обновлены настройки сложности
- [ ] Обновлен фон и визуальные элементы
- [ ] Обновлен UI (убраны элементы пауер-апов)
- [ ] Протестирована игра после всех изменений

## 📝 Пример: Минимальный раннер

Если нужен максимально простой раннер (только препятствия и собираемые предметы):

1. **Удалить:**
   - Все пауер-апы
   - Все враги с горизонтальным движением
   - Систему скинов (или упростить)

2. **Оставить:**
   - Базовое движение игрока
   - Простые статичные препятствия
   - Простые собираемые предметы
   - Фон и параллакс

3. **Упростить:**
   - SpawnManager - только 2-3 типа объектов
   - spawner.gd - только 2-3 таймера
   - Player - убрать всю логику пауер-апов

## 🚀 Быстрый старт для нового концепта

1. Создайте копию проекта
2. Следуйте шагам выше по порядку
3. Тестируйте после каждого большого изменения
4. Используйте Git для отслеживания изменений

---

**Важно:** Сохраняйте бэкапы перед большими изменениями!

