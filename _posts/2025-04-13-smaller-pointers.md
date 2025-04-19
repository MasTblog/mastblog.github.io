---
layout: post
title:  "4-byte Pointers in Competitive Programming"
date:   2025-04-12 00:22:04 -0700
toc: true
---

In this article I discuss a niche trick involving data structures
in competitive programming that use pointers. This trick is specific to competitors who use C++ and online graders that use 64-bit machines, which is the most common.

Some data structures like binary search trees in competitive programming are somewhat rare since "static trees"
such as the binary indexed tree and segment tree usually get the job done. Static trees are tree data structures where the shape of the tree never
changes, so pointers are actually just indexes in an array. For example, in binary indexed trees, the index of the parent of a node can be found by
subtracting the least significant bit, and in segment trees, the left and right children are double the current index and double the current index plus one, respectively.

In balanced binary search trees such as treaps and splay trees, as well as persistent data structures where new nodes are allocated upon update, pointers
are commonly used, with struct definitions much more reminiscent of those seen in an introductory algorithms course:

```cpp
struct node {
    int value;
    node *left, *right;
};

node* make_node(int value, node *left, node *right) {
    return new node { value, left, right };
}

void traverse(node *n) {
    if(!n) {
        return;
    }
    traverse(n->left);
    std::cout << n->value << "\n";
    traverse(n->right);
}
```

In problems where nodes aren't deleted or where there's no need to reclaim the memory from deleted nodes, it's faster and more space-efficient to declare all the nodes in the BSS segment in a single store rather than using the heap:

```cpp
// Maximum number of nodes allocated in the entire program
const int MAX = 1e6;
int current_idx = 0;
node store[MAX];
node* make_node(int value, node *left, node *right) {
    store[current_idx] = { value, left, right };
    node* pointer = &store[current_idx];
    ++current_idx;
    return pointer;
}
```

Now we have the option of just storing the indexes in the struct instead of the pointers, which require us to designate an index as the null pointer and change all of our functions as well:

```cpp
struct node {
    int value;
    int left_idx, right_idx;
};

// Something large so it causes a segfault
// when dereferenced, making it easier to debug
const int null = 1e9;

void traverse(int node_idx) {
    if(node_idx == null) {
        return;
    }
    traverse(store[node_idx].left);
    std::cout << store[node_idx].value << "\n";
    traverse(store[node_idx].right);
}
```

It can get cumbersome to keep having to refer to the store each time
we "deference" the index, especially if we need to do it more than once:

```cpp
// This function is just for demonstration, it has no applications
int zigzag(int node_idx){
    return store[store[store[node_index].right].left].right;
}
```

While we almost always benefit from using BSS-allocated nodes instead of heap nodes where appropriate, in both time and space, the only real benefit to storing indexes instead of pointers is using less space, and it is even at the cost of time since the computer must now do pointer arithmetic every time it "dereferences" the index.

As such, the only appropriate time to do this is if the allowed memory limit is tight, which can happen with persistent segment trees or 2D range trees which use $\Theta(n \log n)$ nodes or with [square root decomposition](https://cp-algorithms.com/data_structures/sqrt_decomposition.html) solutions that use $\Theta(n \sqrt n)$ nodes. Of course, competitors should first consider reducing the number of fields in the node structs
and reorder fields to minimize struct padding. Storing `int` indexes instead of pointers will only save 4 bytes per pointer, plus possibly some padding if the layout becomes more favorable and the struct no longer uses any 8-byte fields.

In these cases, we can take advantage of C++'s operator overloading to keep pointer semantics while using indexes:

```cpp
struct node;

// Pointer to node
struct pnode {
    unsigned int idx;
    explicit operator bool();
    node& operator*();
    node* operator->();
    bool operator==(pnode &other);
};

const int NULL_IDX = 1e9;
const pnode null = { NULL_IDX };

struct node {
    int value;
    pnode left, right;
};

node store[MN];

pnode::operator bool() {
    return idx != NULL_IDX;
}

node& pnode::operator*() {
    return store[idx];
}

node* pnode::operator->() {
    return store + idx;
}

bool pnode::operator==(pnode &other) {
    return idx == other.idx;
}
```

Now we can write all of our functions how we would before, replacing `node*` with `pnode`.

Modern programming contests generally don't test contestants' abilities
to optimize for space, so problems where a lot of memory is intended to be needed will set a generous memory limit, but this trick can be used to get an unintended memory-hungry solution within the memory limit, especially given that a lot of problems admit easier, unintended solutions that use square root decomposition.

In the examples above, the original struct takes up 24 bytes (due to padding), and the struct with `int` indexes takes up only 12. If 10 million nodes are required in a solution, which can easily be the case with solution that use $\Theta(n \log n)$ or $\Theta(n \sqrt n)$ nodes, we can save 120MB, which can make the difference between making the memory limit or not.

In [Database](https://dmoj.ca/problem/tle17c7p4), a problem with a persistent segment tree solution, I was able to save 63MB by switching
from heap allocation to BSS allocation, and 33MB by switching from pointers to indexes, so actual savings may vary.

## Bonus: Packing and 3-Byte Pointers

Outside of re-ordering the fields of a struct, we can also pack structs to force unaligned accesses, giving us better space usage at the cost of time:

```cpp
struct __attribute__((packed)) node {
    long long int key;
    int value;
    pnode left, right;
};
```

In this example, assuming we are using a 4-byte `pnode`, we can save 4 bytes of padding in this example at the cost of having the `key` field possibly being loaded at an unaligned address, which costs more CPU cycles.

To go one step further: if we know that we won't use more than $2^{16}$ nodes (including one value reserved for the null pointer), we can use an `unsigned short`, giving us 2-byte pointers, but more realistically for contest problems, if we will be using between $2^{16}$ and $2^{24}$ nodes, we can instead use a bit field:

```cpp
struct __attribute__((packed)) node {
    long long int key;
    int value;
    pnode left, right;
};

struct __attribute__((packed)) pnode {
    unsigned int idx : 24;

    explicit operator bool();
    node& operator*();
    node* operator->();
    bool operator==(pnode &other);
};
```

This will make all `pnode` pointers 3 bytes, but now potentially every field in `node` will have unaligned reads and writes, considerably slowing things down. On [Database](https://dmoj.ca/problem/tle17c7p4), this approach saved 9MB over 4-byte pointers, at the cost of of a 46% slowdown.