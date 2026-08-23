# C-Extended

A minimal transpiler that adds classes to C. Like C++, but far smaller.

C-Extended (`.cx`) is C with three additions: `class`, **associated functions**, and
**methods** with a `&self` receiver. Everything else is plain C and passes through
untouched. The output is portable C — no runtime, no vtables, no hidden allocations.

---

## Example

**`player.cx`**

```cx
class Player {
    char *name;
    int health;
    int level;

    Player Player::new(char *name) {
        Player p;
        p.name = name;
        p.health = 100;
        p.level = 1;
        return p;
    }

    int is_alive(&self) {
        return self.health > 0;
    }

    void take_damage(&self, int amount) {
        self.health = self.health - amount;
        if (self.health < 0) {
            self.health = 0;
        }
    }

    void level_up(&self) {
        self.level = self.level + 1;
        self.health = 100;
    }
}

int main(void) {
    Player p = Player::new("Eray");

    p.take_damage(30);
    if (p.is_alive()) {
        p.level_up();
    }

    return p.level;
}
```

**`player.c` (generated)**

```c
typedef struct player_t {
    char *name;
    int health;
    int level;
} player_t;

player_t player_new(char *name);
int player_is_alive(player_t *self);
void player_take_damage(player_t *self, int amount);
void player_level_up(player_t *self);

player_t player_new(char *name) {
    player_t p;
    p.name = name;
    p.health = 100;
    p.level = 1;
    return p;
}

int player_is_alive(player_t *self) {
    return self->health > 0;
}

void player_take_damage(player_t *self, int amount) {
    self->health = self->health - amount;
    if (self->health < 0) {
        self->health = 0;
    }
}

void player_level_up(player_t *self) {
    self->level = self->level + 1;
    self->health = 100;
}

int main(void) {
    player_t p = player_new("Eray");

    player_take_damage(&p, 30);
    if (player_is_alive(&p)) {
        player_level_up(&p);
    }

    return p.level;
}
```

---

## Syntax

### Classes

A class declares fields exactly like a `struct` body. It becomes a `typedef`'d struct.

```cx
class Enemy {
    int health;
    int damage;
}
```

```c
typedef struct enemy_t {
    int health;
    int damage;
} enemy_t;
```

### Associated functions

A member function written as `Type::name` is an **associated function**: it has no
receiver and is called through the type itself.

```cx
Player Player::new(char *name) { ... }
Player Player::from_save(int slot) { ... }
```

```c
player_t player_new(char *name);
player_t player_from_save(int slot);
```

Call site:

```cx
Player p = Player::new("Eray");
```

```c
player_t p = player_new("Eray");
```

`new` carries no special meaning — it is a conventional name, not a keyword. There is
no implicit allocation, construction, or destruction.

### Methods

A member function whose first parameter is `&self` is a **method**: it operates on an
instance. `&self` expands to a pointer to the owning class.

```cx
int Player::is_alive(&self) { ... }
```

```c
int player_is_alive(player_t *self);
```

Inside the body, `self.field` refers to a field of the receiver and is emitted as
`self->field`. Field access on any other value keeps its original form.

### Method calls

A call on a value takes the address of that value:

```cx
p.is_alive()          →  player_is_alive(&p)
p.take_damage(30)     →  player_take_damage(&p, 30)
```

A call on a pointer passes it through:

```cx
pp->is_alive()        →  player_is_alive(pp)
```

### Everything else

Any construct that is not a `class` body, an associated-function call, or a method call
is copied to the output unchanged: `#include`, typedefs, free functions, `struct`,
enums, macros, pointers, arrays.

---

## Name mangling

| C-Extended         | C                          | Kind                 |
|--------------------|----------------------------|----------------------|
| `class Player`     | `typedef struct player_t { ... } player_t;` | class |
| `Player::new`      | `player_new`               | associated function  |
| `Player::from_save`| `player_from_save`         | associated function  |
| `take_damage(&self, int)` | `player_take_damage(player_t *self, int)` | method |
| `self.health`      | `self->health`             | field access         |

The class name is lowercased and used as the prefix. Function names are kept verbatim.

---

## Usage

```sh
cx player.cx -o player.c
cc player.c -o player
```

Or pipe straight into the compiler:

```sh
cx player.cx | cc -x c - -o player
```

---

## Non-goals

- inheritance, virtual dispatch, vtables
- templates or generics
- operator overloading
- exceptions
- destructors, RAII, implicit copy or move
- a runtime library of any kind

If a feature cannot be expressed as a struct plus a free function, it does not belong
in C-Extended.
