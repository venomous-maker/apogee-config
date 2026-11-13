# apogee-config

Configuration for [Apogee](https://github.com/venomous-maker) applications:
the environment and `.env` files (`Env`), and the application's settings as
one tree read by dotted path (`Config`), loaded from JSON files or set in
code, with typed sections filled by reflection.

```grav
import "std"
import "apogee-config"

fn main() {
    Env.loadAll()                                   // .env, .env.<APP_ENV>, .env.local
    Config.loadDir("config")                        // config/app.json -> "app.*", ...
    Config.set("app.name", Env.get("APP_NAME", "Apogee"))

    let debug = Config.getBool("app.debug")
    let port = Config.getInt("http.port", 8080)
    let mail = Config.section::<MailConfig>("mail") // fields filled by name / snake_case
}
```

## `.env` files

```sh
# comments, blank lines
APP_NAME=Shop            # an inline comment after whitespace
export APP_ENV=production
APP_KEY='single quotes are literal: ${NOT_EXPANDED}'
GREETING="double quotes: escapes \n \t \" \$, and
multi-line values"
DB_URL=postgres://${DB_HOST}:${DB_PORT:-5432}/app
```

- `Env.load(path = ".env", overwrite = false)`: how many variables it defined
  (0 when the file is missing). A variable already in the real environment
  wins unless `overwrite`.
- `Env.loadAll(dir = ".")`: `.env`, then `.env.<APP_ENV>`, then `.env.local`,
  each over the last.
- `Env.get(key, fallback)`, `getInt`, `getFloat`, `getBool` (true/false,
  1/0, yes/no, on/off), `getList` (comma-separated), `has`, `set`, `reset`.
- `Env.require(keys)`: one `ConfigError` naming every missing variable.
- A malformed line is a `ConfigError` with its file and line (`.env:3:
  expected KEY=value`).

## Settings

`Config` forwards to one `ConfigRepository` (swap it with `Config.use(repo)`,
say for a test).

| | |
|---|---|
| `get(path, fallback)` | the value at a dotted path: null, bool, long, double, string, `Array<any>`, `HashMap<string, any>` |
| `getString` / `getInt` / `getFloat` / `getBool` | typed, converting text where it makes sense (`"8080"` is 8080) |
| `getList(path)` / `getMap(path)` | an array (or a comma-separated string) as text items; a map |
| `section::<T>(path)` | a new `T` whose fields are set from the map at `path`, by field name or its snake_case spelling, converted to each field's type |
| `set(path, value)` / `setMany` / `merge(path, map)` (deep) / `push(path, item)` / `forget(path)` | |
| `loadJson(file, path = "")` / `loadDir(dir)` | JSON files merged in (`config/mail.json` becomes `mail.*`) |
| `require(paths)` | one `ConfigError` naming every missing setting |
| `toJson()` / `leaves()` | the whole tree, for a `config:show` command or a cache file |

In JSON files, a string that is exactly `"${NAME}"` or `"${NAME:-default}"`
becomes the environment variable's value, typed (`"${APP_DEBUG}"` gives a
bool, `"${PORT}"` a number); a placeholder inside longer text is expanded as
text.

## Speed

Every path, leaf or map, is kept in a flat index: a read is one hash lookup
however deep the path, and a read never writes, so any number of request
threads read settings at once without locks. Settings are made at boot; a
`set()` also updates the index (replacing a map drops every path under the
old one).

## Tests

```bash
grav test tests/config.grav
```

MIT licensed.
