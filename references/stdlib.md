# Raven standard library (3.0)

Every module is bundled into the compiler. Import functions by name with
`from std.fs import read, write`, or import the module and call through it
with `import std.fs` then `fs.read(path)`. String methods, `dict`, `set`,
`Result`/`Option` helpers, and `Error` are in the prelude and need no
import. Fallible calls return `Result[T, Error]`; `e.message()` reads the
text of an `Error`.

The stdlib source in `stdlib/std/*.rv` is the authoritative signature list.

## std.io

`print(x)`, `println(x)`, `read_line() -> Option[str]`, `input(prompt)`.
The builtin `print(a, b, ...)` covers most output.

## std.fs

`read(path) -> Result[str, Error]`, `read_lines`, `write(path, text)`,
`append`, `exists`, `is_file`, `is_dir`, `is_symlink`, `size`, `list_dir`,
`walk`, `create_dir`, `create_dir_all`, `remove_file`, `remove_dir`,
`copy`, plus path helpers `join`, `basename`, `dirname`.

## std.path

Path strings without touching the disk: `join`, `basename`, `dirname`,
`stem`, `extension`, `with_extension`, `components`, `normalize`,
`is_absolute`, `is_relative`.

## std.env

`args() -> list[str]`, `arg_at(i)`, `arg_count()`, `get_env(name)`,
`get_env_or(name, default)`, `has_env(name)`, `os_name()`, `arch()`,
`exit(code)`.

## std.process

`run(cmd, args) -> Result[Output, Error]` (`Output` has `code`, `stdout`,
`stderr`), `run_with_input`, `start` for a `Child`.

## std.time

`now() -> int` (seconds), `now_millis()`, `sleep_millis(ms)`,
`from_timestamp(secs) -> DateTime`, `format_timestamp`, `parse_timestamp`,
`weekday`; `Date`, `Time`, `DateTime` classes.

## std.json

`parse(text) -> Result[JsonValue, Error]`, `stringify(value)`. With
`@derive(ToJson, FromJson)` on a class, `x.to_json()` and `T.from_json(v)`.

## std.math

`sqrt`, `pow`, `exp`, `ln`, `log`, `log2`, `log10`, `sin`, `cos`, `tan`,
`asin`, `acos`, `atan`, `atan2`, `hypot`, `cbrt`, `floor`, `ceil`,
`trunc`, `round`, `fabs`, `fmod`, `pi()`, `e()`, `tau()`, `min`, `max`,
`clamp`, `abs`, `sign`, `gcd`, `lcm`, `pow_int`, `to_degrees`,
`to_radians`, `is_nan`, `is_inf`. `sqrt`, `floor`, `ceil`, `trunc`, and
`fabs` compile to single instructions.

## std.random

`Rng.new(seed)` or `Rng.from_entropy()`, then `next_int()`, `next_float()`,
`next_bool()`, `gen_range(a, b)`, `gen_range_float(a, b)`, `shuffle(xs)`,
`choice(xs)`, `sample(xs, k)`, `weighted_choice(xs, weights)`.

## std.fmt

`pad_left(s, width, fill)`, `pad_right`, `center`, `pad_int`, `join`,
`repeat`, `format_float(x, places)`, `to_hex`, `to_binary`, `to_octal`,
`to_radix`, `from_hex`, `from_radix`, and the `Debug` trait (derivable).
Interpolation format specs (`"${x:.2}"`) cover most formatting.

## std.regex

`compile(pattern) -> Result[Regex, Error]`, then `re.is_match(s)`,
`re.find(s)`, `re.find_all(s)`, `re.replace_all(s, with)`, `re.captures(s)`,
`re.split(s)`.

## std.encoding

`hex_encode`, `hex_decode`, `base64_encode`, `base64_decode`,
`base32_encode`, `base32_decode`, `url_encode`, `url_decode`.

## std.hash

`fnv1a`, `djb2`, `crc32`, `checksum`, `combine`, `hash_int`.

## std.cmp

`min`, `max`, `clamp`, `min_of`, `max_of`, `sort`, `sorted_by`. Classes get
ordering with `@derive(Ord)`.

## std.iter

Lazy adapters over an `Iterator[T]`: `from_list(xs)`, then `map`, `filter`,
`take`, `skip`, `enumerate`, and consumers `collect`, `sum`, `product`,
`count`, `fold`, `find`, `any`, `all`, `min`, `max`, `last`, `nth`,
`for_each`. List methods and comprehensions cover most uses.

## std.sync

`channel() -> Channel[T]`, `channel_buffered(n)`, with `ch.send(x)`,
`ch.recv()`; `select_recv`; `mutex()` with `lock()` and
`unlock()`; `wait_group()`; `yield_now()`, `sleep_millis`. Start work with
the `spawn { }` statement.

## std.net

`connect(addr) -> Result[TcpStream, Error]`, `listen(addr) -> TcpListener`
with `accept()`; a stream has `read()`, `read_all()`, `write(s)`, `close()`,
and read/write timeouts. Also `dns_lookup`, `reachable`.

## std.http

Client: `get(url)`, `post(url, body)`, `put`, `patch`, `delete`, `head`,
`request(...)`, `request_with_timeout`, each returning
`Result[Response, Error]` with `status` and `body`. Server:
`Server.new()`, `app.get("/path/:id", handler)`, `app.post(...)`,
`app.listen("127.0.0.1:8080")`; a handler is
`fun(req: Request) -> Response`, built with `Response.text(s)`,
`Response.html(s)`, `Response.json(value)` (any `ToJson`),
`Response.json_raw(s)`, `Response.not_found()`, `Response.with_body(code, s)`;
`req.param("id")`, `req.query_value("q")`, `req.header(name)`,
`req.json[T]()`. Also `app.put`, `app.delete`, `app.static(dir)`,
`with_timeout(ms)`, `shutdown()`.

## std.tls

`connect(host_port, server_name) -> Result[TlsStream, Error]`,
`connect_with(config, ...)`, `upgrade(tcp, server_name)` for STARTTLS,
`config()` for a `TlsConfig`. `TlsStream` has `read`, `read_all`,
`write`, `close`, and timeouts.

## std.ffi

`alloc[T](n) -> CPtr[T]`, `free`, `load(p)`, `store(p, v)`, `offset(p, i)`,
`null_ptr`, `is_null`, `to_cstr(s)`, `from_cstr(p)`, `free_cstr`.
Declare C functions in `extern "C" { ... }` and lay classes out with
`@repr(C)`.

## std.test

`assert_true`, `assert_false`, `assert_eq(a, b)`, `assert_ne`,
`assert_eq_int`, `assert_eq_str`, `assert_eq_float(a, b, eps)`,
`assert_msg(cond, msg)`, `assert_some`, `assert_none`, `assert_ok`,
`assert_err`. The built-in `assert cond, msg` statement needs no import.
Tests are `fun test_*()` in `*_test.rv` files, run by `rvpm test [NAME]`.

## std.error (prelude)

`Error` with `message()`, `kind()`, and `with_context(text)`; `error(msg)`, `error_kind(kind, msg)`,
and the `Result` helpers `ok`, `err`, `is_ok`, `is_err`, `map_ok`,
`map_err`, `unwrap_or`, `unwrap_or_else`.
