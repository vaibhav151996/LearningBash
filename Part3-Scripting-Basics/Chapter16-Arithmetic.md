# Chapter 16: Arithmetic Operations

## Learning Objectives

By the end of this chapter, you will be able to:

- Perform arithmetic in Bash using multiple methods
- Understand the differences between `$((...))`, `let`, and `expr`
- Work with integer arithmetic (Bash doesn't do floating-point!)
- Use arithmetic for practical scripting tasks
- Handle floating-point math when needed (using `bc` or `awk`)

---

## 16.1 Arithmetic Expansion `$(( ))`

The primary way to do math in Bash. This is the recommended method.

```bash
# Basic operations
echo $((5 + 3))       # 8
echo $((10 - 4))      # 6
echo $((6 * 7))       # 42
echo $((20 / 3))      # 6  (integer division — truncates!)
echo $((20 % 3))      # 2  (modulo — remainder)
echo $((2 ** 10))     # 1024  (exponentiation)

# With variables (no $ needed inside $(( )))
x=10
y=3
echo $((x + y))       # 13
echo $((x * y))       # 30
echo $((x / y))       # 3
echo $((x % y))       # 1

# Assignment
result=$((x + y * 2))
echo $result          # 16  (multiplication before addition)

# Increment and decrement
count=0
((count++))
echo $count           # 1
((count += 5))
echo $count           # 6
((count--))
echo $count           # 5
((count *= 2))
echo $count           # 10
```

### Supported Operators

| Operator | Operation | Example | Result |
|----------|-----------|---------|--------|
| `+` | Addition | `$((5 + 3))` | 8 |
| `-` | Subtraction | `$((10 - 4))` | 6 |
| `*` | Multiplication | `$((6 * 7))` | 42 |
| `/` | Integer division | `$((7 / 2))` | 3 |
| `%` | Modulo (remainder) | `$((7 % 2))` | 1 |
| `**` | Exponentiation | `$((2 ** 8))` | 256 |
| `++` | Increment | `((x++))` | |
| `--` | Decrement | `((x--))` | |
| `+=` | Add and assign | `((x += 5))` | |
| `-=` | Subtract and assign | `((x -= 3))` | |
| `*=` | Multiply and assign | `((x *= 2))` | |
| `/=` | Divide and assign | `((x /= 4))` | |

### Bitwise Operators

```bash
echo $((5 & 3))       # 1   (AND)
echo $((5 | 3))       # 7   (OR)
echo $((5 ^ 3))       # 6   (XOR)
echo $((~5))          # -6  (NOT)
echo $((1 << 4))      # 16  (left shift)
echo $((16 >> 2))     # 4   (right shift)
```

### Comparison in Arithmetic Context

Inside `(( ))`, comparisons return 1 (true) or 0 (false):

```bash
# Use (( )) for numeric conditions
if (( x > 5 )); then
    echo "x is greater than 5"
fi

if (( count == 0 )); then
    echo "Count is zero"
fi

# Ternary operator
a=10; b=20
max=$(( a > b ? a : b ))
echo $max    # 20
```

---

## 16.2 The let Command

`let` evaluates arithmetic expressions. It's an alternative to `(( ))`:

```bash
let "x = 5 + 3"
echo $x              # 8

let "x += 10"
echo $x              # 18

let "x = x * 2"
echo $x              # 36
```

`(( ))` is generally preferred over `let` for readability.

---

## 16.3 The expr Command (Legacy)

`expr` is an external program for arithmetic. It's older and slower. Prefer `$(( ))`.

```bash
# External command — note the spaces!
result=$(expr 5 + 3)
echo $result         # 8

# Multiplication requires escaping the *
result=$(expr 6 \* 7)
echo $result         # 42
```

Use `expr` only when you need POSIX compatibility with very old shells.

---

## 16.4 Integer-Only Arithmetic

**Bash only supports integer arithmetic.** Division truncates:

```bash
echo $((7 / 2))      # 3  (not 3.5!)
echo $((1 / 3))      # 0  (not 0.333!)
```

---

## 16.5 Floating-Point Math with bc

For floating-point calculations, use `bc` (basic calculator):

```bash
# Simple calculation
echo "7 / 2" | bc -l          # 3.50000000000000000000

# Scale (decimal places)
echo "scale=2; 7 / 2" | bc    # 3.50

# Store result
result=$(echo "scale=4; 22 / 7" | bc)
echo $result                    # 3.1428

# Complex expressions
echo "scale=2; sqrt(144)" | bc -l   # 12.00
echo "scale=10; 4*a(1)" | bc -l     # 3.1415926535 (pi)

# Multiple calculations
bc -l << EOF
scale = 4
pi = 4 * a(1)
radius = 5
area = pi * radius^2
area
EOF
# 78.5396
```

### Floating-Point with awk

```bash
# awk also handles floating-point
awk "BEGIN {printf \"%.2f\n\", 7/2}"        # 3.50
awk "BEGIN {printf \"%.4f\n\", 22/7}"       # 3.1429

# Using variables
x=7; y=2
awk -v x="$x" -v y="$y" 'BEGIN {printf "%.2f\n", x/y}'  # 3.50
```

---

## 16.6 Practical Arithmetic Examples

### Random Numbers

```bash
# RANDOM generates 0-32767
echo $RANDOM                        # Random number

# Random number in a range (0 to N-1)
echo $((RANDOM % 100))              # 0-99

# Random number in a range (min to max)
min=10; max=50
echo $((RANDOM % (max - min + 1) + min))    # 10-50
```

### Percentage Calculation

```bash
used=15360    # MB
total=51200   # MB
percent=$((used * 100 / total))
echo "Disk usage: ${percent}%"       # Disk usage: 30%
```

### Loop Counter

```bash
#!/bin/bash
count=0
for file in /var/log/*.log; do
    ((count++))
done
echo "Found $count log files"
```

### Converting Units

```bash
bytes=1073741824
kb=$((bytes / 1024))
mb=$((kb / 1024))
gb=$((mb / 1024))
echo "${bytes} bytes = ${gb} GB"     # 1073741824 bytes = 1 GB
```

---

## Common Mistakes

1. **Expecting floating-point results** — `$((7 / 2))` is 3, not 3.5. Use `bc` for decimals.

2. **Division by zero** — `$((x / 0))` causes an error. Always validate.

3. **Octal interpretation** — Numbers starting with `0` are interpreted as octal: `$((010))` = 8, not 10! Use `$((10#010))` to force base-10.

4. **Using `*` unquoted with `expr`** — `expr 6 * 7` globbing errors. Use `expr 6 \* 7`.

---

## Exercises

### Exercise 16.1: Temperature Converter
Write a script that converts Fahrenheit to Celsius. Formula: C = (F - 32) × 5 / 9. Note: you'll need `bc` for accurate results.

### Exercise 16.2: Arithmetic Practice
Calculate using `$(( ))`:
1. 2 to the power of 16
2. 1000000 modulo 7
3. The average of 85, 92, 78, 96, 88 (integer division is fine)

### Exercise 16.3: Random Password
Write a script that generates a random number between 100000 and 999999 (a 6-digit PIN).

---

## Summary

- `$(( ))` is the standard way to do arithmetic in Bash
- Bash arithmetic is **integer only** — division truncates
- Use `(( ))` for arithmetic comparisons and assignments
- Use `bc -l` or `awk` for floating-point calculations
- `$RANDOM` generates pseudo-random integers (0-32767)
- Numbers starting with `0` are treated as octal — beware!

---

**Next Chapter:** [Chapter 17: Reading User Input →](Chapter17-User-Input.md)
