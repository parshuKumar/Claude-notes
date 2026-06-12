# C++ STL — CP Cheat Sheet (Grandmaster Revision)

---

## SECTION 0 — Competition Template

```cpp
#include <bits/stdc++.h>
using namespace std;

// Types
typedef long long ll;
typedef long double ld;
typedef pair<int,int> pii;
typedef pair<ll,ll> pll;
typedef vector<int> vi;
typedef vector<ll> vll;
typedef vector<pii> vpii;

// Macros
#define pb push_back
#define eb emplace_back
#define mp make_pair
#define mt make_tuple
#define fi first
#define se second
#define all(x) (x).begin(),(x).end()
#define rall(x) (x).rbegin(),(x).rend()
#define sz(x) (int)(x).size()
#define rep(i,a,b) for(int i=(a);i<(b);i++)
#define per(i,a,b) for(int i=(b)-1;i>=(a);i--)
#define yes cout<<"YES\n"
#define no  cout<<"NO\n"

// Constants
const int MOD = 1e9+7;
const int MOD2 = 998244353;
const ll INF = 1e18;
const int INF32 = 2e9;
const double EPS = 1e-9;
const double PI = acos(-1.0);

// Fast I/O
int main(){
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);
    // cout.tie(NULL); // rarely needed
    int t; cin>>t;
    while(t--){ /* solve */ }
}
```

---

## SECTION 1 — Vector

```
VECTOR — CP: default container, dp tables, graphs, frequency arrays
──────────────────────────────────────────────────────────────────────
vector<int> v(n,0)            size n, all zeros
vector<int> v(n,x)            size n, all x
vector<vector<int>> g(n,vector<int>(m,0))   2D grid/dp O(n*m)
v.push_back(x)                append O(1) amort
v.emplace_back(x)             construct in-place O(1) amort, faster than pb
v.pop_back()                  remove end O(1)
v.insert(it,x)                insert before it O(n) ⚠️
v.erase(it)                   erase at it O(n) ⚠️
v.erase(it1,it2)              erase range [it1,it2) O(n) ⚠️
v.size()                      element count O(1) → returns size_t ⚠️
v.empty()                     true if size==0 O(1)
v.front() / v.back()          first/last element O(1)
v.begin() / v.end()           iterators
v.rbegin() / v.rend()         reverse iterators
v.clear()                     remove all, capacity unchanged O(n)
v.resize(n)                   change size, zero-fills new O(n)
v.reserve(n)                  pre-alloc n capacity, no size change O(n)
v.assign(n,x)                 replace with n copies of x O(n)
v.swap(u)                     swap two vectors O(1)
v.at(i)                       bounds-checked access O(1) ⚠️ throws
v[i]                          unchecked access O(1)
sort(all(v))                  sort ascending O(n log n)
sort(all(v),greater<int>())   sort descending O(n log n)
reverse(all(v))               reverse in-place O(n)
unique(all(v))                remove consecutive dupes, returns new end ⚠️ sort first

PREFER WHEN: default dynamic array, DP, BFS levels, adjacency lists
AVOID WHEN: need O(log n) ordered lookup → set/map; O(1) front ops → deque
```

---

## SECTION 2 — String

```
STRING — CP: parsing, pattern matching, number conversion, hashing
──────────────────────────────────────────────────────────────────────
s.length() / s.size()         char count O(1)
s.empty()                     true if "" O(1)
s.substr(pos,len)             substring O(len) ⚠️ pos out of range = UB
s.find(t)                     first pos of t, string::npos if absent O(n*m)
s.rfind(t)                    last pos of t O(n*m)
s.find(t,pos)                 search starting at pos
s.replace(pos,len,t)          replace len chars at pos with t O(n)
s.append(t) / s+=t            append O(len) ⚠️ s=s+t is O(n) per op
s.push_back(c)                append char O(1) amort
s.pop_back()                  remove last char O(1)
s.insert(pos,t)               insert string t at pos O(n)
s.erase(pos,len)              erase len chars at pos O(n)
s.clear()                     empty the string O(n)
s[i]                          char access O(1)
s.compare(t)                  0 if equal, <0 if s<t O(n)
s < t                         lexicographic compare O(n)
to_string(x)                  int/ll→string O(digits)
stoi(s) / stol(s) / stoll(s)  string→int/long/long long ⚠️ no bounds check
stod(s) / stof(s)             string→double/float
reverse(all(s))               reverse in-place O(n)
sort(all(s))                  sort chars O(n log n)
string(n,c)                   string of n copies of char c

// Char utilities (cctype header, included in bits/stdc++.h)
isalpha(c) isdigit(c) isalnum(c) isspace(c) islower(c) isupper(c)  → bool
tolower(c) toupper(c)          char→lower/upper ⚠️ cast to char

// Char array interop
string s(arr)                 char* → string
s.c_str()                     string → const char* O(1)
strcpy(dst,s.c_str())         copy to char array

PREFER WHEN: text manipulation, parsing, character frequency
AVOID WHEN: heavy concatenation in loop → use stringstream or reserve ⚠️
```

---

## SECTION 3 — Pair and Tuple

```
PAIR/TUPLE — CP: edge lists, coordinates, sorting multi-criteria
──────────────────────────────────────────────────────────────────────
make_pair(a,b) / {a,b}        create pair (C++11 brace init)
p.first / p.second            access members
pair comparison               lex: first compared, then second
sort(all(v)) on vpii          sorts by first, then second by default

// Sort by second element
sort(all(v),[](auto& a,auto& b){ return a.second < b.second; });

// Tuple
make_tuple(a,b,c) / {a,b,c}   create tuple
get<0>(t) get<1>(t) get<2>(t) access by index
tie(a,b,c) = t                unpack tuple into variables
auto [x,y,z] = t              C++17 structured bindings

PREFER WHEN: need to bundle 2-3 values without a struct
AVOID WHEN: >3 values → define a struct for readability
```

---

## SECTION 4 — Stack

```
STACK — CP: DFS iterative, expression eval, monotonic stack, next-greater
──────────────────────────────────────────────────────────────────────
stack<int> st;
st.push(x)                    push top O(1)
st.pop()                      remove top O(1) ⚠️ UB if empty
st.top()                      peek top O(1) ⚠️ UB if empty
st.empty()                    true if empty O(1)
st.size()                     element count O(1)

PREFER WHEN: LIFO order, iterative DFS, monotonic stack problems
AVOID WHEN: need access to middle elements → use deque/vector
```

---

## SECTION 5 — Queue

```
QUEUE — CP: BFS, level-order traversal, simulation
──────────────────────────────────────────────────────────────────────
queue<int> q;
q.push(x)                     enqueue back O(1)
q.pop()                       dequeue front O(1) ⚠️ UB if empty
q.front()                     peek front O(1) ⚠️ UB if empty
q.back()                      peek back O(1)
q.empty()                     true if empty O(1)
q.size()                      element count O(1)

PREFER WHEN: BFS, FIFO processing
AVOID WHEN: need both-end access → deque; need priority → priority_queue
```

---

## SECTION 6 — Deque

```
DEQUE — CP: sliding window, monotonic deque, palindrome checks
──────────────────────────────────────────────────────────────────────
deque<int> dq;
dq.push_front(x)              O(1)
dq.push_back(x)               O(1)
dq.pop_front()                O(1) ⚠️ UB if empty
dq.pop_back()                 O(1) ⚠️ UB if empty
dq.front() / dq.back()        O(1)
dq[i]                         random access O(1) slower than vector
dq.size() / dq.empty()        O(1)
dq.insert(it,x)               O(n) ⚠️
dq.erase(it)                  O(n) ⚠️

// Monotonic deque sliding window max (window size k):
deque<int> dq; // stores indices
for(int i=0;i<n;i++){
    while(!dq.empty()&&dq.front()<i-k+1) dq.pop_front();
    while(!dq.empty()&&a[dq.back()]<=a[i]) dq.pop_back();
    dq.push_back(i);
    if(i>=k-1) ans[i-k+1]=a[dq.front()];
}

PREFER WHEN: need O(1) at both ends; sliding window max/min
AVOID WHEN: only need one-end ops → stack/queue; cache perf matters → vector
```

---

## SECTION 7 — Priority Queue (Heap)

```
PRIORITY QUEUE — CP: Dijkstra, greedy scheduling, top-K, median
──────────────────────────────────────────────────────────────────────
priority_queue<int> pq;               max-heap (default)
priority_queue<int,vector<int>,greater<int>> pq;  min-heap ⚠️ syntax!
pq.push(x)                    insert O(log n)
pq.pop()                      remove top O(log n) ⚠️ UB if empty
pq.top()                      peek top O(1) ⚠️ UB if empty
pq.empty()                    O(1)
pq.size()                     O(1)

// Custom comparator (min-heap of pairs by second):
auto cmp=[](pii& a,pii& b){ return a.second>b.second; };
priority_queue<pii,vector<pii>,decltype(cmp)> pq(cmp);

// Dijkstra pattern:
priority_queue<pii,vector<pii>,greater<pii>> pq; // {dist,node}
pq.push({0,src}); dist[src]=0;
while(!pq.empty()){
    auto [d,u]=pq.top(); pq.pop();
    if(d>dist[u]) continue;
    for(auto [v,w]:adj[u]) if(dist[u]+w<dist[v]){
        dist[v]=dist[u]+w; pq.push({dist[v],v});
    }
}

PREFER WHEN: always-need-min/max from dynamic set; Dijkstra
AVOID WHEN: need arbitrary element access/delete → set with iterators
```

---

## SECTION 8 — Set and Multiset

```
SET/MULTISET — CP: unique sorted elements, order stats, range queries
──────────────────────────────────────────────────────────────────────
set<int> s;
s.insert(x)                   O(log n)
s.erase(x)                    erase ALL copies by value ⚠️
s.erase(s.find(x))            erase ONE copy (critical for multiset) ⚠️
s.find(x)                     iterator, s.end() if absent O(log n)
s.count(x)                    0 or 1 for set; ≥0 for multiset O(log n)
s.contains(x)                 C++20, bool O(log n)
s.size() / s.empty()          O(1)
s.begin() / s.end()           min/max iterators
*s.begin()                    minimum O(1) via iterator
*s.rbegin()                   maximum O(1) via iterator
s.lower_bound(x)              first it >= x O(log n)
s.upper_bound(x)              first it > x  O(log n)
// ⚠️ Use s.lower_bound(x) NOT std::lower_bound(all(s),x) — same result but member is O(log n)

multiset<int> ms;             allows duplicates; same interface
ms.erase(ms.find(x))          remove ONE occurrence ⚠️ ms.erase(x) removes ALL

// Policy-based ordered set (CP secret weapon — O(log n) rank queries):
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/tree_policy.hpp>
using namespace __gnu_pbds;
typedef tree<int,null_type,less<int>,rb_tree_tag,tree_order_statistics_node_update> ordered_set;
ordered_set os;
os.insert(x)                  insert O(log n)
os.find_by_order(k)           iterator to k-th element (0-indexed) O(log n)
os.order_of_key(x)            # elements strictly < x O(log n)
// ⚠️ No duplicate keys — use ordered_set<pair<int,int>> with unique pairs

PREFER WHEN: sorted unique elements, predecessor/successor queries
AVOID WHEN: only need membership check → unordered_set O(1); duplicates+rank → multiset+BIT
```

---

## SECTION 9 — Map and Multimap

```
MAP — CP: frequency count, memoization, coordinate→value mapping
──────────────────────────────────────────────────────────────────────
map<int,int> m;
m.insert({k,v}) / m[k]=v      insert/update O(log n)
m.erase(k)                    erase by key O(log n)
m.erase(it)                   erase by iterator O(1) amort
m.find(k)                     iterator, m.end() if absent O(log n)
m.count(k)                    0 or 1 O(log n)
m.contains(k)                 C++20 bool O(log n)
m[k]                          access; CREATES key with default 0 if absent ⚠️
m.at(k)                       access; throws if key absent O(log n)
m.size() / m.empty()          O(1)
m.begin() / m.end()           sorted order by key
m.lower_bound(k)              first entry with key >= k O(log n)
m.upper_bound(k)              first entry with key >  k O(log n)
// Iterate with structured bindings (C++17):
for(auto& [k,v] : m) { ... }

multimap<int,int> mm;         allows duplicate keys; no operator[]
mm.equal_range(k)             pair of iterators for all entries with key k

// map vs unordered_map:
// map: O(log n) all ops, ordered, safe on CF
// unordered_map: O(1) avg, O(n) worst, hackable on CF ⚠️ → use custom hash

PREFER WHEN: ordered key traversal, range queries on keys, small n
AVOID WHEN: just need O(1) lookup and no ordering needed → unordered_map
```

---

## SECTION 10 — Unordered Map / Unordered Set

```
UNORDERED MAP/SET — CP: O(1) freq count, memoization when speed critical
──────────────────────────────────────────────────────────────────────
unordered_map<int,int> um;
unordered_set<int> us;
// All map/set methods apply; O(1) avg, O(n) worst ⚠️

// ⚠️ CODEFORCES ANTI-HACK — splitmix64 custom hash (MANDATORY on CF):
struct custom_hash {
    static uint64_t splitmix64(uint64_t x){
        x+=0x9e3779b97f4a7c15; x=(x^(x>>30))*0xbf58476d1ce4e5b9;
        x=(x^(x>>27))*0x94d049bb133111eb; return x^(x>>31);
    }
    size_t operator()(uint64_t x) const {
        static const uint64_t FIXED=chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x+FIXED);
    }
};
unordered_map<int,int,custom_hash> safe_map;
unordered_set<int,custom_hash> safe_set;

// Performance tricks:
um.reserve(1<<17);            pre-alloc to avoid rehash
um.max_load_factor(0.25);     reduce collision rate

PREFER WHEN: O(1) lookup/insert, large n, key ordering not needed
AVOID WHEN: CF hacks possible without custom hash ⚠️; keys need ordering → map
```

---

## SECTION 11 — STL Algorithms

```
ALGORITHMS — CP: core toolkit for every problem type
──────────────────────────────────────────────────────────────────────
// SORTING
sort(all(v))                              O(n log n) unstable introsort
sort(all(v),greater<int>())               descending
sort(all(v),[](int a,int b){return a>b;}) lambda comparator
stable_sort(all(v))                       O(n log n) stable
partial_sort(v.begin(),v.begin()+k,v.end()) top-k sorted O(n log k)
nth_element(all(v),v.begin()+k)           k-th element in O(n) avg ⚠️ underused gem

// BINARY SEARCH (on sorted range)
binary_search(all(v),x)                   bool only ⚠️ use lower_bound for position
lower_bound(all(v),x)                     iterator to first >= x O(log n)
upper_bound(all(v),x)                     iterator to first >  x O(log n)
equal_range(all(v),x)                     pair<it,it> of [lower,upper) O(log n)
// Index from iterator: int idx = lower_bound(all(v),x)-v.begin();

// SEARCH
find(all(v),x)                            iterator to first x, end if absent O(n)
find_if(all(v),[](int x){return x>5;})    first satisfying pred O(n)
count(all(v),x)                           count occurrences O(n)
count_if(all(v),pred)                     count satisfying pred O(n)

// MIN / MAX
min(a,b) / max(a,b)                       O(1)
min({a,b,c}) / max({a,b,c})               initializer list variant O(k)
min_element(all(v))                       iterator to min O(n)
max_element(all(v))                       iterator to max O(n)
minmax(a,b)                               pair{min,max} O(1)
minmax_element(all(v))                    pair{min_it,max_it} O(n)

// PERMUTATION
next_permutation(all(v))                  next lex perm, false if last O(n)
prev_permutation(all(v))                  prev lex perm O(n)

// NUMERIC (numeric header)
accumulate(all(v),0LL)                    sum O(n) ⚠️ init type matters for ll
partial_sum(all(v),out.begin())           prefix sums O(n)
inner_product(a.begin(),a.end(),b.begin(),0LL)  dot product O(n)
iota(all(v),0)                            fill 0,1,2,... O(n)
gcd(a,b) / lcm(a,b)                       C++17 O(log min)
__gcd(a,b)                                pre-C++17 ⚠️ returns 0 if both 0

// MANIPULATION
reverse(all(v))                           O(n)
rotate(v.begin(),v.begin()+k,v.end())     left rotate by k O(n)
unique(all(v))                            remove consec dupes, returns new end ⚠️ sort first
// Erase dupes: v.erase(unique(all(v)),v.end());
remove(all(v),x)                          move non-x forward, returns new end ⚠️ doesn't resize
// Erase: v.erase(remove(all(v),x),v.end()); // erase-remove idiom
fill(all(v),x)                            fill with x O(n)
fill_n(v.begin(),n,x)                     fill n elements O(n)
copy(all(v),out.begin())                  copy range O(n)
swap(a,b)                                 O(1) for containers

// SET OPS (on sorted ranges)
set_union(all(a),all(b),back_inserter(c))        O(n+m)
set_intersection(all(a),all(b),back_inserter(c)) O(n+m)
set_difference(all(a),all(b),back_inserter(c))   O(n+m)
includes(all(a),all(b))                          bool: b⊆a O(n+m)
```

---

## SECTION 12 — Numeric Utilities

```
NUMERIC UTILITIES — CP: overflow safety, bit tricks, math ops
──────────────────────────────────────────────────────────────────────
// INTEGER LIMITS
INT_MAX = 2147483647 (~2e9)    INT_MIN = -2147483648
LLONG_MAX = 9223372036854775807 (~9.2e18)   LLONG_MIN
// ⚠️ LLONG_MAX+1 overflows! Use INF=1e18 or 4e18

// BIT MANIPULATION
__builtin_popcount(x)          set bit count (int) O(1)
__builtin_popcountll(x)        set bit count (ll) O(1)
__builtin_clz(x)               leading zeros (int, ⚠️ UB if x==0)
__builtin_ctz(x)               trailing zeros (int, ⚠️ UB if x==0)
__builtin_parity(x)            parity of set bits O(1)
x & (x-1)                      clear lowest set bit; x&(x-1)==0 iff power of 2
x & (-x)                       isolate lowest set bit (LSB)
x | (x-1)                      set all bits at/below LSB
(x >> k) & 1                   check k-th bit
x | (1 << k)                   set k-th bit
x & ~(1 << k)                  clear k-th bit
x ^ (1 << k)                   toggle k-th bit
1 << k                         ⚠️ use 1LL << k if k >= 31

// MATH
abs(x) / llabs(x)              ⚠️ abs on ll needs llabs or cast; or use -x<0?-x:x
fabs(x)                        float abs
pow(base,exp)                  ⚠️ returns double; use modpow for integers
sqrt(x)                        ⚠️ float error; use (ll)sqrt(x) then verify ±1
ceil(x) / floor(x) / round(x)  ⚠️ float ops; integer ceil: (a+b-1)/b
log2(x)                        ⚠️ float; integer floor log2: 31-__builtin_clz(x)
log(x) / log10(x)              natural/base-10 log

// __int128 (when ll overflows):
__int128 x = (__int128)a * b;
// ⚠️ No cin/cout support — print manually:
void print128(__int128 x){
    if(x<0){putchar('-');x=-x;}
    if(x>9) print128(x/10);
    putchar('0'+x%10);
}
```

---

## SECTION 13 — Essential CP Patterns

```cpp
// COORDINATE COMPRESSION
vector<int> vals(all(a)); sort(all(vals)); vals.erase(unique(all(vals)),vals.end());
auto compress=[&](int x){ return lower_bound(all(vals),x)-vals.begin(); };

// FREQUENCY MAP
unordered_map<int,int,custom_hash> freq;
for(int x:a) freq[x]++;

// ADJACENCY LIST
vector<vector<pair<int,int>>> adj(n); // adj[u]={v,weight}
adj[u].push_back({v,w});

// BFS TEMPLATE
vi dist(n,-1); queue<int> q;
dist[src]=0; q.push(src);
while(!q.empty()){
    int u=q.front(); q.pop();
    for(auto [v,w]:adj[u]) if(dist[v]==-1){ dist[v]=dist[u]+1; q.push(v); }
}

// DFS RECURSIVE
void dfs(int u,int p){ vis[u]=1; for(auto [v,w]:adj[u]) if(!vis[v]) dfs(v,u); }

// DFS ITERATIVE
stack<int> st; st.push(src); vis[src]=1;
while(!st.empty()){ int u=st.top(); st.pop(); for(auto [v,w]:adj[u]) if(!vis[v]){vis[v]=1;st.push(v);} }

// DIJKSTRA
vll dist(n,INF); priority_queue<pll,vector<pll>,greater<pll>> pq;
dist[src]=0; pq.push({0,src});
while(!pq.empty()){ auto [d,u]=pq.top(); pq.pop(); if(d>dist[u]) continue;
    for(auto [v,w]:adj[u]) if(dist[u]+w<dist[v]){ dist[v]=dist[u]+w; pq.push({dist[v],v}); } }

// UNION-FIND (DSU) — path compression + union by rank
struct DSU {
    vi par,rnk;
    DSU(int n):par(n),rnk(n,0){ iota(all(par),0); }
    int find(int x){ return par[x]==x?x:par[x]=find(par[x]); }
    bool unite(int a,int b){ a=find(a);b=find(b); if(a==b)return false;
        if(rnk[a]<rnk[b]) swap(a,b); par[b]=a; if(rnk[a]==rnk[b]) rnk[a]++; return true; }
    bool same(int a,int b){ return find(a)==find(b); }
};

// BINARY SEARCH — leftmost condition (find first x where cond(x) true)
int lo=0,hi=n-1,ans=-1;
while(lo<=hi){ int mid=lo+(hi-lo)/2; if(cond(mid)){ans=mid;hi=mid-1;} else lo=mid+1; }

// BINARY SEARCH ON ANSWER
int lo=0,hi=1e9,ans=0;
while(lo<=hi){ int mid=lo+(hi-lo)/2; if(feasible(mid)){ans=mid;lo=mid+1;} else hi=mid-1; }

// PREFIX SUM 1D
vi pre(n+1,0); for(int i=0;i<n;i++) pre[i+1]=pre[i]+a[i];
// sum [l,r] = pre[r+1]-pre[l]

// PREFIX SUM 2D
vector<vll> pre(n+1,vll(m+1,0));
for(int i=1;i<=n;i++) for(int j=1;j<=m;j++) pre[i][j]=a[i-1][j-1]+pre[i-1][j]+pre[i][j-1]-pre[i-1][j-1];
// sum [r1,c1]..[r2,c2] = pre[r2+1][c2+1]-pre[r1][c2+1]-pre[r2+1][c1]+pre[r1][c1]

// SPARSE TABLE (RMQ, O(n log n) build, O(1) query)
int LOG=20; vector<vi> sp(LOG,vi(n));
sp[0]=a; for(int j=1;j<LOG;j++) for(int i=0;i+(1<<j)<=n;i++) sp[j][i]=min(sp[j-1][i],sp[j-1][i+(1<<(j-1))]);
auto rmq=[&](int l,int r){ int k=__lg(r-l+1); return min(sp[k][l],sp[k][r-(1<<k)+1]); };

// SIEVE OF ERATOSTHENES
const int MAXN=1e6+5; vector<bool> is_prime(MAXN,true); vi primes;
is_prime[0]=is_prime[1]=false;
for(int i=2;i<MAXN;i++){ if(is_prime[i]){ primes.pb(i); for(ll j=(ll)i*i;j<MAXN;j+=i) is_prime[j]=false; } }

// MODULAR EXPONENTIATION
ll modpow(ll base,ll exp,ll mod){ ll res=1; base%=mod;
    while(exp>0){ if(exp&1) res=res*base%mod; base=base*base%mod; exp>>=1; } return res; }

// MODULAR INVERSE (Fermat — mod must be prime)
ll modinv(ll a,ll mod){ return modpow(a,mod-2,mod); }

// nCr MOD P
const int MAXF=2e6+5; vll fact(MAXF),inv_fact(MAXF);
void precompute(){ fact[0]=1; for(int i=1;i<MAXF;i++) fact[i]=fact[i-1]*i%MOD;
    inv_fact[MAXF-1]=modinv(fact[MAXF-1],MOD);
    for(int i=MAXF-2;i>=0;i--) inv_fact[i]=inv_fact[i+1]*(i+1)%MOD; }
ll C(int n,int r){ if(r<0||r>n) return 0; return fact[n]%MOD*inv_fact[r]%MOD*inv_fact[n-r]%MOD; }
```

---

## SECTION 14 — CP Gotchas and Common Bugs

```
BUG → FIX
──────────────────────────────────────────────────────────────────────
int*int overflow           → cast: (ll)a*b; or use ll from start
endl in tight loop         → use "\n"; endl flushes buffer O(1)→O(flush) ⚠️
int i; i<v.size()          → size() returns size_t; i<(int)v.size() or use sz(v) ⚠️
binary_search returns bool → use lower_bound for position ⚠️
unique without sort        → sort first; unique only removes consecutive dupes ⚠️
multiset.erase(val)        → erases ALL copies; use erase(ms.find(val)) ⚠️
pq min-heap syntax         → priority_queue<int,vector<int>,greater<int>> ⚠️
unordered_map on CF        → use custom_hash (splitmix64) to prevent hack ⚠️
map[key] on const map      → use .at(key) or .find(key); [] inserts ⚠️
not clearing global arrays → memset(arr,0,sizeof(arr)) or re-init each testcase ⚠️
float == comparison        → use fabs(a-b)<EPS ⚠️
sqrt float error           → (ll)sqrt(x); verify with (ans-1)^2,(ans)^2,(ans+1)^2 ⚠️
1<<k for k>=31             → use 1LL<<k ⚠️
deep recursion stack OVF   → increase stack / convert to iterative ⚠️
off-by-one binary search   → use lo+(hi-lo)/2; test on n=1,2 cases ⚠️
reading input order        → cin reads left-to-right; function arg order UB in C++ ⚠️
signed/unsigned compare    → warning-as-bug; always cast ⚠️
global array auto-zeroed   → local arrays are NOT zero-initialized ⚠️
modular arithmetic neg     → ((a%MOD)+MOD)%MOD to handle negatives ⚠️
LLONG_MAX + anything       → overflow; use INF=4e18 or 1e18 ⚠️
stringstream reuse         → ss.str(""); ss.clear(); to reset ⚠️
lower_bound on set         → use s.lower_bound(x) not std::lower_bound(all(s),x) — both O(log n) but member preferred ⚠️
__builtin_clz(0)           → UB; always check x!=0 first ⚠️
partial_sum output range   → output range must have enough space ⚠️
accumulate with int init   → accumulate(all(v),0LL) for ll sum; 0 truncates ⚠️
pow(2,31) as int           → returns double; use 1LL<<31 ⚠️
multiple test cases        → reset ALL state: vis[], dist[], DSU, memo, ans ⚠️
```

---

*One file. Open five minutes before contest. Win.*
