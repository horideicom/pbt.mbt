# horideicom/pbt

Property-based testing examples for MoonBit using `@quickcheck` with a small
local harness.

- Generators: `pbt_gen.mbt`
- Shrinkers: `pbt_shrink.mbt`
- Runner: `pbt_check.mbt`
- Examples: `codec.mbt`, `stack.mbt`, `pbt_property_test.mbt`

## Run tests

```
moon test
```

## README examples (tested)

```mbt check
///|
test "readme codec roundtrip" {
  let gen = Gen::array_of(Gen::choose_int(-10, 10))
  let config = CheckConfig::new(50, 12, 7, 20)
  let result = check(
    gen,
    fn(values) {
      let encoded = encode_int_list(values)
      match decode_int_list(encoded) {
        Ok(decoded) => if decoded == values { Ok(()) } else { Err("mismatch") }
        Err(err) => Err(err)
      }
    },
    config~,
    shrink=shrink_array,
  )
  guard result is Ok(_) else { fail("\{result}") }
}
```

```mbt check
///|
test "readme stack commands" {
  let cmds : Array[Command] = [Push(1), Push(2), Top, Pop, Size, Clear]
  let result = run_stack_commands(cmds)
  guard result is Ok(_) else { fail("\{result}") }
}
```

### Gen::sized

```mbt check
///|
test "readme gen sized" {
  let gen = Gen::sized(fn(size) { Gen::pure(size) })
  let rs = @quickcheck/splitmix.new(seed=0)
  let small = (gen.run)(0, rs)
  let large = (gen.run)(5, rs)
  assert_eq(small, 0)
  assert_eq(large, 5)
}
```

### Gen::frequency

```mbt check
///|
test "readme gen frequency" {
  let gen = Gen::frequency([(0, Gen::pure("skip")), (1, Gen::pure("keep"))])
  let rs = @quickcheck/splitmix.new(seed=1)
  let sample = (gen.run)(10, rs)
  assert_eq(sample, "keep")
}
```
