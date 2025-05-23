# Understanding CRDTs: Conflict-free Replicated Data Types

## What are CRDTs?

CRDTs (Conflict-free Replicated Data Types) are data structures that can be replicated across multiple nodes in a distributed system, where each replica can be updated independently and concurrently without coordination between replicas. The key property is that when replicas are eventually merged, they always converge to the same state regardless of the order of operations.

## The Core Problem CRDTs Solve

**Traditional Distributed Systems Problem:**
```
Node A: counter = 5
Node B: counter = 5

Node A increments: counter = 6
Node B increments: counter = 6

When they sync: What should the final value be?
- Last-writer-wins: 6 (we lose one increment!)
- Both increments: 7 (correct!)
```

CRDTs guarantee that concurrent operations can be merged without conflicts or data loss.

## Mathematical Foundation

CRDTs rely on **semilattices** - algebraic structures where:
1. **Associative**: (a ⊔ b) ⊔ c = a ⊔ (b ⊔ c)
2. **Commutative**: a ⊔ b = b ⊔ a  
3. **Idempotent**: a ⊔ a = a

The merge operation (⊔) ensures convergence regardless of operation order.

## Types of CRDTs

### 1. State-based CRDTs (CvRDTs)
Replicas exchange entire state and merge using a join function.

### 2. Operation-based CRDTs (CmRDTs)  
Replicas exchange operations that are applied in causal order.

## Common CRDT Implementations

### G-Counter (Grow-only Counter)

**Concept**: A counter that can only increment, implemented as a vector of per-replica counters.

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct GCounter {
    counts: HashMap<String, u64>, // replica_id -> count
    replica_id: String,
}

impl GCounter {
    fn new(replica_id: String) -> Self {
        let mut counts = HashMap::new();
        counts.insert(replica_id.clone(), 0);
        Self { counts, replica_id }
    }
    
    // Increment this replica's counter
    fn increment(&mut self) {
        *self.counts.entry(self.replica_id.clone()).or_insert(0) += 1;
    }
    
    // Get total value across all replicas
    fn value(&self) -> u64 {
        self.counts.values().sum()
    }
    
    // Merge with another replica's state
    fn merge(&mut self, other: &GCounter) {
        for (replica, &count) in &other.counts {
            let current = self.counts.get(replica).unwrap_or(&0);
            self.counts.insert(replica.clone(), (*current).max(count));
        }
    }
}

// Example usage
fn main() {
    let mut counter_a = GCounter::new("replica_a".to_string());
    let mut counter_b = GCounter::new("replica_b".to_string());
    
    // Both replicas increment independently
    counter_a.increment();
    counter_a.increment();
    counter_b.increment();
    
    println!("Counter A value: {}", counter_a.value()); // 2
    println!("Counter B value: {}", counter_b.value()); // 1
    
    // Merge states
    counter_a.merge(&counter_b);
    counter_b.merge(&counter_a);
    
    println!("After merge - A: {}", counter_a.value()); // 3
    println!("After merge - B: {}", counter_b.value()); // 3
}
```

### OR-Set (Observed-Remove Set)

**Concept**: A set that supports both additions and removals by tagging each element with unique identifiers.

```rust
use std::collections::{HashMap, HashSet};
use uuid::Uuid;

#[derive(Debug, Clone)]
struct ORSet<T: Clone + Eq + std::hash::Hash> {
    added: HashMap<T, HashSet<Uuid>>, // element -> set of add tags
    removed: HashSet<Uuid>,           // set of removed tags
}

impl<T: Clone + Eq + std::hash::Hash> ORSet<T> {
    fn new() -> Self {
        Self {
            added: HashMap::new(),
            removed: HashSet::new(),
        }
    }
    
    // Add element with unique tag
    fn add(&mut self, element: T) -> Uuid {
        let tag = Uuid::new_v4();
        self.added.entry(element).or_insert_with(HashSet::new).insert(tag);
        tag
    }
    
    // Remove element by removing all its current tags
    fn remove(&mut self, element: &T) {
        if let Some(tags) = self.added.get(element) {
            for &tag in tags {
                self.removed.insert(tag);
            }
        }
    }
    
    // Check if element is in set
    fn contains(&self, element: &T) -> bool {
        if let Some(tags) = self.added.get(element) {
            tags.iter().any(|tag| !self.removed.contains(tag))
        } else {
            false
        }
    }
    
    // Get all elements currently in set
    fn elements(&self) -> Vec<T> {
        self.added.iter()
            .filter_map(|(element, tags)| {
                if tags.iter().any(|tag| !self.removed.contains(tag)) {
                    Some(element.clone())
                } else {
                    None
                }
            })
            .collect()
    }
    
    // Merge with another OR-Set
    fn merge(&mut self, other: &ORSet<T>) {
        // Merge added elements
        for (element, tags) in &other.added {
            let entry = self.added.entry(element.clone()).or_insert_with(HashSet::new);
            for &tag in tags {
                entry.insert(tag);
            }
        }
        
        // Merge removed tags
        for &tag in &other.removed {
            self.removed.insert(tag);
        }
    }
}

// Example usage
fn main() {
    let mut set_a = ORSet::new();
    let mut set_b = ORSet::new();
    
    // Both replicas add elements
    set_a.add("apple".to_string());
    set_a.add("banana".to_string());
    set_b.add("cherry".to_string());
    
    // Replica A removes banana
    set_a.remove(&"banana".to_string());
    
    println!("Set A: {:?}", set_a.elements()); // ["apple"]
    println!("Set B: {:?}", set_b.elements()); // ["cherry"]
    
    // Merge states
    set_a.merge(&set_b);
    set_b.merge(&set_a);
    
    println!("After merge - A: {:?}", set_a.elements()); // ["apple", "cherry"]
    println!("After merge - B: {:?}", set_b.elements()); // ["apple", "cherry"]
}
```

### LWW-Map (Last-Writer-Wins Map)

**Concept**: A key-value map where each update includes a timestamp, and conflicts are resolved by choosing the most recent write.

```rust
use std::collections::HashMap;
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug, Clone)]
struct LWWMap<K, V> 
where 
    K: Clone + Eq + std::hash::Hash,
    V: Clone,
{
    data: HashMap<K, (V, u64)>, // key -> (value, timestamp)
}

impl<K, V> LWWMap<K, V>
where
    K: Clone + Eq + std::hash::Hash,
    V: Clone,
{
    fn new() -> Self {
        Self {
            data: HashMap::new(),
        }
    }
    
    fn current_timestamp() -> u64 {
        SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_millis() as u64
    }
    
    // Set key-value pair with current timestamp
    fn set(&mut self, key: K, value: V) {
        let timestamp = Self::current_timestamp();
        self.set_with_timestamp(key, value, timestamp);
    }
    
    // Set with explicit timestamp (useful for testing)
    fn set_with_timestamp(&mut self, key: K, value: V, timestamp: u64) {
        match self.data.get(&key) {
            Some((_, existing_timestamp)) => {
                if timestamp >= *existing_timestamp {
                    self.data.insert(key, (value, timestamp));
                }
            }
            None => {
                self.data.insert(key, (value, timestamp));
            }
        }
    }
    
    // Get value for key
    fn get(&self, key: &K) -> Option<&V> {
        self.data.get(key).map(|(value, _)| value)
    }
    
    // Merge with another LWW-Map
    fn merge(&mut self, other: &LWWMap<K, V>) {
        for (key, (value, timestamp)) in &other.data {
            self.set_with_timestamp(key.clone(), value.clone(), *timestamp);
        }
    }
}

// Example usage
fn main() {
    let mut map_a = LWWMap::new();
    let mut map_b = LWWMap::new();
    
    // Both replicas update independently
    map_a.set_with_timestamp("user1".to_string(), "Alice".to_string(), 1000);
    map_b.set_with_timestamp("user1".to_string(), "Bob".to_string(), 2000);
    map_a.set_with_timestamp("user2".to_string(), "Charlie".to_string(), 1500);
    
    println!("Map A user1: {:?}", map_a.get(&"user1".to_string())); // Some("Alice")
    println!("Map B user1: {:?}", map_b.get(&"user1".to_string())); // Some("Bob")
    
    // Merge states - Bob wins because timestamp 2000 > 1000
    map_a.merge(&map_b);
    map_b.merge(&map_a);
    
    println!("After merge - A user1: {:?}", map_a.get(&"user1".to_string())); // Some("Bob")
    println!("After merge - B user1: {:?}", map_b.get(&"user1".to_string())); // Some("Bob")
}
```

## CRDT Properties and Guarantees

### Strong Eventual Consistency (SEC)
1. **Eventual Delivery**: All updates are eventually delivered to all replicas
2. **Convergence**: Replicas that have received the same set of updates have equivalent state
3. **Termination**: All method executions terminate

### Mathematical Properties
- **Monotonicity**: State can only grow (for state-based CRDTs)
- **Commutativity**: Operation order doesn't matter for final result
- **Associativity**: Grouping of operations doesn't matter

## Challenges and Limitations

### Memory Growth
CRDTs often grow monotonically - they accumulate metadata that can't be garbage collected without coordination.

**Example**: OR-Set keeps all addition tags forever to handle late removals.

### Semantic Limitations
Not all data types can be made into CRDTs. Counter subtraction, for example, requires careful design (PN-Counter with separate increment/decrement counters).

### Network Overhead
State-based CRDTs may transfer large amounts of data during synchronization.

## CRDTs in the ZK Context (Prism's Challenge)

The Prism proposal faces unique challenges adapting CRDTs for zero-knowledge circuits:

### Circuit Constraint Issues
```rust
// Traditional CRDT merge operation
fn merge_gcounter(local: &HashMap<String, u64>, remote: &HashMap<String, u64>) -> HashMap<String, u64> {
    let mut result = local.clone();
    for (replica, &count) in remote {
        let current = result.get(replica).unwrap_or(&0);
        result.insert(replica.clone(), (*current).max(count)); // max() is expensive in circuits
    }
    result
}
```

**ZK-Specific Optimizations Needed:**
1. **Fixed-size structures** instead of dynamic HashMap
2. **Field arithmetic** instead of arbitrary precision integers
3. **Merkle tree commitments** instead of full state replication
4. **Batched operations** to amortize proof generation costs

### Example ZK-Optimized G-Counter
```rust
// Simplified ZK-friendly version
struct ZKGCounter {
    counts: [u64; MAX_REPLICAS], // Fixed-size array
    replica_mask: u64,           // Bitfield for active replicas
}

impl ZKGCounter {
    // Merge operation optimized for circuit constraints
    fn merge_in_circuit(&self, other: &ZKGCounter) -> ZKGCounter {
        let mut result = *self;
        for i in 0..MAX_REPLICAS {
            // Use conditional assignment instead of max()
            let should_update = other.counts[i] > result.counts[i];
            result.counts[i] = if should_update { 
                other.counts[i] 
            } else { 
                result.counts[i] 
            };
        }
        result.replica_mask |= other.replica_mask;
        result
    }
}
```

## Real-World CRDT Applications

1. **Redis**: Implements various CRDTs for distributed caching
2. **Riak**: Uses CRDTs for conflict resolution in distributed databases
3. **Collaborative Editors**: Google Docs, Figma use CRDT-like algorithms
4. **Blockchain**: Some consensus mechanisms use CRDT principles
5. **Gaming**: Real-time multiplayer games for state synchronization

CRDTs provide the mathematical foundation for building distributed systems that "just work" without complex consensus protocols - exactly what Prism needs for multi-party ZK applications.
