## tl;dr

Learnt Rust to implement an order book and compare against another implementation in C++17. 

## Lightning talk

Gave a 5-minute talk about this project at the London Rust meetup. 

[Slides](https://docs.google.com/presentation/d/e/2PACX-1vShyqTQMgiZyg7GpxN5cqOqKM-cLAVhvymcDQFCp4gRcLubBz7OuoL3houVt_HdDmsCbOxMbF3KbWyl/pub?start=false&loop=false&delayms=3000)

## Requirements

  * Rust 1.27
  * fnv 1.0.6
  * Cargo

## Test and run

Git clone and change into the directory, before running the command below. The first argument is the target size of the order book. After that the executable will wait for market data feed on stdin. Conveniently packaged in a shell script.

```bash
cargo test
cargo build --release
cargo run --release <target_size> < data/<market_data_file>
```

Test harness from the problem statement. Writes output to a tmp file and compares to expected output. 

```bash
./run_basic_test.sh
```

## Design

The order book allows adding new orders, reducing current ones and printing the amount earned from selling <target\_size> of shares or amount spent on buying <target\_size> of shares.

```rust
type Depth = i64;

struct OrderBook<T: IdPriceCache + Sized> {
    cache: T,
    bids_at_price: BTreeMap<Amount, Depth>,
    bids_total_size: i64,
    asks_at_price: BTreeMap<Amount, Depth>,
    asks_total_size: i64,
    target_size: i64,
    // only 1 side is affected on Reduce or Limit order
    last_action_side: OrderSide,   // which side was touched last
    last_action_timestamp: String, // timestamp of last touched side
}
```

## API

### Adding a new limit order

Parse the price point (input as float) and quantity (relevant optimisation below). Find the relevant order side and insert the key-value pair of price - depth. Update the last action side and last action timestamp as well. 


### Reducing an order

Using the order id, look up in cache, the side and price of the order. Find the relevant bucket inside the ordered map of the given side, decrement the depth of the bucket.

### Checking and reporting

To check if the order book needs to report income/expense, you need to see if the last affected side now has total depth more than target_size. Only if it does, do we calculate the amount. 

Keeping total depth per price, gives us a shortcut to quickly calculate how much we can make/spend on each bucket as `size * price`.

### Storage

Orders are stored in:

  * Cache to look up price and side by order id
  * Ordered map (BTreeMap) of prices to depths.

## Motivation

Inspired by [Ludwig Pacifici's implementation using C++17](https://github.com/ludwigpacifici/order-book-pricer), I decided to learn Rust and implement an order book.

### Implemented perf improvements

Benchmarking my first implementation against Ludwig's C++17 version showed that my design was terrible. Performance optimisations that I made (chronological order):

  0. Realised this from the start - store float prices as ints. FP arithmetic is more CPU-intensive. For printing - implemeted own Display trait turns int into a string and prints the string with a separator <decimal_points> chars away from the right-hand side. Input: 44.25, stored as 4425, printed as "44.25".

  1. First implementation stored full limit order structs in Linked Lists in BTreeMaps. Linked list nodes were heap-allocated and blew the cache efficiency of my algrorithm. Ultimatelly, it's not necessary to keep the exact order. I now use the BTreeMap as a key value store between price point and depth of order book at that price point.

  2. After running `collect_perf` and `perf report`, I found that println! was taking 8.96% of time. Googling for efficient stdout printing in Rust suggested replacing println! with writeln! with a stdout lock as one of the args. 

  3. Replaced Strings for timestamps with int64. Strings are heap-allocated, require malloc and free. Ints should be faster to allocate. Updated the benchmark.

  4. Since order IDs aren't required for stdout, we don't need to keep the string representation of each order id. I implemented a hash function (using a FNVHasher) and changed order id from `String` to `u64` in ReduceOrder and LimitOrder. Also changed the IdPriceCache signature to make sure cache looks `hash(id)` rather than `id: String`. 

```bash 
./time_rust_pricer.sh
...
real	0m1.712s
user	0m1.590s
sys	0m0.116s
```

  5. Enabled LTO and added compile-time detail to heap-allocated data structures that expand at runtime. Calling `malloc` often will increase time spent in kernel-space. Given that we know the size of input data, we can call malloc at application start-up, request a lot of memory at once to reduce future calls for additional memory. The trade-off between requesting too much memory at start-up that you will never need vs. calling malloc for every expansion of BTreeMap can be investigated with different input sizes. 
  
```bash 
./time_rust_pricer.sh
...
real	0m1.643s
user	0m1.532s
sys	0m0.104s
```

### Potential perf improvements - yet to be investigated

1. Wherever feasible replace `String` with `&str`. Consider the lifetime of strings like order ID and order timestamps and implement an efficient way instead of using `to_string()` and `clone()`, which defeat the advantage of rust. 

eg. Order ID is only used in input and inside the order book, no need to print it back out. It might be more efficient to convert/cast string into a i32 value and use that, wherever order ID is used. 

Pros: 

  * most of my strings are read-only - should work well and reduce memory usage.
  * Learn about lifetimes and borrowing in Rust

Cons: 
  * learning about said lifetimes and borrowing will pit me against the infamous borrow checker.

2. Currently - reducing an order into oblivion (eg. reduce an order of size 100, by >100) doesn't remove its key from the IdPriceCache. This leads to higher memory usage, if unused keys persist in the cache. It might be useful to remove the key-value pair, if the order is ever completely reduced. 

Requires: 

  * Adding order size to the IdOrderPrice cache and decrementing it after every reduce. 
  * Turning OrderBook.reduce() into reducing 2 internal states - not a pretty abstraction.

Pros:

  * if a lookup of previously-deleted key occurs, we can end that branch of logic quickly. Unlikely to occur - clients shouldn't ask to reduce the same order twice.
  * Prevents the BTreeMap from growing too much. Shouldn't matter too much, but on big applications, it's worth preserving heap space for ids with valid data.

3. Check if using a vector for bids and asks is better than a BTreeMap. Perf shows BTreeMap iterators to be one of the most expensive parts of the code and if the vector is cheaper to rewrite in practice - use the vector for cache locality.

