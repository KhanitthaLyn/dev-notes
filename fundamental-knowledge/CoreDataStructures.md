# The Big Picture & Analogy

This topic is about understanding **Core Data Structures** — the foundation underlying everything in the Collections Framework you already learned. This time we go deeper, looking "under the hood" at how each one actually works.

**Analogy:**

- **Array** = a row of lockers lined up in a fixed sequence, numbered 0, 1, 2, 3... — know the locker number, and you can walk straight to it. But if you want to insert a new locker in the middle, every locker after it has to shift over.
- **Linked List** = a train, where each car (node) only has a "hook connecting to the next car." To insert a new car in the middle, you just unhook and re-hook — no other cars need to move. But if you want to reach car #500, you have to walk through every car from the front.
- **HashMap** = a library with a system that "converts a book's title into a shelf number" (hash function). Know the title, calculate the shelf number instantly — no searching book by book.
- **Stack** = a stack of plates — the most recently placed plate always comes off first (Last In, First Out).
- **Queue** = a checkout line — first come, first served (First In, First Out).

---

# Why Do We Need It?

**Why do we need multiple structures instead of just one?**

The problem is that **no single data structure is fast at everything simultaneously** — this is a fundamental trade-off in computer science.

- If you want **fast index-based access** (random access) → Array is best, but at the cost of slow insert/delete in the middle.
- If you want to **insert/delete frequently without shifting other data** → Linked List excels, but at the cost of slow access to elements in the middle (must traverse one by one).
- If you want **very fast lookups by key** → HashMap is best, but you trade away ordering (by default) and need to understand edge cases around collisions.
- If your business logic has a specific access pattern (e.g. "undo the most recent change first," or "process in the order received") → Stack/Queue provide an interface that matches that pattern exactly, preventing misuse.

**Without understanding this deeply:** you'll pick the wrong data structure without realizing it — e.g. using an ArrayList and frequently inserting at the front (which is expensive, O(n) every time) when a LinkedList would be far better, or not understanding why a HashMap sometimes slows down when collisions pile up.

---

# Core Logic & How It Works

## Array vs Linked List

### **Array**
Data is stored in **contiguous memory** — knowing the index lets you calculate the exact address immediately, using `base_address + (index × element_size)`.

```
Index:   [0]  [1]  [2]  [3]
Memory:  [A]  [B]  [C]  [D]   ← packed together as one block
```

- **Access by index:** O(1) — the address is calculated directly.
- **Insert/Delete in the middle:** O(n) — every element after that point must shift.

### **Linked List**
Data is scattered across different memory locations. Each **node** stores a "value" + a "pointer to the next node."

```
[A|next] → [B|next] → [C|next] → [D|null]
   (located in different, non-contiguous places in memory)
```

- **Access by index:** O(n) — you must walk from the head, node by node, until reaching the target position.
- **Insert/Delete once you already have the node:** O(1) — you just change pointers, nothing needs to shift.

```java
// Code comparison
List<Integer> arrayList = new ArrayList<>();  // backed by an array internally
arrayList.add(0, 999); // insert at index 0 → O(n), because everything shifts

List<Integer> linkedList = new LinkedList<>(); // backed by a linked list internally
linkedList.add(0, 999); // insert at index 0 → O(1), just changes the head pointer
```

## HashMap Internals

**How it works:**
1. On `put(key, value)` → calls `key.hashCode()` to get a hash number.
2. Takes that hash and applies `%` (mod) against the size of the internal array (called a **bucket** array) to determine which bucket to store it in.
3. Stores the key-value pair at that bucket.

```
key.hashCode() → hash value → hash % bucket_count → bucket index

"apple".hashCode() = 93029210
93029210 % 16 = bucket #10  → stored at bucket 10
```

**What's a collision:** when 2 different keys end up computing the same bucket index (this always happens eventually, because the number of possible keys exceeds the number of buckets).

**How Java's HashMap handles collisions:**
- Each bucket actually stores a **LinkedList of entries** (or a Red-Black Tree if entries in that bucket exceed a threshold of 8 — since Java 8).
- If 2 keys collide into the same bucket → both are stored in that bucket's linked list.
- On `get(key)` → find the bucket first (O(1)), then walk through that bucket's linked list comparing with `.equals()` until the matching key is found.

```
Bucket 10: [ ("apple", 5) ] → [ ("grape", 3) ]  ← if apple and grape collide into the same bucket
```

**This is why HashMap averages O(1) but has an O(n) worst case:** with few collisions, it's nearly always O(1). But if the hash function is terrible and every key collides into the same bucket (worst case), it degrades into searching through a long linked list = O(n).

## Stack & Queue

**Stack — LIFO (Last In, First Out)**
```java
Stack<Order> undoStack = new Stack<>();
undoStack.push(order1);  // in
undoStack.push(order2);  // in
undoStack.pop();         // order2 comes out first (most recently added)
```
Core operations: `push()` (add to top), `pop()` (remove the top), `peek()` (view the top without removing) — all O(1)

**Queue — FIFO (First In, First Out)**
```java
Queue<Order> processingQueue = new LinkedList<>();
processingQueue.offer(order1); // enqueue
processingQueue.offer(order2); // enqueue
processingQueue.poll();        // order1 comes out first (arrived first)
```
Core operations: `offer()`/`enqueue()` (add to the back), `poll()`/`dequeue()` (remove from the front) — all O(1)

---

# Trade-offs & When to Use

| Structure | Use when | Avoid when |
|---|---|---|
| **Array/ArrayList** | Frequent reads, roughly known size upfront, frequent index-based access | Frequent insert/delete in the middle |
| **LinkedList** | Frequent insert/delete (especially at head/tail), no need for random access | Frequent index-based access |
| **HashMap** | Frequent lookups by key, order doesn't matter | Order matters (use LinkedHashMap/TreeMap instead) |
| **Stack** | Need "reverse most-recent-first" behavior, e.g. undo, backtracking, browser history | Need to process in arrival order |
| **Queue** | Need to "process in arrival order," e.g. task queues, message processing, BFS | Need LIFO behavior |

**Trade-offs:**
- Array: fast access, but expensive insert/delete, and requires contiguous memory (can be problematic with frequent resizing).
- LinkedList: fast insert/delete once you have the node, but slow access, and uses more memory (extra pointer per node).
- HashMap: very fast lookups, but no ordering, and needs a good hash function to minimize collisions.

---

# Real-World Scenario / Mini Example

**Implementing a Stack from scratch (backed by an Array):**

```java
public class ArrayStack<T> {
    private Object[] elements;
    private int size = 0;

    public ArrayStack(int capacity) {
        elements = new Object[capacity];
    }

    public void push(T item) {
        if (size == elements.length) {
            throw new IllegalStateException("Stack is full");
        }
        elements[size++] = item; // place at position `size`, then increment
    }

    @SuppressWarnings("unchecked")
    public T pop() {
        if (size == 0) throw new IllegalStateException("Stack is empty");
        T item = (T) elements[--size]; // decrement first, then read from the new position
        elements[size] = null; // clear the old reference to prevent memory leaks
        return item;
    }

    @SuppressWarnings("unchecked")
    public T peek() {
        if (size == 0) throw new IllegalStateException("Stack is empty");
        return (T) elements[size - 1];
    }
}
```

**Implementing a Queue from scratch (backed by linked nodes):**

```java
public class LinkedQueue<T> {
    private Node<T> head, tail;

    private static class Node<T> {
        T value;
        Node<T> next;
        Node(T value) { this.value = value; }
    }

    public void offer(T item) { // enqueue at the tail
        Node<T> newNode = new Node<>(item);
        if (tail == null) {
            head = tail = newNode; // empty queue — this single node is both head and tail
        } else {
            tail.next = newNode; // attach after the current tail
            tail = newNode;       // update tail
        }
    }

    public T poll() { // dequeue from the head
        if (head == null) throw new IllegalStateException("Queue is empty");
        T value = head.value;
        head = head.next;
        if (head == null) tail = null; // queue is now empty — reset tail too
        return value;
    }
}
```

**Real ecommerce scenarios:**
- **Stack:** stores "order edit history" to implement undo — the most recent edit must be undone first.
- **Queue:** acts as an "order processing queue" — orders placed earlier by customers should be processed first (FIFO) for fairness.
- **HashMap:** stores `productId → stock quantity` for fast lookups (as discussed earlier in the Collections topic).

**Explaining HashMap collision handling, interview-style:**
> "A HashMap uses a hash function to convert a key into a bucket index. When two keys collide into the same bucket, Java stores both in a linked list at that bucket (or a tree, if entries exceed a threshold). On lookup, it hashes to find the bucket first, then walks through that bucket's list comparing with equals() until it finds the matching key. This gives an average case of O(1), but a worst case (heavy collisions) of O(n)."

---

# Lead's Key Takeaway

1. **Every data structure is a choice of "what to be fast at, in exchange for what being slow" — none of them are fast at everything.** Before choosing one, ask yourself: what's your access pattern? Read-heavy or write-heavy? Random access or sequential? Where do inserts/deletes happen most often?
2. **Understand "why" HashMap is O(1) deeply enough to explain the worst case too.** This is a classic interview question that separates people who truly understand it from those who just memorized the fact. Knowing about collision handling and resizing (rehashing) lets you answer follow-up questions confidently, instead of just reciting "HashMap is O(1) fast."
