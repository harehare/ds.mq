<h1 align="center">ds.mq</h1>

> [!WARNING]
> This project is currently under active development. APIs and features may change without notice.

> [!NOTE]
> This module is tuned for the development version of mq, which is not released yet: the heap relies on `slice` cutting an unshared array in place. Released versions give the same results but copy the heap on every `pop`.

Data structures for [mq](https://github.com/harehare/mq): a binary heap, a trie, a union-find, a deque and a sorted list, written as an mq module.

## Modules

| Module | Use it for |
| --- | --- |
| `ds::heap` | priority queues, top-k selection |
| `ds::trie` | prefix lookup, longest-match scanning of text |
| `ds::unionfind` | grouping items by pairwise links |
| `ds::deque` | queues, breadth-first search, sliding windows |
| `ds::sorted` | ordered sets, nearest-key lookup, range queries |

## Installation

Copy `ds.mq` to your mq module directory, or place it anywhere and reference it with `-L`.

```sh
cp ds.mq ~/.local/mq/config/
```

HTTP imports are disabled by default; pass `--allow-http-import` to import directly from GitHub:

```sh
mq --allow-http-import -I null 'import "github.com/harehare/ds.mq" | ds::heap::largest([5, 1, 9, 3], 2)'
```

## Usage

### Heap

```sh
mq -L . -I null '
  import "ds"
  | let h = ds::heap::from([5, 3, 8, 1], false)
  | let r = ds::heap::pop(h)
  | r[0]
'
# 1
```

| Function | Description |
| --- | --- |
| `new()` / `new_max()` | empty min-heap / max-heap |
| `from(arr, max = false)` | heap from an array in O(n) |
| `push(h, x)` | add an item |
| `pop(h)` | `[root, heap]`, fails when empty |
| `peek(h)` | the root, or `None` |
| `size(h)` | number of items |
| `to_sorted(h)` | items in pop order |
| `largest(arr, k)` / `smallest(arr, k)` | top-k with a heap bounded by `k` |

Items are compared with `<` and `>`, so pairs such as `[priority, value]` order by priority first.

### Trie

```sh
mq -L . -I null '
  import "ds"
  | let t = ds::trie::put(ds::trie::put(ds::trie::new(), "car", 1), "card", 2)
  | [ds::trie::lookup(t, "card"), ds::trie::words(t, "ca"), ds::trie::longest_match(t, "cards", 0)]
'
# [2, ["car", "card"], 4]
```

| Function | Description |
| --- | --- |
| `new()` | empty trie |
| `insert(t, word)` / `put(t, word, value)` | store a word, optionally with a value |
| `contains(t, word)` / `lookup(t, word)` | membership / stored value |
| `has_prefix(t, prefix)` | whether any word starts with `prefix` |
| `words(t, prefix)` | stored words under `prefix`, sorted |
| `longest_match(t, s, start)` | end of the longest word starting at `start` in `s`, or `None` |

### Union-find

```sh
mq -L . -I null '
  import "ds"
  | ds::unionfind::groups(ds::unionfind::from_pairs(6, [[0, 2], [1, 4], [2, 5]]))
'
# [[0, 2, 5], [1, 4], [3]]
```

| Function | Description |
| --- | --- |
| `new(n)` | `n` singleton sets over `0..n-1` |
| `union(uf, x, y)` | merge two sets |
| `find(uf, x)` / `connected(uf, x, y)` | set root / same-set test |
| `size_of(uf, x)` / `count(uf)` | size of one set / number of sets |
| `groups(uf)` | the sets as arrays of members |
| `from_pairs(n, pairs)` | build from `[x, y]` links |

### Deque

```sh
mq -L . -I null '
  import "ds"
  | let d = ds::deque::push_front(ds::deque::from([2, 3]), 1)
  | ds::deque::to_array(d)
'
# [1, 2, 3]
```

| Function | Description |
| --- | --- |
| `new()` / `from(arr)` | empty deque / deque holding an array, first item at the front |
| `push_back(d, x)` / `push_front(d, x)` | add at one end |
| `pop_front(d)` / `pop_back(d)` | `[item, deque]`, fails when empty |
| `front(d)` / `back(d)` | end items, `None` when empty |
| `get(d, i)` | item `i` counted from the front |
| `size(d)` / `is_empty(d)` | number of items / emptiness test |
| `to_array(d)` | the items in order |

### Sorted list

```sh
mq -L . -I null '
  import "ds"
  | let s = ds::sorted::insert(ds::sorted::from([40, 10, 30]), 20)
  | [s, ds::sorted::ceiling(s, 25), ds::sorted::between(s, 15, 35)]
'
# [[10, 20, 30, 40], 30, [20, 30]]
```

A sorted list is a plain array in ascending order, so `len(s)` and `s[i]` work on it directly.

| Function | Description |
| --- | --- |
| `new()` / `from(arr)` | empty list / sorted copy of an array |
| `insert(s, x)` / `remove(s, x)` | add an item / drop one equal item |
| `contains(s, x)` | membership test |
| `lower_bound(s, x)` / `upper_bound(s, x)` | position of the first item `>= x` / `> x` |
| `ceiling(s, x)` / `floor_item(s, x)` | nearest item `>= x` / `<= x`, `None` if there is none |
| `between(s, lo, hi)` | items from `lo` up to but not including `hi` |
| `first(s)` / `last(s)` / `get(s, i)` / `size(s)` | ends, item at a position, number of items |

## Updating in place

Every structure is a plain array or dict, and each operation returns the updated value. Assign the result back to the variable you passed in, and do not read the old value afterwards:

```
h = ds::heap::push(h, x)
```

The old value is then released and the update happens in place. If the old value is still referenced, mq copies the whole structure first, which makes each operation O(n).

## Complexity

| Operation | Cost |
| --- | --- |
| heap `push`, `pop` | O(log n) |
| heap `from` | O(n) |
| heap `largest`, `smallest` | O(n log k) |
| trie `put`, `contains`, `has_prefix` | O(length of the word) |
| trie `words` | O(matches) |
| union-find `find`, `union` | O(log n) |
| deque push, pop, `get`, `front`, `back` | O(1) amortized |
| deque `from`, `to_array` | O(n) |
| sorted `contains`, `lower_bound`, `ceiling`, `floor_item` | O(log n) |
| sorted `insert`, `remove` | O(n) |

## Development

```sh
mq-test ds_tests.mq
mq-bench -L . ds_bench.mq
```

Each `_small` / `_large` pair in `ds_bench.mq` grows one structure at two sizes. A ratio close to the size ratio means updates run in place.
