# PATTERN 35 — MATH AND NUMBER THEORY
## GCD/LCM, Sieves, Modular Arithmetic, Combinatorics, Inclusion-Exclusion — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**Why this pattern is different from every other pattern in this curriculum:**

Every other pattern (sliding window, DP, graphs, trees) is a *technique* you apply to a problem. Number theory is different — it is a **body of facts** you either know cold or you don't. There's no clever insight that reconstructs the extended Euclidean algorithm from first principles in a 30-minute interview. You either have it memorized, or you spend 15 minutes deriving it and run out of time.

**The bar for a 1900+ rated contestant:**

- **GCD in one line, without thinking.** `__gcd(a, b)` or the recursive Euclidean form. If you have to pause to remember whether it's `gcd(a, b) = gcd(b, a % b)` or `gcd(a % b, b)`, you are not ready.
- **Sieve of Eratosthenes from memory**, including the `i * i` optimization and knowing why the outer loop only needs to run to `sqrt(n)`.
- **Modular exponentiation (fast power) from memory** — this single template solves Pow(x,n), Super Pow, Count Good Numbers, and half the "compute X mod 1e9+7" problems on Codeforces Div 2.
- **nCr mod p via Fermat's little theorem** — precompute factorials once, answer every query in O(log p).
- **Knowing when modular inverse via Fermat is valid** (p must be prime) **vs. when you need extended Euclid** (any modulus, as long as gcd(a, m) = 1).

**Why it shows up in CF constantly:**

Number theory is the *cheapest* way for a problem setter to add a layer of difficulty on top of an otherwise easy problem. A trivial counting problem becomes a Div 2 C/D the moment the answer must be reported "mod 1e9+7" and the counting itself involves combinatorics (`nCr`), because now you need modular inverse — not just modular arithmetic. This is the single most common "free difficulty knob" in competitive programming, and it is *pure recall*, not creativity. That is exactly why grandmasters treat this pattern as non-negotiable memorization, not something to "figure out" live.

**The key skill:**

Before coding any math-heavy problem, answer:
1. Is the modulus prime? (Determines whether Fermat's little theorem applies for inverses.)
2. Does the answer require division under a modulus? (If yes, you need modular inverse — division does not distribute over mod naturally.)
3. Am I at risk of `int` overflow before taking `% mod`? (Almost always yes if two numbers up to `1e9` are multiplied — cast to `long long` or use `__int128`.)
4. Is this a "count objects with property X" problem where inclusion-exclusion or a sieve-based multiplicity count is faster than brute enumeration?

If you answer these correctly, the code is boilerplate you've typed a hundred times.

---

## SECTION 2 — THE TOOLKIT TAXONOMY

### Tool 1: GCD / LCM
Use: reducing fractions, checking divisibility relationships, finding a common "step size" (Bezout-style problems), string period problems (GCD of Strings).
Complexity: O(log(min(a,b))) via Euclidean algorithm.

### Tool 2: Sieve of Eratosthenes (+ variants)
Use: need primality or smallest-prime-factor for MANY numbers up to N — never call `isPrime()` in a loop when N is large.
Variants: basic sieve, linear (Euler) sieve for O(n) SPF computation, segmented sieve for ranges beyond memory limits.

### Tool 3: Prime Factorization
Use: decomposing a single number into its prime power representation — needed for divisor counting, Euler's totient, or "numbers connected by a shared prime factor" (Union-Find + primes).
Complexity: O(sqrt(n)) trial division per number, or O(log n) per number if SPF table is precomputed.

### Tool 4: Modular Arithmetic + Fast Power
Use: ANY problem where the answer must be reported mod a large prime (usually 1e9+7 or 998244353), or where exponents are astronomically large (Super Pow, Count Good Numbers).
Complexity: O(log exponent).

### Tool 5: Modular Inverse
Use: division under a modulus — computing `nCr mod p`, dividing a running product, DP transitions involving division.
Two methods: Fermat's little theorem (modulus must be PRIME), Extended Euclidean algorithm (works for ANY modulus as long as gcd(a, m) = 1).

### Tool 6: Combinatorics mod p (nCr, nPr)
Use: counting problems — "how many ways," "how many arrangements" — reported mod a prime.
Complexity: O(N) precompute, O(log p) per query (or O(1) per query after precompute).

### Tool 7: Inclusion-Exclusion
Use: counting elements satisfying AT LEAST ONE of several overlapping conditions (avoids double counting) — e.g., "count numbers ≤ N divisible by 2, 3, or 5."
Complexity: O(2^k) for k conditions — fine for small k (≤ 20).

---

## SECTION 3 — TEMPLATE 1: GCD AND LCM

```cpp
// ─────────────────────────────────────────────────────────────────
// Euclidean Algorithm — GCD (Greatest Common Divisor)
//
// CLAIM: gcd(a, b) = gcd(b, a % b)
// PROOF SKETCH: Let d = gcd(a, b). Write a = q*b + r where r = a % b.
//   d | a and d | b  =>  d | (a - q*b) = r.  So d | gcd(b, r).
//   Conversely, let e = gcd(b, r). e | b and e | r
//   => e | (q*b + r) = a.  So e | gcd(a, b).
//   Both divide each other => gcd(a, b) = gcd(b, r) = gcd(b, a % b). QED.
//
// Base case: gcd(a, 0) = a.
// ─────────────────────────────────────────────────────────────────

long long gcd(long long a, long long b) {
    while (b) {
        a %= b;
        swap(a, b);
    }
    return a;   // when b == 0, a holds the gcd
}

// Recursive form (equivalent, sometimes clearer in interviews)
long long gcdRec(long long a, long long b) {
    return b == 0 ? a : gcdRec(b, a % b);
}

// Built-in: C++17 provides __gcd(a, b) in <algorithm> (GCC extension)
// C++17 also provides std::gcd(a, b) in <numeric> (portable, standard)
#include <numeric>
long long g = std::gcd(48LL, 18LL);   // 6

// ─────────────────────────────────────────────────────────────────
// LCM (Least Common Multiple)
//
// LCM(a,b) * GCD(a,b) = a * b   (fundamental identity)
// => LCM(a,b) = a * b / GCD(a,b)
//
// OVERFLOW TRAP: compute a / gcd(a,b) FIRST, then multiply by b.
// If you compute a*b first, you risk overflow even though the
// final LCM fits in the type (a*b can overflow long long when
// a, b are each up to ~3e9, but a/g * b will not if LCM itself fits).
// ─────────────────────────────────────────────────────────────────

long long lcm(long long a, long long b) {
    return (a / gcd(a, b)) * b;   // divide BEFORE multiplying
}
```

**TRACE for gcd(48, 18):**
```
a=48, b=18: a%b = 48%18 = 12.  swap -> a=18, b=12
a=18, b=12: a%b = 18%12 = 6.   swap -> a=12, b=6
a=12, b=6:  a%b = 12%6  = 0.   swap -> a=6,  b=0
b == 0  ->  return a = 6

gcd(48, 18) = 6 ✓  (48 = 6*8, 18 = 6*3, gcd(8,3)=1 ✓)
```

**TRACE for lcm(48, 18):**
```
gcd(48,18) = 6
lcm = (48 / 6) * 18 = 8 * 18 = 144

Verify: 144 / 48 = 3 ✓,  144 / 18 = 8 ✓  (144 is the smallest common multiple)
```

**Why GCD is O(log(min(a,b))):** Each step at least halves `max(a,b)` within two iterations (Fibonacci-adjacent argument — the worst case input is consecutive Fibonacci numbers, which is why Euclid's algorithm on Fibonacci pairs is the textbook example of its slowest convergence, yet still O(log n)).

---

## SECTION 4 — TEMPLATE 2: SIEVE OF ERATOSTHENES (+ LINEAR SIEVE, + SPF)

```cpp
// ─────────────────────────────────────────────────────────────────
// SIEVE OF ERATOSTHENES — classic O(n log log n)
//
// Idea: start with everything marked "prime." For each prime p found,
// mark all its multiples (starting at p*p, since smaller multiples
// were already marked by smaller primes) as composite.
//
// WHY start at p*p, not 2*p?
//   Any composite c = p*k with k < p was already marked when we
//   processed the prime factor of c that is smaller than p.
//   Example: 2*7=14 was marked when p=2. When p=7, we can skip
//   straight to 7*7=49 — 7*2..7*6 are already handled.
// ─────────────────────────────────────────────────────────────────

vector<bool> sieveOfEratosthenes(int n) {
    vector<bool> isPrime(n + 1, true);
    isPrime[0] = isPrime[1] = false;

    for (long long i = 2; i * i <= n; i++) {
        if (isPrime[i]) {
            for (long long j = i * i; j <= n; j += i) {
                isPrime[j] = false;
            }
        }
    }
    return isPrime;
}

// ─────────────────────────────────────────────────────────────────
// LINEAR (EULER) SIEVE — O(n), each composite marked EXACTLY ONCE
//
// Bonus: computes the Smallest Prime Factor (SPF) of every number,
// which is the key tool for O(log n) prime factorization.
//
// Invariant: for composite c = spf[c] * (c / spf[c]),
//   we mark c exactly once — when processing i = c / p for the
//   SMALLEST prime p that divides c. The inner break condition
//   `i % primes[j] == 0` is what guarantees "exactly once."
// ─────────────────────────────────────────────────────────────────

vector<int> linearSieveSPF(int n) {
    vector<int> spf(n + 1, 0);       // smallest prime factor
    vector<int> primes;
    primes.reserve(n / 10);

    for (int i = 2; i <= n; i++) {
        if (spf[i] == 0) {           // i is prime — no smaller factor found it
            spf[i] = i;
            primes.push_back(i);
        }
        // Propagate i's prime multiples, tagged with the correct SPF
        for (int p : primes) {
            if ((long long)p * i > n) break;
            spf[p * i] = p;                // p is the SPF of p*i by construction
            if (i % p == 0) break;         // CRITICAL: stops double-marking
            // Once p divides i, any larger prime q > p would give
            // q*i a smaller factor p (via p | i), so we must stop here
            // to guarantee each composite is visited only once.
        }
    }
    return spf;   // spf[x] = smallest prime factor of x (spf[1] = 0, undefined)
}
```

**TRACE for sieveOfEratosthenes(30):**
```
Initial: all true except 0,1

i=2 (prime): mark 4,6,8,10,12,14,16,18,20,22,24,26,28,30 = false
i=3 (prime): mark 9,12,15,18,21,24,27,30 = false  (starts at 9=3*3)
i=4: not prime (already marked) — skip
i=5 (prime): 5*5=25 <= 30, mark 25,30 = false
i=6: i*i=36 > 30 — loop ends

Primes ≤ 30: 2,3,5,7,11,13,17,19,23,29   (10 primes)
```

**TRACE for linearSieveSPF(12):**
```
i=2: spf[2]=0 -> prime. spf[2]=2. primes=[2]
     inner: p=2, 2*2=4<=12: spf[4]=2. i%p = 2%2==0 -> break
i=3: spf[3]=0 -> prime. spf[3]=3. primes=[2,3]
     inner: p=2, 2*3=6<=12: spf[6]=2. i%p=3%2=1, continue
            p=3, 3*3=9<=12: spf[9]=3. i%p=3%3==0 -> break
i=4: spf[4]=2 already set -> NOT prime, skip "prime" branch
     inner: p=2, 2*4=8<=12: spf[8]=2. i%p=4%2==0 -> break
i=5: spf[5]=0 -> prime. spf[5]=5. primes=[2,3,5]
     inner: p=2, 2*5=10<=12: spf[10]=2. i%2=1, continue
            p=3, 3*5=15>12 -> break (out of range)
i=6: spf[6]=2 already -> not prime
     inner: p=2, 2*6=12<=12: spf[12]=2. i%2==0 -> break

Final spf[1..12] = [_,0,2,3,2,5,2,7,2,3,2,_,2]
(spf[7] and spf[11] never got touched by inner loops because
 their multiples exceed 12 or weren't reached — they get spf[7]=7,
 spf[11]=11 when i=7 and i=11 hit the "prime" branch directly)
```

**Why the linear sieve is O(n):** Every composite number `c` has a UNIQUE decomposition `c = spf[c] * (c / spf[c])`. The inner loop only ever sets `spf[p*i] = p` for `p <= spf[i]` (the break condition enforces this), so each composite is visited by exactly one `(i, p)` pair — the one where `p = spf[c]`. Total work = O(n).

---

## SECTION 5 — TEMPLATE 3: PRIME FACTORIZATION

```cpp
// ─────────────────────────────────────────────────────────────────
// METHOD A: Trial division to sqrt(n) — O(sqrt(n)) per number
// Use when: factorizing a SMALL NUMBER of large numbers (no sieve
// precompute needed, or n exceeds sieve memory limits).
// ─────────────────────────────────────────────────────────────────

vector<pair<long long,int>> factorizeTrial(long long n) {
    vector<pair<long long,int>> factors;   // (prime, exponent)

    for (long long p = 2; p * p <= n; p++) {
        if (n % p == 0) {
            int exp = 0;
            while (n % p == 0) { n /= p; exp++; }
            factors.push_back({p, exp});
        }
    }
    if (n > 1) factors.push_back({n, 1});  // leftover factor > sqrt(original n)
    return factors;
}

// ─────────────────────────────────────────────────────────────────
// METHOD B: SPF table lookup — O(log n) per number AFTER O(n) precompute
// Use when: factorizing MANY numbers, all bounded by the same N
// (e.g., LC 952 Largest Component Size by Common Factor).
// ─────────────────────────────────────────────────────────────────

vector<pair<int,int>> factorizeSPF(int n, vector<int>& spf) {
    vector<pair<int,int>> factors;
    while (n > 1) {
        int p = spf[n];
        int exp = 0;
        while (n % p == 0) { n /= p; exp++; }
        factors.push_back({p, exp});
    }
    return factors;
}
```

**TRACE factorizeTrial(360):**
```
n=360
p=2: 360%2==0. divide: 360->180->90->45. exp=3. factors=[(2,3)]. n=45
p=3: p*p=9<=45. 45%3==0. divide: 45->15->5. exp=2. factors=[(2,3),(3,2)]. n=5
p=4: p*p=16>5 -> loop ends
n=5 > 1 -> factors.push_back((5,1))

360 = 2^3 * 3^2 * 5^1  ✓  (8 * 9 * 5 = 360)
```

**TRACE factorizeSPF(360, spf) assuming spf precomputed to 360:**
```
n=360: spf[360]=2. divide by 2: 360->180->90->45 (exp=3). factors=[(2,3)]
n=45:  spf[45]=3.  divide by 3: 45->15->5 (exp=2).       factors=[(2,3),(3,2)]
n=5:   spf[5]=5.   divide by 5: 5->1 (exp=1).             factors=[(2,3),(3,2),(5,1)]
n=1 -> stop

Same result, but only 3 SPF lookups instead of trial-dividing up to sqrt(360)≈19.
```

**When to choose which:** If you factorize ONE number (or a handful) up to 1e12, trial division to sqrt(n) is simpler and fast enough. If you factorize thousands/millions of numbers all bounded by the same N ≤ 1e6 or 1e7, precompute SPF once (O(N)) and factorize each number in O(log N) — this is the difference between AC and TLE on problems like LC 952.

---

## SECTION 6 — TEMPLATE 4: MODULAR ARITHMETIC AND FAST POWER

```cpp
// ─────────────────────────────────────────────────────────────────
// BASIC MODULAR ARITHMETIC — the three identities you must know cold
// ─────────────────────────────────────────────────────────────────
const long long MOD = 1e9 + 7;

// (a + b) mod m  — safe as long as a, b < MOD (no overflow for long long)
long long addMod(long long a, long long b, long long m = MOD) {
    return ((a % m) + (b % m)) % m;
}

// (a - b) mod m  — must add m before mod to avoid negative results
// C++'s % can return negative values when a - b < 0!
long long subMod(long long a, long long b, long long m = MOD) {
    return ((a % m) - (b % m) + m) % m;
}

// (a * b) mod m  — OVERFLOW TRAP: a, b up to ~1e9 each => a*b up to ~1e18
// long long max is ~9.2e18, so a*b (up to 1e18) FITS in long long —
// but if a, b can be up to ~1e18 themselves (rare, but happens with
// large moduli), a*b OVERFLOWS long long and you need __int128.
long long mulMod(long long a, long long b, long long m = MOD) {
    return ( (a % m) * (b % m) ) % m;   // safe: both operands < 1e9+7 here
}
// For moduli up to ~1e18 (rare in interviews, common in some CF problems):
long long mulModSafe(long long a, long long b, long long m) {
    return (long long)(( (__int128)a * b ) % m);
}

// ─────────────────────────────────────────────────────────────────
// MODULAR EXPONENTIATION (FAST POWER / BINARY EXPONENTIATION)
//
// Naive: multiply base by itself n times = O(n). Too slow for n ~ 1e18.
// Fast: use the binary representation of the exponent.
//   x^13 = x^(1101 in binary) = x^8 * x^4 * x^1
// At each step: square the base, and if the current LOW BIT of the
// exponent is 1, multiply the running result by the current base power.
// ─────────────────────────────────────────────────────────────────

long long powMod(long long base, long long exp, long long m = MOD) {
    base %= m;
    if (base < 0) base += m;
    long long result = 1;

    while (exp > 0) {
        if (exp & 1) {                       // low bit set — include this power
            result = mulMod(result, base, m);
        }
        base = mulMod(base, base, m);        // square the base
        exp >>= 1;                           // shift to next bit
    }
    return result;
}
```

**TRACE for powMod(3, 13, 1000000007):**
```
exp=13 in binary = 1101

result=1, base=3, exp=13 (binary 1101)
  exp&1=1 -> result = 1*3 = 3.        base = 3*3=9.    exp=6  (110)
  exp&1=0 -> skip.                     base = 9*9=81.   exp=3  (11)
  exp&1=1 -> result = 3*81=243.       base = 81*81=6561. exp=1 (1)
  exp&1=1 -> result = 243*6561=1594323. base=... exp=0 -> loop ends

3^13 = 1594323.  Verify: 3^13 = 1,594,323 ✓ (fits comfortably, no mod wraparound
here since 3^13 < 1e9+7; the algorithm is identical when wraparound occurs)
```

**Why O(log exp):** The exponent is halved every iteration (`exp >>= 1`), so the loop runs `⌊log2(exp)⌋ + 1` times. Each iteration does O(1) multiplications (at most 2). Total: O(log exp) multiplications, each O(1) under fixed-width modular arithmetic.

---

## SECTION 7 — TEMPLATE 5: MODULAR INVERSE

```cpp
// ─────────────────────────────────────────────────────────────────
// METHOD A: Fermat's Little Theorem — ONLY valid when m is PRIME
//
// Fermat: if p is prime and gcd(a, p) = 1, then a^(p-1) ≡ 1 (mod p).
// Multiply both sides by a^(-1):  a^(p-2) ≡ a^(-1) (mod p).
// So the modular inverse of a is simply a^(p-2) mod p — a single
// fast-power call. This is why 1e9+7 and 998244353 (both prime)
// are the go-to competitive programming moduli — they make inverses trivial.
// ─────────────────────────────────────────────────────────────────

long long modInverseFermat(long long a, long long p = MOD) {
    // REQUIRES: p is prime, and a is NOT a multiple of p (gcd(a,p)=1)
    return powMod(a, p - 2, p);
}

// ─────────────────────────────────────────────────────────────────
// METHOD B: Extended Euclidean Algorithm — works for ANY modulus m,
// as long as gcd(a, m) = 1 (the inverse exists iff this holds).
// Use when: m is NOT prime (e.g., m = 1000000, or an arbitrary composite).
//
// Extended Euclid finds x, y such that: a*x + m*y = gcd(a, m).
// If gcd(a, m) = 1, then a*x + m*y = 1  =>  a*x ≡ 1 (mod m)
// => x is the modular inverse of a mod m.
// ─────────────────────────────────────────────────────────────────

// Returns gcd(a,b) and sets x,y such that a*x + b*y = gcd(a,b)
long long extendedGCD(long long a, long long b, long long &x, long long &y) {
    if (b == 0) { x = 1; y = 0; return a; }
    long long x1, y1;
    long long g = extendedGCD(b, a % b, x1, y1);
    x = y1;
    y = x1 - (a / b) * y1;
    return g;
}

long long modInverseExtEuclid(long long a, long long m) {
    long long x, y;
    long long g = extendedGCD(a, m, x, y);
    if (g != 1) return -1;              // inverse does not exist
    return ((x % m) + m) % m;           // normalize to [0, m)
}
```

**TRACE for modInverseFermat(3, 7):** (p=7 is prime)
```
inverse = 3^(7-2) mod 7 = 3^5 mod 7

3^1=3, 3^2=9%7=2, 3^3=6, 3^4=18%7=4, 3^5=12%7=5

modInverseFermat(3,7) = 5
Verify: 3 * 5 = 15 = 2*7 + 1 ≡ 1 (mod 7) ✓
```

**TRACE for extendedGCD(3, 11) → inverse of 3 mod 11:**

The recursion bottoms out at `extendedGCD(1, 0)` and unwinds, computing
`x = y1`, `y = x1 - (a/b)*y1` at each frame (checked via `a*x + b*y = g`):

| Call (a, b) | recurses into | x1, y1 (child's x,y) | computed x, y | check: a·x + b·y |
|---|---|---|---|---|
| (1, 0) | base case | — | x=1, y=0 | 1·1 + 0·0 = 1 ✓ |
| (2, 1) | (1, 0) | 1, 0 | x=0, y=1-2·0=1 | 2·0 + 1·1 = 1 ✓ |
| (3, 2) | (2, 1) | 0, 1 | x=1, y=0-1·1=-1 | 3·1 + 2·(-1) = 1 ✓ |
| (11, 3) | (3, 2) | 1, -1 | x=-1, y=1-3·(-1)=4 | 11·(-1) + 3·4 = 1 ✓ |
| (3, 11) | (11, 3) | -1, 4 | x=4, y=-1-0·4=-1 | 3·4 + 11·(-1) = 1 ✓ |

Top-level result: `x = 4`.
`modInverseExtEuclid(3,11) = ((4 % 11) + 11) % 11 = 4`
Verify: `3 * 4 = 12 ≡ 1 (mod 11)` ✓

**When to use which — the ONE rule:**
- Modulus is a known prime (1e9+7, 998244353, or any prime you can verify) → **Fermat**, it's a one-liner (`powMod(a, m-2, m)`).
- Modulus is composite, or not guaranteed prime → **Extended Euclid** is the only correct method; Fermat's theorem simply does not apply (a^(m-2) mod m is NOT the inverse if m isn't prime).
- Either way: **the inverse only exists if gcd(a, m) = 1.** If a and m share a common factor, no inverse exists — division by a is undefined mod m.

---

## SECTION 8 — TEMPLATE 6: nCr MOD p (COMBINATORICS)

```cpp
// ─────────────────────────────────────────────────────────────────
// nCr mod p — precompute factorials AND inverse factorials once,
// then answer each query in O(1) (after O(log p) per inverse factorial
// during precompute, or O(N) total using the trick below).
//
// nCr = n! / (r! * (n-r)!)
// Under a modulus, division becomes multiplication by modular inverse:
// nCr mod p = fact[n] * invFact[r] * invFact[n-r] mod p
// ─────────────────────────────────────────────────────────────────

const int MAXN = 200005;
long long fact[MAXN], invFact[MAXN];

void precomputeFactorials(long long p = MOD) {
    fact[0] = 1;
    for (int i = 1; i < MAXN; i++)
        fact[i] = mulMod(fact[i-1], i, p);

    // Compute invFact[MAXN-1] via Fermat, then walk DOWNWARD:
    // invFact[i-1] = invFact[i] * i   (since (i-1)! = i! / i)
    // This computes ALL inverse factorials with just ONE powMod call —
    // much faster than calling modInverseFermat for every i individually.
    invFact[MAXN-1] = powMod(fact[MAXN-1], p - 2, p);
    for (int i = MAXN - 1; i > 0; i--)
        invFact[i-1] = mulMod(invFact[i], i, p);
}

long long nCr(int n, int r, long long p = MOD) {
    if (r < 0 || r > n) return 0;
    return mulMod(mulMod(fact[n], invFact[r], p), invFact[n-r], p);
}
```

**Why walking `invFact` downward with ONE powMod call is correct:**
```
Claim: invFact[i-1] = invFact[i] * i (mod p)

Proof: invFact[i] = (i!)^-1.  invFact[i] * i = (i!)^-1 * i.
  i! = i * (i-1)!  =>  (i!)^-1 = (i-1)!^-1 * i^-1
  So invFact[i] * i = (i-1)!^-1 * i^-1 * i = (i-1)!^-1 = invFact[i-1].  QED.

This turns MAXN individual powMod calls (O(MAXN log p)) into ONE powMod
call (O(log p)) plus MAXN O(1) multiplications — O(MAXN) total precompute.
```

**TRACE for nCr(5, 2) mod 1000000007:**
```
fact[0]=1, fact[1]=1, fact[2]=2, fact[3]=6, fact[4]=24, fact[5]=120

invFact[5] = powMod(120, MOD-2, MOD)   (some large number ≡ 120^-1 mod p)
invFact[4] = invFact[5]*5 mod p  (= 24^-1 mod p, since 5!/(5)=4!)
invFact[3] = invFact[4]*4 mod p  (= 6^-1 mod p)
invFact[2] = invFact[3]*3 mod p  (= 2^-1 mod p)

nCr(5,2) = fact[5] * invFact[2] * invFact[3] mod p
         = 120 * (2^-1 mod p) * (6^-1 mod p) mod p
         = 120 / (2*6) = 120 / 12 = 10   (verified via modular inverse arithmetic)

Direct check: C(5,2) = 10 ✓  (5!/(2!*3!) = 120/(2*6) = 10)
```

---

## SECTION 9 — TEMPLATE 7: INCLUSION-EXCLUSION PRINCIPLE

```cpp
// ─────────────────────────────────────────────────────────────────
// INCLUSION-EXCLUSION — count elements satisfying AT LEAST ONE
// of k conditions, without double-counting overlaps.
//
// |A ∪ B| = |A| + |B| - |A ∩ B|
// |A ∪ B ∪ C| = |A|+|B|+|C| - |A∩B|-|A∩C|-|B∩C| + |A∩B∩C|
//
// GENERAL FORM (k sets): sum over all NON-EMPTY subsets S of {1..k},
// with sign (-1)^(|S|+1), of the count of elements satisfying ALL
// conditions in S simultaneously.
// ─────────────────────────────────────────────────────────────────

// EXAMPLE: count integers in [1, n] divisible by AT LEAST ONE of
// the primes in a given list (classic inclusion-exclusion problem,
// e.g. LeetCode 1201 "Ugly Number III" style or divisor-counting).

long long countDivisibleByAtLeastOne(long long n, vector<long long>& primes) {
    int k = primes.size();
    long long count = 0;

    // Iterate over all 2^k non-empty subsets via bitmask
    for (int mask = 1; mask < (1 << k); mask++) {
        long long lcmOfSubset = 1;
        int bits = __builtin_popcount(mask);
        bool overflow = false;

        for (int i = 0; i < k; i++) {
            if (mask & (1 << i)) {
                // lcm of pairwise-coprime primes = product (safe here
                // since inputs are primes; general LCM needs gcd)
                if (lcmOfSubset > n / primes[i]) { overflow = true; break; }
                lcmOfSubset *= primes[i];
            }
        }
        if (overflow) continue;

        long long multiplesInRange = n / lcmOfSubset;
        // Odd subset size -> add (inclusion), even -> subtract (exclusion)
        count += (bits % 2 == 1) ? multiplesInRange : -multiplesInRange;
    }
    return count;
}
```

**WORKED EXAMPLE: count integers in [1,100] divisible by 2, 3, or 5.**
```
A = divisible by 2: floor(100/2) = 50
B = divisible by 3: floor(100/3) = 33
C = divisible by 5: floor(100/5) = 20

A∩B = divisible by lcm(2,3)=6:  floor(100/6) = 16
A∩C = divisible by lcm(2,5)=10: floor(100/10) = 10
B∩C = divisible by lcm(3,5)=15: floor(100/15) = 6

A∩B∩C = divisible by lcm(2,3,5)=30: floor(100/30) = 3

|A∪B∪C| = (50+33+20) - (16+10+6) + 3
        = 103 - 32 + 3
        = 74

Answer: 74 integers in [1,100] are divisible by 2, 3, or 5.
(Sanity: numbers NOT divisible by any of 2,3,5 are the ones coprime-ish
to 30 within each block of 30 — 100-74=26 such numbers, which matches
known density calculations for this classic problem.)
```

**Why the alternating sign works (intuition):** When you sum `|A|+|B|+|C|`, every element in exactly one set is counted once (correct), but an element in two sets (say A∩B) is counted TWICE. Subtracting `|A∩B|` etc. fixes double-counted pairs — but now an element in all three sets was added 3 times, subtracted 3 times (once per pair), netting to 0 — so you add back `|A∩B∩C|` once to restore the correct count of 1.

---

## SECTION 10 — APPLICATIONS

### Application 1 — LC 204: Count Primes

```cpp
// Count primes strictly less than n.
// Pure sieve application — the entire pattern in one function.
int countPrimes(int n) {
    if (n < 3) return 0;
    vector<bool> isComposite(n, false);
    int count = 0;

    for (int i = 2; i < n; i++) {
        if (!isComposite[i]) {
            count++;
            for (long long j = (long long)i * i; j < n; j += i)
                isComposite[j] = true;
        }
    }
    return count;
}
```

**TRACE for n=10:**
```
Check i=2..9 (strictly less than 10)
i=2: prime. count=1. mark 4,6,8
i=3: prime. count=2. mark 9  (3*3=9)
i=4: composite (marked) — skip
i=5: prime. count=3. 5*5=25>=10, no marking needed
i=6: composite — skip
i=7: prime. count=4. 7*7=49>=10, no marking
i=8: composite — skip
i=9: composite (marked by i=3) — skip

Answer: 4  (primes: 2,3,5,7) ✓
```

---

### Application 2 — LC 50: Pow(x, n)

```cpp
// Compute x^n. n can be NEGATIVE. n can be INT_MIN (the classic trap:
// -INT_MIN overflows int, since |INT_MIN| > INT_MAX).
double myPow(double x, int n) {
    long long N = n;              // widen to long long FIRST — before negating
    if (N < 0) { x = 1 / x; N = -N; }   // now -N is safe since N is long long

    double result = 1.0;
    double base = x;
    while (N > 0) {
        if (N & 1) result *= base;
        base *= base;
        N >>= 1;
    }
    return result;
}
```

**TRACE for x=2.0, n=10:**
```
N=10 (binary 1010), x=2.0 positive so no inversion

result=1, base=2, N=10
  N&1=0 -> skip.           base=4.   N=5 (101)
  N&1=1 -> result=1*4=4.   base=16.  N=2 (10)
  N&1=0 -> skip.           base=256. N=1 (1)
  N&1=1 -> result=4*256=1024. base=...  N=0 -> stop

2^10 = 1024 ✓
```

**Why `long long N = n` BEFORE `N = -N`:** If `n = INT_MIN = -2147483648`, then `-n` overflows `int` (max is 2147483647) — undefined behavior. By assigning to `long long N` first, `-N` is computed in `long long` range, where it fits comfortably (`2147483648` is well within `long long` range).

---

### Application 3 — LC 1922: Count Good Numbers

```cpp
// A "good number" at even indices (0,2,4,...) must be a prime digit
// (2,3,5,7) — 4 choices. At odd indices, must be an even digit
// (0,2,4,6,8) — 5 choices. Count good strings of length n, mod 1e9+7.
//
// n positions total: ceil(n/2) even-indexed positions (5 choices... wait,
// careful: even index -> prime digit -> 4 choices; odd index -> even digit
// -> 5 choices.
// #even-indexed positions = ceil(n/2), #odd-indexed = floor(n/2)
// Answer = 4^ceil(n/2) * 5^floor(n/2) mod p — two fast-power calls.
int countGoodNumbers(long long n) {
    const long long MOD = 1e9+7;
    long long evenCount = (n + 1) / 2;   // positions 0,2,4,... => ceil(n/2)
    long long oddCount  = n / 2;          // positions 1,3,5,... => floor(n/2)

    return (int)mulMod(powMod(4, evenCount, MOD), powMod(5, oddCount, MOD), MOD);
}
```

**TRACE for n=4:**
```
evenCount = (4+1)/2 = 2   (positions 0, 2)
oddCount  = 4/2 = 2       (positions 1, 3)

answer = 4^2 * 5^2 mod p = 16 * 25 = 400

Verify by enumeration logic: position 0 (prime digit, 4 choices) *
position 1 (even digit, 5 choices) * position 2 (prime digit, 4 choices) *
position 3 (even digit, 5 choices) = 4*5*4*5 = 400 ✓
```

---

### Application 4 — LC 372: Super Pow

```cpp
// Compute a^b mod 1337, where b is given as an ARRAY of digits
// (can represent astronomically large exponents, e.g. b has 2000 digits).
//
// KEY IDENTITY: a^(10*x + d) = (a^x)^10 * a^d
// Process b's digits LEFT TO RIGHT, building up the exponent digit by digit:
//   result = (result^10 * a^(next digit)) mod 1337
const int SP_MOD = 1337;

int superPow(int a, vector<int>& b) {
    long long result = 1;
    a %= SP_MOD;

    for (int digit : b) {
        // result currently represents a^(prefix so far).
        // Shift the exponent by one decimal digit (raise to 10th power),
        // then fold in the new digit.
        result = powMod(result, 10, SP_MOD) * powMod(a, digit, SP_MOD) % SP_MOD;
    }
    return (int)result;
}
```

**TRACE for a=2, b=[1,0]  (meaning exponent = 10, so answer = 2^10 mod 1337):**
```
a=2, result=1

digit=1: result = powMod(1,10,1337) * powMod(2,1,1337) mod 1337
                = 1 * 2 mod 1337 = 2
        (result now represents 2^1)

digit=0: result = powMod(2,10,1337) * powMod(2,0,1337) mod 1337
                = 1024 * 1 mod 1337 = 1024
        (result now represents 2^(1*10 + 0) = 2^10)

Answer: 1024.  Verify: 2^10 = 1024, and 1024 < 1337 so no wraparound. ✓
```

**Why left-to-right digit processing works:** After processing prefix digits `d0, d1, ..., dk`, `result = a^(d0 d1 ... dk interpreted as a number)`. Appending digit `d(k+1)` transforms the exponent from `X` to `10*X + d(k+1)`, which by the identity `a^(10X+d) = (a^X)^10 * a^d` is exactly `result^10 * a^d`.

---

### Application 5 — LC 1071: GCD of Strings

```cpp
// Find the largest string that divides BOTH str1 and str2 (i.e., str1
// and str2 are each formed by repeating this string some number of times).
//
// KEY INSIGHT: if such a string X exists, then str1 + str2 == str2 + str1
// (concatenation must commute — this is the necessary AND sufficient
// condition). The answer, if it exists, is the prefix of length
// gcd(len(str1), len(str2)).
string gcdOfStrings(string str1, string str2) {
    if (str1 + str2 != str2 + str1) return "";   // no common "divisor" string

    int g = gcd((long long)str1.size(), (long long)str2.size());
    return str1.substr(0, g);
}
```

**TRACE for str1="ABCABC", str2="ABC":**
```
str1+str2 = "ABCABCABC"
str2+str1 = "ABCABCABC"
Equal -> a common divisor string exists.

gcd(6, 3) = 3
Answer: str1.substr(0,3) = "ABC" ✓
```

**TRACE for str1="ABABAB", str2="ABAB" (a trickier case):**
```
str1+str2 = "ABABAB" + "ABAB" = "ABABABABAB"   (len 10)
str2+str1 = "ABAB" + "ABABAB" = "ABABABABAB"   (len 10)
Equal -> divisor string exists.

gcd(6, 4) = 2
Answer: str1.substr(0,2) = "AB" ✓  (ABABAB = AB*3, ABAB = AB*2)
```

**Why the concatenation check is both necessary and sufficient:** If a common "unit" string `u` of length `g` exists with `str1 = u * (len1/g)` and `str2 = u * (len2/g)`, then any concatenation order of str1 and str2 reduces to a repetition of `u` — hence they must be equal. This is a classical result (related to the Fine–Wilf theorem on periodic strings): two strings commute under concatenation if and only if they are powers of a common string.

---

### Application 6 — LC 62: Unique Paths (via nCr, closed form)

```cpp
// m x n grid, moves only right or down, from (0,0) to (m-1,n-1).
// Total moves needed: (m-1) downs + (n-1) rights = (m+n-2) moves total.
// The path is FULLY DETERMINED by WHICH (m-1) of those (m+n-2) moves
// are "down" (the rest are "right"). So the answer is simply:
//   C(m+n-2, m-1)   — choose positions for the down-moves.
//
// This is an O(min(m,n)) closed-form alternative to the O(mn) DP
// from Pattern 18 — useful when m, n are huge (e.g. up to 1e9) and a
// DP table would not fit in memory or time.
int uniquePaths(int m, int n) {
    long long totalMoves = m + n - 2;
    long long downMoves  = m - 1;
    // Compute C(totalMoves, downMoves) directly with incremental
    // multiplication (no modulus needed here — LC 62 fits in double/int64).
    long double result = 1;
    for (int i = 1; i <= downMoves; i++) {
        result = result * (totalMoves - downMoves + i) / i;
    }
    return (int)round((double)result);
}
```

**TRACE for m=3, n=3:**
```
totalMoves = 3+3-2 = 4
downMoves  = 3-1 = 2
C(4, 2) = 6

Incremental computation:
  i=1: result = 1 * (4-2+1)/1 = 3/1 = 3
  i=2: result = 3 * (4-2+2)/2 = 3*4/2 = 6

Answer: 6 ✓  (matches the DP trace from Pattern 18 exactly)
```

**Why this is "math" and not "just DP":** Recognizing that the DP table collapses into a single combinatorial coefficient is a classic 1900+ pattern-recognition skill — many grid/path counting problems that look like they need O(mn) DP actually have an O(min(m,n)) or O(1) closed form once you see the "choose which steps are down" reformulation.

---

### Application 7 — LC 952: Largest Component Size by Common Factor

```cpp
// Two numbers are "connected" if they share a common factor > 1.
// Find the size of the largest connected component.
//
// KEY INSIGHT: don't build edges between numbers directly (O(n^2) —
// too slow). Instead, treat each PRIME as a virtual "hub" node.
// Union each number with every prime factor it has. Numbers sharing
// a prime factor end up in the same DSU component transitively.
// (This directly combines Pattern 13 — DSU — with prime factorization.)

class DSU {
public:
    vector<int> parent, rank_;
    DSU(int n) : parent(n), rank_(n, 0) {
        iota(parent.begin(), parent.end(), 0);
    }
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);   // path compression
        return parent[x];
    }
    void unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return;
        if (rank_[a] < rank_[b]) swap(a, b);
        parent[b] = a;
        if (rank_[a] == rank_[b]) rank_[a]++;
    }
};

int largestComponentSize(vector<int>& nums) {
    int maxVal = *max_element(nums.begin(), nums.end());
    DSU dsu(maxVal + 1);              // index space: [0, maxVal] doubles as
                                        // "prime hub" IDs (a prime p uses index p)

    for (int num : nums) {
        int x = num;
        for (int p = 2; (long long)p * p <= x; p++) {
            if (x % p == 0) {
                dsu.unite(num, p);           // union num with prime hub p
                while (x % p == 0) x /= p;
            }
        }
        if (x > 1) dsu.unite(num, x);        // leftover prime factor > sqrt
    }

    unordered_map<int,int> compSize;
    int best = 1;
    for (int num : nums) {
        int root = dsu.find(num);
        best = max(best, ++compSize[root]);
    }
    return best;
}
```

**TRACE for nums=[4, 6, 15, 35]:**
```
Factorize each and union with prime hubs:
  4  = 2^2       -> unite(4, 2)
  6  = 2*3        -> unite(6, 2), unite(6, 3)
  15 = 3*5        -> unite(15, 3), unite(15, 5)
  35 = 5*7        -> unite(35, 5), unite(35, 7)

Union chain: 4-2-6-3-15-5-35-7  (all connected transitively via shared primes)
  4 and 6 share prime 2 -> same component
  6 and 15 share prime 3 -> same component
  15 and 35 share prime 5 -> same component

Final: all four numbers {4,6,15,35} end up in ONE component (size 4),
even though 4 and 35 share NO direct common factor — they're connected
through the chain 4~6~15~35.

Answer: 4
```

**Why prime hubs avoid O(n^2) edge construction:** Directly checking every pair `(numsE[i], nums[j])` for `gcd > 1` is O(n^2 log(maxVal)). By instead unioning each number with its OWN prime factors (each number has O(log maxVal) distinct prime factors), total unions are O(n log maxVal * α(n)) — the primes act as shared "meeting points" so numbers with a common factor automatically land in the same DSU tree without ever being compared directly.

---

### Application 8 — LC 2514: Count Anagrams

```cpp
// Given a string s of space-separated words, count how many strings
// can be formed by independently permuting the letters of each word
// (order of words is fixed), mod 1e9+7.
//
// For a single word of length L with character counts c[0..25]:
//   distinct permutations = L! / (c[0]! * c[1]! * ... * c[25]!)
// Multiply this across all words (independent choices per word).
// Requires factorials AND inverse factorials mod p -> Template 6 directly.

int countAnagrams(string s) {
    // Assume fact[] and invFact[] precomputed up to s.size() via
    // precomputeFactorials() from Template 6.
    long long answer = 1;
    stringstream ss(s);
    string word;

    while (ss >> word) {
        int cnt[26] = {0};
        for (char c : word) cnt[c - 'a']++;

        long long wordWays = fact[word.size()];
        for (int i = 0; i < 26; i++) {
            wordWays = mulMod(wordWays, invFact[cnt[i]], MOD);
        }
        answer = mulMod(answer, wordWays, MOD);
    }
    return (int)answer;
}
```

**TRACE for s="two two":**
```
Word "two": length 3, counts: t=1, w=1, o=1 (all distinct)
  ways = fact[3] * invFact[1] * invFact[1] * invFact[1]
       = 6 * 1 * 1 * 1 = 6           (3! / (1!1!1!) = 6, all permutations distinct)

Word "two" (again, independent): ways = 6

answer = 6 * 6 = 36

Verify: "two" has 3 distinct letters -> 3! = 6 arrangements. Two independent
words -> 6 * 6 = 36 total anagram-strings. ✓
```

**Why we need modular inverse here (not just factorials):** The formula divides by `c[i]!` for each letter's count — under a modulus, "divide by c[i]!" must become "multiply by `invFact[c[i]]`." Attempting `fact[L] / (c[0]! * ... )` with integer division mod p is simply WRONG — modular division is not the same operation as real-number division, and standard `/` under a modulus does not exist without inverse arithmetic.

---

## SECTION 11 — COMPLEXITY TABLE

| Tool | Time | Space | Notes |
|------|------|-------|-------|
| GCD (Euclidean) | O(log(min(a,b))) | O(1) | Also LCM, since LCM = a/gcd*b |
| Sieve of Eratosthenes | O(n log log n) | O(n) | Standard primality up to n |
| Linear (Euler) sieve + SPF | O(n) | O(n) | Also gives SPF for O(log n) factorization |
| Trial division factorization | O(sqrt(n)) | O(1) | Best for 1-off large n (up to ~1e12) |
| SPF-table factorization | O(log n) | O(n) precompute | Best for many numbers ≤ N |
| Modular exponentiation | O(log exp) | O(1) | The single most-reused template here |
| Modular inverse (Fermat) | O(log p) | O(1) | REQUIRES p prime |
| Modular inverse (ext. Euclid) | O(log(min(a,m))) | O(1) | Works for any modulus, gcd(a,m)=1 |
| nCr mod p (after precompute) | O(1) per query | O(N) precompute | N = max n needed |
| nCr precompute (fact + invFact) | O(N) | O(N) | One powMod call total (not N calls) |
| Inclusion-Exclusion (k sets) | O(2^k * k) | O(1) | Only viable for small k (≤ ~20) |

---

## SECTION 12 — VARIANTS

**Variant 1 — Euler's Totient Function φ(n):** Count integers in [1,n] coprime to n. Computable via the same SPF/sieve infrastructure: `φ(n) = n * Π(1 - 1/p)` over distinct prime factors p of n. Useful for problems counting coprime pairs.

**Variant 2 — Segmented Sieve:** When you need primes in a range `[L, R]` where `R` is too large to sieve from 0 (e.g., R up to 1e12 but R-L ≤ 1e6), sieve only the range using primes up to `sqrt(R)` precomputed separately.

**Variant 3 — Matrix Exponentiation:** Generalizes fast power to matrices — used for linear recurrences (Fibonacci in O(log n), tiling counts, etc.). Same binary-exponentiation skeleton, multiplication replaced with matrix multiplication.

**Variant 4 — Chinese Remainder Theorem (CRT):** Solve systems of congruences `x ≡ a_i (mod m_i)` for pairwise coprime moduli — used when a modulus is a product of primes and you want to combine results computed modulo each prime factor separately.

**Variant 5 — Catalan Numbers:** `C_n = C(2n,n)/(n+1)` — counts balanced parenthesizations, BST shapes, etc. Computed via the nCr-mod-p template plus one extra modular inverse (of `n+1`).

**Variant 6 — Digit DP with modular counting:** Counting numbers with digit properties in a range often combines Pattern 25 (Digit DP) with modular arithmetic when counts must be reported mod p — the two patterns overlap directly.

---

## SECTION 13 — COMMON MISTAKES

### Mistake 1: Overflow from `a * b` before taking `% mod`

```cpp
// WRONG — int overflow. a, b up to 1e9 each -> a*b up to 1e18,
// but if a, b are declared as `int`, the multiplication overflows
// BEFORE the result is even assigned to a wider type.
int a = 999999937, b = 999999937;
int product = (a * b) % MOD;      // UNDEFINED BEHAVIOR — overflows int

// CORRECT — cast to long long BEFORE multiplying
long long product = ((long long)a * b) % MOD;

// For moduli that can themselves be near long long's limit (~9.2e18),
// even long long*long long can overflow — use __int128 or Barrett/Montgomery
// reduction in that regime:
long long productSafe = (long long)(( (__int128)a * b ) % MOD);
```

---

### Mistake 2: Pow(x, n) — mishandling INT_MIN

```cpp
// WRONG — negating INT_MIN overflows int (UB)
double myPow(double x, int n) {
    if (n < 0) { x = 1/x; n = -n; }   // BUG: -n overflows when n == INT_MIN
    ...
}

// CORRECT — widen to long long BEFORE negating
double myPow(double x, int n) {
    long long N = n;
    if (N < 0) { x = 1/x; N = -N; }   // safe: long long comfortably holds
                                        // 2147483648, unlike int
    ...
}
```

---

### Mistake 3: Using Fermat's little theorem when the modulus is NOT prime

```cpp
// WRONG — modulus 12 is NOT prime; a^(m-2) mod m is meaningless here
long long inv = powMod(5, 12 - 2, 12);   // garbage — Fermat requires prime modulus!

// gcd(5, 12) = 1, so an inverse DOES exist — but Fermat's formula
// simply does not compute it when the modulus is composite.

// CORRECT — use extended Euclidean algorithm for composite moduli
long long inv = modInverseExtEuclid(5, 12);   // correctly returns 5
                                                 // (5*5=25=2*12+1 ≡ 1 mod 12)
```

---

### Mistake 4: Assuming modular inverse always exists

```cpp
// WRONG — blindly computing an inverse without checking gcd(a, m) == 1
long long inv = modInverseExtEuclid(6, 9);   // gcd(6,9) = 3 != 1 -> NO INVERSE EXISTS
// The function returns -1 in this case (per our implementation), but if
// you don't check for -1, you'll silently use garbage downstream.

// CORRECT — always verify the inverse exists (or, when the modulus is a
// prime p, verify a % p != 0, which guarantees gcd(a,p)=1 automatically)
long long inv = modInverseExtEuclid(6, 9);
if (inv == -1) { /* handle: no valid inverse, problem-specific logic */ }
```

---

### Mistake 5: LCM overflow from multiplying before dividing

```cpp
// WRONG — a*b computed first can overflow even when the true LCM fits
long long lcmBad(long long a, long long b) {
    return (a * b) / gcd(a, b);     // a*b can overflow long long
                                      // even if (a*b)/gcd(a,b) would not!
}

// CORRECT — divide by gcd FIRST, then multiply
long long lcmGood(long long a, long long b) {
    return (a / gcd(a, b)) * b;     // each intermediate value stays smaller
}
```

---

### Mistake 6: Off-by-one sieve bounds / marking primes wrong

```cpp
// WRONG — starting the inner marking loop at 2*i instead of i*i
// (correctness is preserved, but it's O(n log n) instead of O(n log log n) —
// often still passes, but the REAL bug below is worse)
for (int i = 2; i * i <= n; i++)
    if (isPrime[i])
        for (int j = 2 * i; j <= n; j += i)   // wasteful but not wrong
            isPrime[j] = false;

// ACTUAL BUG — outer loop condition off-by-one, using i <= n instead of
// i * i <= n, causing either a silent slowdown (harmless) or, worse,
// forgetting to mark isPrime[0] and isPrime[1] as false:
vector<bool> isPrime(n + 1, true);   // BUG: isPrime[0] and isPrime[1]
                                        // are left `true` — 0 and 1 are
                                        // NOT prime by definition!

// CORRECT — always explicitly clear the base cases
vector<bool> isPrime(n + 1, true);
isPrime[0] = isPrime[1] = false;      // 0 and 1 are never prime
for (long long i = 2; i * i <= n; i++)
    if (isPrime[i])
        for (long long j = i * i; j <= n; j += i)
            isPrime[j] = false;
```

---

## SECTION 14 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Count Primes | 204 | Basic sieve, count while marking |
| 2 | Power of Two / Power of Three | 231 / 326 | Bit tricks (n & (n-1)) or repeated division / log trick |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | GCD of Strings | 1071 | GCD applied to string periodicity, concatenation check |
| 4 | Pow(x, n) | 50 | Fast power, negative n, INT_MIN overflow trap |
| 5 | Count Good Numbers | 1922 | Fast power twice, even/odd index counting |
| 6 | Unique Paths (math) | 62 | Closed-form nCr instead of O(mn) DP |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Super Pow | 372 | Modular exponentiation with an array-encoded exponent |
| 8 | Largest Component Size by Common Factor | 952 | Prime factorization + DSU (Pattern 13 combo) |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 9 | Count Anagrams | 2514 | Factorials + modular inverse, per-word multinomial |
| 10 | Apply Operations to Make Sum of Array ≥ k | ~3229 (contest) | Optimization over divisor-like search space (n·ceil(k/n)) |

---

## SECTION 15 — PATTERN CONNECTIONS

1. **Pattern 13 (Union-Find / DSU):** LC 952 (Largest Component Size by Common Factor) is a direct fusion of Pattern 35's prime factorization with Pattern 13's DSU — primes act as virtual hub nodes that let numbers merge transitively without pairwise comparison.

2. **Pattern 16 (DP Combinatorics):** Any DP that counts arrangements/paths and must report the answer mod a large prime (0/1 Knapsack counting variants, path-counting DPs) needs Template 6 (nCr mod p) or at minimum Template 4 (modular arithmetic) layered on top of the DP recurrence — the DP structure and the modular arithmetic are separate, composable concerns.

3. **Pattern 25 (Digit DP):** Digit DP problems that ask "count numbers with property X in [1, N] mod p" combine digit-by-digit DP transitions with the modular arithmetic template from this pattern — the DP determines WHAT to count, this pattern determines HOW to report it safely.

4. **Pattern 18 (2D Grid DP):** LC 62 (Unique Paths) appears in BOTH patterns — Pattern 18 solves it with O(mn) DP, this pattern solves it with O(min(m,n)) closed-form combinatorics (`C(m+n-2, m-1)`). Recognizing when a DP table collapses into a single binomial coefficient is a hallmark 1900+ skill.

5. **Pattern 04 (Two Pointers / Binary Search on Answer):** Problem 10 in this pattern's problem set (Apply Operations to Make Sum ≥ k) is fundamentally an optimization over a small parameter space (number of array elements, bounded by O(sqrt(k))) — the same "search over a bounded parameter, compute the rest via a closed-form formula" idea that appears in binary-search-on-answer problems.

---

## SECTION 16 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 50 — Pow(x, n)

**Interviewer:** "Implement pow(x, n) without using the built-in power function."

**Candidate:**

*[First establish the naive approach and why it's insufficient]*
> "Naive approach: multiply x by itself n times — O(n). For n up to 2^31, that's too slow if we want sub-linear time, though it would technically pass on small n. The standard optimization is binary exponentiation: O(log n)."

*[State the recursive idea, then commit to iterative for clean code]*
> "The idea: x^n = (x^(n/2))^2 if n is even, or x * (x^((n-1)/2))^2 if n is odd. I'll implement this iteratively using the bits of n."

*[Flag the edge case BEFORE coding — this is what separates 1900+ from 1600]*
> "One trap: n can be negative, and n can be INT_MIN specifically. If I do `n = -n` on an `int`, that's undefined behavior for INT_MIN because `-INT_MIN` doesn't fit in `int`. I'll widen n to `long long` before negating."

*[Code — 5 minutes]*
```cpp
double myPow(double x, int n) {
    long long N = n;
    if (N < 0) { x = 1 / x; N = -N; }
    double result = 1.0, base = x;
    while (N > 0) {
        if (N & 1) result *= base;
        base *= base;
        N >>= 1;
    }
    return result;
}
```

*[Interviewer]: "What if x is 0 and n is negative?"*
> "That's division by zero — `1/x` becomes infinity. LeetCode's constraints actually guarantee x != 0 when n < 0 implicitly through the answer bounds, but in a general-purpose library I'd add an explicit check and either throw or return a sentinel."

**RED FLAGS:**
- Not catching the INT_MIN overflow trap unprompted
- Writing O(n) iterative multiplication instead of O(log n)
- Forgetting to handle negative n at all (returning garbage for x^-3)

---

### Interview Simulation 2: LC 2514 — Count Anagrams (or equivalent "count arrangements mod p")

**Interviewer:** "Given a sentence, count how many strings you can form by permuting letters within each word independently. Answer mod 1e9+7."

**Candidate:**

*[State the combinatorial formula first, in plain English]*
> "For one word of length L, the number of distinct permutations is L! divided by the product of (count of each letter)!. This is the standard multinomial coefficient formula — it corrects for permutations that look identical because of repeated letters. I multiply this across all words since they're chosen independently."

*[Immediately flag WHY this needs modular inverse, not just modular multiplication]*
> "Since the answer needs to be mod 1e9+7, and the formula involves DIVISION by factorials, I can't just do `(L! / product) % MOD` — integer division doesn't distribute over mod like that. I need to precompute factorials and INVERSE factorials, and multiply by the inverse instead of dividing."

*[Explain the inverse computation approach]*
> "Since 1e9+7 is prime, I can compute each inverse via Fermat's little theorem: `a^(p-2) mod p`. For efficiency, I precompute factorials up to the max word length, then compute `invFact[max]` with ONE powMod call, and fill the rest downward using `invFact[i-1] = invFact[i] * i mod p` — avoids O(n) separate powMod calls."

*[Code — 6 minutes, using the precompute + per-word template]*

*[Interviewer]: "Why can't a naive `fact[L] / (c1! * c2! * ...)` work under the modulus?"*
> "Because modular arithmetic isn't a field under plain integer division — `/` mod p is only well-defined via multiplication by the modular inverse, which requires the divisor to be coprime with p. Since p is prime and none of these factorials are multiples of p (word lengths are far smaller than 1e9+7), every inverse exists — but I still must compute it explicitly via powMod, not via C++'s `/` operator."

**RED FLAGS:**
- Attempting `%` and `/` together naively without ever mentioning modular inverse
- Not recognizing this is fundamentally the same template as nCr mod p
- Recomputing `powMod` from scratch for every inverse instead of the O(1)-per-step downward-fill trick

---

## SECTION 17 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why does the Euclidean algorithm terminate, and why is it O(log(min(a,b)))?**

> It terminates because `b` strictly decreases every iteration (it becomes `a % b < b`), hitting 0 eventually — a is a non-negative integer, so this is a well-founded descent. The O(log) bound comes from the fact that `a % b < a / 2` whenever `b <= a / 2` (halves immediately), and when `b > a/2`, `a % b = a - b < a/2` (also halves). So every TWO steps, the larger value at least halves — giving O(log(min(a,b))) total iterations. The worst case (slowest convergence) occurs on consecutive Fibonacci numbers, where each step only barely reduces the pair — yet even then it's still O(log n).

---

**Q2: Why must you compute `a / gcd(a,b) * b` instead of `a * b / gcd(a,b)` for LCM?**

> Both are mathematically equivalent, but the FIRST avoids overflow: `a` divided by `gcd(a,b)` is guaranteed to be an integer ≤ `a` (often much smaller), so multiplying by `b` afterward keeps the intermediate value close to the final LCM (which is guaranteed to fit if the problem guarantees the answer fits). The SECOND computes `a*b` first — an intermediate value that can be far larger than the final LCM and can overflow even when the true LCM would have fit comfortably in the target type.

---

**Q3: You need to compute nCr mod m for many (n, r) pairs, but m is NOT guaranteed to be prime. What changes in your precompute?**

> You cannot use Fermat's little theorem for the inverse factorials (it only works for prime moduli). You'd need the extended Euclidean algorithm to compute each `invFact[i]` individually (since the "downward fill" trick `invFact[i-1] = invFact[i]*i` still works — it's just arithmetic, independent of primality — but the SEED value `invFact[MAXN-1]` must be computed via extended Euclid instead of `powMod(fact[MAXN-1], m-2, m)`). Also, you must verify `gcd(fact[MAXN-1], m) == 1` — if the factorial shares a factor with a composite `m`, no inverse exists at all, and you'd need a different approach entirely (e.g., computing nCr via prime-factorization/Legendre's formula and taking the count of each prime factor mod the prime powers dividing m, combined via CRT).

---

**Q4: In `powMod`, why do we square the base EVERY iteration, even when the current bit is 0?**

> Because the base always represents `x^(2^k)` at iteration `k` — this doubling must happen every step regardless of whether that particular power-of-two term is "used" in the final product. Skipping the squaring on a 0-bit would desynchronize `base` from the exponent's bit position, and the next 1-bit encountered would multiply in the WRONG power of x. The squaring tracks the exponent's bit position; the conditional multiplication is what actually selects which powers-of-two contribute to the final answer.

---

**Q5: Why does the linear sieve's inner loop `break` when `i % primes[j] == 0`?**

> This break enforces that every composite number is marked by EXACTLY its smallest prime factor, not by every prime factor. Once `primes[j]` divides `i`, continuing to the next larger prime `q` would compute `spf[q * i] = q` — but `q * i` actually has `primes[j]` (smaller than q) as a factor too (since `primes[j] | i`), meaning `primes[j]` is the TRUE smallest prime factor of `q*i`, not `q`. Continuing past this point would either give WRONG spf values or re-mark numbers that a smaller prime will correctly mark later — breaking the O(n) "each composite touched once" guarantee.

---

**Q6: Give a concrete example where `a^n mod p` computed via Fermat's little theorem inverse would silently give a WRONG answer.**

> Take `p = 8` (NOT prime), `a = 3`. `gcd(3, 8) = 1`, so an inverse of 3 mod 8 genuinely exists (it's 3, since `3*3=9≡1 mod 8`). Fermat's formula would compute `3^(8-2) mod 8 = 3^6 mod 8`. Compute: `3^2=9≡1 mod 8`, so `3^6 = (3^2)^3 ≡ 1^3 = 1 mod 8`. That gives 1, NOT 3 — the WRONG inverse (since `3 * 1 = 3 ≢ 1 mod 8`). Fermat's theorem simply doesn't apply when the modulus isn't prime; the formula `a^(p-2)` has no mathematical justification outside that context, and this example shows it produces silently plausible-looking but incorrect output.

---

**Q7: When counting via inclusion-exclusion over k conditions, why is the complexity O(2^k) and not O(k^2) or O(k!)?**

> Inclusion-exclusion requires evaluating the intersection size for EVERY non-empty subset of the k conditions — there are exactly `2^k - 1` such subsets (every condition is independently either "in" or "out" of a given subset, hence `2^k` total subsets including the empty one). This is fundamentally different from choosing PAIRS (O(k^2), which would only handle 2-way overlaps) or permutations (O(k!), irrelevant here since subsets are unordered). The `2^k` growth is why inclusion-exclusion is only practical for small k (typically ≤ 20, since `2^20 ≈ 1e6` is the practical ceiling for a fast per-subset computation).

---

## SECTION 18 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. GCD / LCM
// ─────────────────────────────────────────────────────────────────
long long g = std::gcd(a, b);              // C++17, <numeric>
long long l = (a / std::gcd(a,b)) * b;     // divide before multiply!

// ─────────────────────────────────────────────────────────────────
// 2. SIEVE OF ERATOSTHENES
// ─────────────────────────────────────────────────────────────────
vector<bool> isPrime(n+1, true);
isPrime[0] = isPrime[1] = false;
for (long long i = 2; i * i <= n; i++)
    if (isPrime[i])
        for (long long j = i*i; j <= n; j += i)
            isPrime[j] = false;

// ─────────────────────────────────────────────────────────────────
// 3. FAST POWER (MODULAR EXPONENTIATION)
// ─────────────────────────────────────────────────────────────────
long long powMod(long long base, long long exp, long long m) {
    base %= m; if (base < 0) base += m;
    long long result = 1;
    while (exp > 0) {
        if (exp & 1) result = (__int128)result * base % m;
        base = (__int128)base * base % m;
        exp >>= 1;
    }
    return result;
}

// ─────────────────────────────────────────────────────────────────
// 4. MODULAR INVERSE
// ─────────────────────────────────────────────────────────────────
// prime modulus:
long long inv = powMod(a, p - 2, p);
// any modulus (gcd(a,m)=1 required):
long long extendedGCD(long long a, long long b, long long &x, long long &y) {
    if (!b) { x = 1; y = 0; return a; }
    long long x1, y1, g = extendedGCD(b, a % b, x1, y1);
    x = y1; y = x1 - (a/b) * y1;
    return g;
}

// ─────────────────────────────────────────────────────────────────
// 5. nCr MOD p
// ─────────────────────────────────────────────────────────────────
fact[0] = 1;
for (int i = 1; i < N; i++) fact[i] = fact[i-1] * i % MOD;
invFact[N-1] = powMod(fact[N-1], MOD-2, MOD);
for (int i = N-1; i > 0; i--) invFact[i-1] = invFact[i] * i % MOD;
long long nCr(int n, int r) {
    if (r < 0 || r > n) return 0;
    return fact[n] * invFact[r] % MOD * invFact[n-r] % MOD;
}

// ─────────────────────────────────────────────────────────────────
// 6. PRIME FACTORIZATION (trial division)
// ─────────────────────────────────────────────────────────────────
vector<pair<long long,int>> factorize(long long n) {
    vector<pair<long long,int>> f;
    for (long long p = 2; p*p <= n; p++) {
        if (n % p == 0) {
            int e = 0;
            while (n % p == 0) { n /= p; e++; }
            f.push_back({p, e});
        }
    }
    if (n > 1) f.push_back({n, 1});
    return f;
}

// ─────────────────────────────────────────────────────────────────
// 7. DECISION TABLE
// ─────────────────────────────────────────────────────────────────
// Need primality/SPF for MANY numbers up to N?      -> sieve (linear if SPF needed)
// Need to factorize ONE big number (up to ~1e12)?    -> trial division to sqrt(n)
// Need a^n mod p, n huge?                             -> fast power O(log n)
// Need division under a modulus?                      -> modular inverse
//   modulus prime?      -> Fermat: powMod(a, p-2, p)
//   modulus composite?  -> extended Euclid
// Need nCr for many (n,r) queries, same modulus?      -> precompute fact + invFact
// Need "at least one of k conditions" count?           -> inclusion-exclusion (k small)
```

---

## SECTION 19 — WHAT TO DO AFTER THIS PATTERN

1. **Memorize the fast-power template until you can type it blind.** This is non-negotiable — it is the single most reused piece of code across this entire pattern, and shows up embedded inside Super Pow, Count Good Numbers, nCr mod p (via the inverse), and dozens of Codeforces problems.

2. **Solve all 10 problems in order.** Problems 7-9 (Super Pow, Largest Component Size, Count Anagrams) are the ones that separate 1600 from 2000 — they require composing this pattern with another (DSU for #8, modular inverse for #9).

3. **After LC 952, revisit Pattern 13 (Union-Find).** The "primes as virtual hub nodes" trick generalizes: any time you need to connect entities via a SHARED PROPERTY rather than direct pairwise edges, consider whether the property itself can become a DSU node.

4. **After nCr mod p, learn Lucas' Theorem.** When `n` or `r` can exceed the modulus `p` itself (so precomputed factorial tables aren't large enough), Lucas' theorem lets you compute `nCr mod p` by decomposing n and r into base-p digits — a natural extension once the base template is solid.

5. **Preview Pattern 25 (Digit DP):** Several digit DP problems require reporting counts mod a large prime, directly reusing Template 4 (modular arithmetic) inside the DP transition. The two patterns are frequently combined in Div 1 problems.

6. **Practice recognizing "hidden nCr":** Many DP problems (grid paths, distributing identical items into distinct bins, stars-and-bars counting) have closed-form combinatorial solutions that are O(1) or O(log n) instead of O(n^2) DP. After finishing this pattern, revisit 2-3 DP problems you've already solved and ask: "does this collapse into a binomial coefficient?"

---

## SECTION 20 — SIGN-OFF CRITERIA

### Tier 1 — Foundations mastered
You write GCD, LCM, and the Sieve of Eratosthenes from memory, with correct base cases (isPrime[0]=isPrime[1]=false) and the `i*i` optimization, in under 3 minutes combined.

### Tier 2 — Modular arithmetic solid
You write fast power (binary exponentiation) from memory in under 2 minutes. You can explain, without hesitation, why `(a*b) % mod` needs a `long long` cast and why LCM must divide before multiplying.

### Tier 3 — Inverse and combinatorics fluent
You correctly choose Fermat vs. extended Euclid for modular inverse depending on whether the modulus is prime, and can justify why in one sentence. You precompute factorials + inverse factorials and answer nCr mod p queries in O(1) each.

### Tier 4 — Composition with other patterns
You solve LC 952 (prime factorization + DSU) and LC 2514 (factorials + modular inverse in a real counting problem) without hints. You can explain the "primes as virtual hub nodes" trick and identify at least one other problem type where a shared-property DSU trick would apply.

**Sign-off threshold:** Solve 8 of 10 problems. Mandatory: LC 50 (Pow — INT_MIN and negative-exponent handling), LC 1922 or LC 372 (fast power applied to a non-trivial problem), and LC 952 (prime factorization combined with DSU). These three represent the three core competencies — safe fast power, applied fast power, and math-pattern composition — that this pattern exists to build.

---

## REFERENCES

- CP-Algorithms — Algebra section (GCD, modular arithmetic, exponentiation, inverses, combinatorics): https://cp-algorithms.com/#algebra
- *Competitive Programmer's Handbook* by Antti Laaksonen — Chapter 21 (Number Theory) and Chapter 22 (Combinatorics)

---

*Pattern 35 Complete — Math and Number Theory*
*Next: Pattern 36 — Bit Manipulation Deep Dive (XOR tricks, subset enumeration, bitmask DP)*
