# Raven built-in functions

These functions are always in scope without an `import`. They are implemented in Rust inside the interpreter (see `src/code_gen.rs`). All signatures use Raven's notation: `name(param: type, ...) -> return_type`.

## I/O

### `print(template: string, args...) -> void`
Prints `template` to stdout followed by a newline. `{}` placeholders are replaced left-to-right by `args`. With a single argument and no `{}`, prints the value as a string.

```raven
print("hello");
print("x = {}, y = {}", 1, 2);   // "x = 1, y = 2"
print(format("answer: {}", 42)); // single string arg
```

There are no `{:d}`/`{:.2f}` style specifiers. All values stringify with their default representation.

### `format(template: string, args...) -> string`
Same substitution as `print` but returns the resulting string. The most ergonomic way to coerce an int/float/bool/struct into a string.

```raven
let s: string = format("{} + {} = {}", 1, 2, 3);
```

### `input(prompt: string) -> string` (or `input() -> string`)
Prints the prompt (no newline), reads a line, returns it without the trailing newline.

## Lengths and types

### `len(value) -> int`
Length of an array (number of elements) or a string (number of characters).

### `type(value) -> string`
Type name as a string. Useful for debugging. Returns `"int"`, `"string"`, struct or enum name, `"array"`, `"void"`, etc.

## Conversions

### `parse_int(s: string) -> int`
Parses `s` as a 64-bit integer. **Returns 0** on failure — no exception, no `Option`. If you need to distinguish "the string was 0" from "parse failed", check the input first.

### `char_code(s: string) -> int`
ASCII code of the first character in `s`. `char_code("A")` is 65. Empty string returns 0.

There is no inverse `int_to_char` built-in. To produce a single character, use a lookup table or `format("{}", n)` for the digit form.

## Files

### `read_file(path: string) -> string`
Returns the entire file as a string. If the file doesn't exist, the call errors out (fatal — no exception handling exists in Raven). Always guard with `file_exists` first.

### `write_file(path: string, content: string) -> void`
Writes `content` to `path`, overwriting if it already exists. Backslash-`n` sequences in `content` (`"line1\\nline2"`) are converted to actual newlines.

### `append_file(path: string, content: string) -> void`
Appends to (or creates) `path`. Same escape handling as `write_file`.

### `file_exists(path: string) -> bool`
True if a file or directory exists at `path`.

### `is_dir(path: string) -> bool`
True if `path` is a directory.

### `list_directory(path: string) -> string[]`
Names of entries in the directory. Empty array if the directory is empty or doesn't exist.

### `create_directory(path: string) -> bool`
Creates the directory and any missing parents. Returns true on success, false on error.

### `remove_file(path: string) -> bool`
Deletes a file. Returns false if it doesn't exist.

### `remove_directory(path: string) -> bool`
Recursively deletes a directory (everything inside). Use carefully.

### `get_file_size(path: string) -> int`
File size in bytes. Returns 0 if the file is missing.

## Time

### `sys_time() -> string`
Current local time as `"HH:MM:SS"`.

### `sys_date() -> string`
Current local date as `"YYYY-MM-DD"`.

### `sys_timestamp() -> float`
Unix timestamp in seconds, with sub-second precision. Useful as a cheap source of entropy: take the fractional part as a pseudo-random integer.

## Networking

### `http_fetch(method: string, url: string, headers: string[], body: string) -> HttpResponse`
Synchronously perform an HTTP request. `method` is `"GET"`, `"POST"`, etc. `headers` is an array of `"Header-Name: value"` strings. `body` is the request body (use `""` for GET).

The response struct:

```raven
struct HttpResponse {
    status_code: int,
    status_text: string,
    headers: string[],
    body: string,
}
```

### `tcp_listen(addr: string, backlog: int) -> TcpListener`
Bind a TCP listener. `addr` is `"host:port"`, e.g. `"127.0.0.1:8080"`. `backlog` is the connection queue depth.

### `tcp_accept(listener: TcpListener) -> TcpStream`
Block until a connection arrives.

### `tcp_read(stream: TcpStream, max_bytes: int) -> string`
Read up to `max_bytes` from the stream.

### `tcp_write(stream: TcpStream, data: string) -> void`
Write `data` to the stream.

### `tcp_close_stream(stream: TcpStream) -> void`
Close one client connection.

### `tcp_close_listener(listener: TcpListener) -> void`
Stop accepting new connections.

### `dns_lookup(host: string) -> string`
Resolve a hostname to an IP string.

### `reachable(host: string) -> bool`
Quick reachability ping.

## Enums

### `enum_from_string(enum_name: string, variant_name: string) -> Enum`
Construct an enum value from two strings. Errors at runtime if the enum or variant doesn't exist.

```raven
enum Status { OK, Error }
let s: Status = enum_from_string("Status", "OK");
```

## Errors

### `panic(message: string, args...) -> never`
Concatenate the arguments into an error message and abort the program. There is no `try`/`catch` in Raven — `panic` ends execution immediately.

## String methods (built into the language)

These behave like methods on string values. They're not user-defined functions but the parser/interpreter knows them.

| Method                              | Returns       | Notes                                            |
| ----------------------------------- | ------------- | ------------------------------------------------ |
| `s.slice(start: int, end: int)`     | `string`      | Substring `[start, end)`                         |
| `s.split(sep: string)`              | `string[]`    | Split on every occurrence of `sep`               |
| `s.replace(from: string, to: string)` | `string`    | Replace all occurrences                          |
| `s.index_of(sub: string)`           | `int`         | -1 if not found                                  |
| `s.last_index_of(sub: string)`      | `int`         | -1 if not found                                  |
| `s.contains(sub: string)`           | `bool`        |                                                  |
| `s.starts_with(p: string)`          | `bool`        |                                                  |
| `s.ends_with(p: string)`            | `bool`        |                                                  |
| `s.to_upper()`                      | `string`      |                                                  |
| `s.to_lower()`                      | `string`      |                                                  |
| `s.trim()`                          | `string`      | Strips leading and trailing whitespace           |

Strings are immutable — every method returns a new string. Indexing a string with `s[i]` returns a single-character string.

## Array methods (built-in)

| Method                                | Returns        | Notes                                             |
| ------------------------------------- | -------------- | ------------------------------------------------- |
| `arr.push(value)`                     | array          | Mutates `arr` when called on a local variable     |
| `arr.pop()`                           | element type   | Removes and returns the last element; errors if empty |
| `arr.slice(start: int, end: int)`     | array          | Sub-array `[start, end)`                          |
| `arr.join(sep: string)`               | `string`       | Only meaningful for `string[]`                    |

Mutation caveat: pushing to an array **received as a function parameter** does not always propagate back to the caller. The reliable pattern is to build the array locally and `return` it. See SKILL.md "Array mutation pitfall".
