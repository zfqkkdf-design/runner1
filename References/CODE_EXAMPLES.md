# 💻 Примеры кода для адаптации

Готовые примеры кода для быстрой адаптации движка под новый концепт.

## 🔄 Пример 1: Упрощенный Player (без пауер-апов)

```gdscript
# Player/player.gd
extends Area2D

signal player_hurt

const MAX_SPEED: float = 100
const ACCELERATION: float = 70
const FRICTION = 90

@onready var player_collision_shape: CollisionShape2D = $CollisionShape2D
@onready var sprite_node: Sprite2D = $Sprite
@onready var animated_sprite: AnimatedSprite2D = $AnimatedSprite2D  # Если используете анимации

var recently_collided_obstacles: Array[Node2D] = []
var is_game_running: bool = false
var is_free_falling: bool = false
var move_vec: Vector2
var half_of_player_width: float

func _ready() -> void:
    self.hide()
    GameManager.game_started.connect(_on_game_start)
    GameManager.game_over.connect(_on_game_over)
    self.player_hurt.connect(_on_hurt)
    
    half_of_player_width = player_collision_shape.shape.radius + 5

func _reset_properties() -> void:
    self.show()
    $CollisionShape2D.disabled = false
    move_vec = Vector2.ZERO
    is_free_falling = false
    self.rotation = 0
    
    var screen_size = get_viewport_rect().size
    self.position.x = screen_size.x / 2
    self.position.y = 0.8 * screen_size.y

func apply_friction(velocity: float, delta: float) -> float:
    var magnitude_of_vec = abs(velocity)
    var direction_of_vec = (velocity / magnitude_of_vec)
    magnitude_of_vec -= FRICTION * delta
    
    return (direction_of_vec * magnitude_of_vec)

func move(delta: float):
    if is_free_falling:
        return

    if GameManager.is_left_button_pressed or GameManager.is_right_button_pressed:
        if (GameManager.is_left_button_pressed and GameManager.is_right_button_pressed):
            move_vec.x = apply_friction(move_vec.x, delta)
        else:
            if GameManager.is_left_button_pressed:
                move_vec.x += -1 * (ACCELERATION * delta)
            elif GameManager.is_right_button_pressed:
                move_vec.x += (ACCELERATION * delta)
    elif move_vec.length() > (FRICTION * delta):
        move_vec.x = apply_friction(move_vec.x, delta)
    else:
        move_vec = Vector2.ZERO

    move_vec = move_vec.limit_length(MAX_SPEED)
    self.position += move_vec

    # Предотвращение выхода за экран
    var horizontal_screen_size = get_viewport_rect().size.x
    if (self.position.x >= horizontal_screen_size - half_of_player_width) or (self.position.x <= half_of_player_width):
        self.position.x = clamp(self.position.x, half_of_player_width, horizontal_screen_size - half_of_player_width)
        move_vec = Vector2.ZERO

    GameManager.current_player_x_pos = self.position.x
    
    # Анимации (если используете)
    if animated_sprite:
        if move_vec.length() > 0:
            animated_sprite.play("run")
        else:
            animated_sprite.play("idle")
        
        if move_vec.x > 0:
            animated_sprite.flip_h = false
        elif move_vec.x < 0:
            animated_sprite.flip_h = true

func _physics_process(delta: float) -> void:
    if is_game_running and not is_free_falling:
        move(delta)
    elif is_free_falling:
        _free_fall(delta)

func _free_fall(delta) -> void:
    self.position.y += (GameManager.player_speed * delta)

func _on_hurt():
    if is_free_falling:
        return
    
    UiManager.emit_signal("triggered_menu_ui_setup")
    is_free_falling = true
    $CollisionShape2D.set_deferred("disabled", true)
    
    var faceplant_tween = create_tween()
    faceplant_tween.tween_property(self, "rotation", deg_to_rad(180), 0.5)

func _on_area_shape_entered(_area_rid: RID, area: Area2D, _area_shape_index: int, _local_shape_index: int) -> void:
    if not recently_collided_obstacles.has(area):
        recently_collided_obstacles.append(area)
        
        var removal_timer = get_tree().create_timer(0.5)
        removal_timer.timeout.connect(func (): recently_collided_obstacles.erase(area))
        
        if area.has_method("_on_hit"):
            area.call("_on_hit")
        
        if area is Obstacle:
            emit_signal("player_hurt")

func _on_game_start() -> void:
    _reset_properties()
    is_game_running = true

func _on_game_over() -> void:
    self.hide()
```

## 🚫 Пример 2: Упрощенный SpawnManager (только препятствия и собираемые)

```gdscript
# Managers/spawn_manager.gd
extends Node

enum {
    OBSTACLE_1,      # Простое препятствие 1
    OBSTACLE_2,      # Простое препятствие 2
    COLLECTABLE_1,   # Собираемый предмет 1
}

@onready var SPAWNABLE_SCENES: Dictionary = {
    OBSTACLE_1: preload("res://Enemies/Obstacle1/obstacle1.tscn"),
    OBSTACLE_2: preload("res://Enemies/Obstacle2/obstacle2.tscn"),
    COLLECTABLE_1: preload("res://Collectables/Collectable1/collectable1.tscn"),
}

var active_spawns: Dictionary = {
    OBSTACLE_1: [],
    OBSTACLE_2: [],
    COLLECTABLE_1: [],
}

var difficulty_level_values = {
    GameManager.DIFFICULTY_LEVELS.EASY: {
        OBSTACLE_1: {
            "spawning_interval" = {
                "min": 2.0,
                "max": 4.0,
            },
        },
        OBSTACLE_2: {
            "spawning_interval" = {
                "min": 3.0,
                "max": 5.0,
            },
        },
        COLLECTABLE_1: {
            "spawning_interval" = {
                "min": 1.5,
                "max": 3.0,
            },
        }
    },
    GameManager.DIFFICULTY_LEVELS.NORMAL: {
        OBSTACLE_1: {
            "spawning_interval" = {
                "min": 1.0,
                "max": 3.0,
            },
        },
        OBSTACLE_2: {
            "spawning_interval" = {
                "min": 2.0,
                "max": 4.0,
            },
        },
        COLLECTABLE_1: {
            "spawning_interval" = {
                "min": 1.0,
                "max": 2.5,
            },
        }
    },
    GameManager.DIFFICULTY_LEVELS.HARD: {
        OBSTACLE_1: {
            "spawning_interval" = {
                "min": 0.5,
                "max": 2.0,
            },
        },
        OBSTACLE_2: {
            "spawning_interval" = {
                "min": 1.0,
                "max": 3.0,
            },
        },
        COLLECTABLE_1: {
            "spawning_interval" = {
                "min": 0.8,
                "max": 2.0,
            },
        }
    },
}

var current_obstacle1_spawning_interval: Vector2
var current_obstacle2_spawning_interval: Vector2
var current_collectable1_spawning_interval: Vector2

func _ready() -> void:
    GameManager.level_up.connect(_on_level_up)
    GameManager.difficulty_level_changed.connect(_reload_difficulty_data)
    _reload_difficulty_data()

func _reload_difficulty_data(difficulty_level: int = GameManager.current_difficulty_level):
    var current_difficulty_data: Dictionary = difficulty_level_values.get(difficulty_level)
    
    var obstacle1_data: Dictionary = current_difficulty_data.get(OBSTACLE_1)
    current_obstacle1_spawning_interval = Vector2(
        obstacle1_data.get("spawning_interval").min,
        obstacle1_data.get("spawning_interval").max
    )
    
    var obstacle2_data: Dictionary = current_difficulty_data.get(OBSTACLE_2)
    current_obstacle2_spawning_interval = Vector2(
        obstacle2_data.get("spawning_interval").min,
        obstacle2_data.get("spawning_interval").max
    )
    
    var collectable1_data: Dictionary = current_difficulty_data.get(COLLECTABLE_1)
    current_collectable1_spawning_interval = Vector2(
        collectable1_data.get("spawning_interval").min,
        collectable1_data.get("spawning_interval").max
    )

func _on_level_up() -> void:
    return  # Можно добавить логику увеличения сложности

func get_free_obstacle(obstacle_type: int) -> Node2D:
    var instantiable_res: PackedScene = SPAWNABLE_SCENES.get(obstacle_type)
    
    if not instantiable_res:
        push_error("No such obstacle type: " + str(obstacle_type))
        return null
    else:
        var new_obstacle: Node2D = instantiable_res.instantiate()
        (active_spawns.get(obstacle_type) as Array).append(new_obstacle)
        return new_obstacle

func make_obstacle_free(obstacle_node: Node2D, obstacle_type: int) -> void:
    (active_spawns.get(obstacle_type) as Array).erase(obstacle_node)
    obstacle_node.queue_free()
```

## 🎯 Пример 3: Упрощенный Spawner

```gdscript
# spawner.gd
extends Path2D

@onready var obstacle_path = $PathFollow2D2

@onready var obstacle1_timer: Timer = $Obstacle1Timer
@onready var obstacle2_timer: Timer = $Obstacle2Timer
@onready var collectable1_timer: Timer = $Collectable1Timer

const OBSTACLE_SPAWN_MARGIN: float = 20

func _ready() -> void:
    GameManager.screen_size_updated.connect(_on_screen_size_updated)
    GameManager.start_spawning.connect(_on_start_spawning)
    GameManager.stop_spawning.connect(_on_stop_spawning)
    
    obstacle1_timer.timeout.connect(_on_obstacle1_timer_timeout)
    obstacle2_timer.timeout.connect(_on_obstacle2_timer_timeout)
    collectable1_timer.timeout.connect(_on_collectable1_timer_timeout)
    
    _setup_spawn_line()

func _on_screen_size_updated(_screen_size: Vector2) -> void:
    _setup_spawn_line()

func _setup_spawn_line() -> void:
    self.curve.clear_points()
    var horizontal_screen_size = GameManager.game_screen_size.x
    self.curve.add_point(Vector2(0 + OBSTACLE_SPAWN_MARGIN, 0))
    self.curve.add_point(Vector2(horizontal_screen_size - OBSTACLE_SPAWN_MARGIN, 0))

func spawn_obstacle(obstacle_type: int):
    var obstacle_node: Node2D = SpawnManager.get_free_obstacle(obstacle_type)
    
    obstacle_path.progress_ratio = randf()
    var rand_xpos_on_path = obstacle_path.position.x
    var spawn_location = Vector2(rand_xpos_on_path, -250)
    
    obstacle_node.position = spawn_location
    add_child(obstacle_node)

func _on_stop_spawning() -> void:
    obstacle1_timer.stop()
    obstacle2_timer.stop()
    collectable1_timer.stop()

func _on_start_spawning() -> void:
    obstacle1_timer.start(2.0)
    obstacle2_timer.start(3.0)
    collectable1_timer.start(1.5)

func _on_obstacle1_timer_timeout() -> void:
    spawn_obstacle(SpawnManager.OBSTACLE_1)
    obstacle1_timer.start(randf_range(
        SpawnManager.current_obstacle1_spawning_interval.x,
        SpawnManager.current_obstacle1_spawning_interval.y
    ))

func _on_obstacle2_timer_timeout() -> void:
    spawn_obstacle(SpawnManager.OBSTACLE_2)
    obstacle2_timer.start(randf_range(
        SpawnManager.current_obstacle2_spawning_interval.x,
        SpawnManager.current_obstacle2_spawning_interval.y
    ))

func _on_collectable1_timer_timeout() -> void:
    spawn_obstacle(SpawnManager.COLLECTABLE_1)
    collectable1_timer.start(randf_range(
        SpawnManager.current_collectable1_spawning_interval.x,
        SpawnManager.current_collectable1_spawning_interval.y
    ))
```

## 🚧 Пример 4: Простое препятствие (без горизонтального движения)

```gdscript
# Enemies/SimpleObstacle/obstacle.gd
extends Area2D

@export var free_fall_multiplier: float = 1

@onready var collision_shape: CollisionShape2D = $CollisionShape2D
@onready var sprite: Sprite2D = $Sprite

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
    collision_shape.set_deferred("disabled", true)
    # Добавьте эффекты при попадании (частицы, звук и т.д.)

func _on_game_over() -> void:
    queue_free()
```

## ⭐ Пример 5: Простой собираемый предмет

```gdscript
# Collectables/SimpleCollectable/collectable.gd
extends Area2D

@onready var sprite: Sprite2D = $Sprite
@onready var collision_shape: CollisionShape2D = $CollisionShape2D
@onready var collection_sound: AudioStreamPlayer = $CollectedSound

@export_enum("Collectable1:0") var collectable_type: int = 0

func _ready() -> void:
    GameManager.game_over.connect(_on_game_over)

func free_fall(delta) -> void:
    var moveable_y: float = (GameManager.player_speed * delta)
    self.position.y += moveable_y

func _physics_process(delta: float) -> void:
    free_fall(delta)

func _on_visible_on_screen_notifier_2d_screen_exited() -> void:
    free_obstacle()

func _on_hit() -> void:
    collision_shape.set_deferred("disabled", true)
    collection_sound.play()
    
    # Эффекты при сборе
    var tween = create_tween().set_parallel(true)
    tween.tween_property(sprite, "scale", Vector2(1.5, 1.5), 0.2)
    tween.tween_property(sprite, "modulate:a", 0, 0.2)
    
    await tween.finished
    _on_collection()

func _on_collection() -> void:
    # Логика при сборе (увеличить счет, дать бонус и т.д.)
    StatManager.add_score(10)  # Пример
    free_obstacle()

func _on_game_over() -> void:
    free_obstacle()

func free_obstacle() -> void:
    if not self.is_queued_for_deletion():
        SpawnManager.make_obstacle_free(self, collectable_type)
```

## 🔄 Пример 6: Обновленный GameManager

```gdscript
# Managers/game_manager.gd
extends Node

signal screen_size_updated(new_screen_size: Vector2)
signal game_started
signal game_over

signal player_speed_changed(new_speed: float)
signal difficulty_level_changed(new_difficulty_level: int)

signal ordered_player_to_target(new_target_x_pos: float, side_expected: int)
signal player_has_reached_target

signal level_up

signal stop_spawning
signal start_spawning

enum DIFFICULTY_LEVELS {
    EASY,
    NORMAL,
    HARD
}

var game_screen_size: Vector2:
    set(value):
        if value != game_screen_size:
            game_screen_size = value
            self.emit_signal("screen_size_updated", value)

var current_difficulty_level: int = DIFFICULTY_LEVELS.EASY:
    set(new_difficulty_level):
        current_difficulty_level = new_difficulty_level
        self.emit_signal("difficulty_level_changed", current_difficulty_level)

var player_speed: float = 600
var current_player_x_pos: float

var is_left_button_pressed: bool = false
var is_right_button_pressed: bool = false
var screen_mid_x_pos: float

func _ready() -> void:
    self.level_up.connect(_increase_difficulty)
    self.player_speed_changed.connect(_on_player_speed_changed)
    self.game_started.connect(_reset_values)
    self.game_over.connect(_on_game_over)
    DataManager.data_reloaded.connect(_reload_data)

func _on_player_speed_changed(new_speed: float):
    player_speed = new_speed

func _increase_difficulty() -> void:
    emit_signal("player_speed_changed", player_speed + 100)

func _reset_values() -> void:
    is_left_button_pressed = false
    is_right_button_pressed = false

func _reload_data() -> void:
    current_difficulty_level = DataManager.gameplay.current_difficulty

func _on_game_over() -> void:
    self.emit_signal("stop_spawning")
```

## 📝 Примечания

- Все примеры упрощены и готовы к использованию
- Замените пути к сценам на свои
- Настройте параметры под свой баланс
- Добавьте свои эффекты и анимации
- Не забудьте обновить сцены (.tscn) под новые скрипты

