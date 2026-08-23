# C-Extended

A minimal transpiler that adds classes to C.

C-Extended (`.cx`) extends C with three constructs: `class`, associated functions, and
methods with a `&self` receiver. Everything else is plain C and passes through
unchanged. The output is portable C — no runtime, no vtables, no hidden allocations.

## Example

`player.cx`

```cx
class Player {
    char *name;
    int health;

    Player Player::new(char *name) {
        Player p;
        p.name = name;
        p.health = 100;
        return p;
    }

    int is_alive(&self) {
        return self.health > 0;
    }

    void take_damage(&self, int amount) {
        self.health = self.health - amount;
    }
}

int main(void) {
    Player p = Player::new("Eray");
    p.take_damage(30);
    return p.is_alive();
}
```

`player.c` (generated)

```c
typedef struct player_t {
    char *name;
    int health;
} player_t;

player_t player_new(char *name);
int player_is_alive(player_t *self);
void player_take_damage(player_t *self, int amount);

player_t player_new(char *name) {
    player_t p;
    p.name = name;
    p.health = 100;
    return p;
}

int player_is_alive(player_t *self) {
    return self->health > 0;
}

void player_take_damage(player_t *self, int amount) {
    self->health = self->health - amount;
}

int main(void) {
    player_t p = player_new("Eray");
    player_take_damage(&p, 30);
    return player_is_alive(&p);
}
```

## Syntax

**Classes** declare fields like a struct body and become a `typedef`'d struct.

**Associated functions** are written as `Type::name`. They take no receiver and are
called through the type: `Player::new(...)` becomes `player_new(...)`. `new` is a
convention, not a keyword — there is no implicit allocation or destruction.

**Methods** take `&self` as their first parameter, which expands to a pointer to the
owning class. Inside the body, `self.field` is emitted as `self->field`.

**Calls** on a value take its address, calls on a pointer pass it through:

```
p.is_alive()      →  player_is_alive(&p)
pp->is_alive()    →  player_is_alive(pp)
```

Everything else — `#include`, free functions, `struct`, enums, macros — is copied to
the output verbatim.

## Name mangling

The class name is lowercased and used as a prefix. Function names are kept verbatim.

| C-Extended        | C                     |
|-------------------|-----------------------|
| `class Player`    | `player_t`            |
| `Player::new`     | `player_new`          |
| `take_damage`     | `player_take_damage`  |
| `&self`           | `player_t *self`      |
| `self.health`     | `self->health`        |

## Usage

```sh
cx player.cx -o player.c
cc player.c -o player
```

## Non-goals

Inheritance, virtual dispatch, generics, operator overloading, exceptions, destructors,
RAII, and a runtime library. If a feature cannot be expressed as a struct plus a free
function, it does not belong in C-Extended.

## License

MIT
