Zig's arrays and slices can be confusing to people coming from other languages.  This primer explains the differences and compares to equivalents in C.

In C we use the word "array" for a sequence of same-sized items laid out sequentially in memory.  A pointer to the first item is the same as a pointer to the array.  The length or size of the array is either stored in a separate variable, or indicated by a sentinel value, usually 0 (null).

Zig has a few different ways of talking about an "array" sequence depending on what the zig compiler and type system knows about it.

## Pointers
Pointers are split into 2 types:
- `p: *T` - pointer to a single T, not a sequence
  - can only dereference `p.*`
- `p: [*]T` - pointer to a sequence of T of unknown length
  - can index `p[i]`
  - can slice `p[i..n]`
  - can pointer math `p + x`

## Slices
Slice - combines a pointer to a sequence with a length.
`[]T`
is very much like
```
struct {
  ptr: [*]T,
  len: usize,
}
```
- `s: []T` - sequence of T of runtime-known length
  - can index `s[i]`
  - can slice `s[i..n]` or `s[i..]`
    - if slice length is comptime-known, returns a pointer to an array (coerces to a slice)
    - if slice length is runtime-known, returns a slice
  - can copy data from another slice `s[i..n][1..3].* = b[j..m][2..5].*`
    - slice length must be comptime-known, giving a pointer to an array which requires the pointer dereference
  - can get pointer `s.ptr` with type `[*]T`
  - can get length `s.len`
  - can iterate `for (s) |x, i| {}`
  - can get by coercing from a pointer to an array `s = &arr` or `s = arr[0..]`
  - duplicating the slice `var s2 = s` does not copy the data
- `s: [:X]T` - sequence of T of known length plus sentinel value X after
  - `s[s.len] == X`
  - use for string interop with C

Slices are used in most of the places you would normally have an "array" in C.  They combine pointer and length so you don't need a separate variable.

Allocating memory returns a slice:
- `var s: []T = allocator.alloc(T, n)`
- `var s: [:X]T = allocator.allocSentinel(T, n, X)`

A struct with a slice field only holds the slice pointer and length.  The pointed-to data is not stored in the memory of the struct.

Experimenting with slices can be confusing if you are starting with arrays.  Try starting with this:
```
const std = @import("std");

var gpa_instance = std.heap.GeneralPurposeAllocator(.{}){};
const gpa = &gpa_instance.allocator;

pub fn main() !void {
  var s = try gpa.alloc(u8, 5);
  std.debug.print("s type {}\n", .{@TypeOf(s)});
}
```

## Arrays
Array - similar to a slice but length is known at compile time.  A pointer to an array is very similar to a slice and can be coerced to a slice.
- `a: [N]T` - sequence of T of length N
  - can index `a[i]`
  - can get length `a.len`
  - can iterate `for (a) |x, i| {}`
  - can slice `a[i..n]` or `a[i..]`
    - if slice length is comptime-known, returns a pointer to an array (coerces to a slice)
    - if slice length is runtime-known, returns a slice
  - can copy data from another array `a[1..3].* = b[2..5].*`
    - slice length must be comptime-known, giving a pointer to an array which requires the pointer dereference
- `a: [N:X]T` - sequence of T of length N plus sentinel value X after
  - `a[a.len] == X`

Array Literals
- `var a = [5]u8{'h', 'e', 'l', 'l', 'o'};`
  - @TypeOf(a) == [5]u8
- `var a = [_]u8{'h', 'e', 'l', 'l', 'o'};`
  - @TypeOf(a) == [5]u8
  - length inferred from literal
- `var a = [_:0]u8{'h', 'e', 'l', 'l', 'o'};`
  - @TypeOf(a) == [5:0]u8
- `var a: [100]u8 = undefined;`
  - use with `std.fmt.bufPrint` and `std.fmt.bufPrintZ` to create strings at runtime
- `var buf = std.mem.zeroes([100:0]u8);`
  - stack allocated zeroed buffer

A struct with an array field holds all the array elements in the memory of the struct.


## Working with strings
- string literals are type `*const [N:0]u8` (pointer to a constant array of bytes with terminating null byte) for easier interop with C
  - can get array `str.*`
  - can slice `str[i..n:0]` gives type `[:0]const u8`
  - can slice `str[i..n]` gives type `[]const u8` (drop sentinel from type)
  - coerce to `[]const u8` (drop sentinel)
  - coerce to `[:0]const u8` (can pass to C)
  - coerce to `[*:0]const u8` (forget length, can pass to C)
- Function that accepts strings and string literals
  - `fn foo(str: []const u8) void {}`
  - `fn foo(str: [:0]const u8) void {}`
  - `fn foo(str: [*:0]const u8) void {}`
- Create strings at runtime:
```
var buf: [100]u8 = undefined;
var buf_slice: [:0]u8 = try std.fmt.bufPrintZ(&buf, "a {d} and a {d}", .{1, 2});
std.debug.print("buf_slice \"{s}\"\n", .{buf_slice});
```
