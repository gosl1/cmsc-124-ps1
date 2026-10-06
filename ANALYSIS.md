# ANALYSIS

## Question 1


## Question 2

**1. You wrote the tag check in dt_value_as_int by hand. Some languages don't let you. They make the tagged union a language construct, so the compiler writes the check for you, refuses to compile a read that skips it, and refuses to compile a set of cases that misses one. Rust's enum and match work this way, and so do ML's datatypes and Swift's enumerations with associated values. What does the C version let you do that a compiler enforcing the check wouldn't, and is any of it worth wanting?**

The C version lets you skip the check entirely, I could delete the if in dt_value_as_int, or read v.as.integer from a string, and the code would still compile. The if and DT_ERR_TAG is something that is needed to be added by hand, not something C requires. In C the tag check is a convention, not a requirement. The if in dt_value_as_int that returns DT_ERR_TAG is code we chose to write. We could delete it, or read `v.as.integer` from a string value, and the program would still compile. C also lets us build a value whose tag doesn't match its payload, and switch on a tag without covering every case.
 

**Example for C**

- For example, I could delete the "if" statement in dt_value_as_int and C would still compile without error.
- Given the case `as int "sixseven"`, this asks for the int but the value is a string, so the tag check here would fail, meaning it would go to DT_ERR_TAG.
- If the "if" statement were removed, C would just read the memory address of that string, let's say the address is 6767, and it would return that number as a normal result without error.
- The value holds a string, so `v.as` contains a pointer, the address of where the letters are stored.
- `*out = v.as.integer` reads those same bytes as a number, so the result isn't a number made from the LETTERS. It's a number made from the ADDRESS of the string, which is 6767, returned with DT_OK and no error.
- What you get here is just a meaningless number that "looks" valid, and the program carries on as if it were right.

Some other languages would not need to manually create a condition to read the tag because compilers already check the tag for you. Rust's match works this way. For example, given the Rust version of the code:

```rust
match v {
    Value::Int(n) => Some(*n),
    _ => None,
}
```
Rust cannot stop `as int "sixseven"` from reaching this function, because the tag is only known at runtime. The match compiles and returns None for the string. What the compiler enforces is that we cannot write a version that reads the integer without matching on the tag first, the payload `n` only exists inside the `Int`. If a `match` over all the kinds leaves one out, the compiler rejects it until it is handled. C is not entirely silent here but it is only a warning, and a `default:` label silences it.

The unchecked read is not purely a defect, because the same mechanism makes several techniques possible. It allows type punning, such as reading a float's bits as an integer in order to hash or serialize it. It allows NaN-boxing and pointer tagging, where some Lua and JavaScript engines pack a type tag and a payload into a single 64-bit word, a layout that a language enforcing a separate tag field can only reproduce through escape hatches like Rust's `unsafe`. And it allows skipping a redundant check in a hot loop once the tag has already been validated. The enforced version has costs of its own. A checked match is heavy when we only care about one variant, and Rust's recursive types need a Box, which adds an allocation and a pointer hop that a C struct avoids.

**Is it worth wanting?**

There are tradeoffs between the two implementations, but I think C's freedom isn't worth wanting here, because I don't want to read the wrong value on purpose, and the check is what keeps `as int "sixseven"` safe. C lets you read a union without checking the tag, build values with a mismatched tag, and skip cases in a switch, and the compiler allows it. That freedom is only useful in low-level work where you need raw bytes, tiny memory, or maximum speed.

In this problem set, it would just create bugs without the check, `as int "sixseven"` returns the string's address as a number with DT_OK and no error. We'd rather the compiler write the check, and keep C's freedom for low-level work where raw bytes or memory matter more than safety.


## Question 3

**Your dt_map keeps insertion order separately from the hash buckets, which is memory spent on something no lookup uses. Argue the other side: describe a design that drops it, say what breaks, and say whether you'd ship it.**

In the dt_map implementation, no lookup ever touches the order list. It costs extra memory, one pointer per key and it makes put and remove do extra work, because every entry has to be added to or removed from two places.

The map has the order list, so listing the keys is easy. You go down the list: first key, second key, third key. That's what `dt_map_key_at(m, 0)`, `dt_map_key_at(m, 1)` and so on do.

In our dt_map implementation, specifically in the "find_entry" function, it uses the hash to jump to one bucket, then follows that bucket's chain looking for one specific key. 

**Alternative design:** Drop the order list and use the buckets only.

In this design, the iteration would traverse the buckets and their chains directly. The only place the entries live is the buckets. So to list every key, you'd have to visit every bucket, one after another, and inside each bucket, follow its chain from the first entry to the last. For example:

```c
for (size_t b = 0; b < m->bucket_count; b++) {
    for (dt_map_entry *e = m->buckets[b]; e != NULL; e = e->next) {
    }
}
```

It visits every bucket and walks every chain. It's a listing of all the keys. This code snippet would be the starting point for `dt_map_key_at` if the order list were gone.

**What breaks:** The iteration will not follow the insertion order anymore. The output order would depend on the hash function and bucket layout, so adding entries can show up anywhere.

With the order list, print.c calls `dt_map_key_at(m, 0)`, then 1, then 2, and gets the keys in the order they were added:

```
{"alpha" -> 1, "beta" -> 2, "gamma" -> 3}
```

With buckets only, the only way to list the keys is to visit bucket 0, then 1, then 2, and so on up to 15. That finds beta (bucket 7) first, then gamma (bucket 10), then alpha (bucket 11). Note: bucket values are only an example.

```
{"beta" -> 2, "gamma" -> 3, "alpha" -> 1}
```

**Additional note:** A removed key that is put back again goes to the end. With buckets only, that key would go back into its same bucket and show up in the same spot as before, because the computation of the hash stays the same.

**The tradeoff:**
Dropping the order list makes the map simpler, since it saves a pointer per key and put and remove only update the buckets. The cost is that the map loses insertion order, keys come out in the order the hash placed them, `dt_map_key_at(i)` has to walk the buckets and count each time.

So buckets-only is better when order never matters, like a cache or a word counter, where keys are only looked up and never printed. The order list is better when the order is visible, like printing a map or comparing test output exactly. The thing to note here is that the order list costs a little memory and some extra work in put and remove, but it gives output in insertion order and fast access.

**Would I ship it?**

No, I would not ship it. Although insertion order is not needed for lookup, it is better to see the map's behavior in printing the output. `print.c` lists the keys in the order they were added, and the tests compare the output exactly, so without the order list the keys would come out in hash order, and the test case `normal/map_basics` would fail.

The cost of keeping it is minimal, it's just one extra pointer per key, plus some extra work in the put remove functions. For us, that is already a reasonable cost for iteration in insertion order, and it does not really slow down lookup, because lookup only uses the buckets. I would only drop the order list for a map that is used purely for lookup and never printed or listed.

Python shows the same tradeoff. Python's dict keeps insertion order. Before 3.7 the order was not guaranteed, but programmers relied on it anyway, so Python made it part of the language. Python also shows we can get that order without a second linked list, it stores entries in an array in the order they were added, and a small table maps each hash to a position in that array. Lookup still takes two quick steps, printing just reads the array in order, and key_at(i) is simply array[i]. The tradeoff is that deleting is harder, since removing an entry leaves a gap that has to be tracked and cleaned up later.

## Question 4

