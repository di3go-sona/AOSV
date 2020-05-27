### Data Structures

In order to achieve the best performance and efficiency, it is better to handle data types independently of their particular data structure. This is a typical feature of Object Oriented programming languages, so kernel developers needed to create a way for C to handle this kind of abstraction.

An abstract data type has the following characteristics:
- it encapsulates the entire implementation of a data structure
- it provides only a well-defined interface to manipulate objects/collections
- optimizations made in the implementation are directly spread across the whole source

#### Circular Doubly-Linked Lists

The circular doubly-linked list is kept inside the data structure of the struct that needs to be in a linked list. This way you can have the tiniest sentinel node possible (the one which represents both the HEAD and the TAIL nodes), since it only needs a `next` and `prev` pointer, without having to also replicate the full data structure.

<img src=".\images\list_as_a_member_of_the_struct.png" alt="list as a member of the struct" style="zoom: 50%;" />

From include/linux/list.h:
```C
struct list_head {
    struct list_head *next, *prev;
};
```

This way, it's also possible to keep multiple lists in a single struct, like this:

```C
struct my_struct {
    /* ... */
    struct list_head list1;
    struct list_head list2;
    /* ... */
}
```

#### List Macros

In order to **access the container struct** when navigating a linked list, we need to use the `container_of()` macro.

<img src=".\images\macro_container_of.png" alt="list as a member of the struct" style="zoom: 80%;" />

List traversal is done with the macro:

```C
list_for_each(ptr, &todo_list) {
    /*...*/
}
```

Where `ptr` is the pointer to the first element of the list (the one right after the sentinel node) and `todo_list` is the pointer to the current node

----
#### Hash Lists

Hash Lists are just like Doubly Linked Lists, but the sentinel node doesn't have a pointer to the tail node. They are used to save space when a pointer to the tail node would be unnecessary.

```C
struct hlist_head {
    struct hlist_node *first;
}

struct hlist_node {
    struct hlist_node *next;
    struct hlist_node **prev;
}
```

In order to not break anything, `**prev` points to the `*next` element of the previous node. This way, the `**prev` of the first `hlist_node` (the one just after `hlist_head`) will point to the only member of `hlist_head`.

----
#### Lockless Lists

Singly-linked NULL-terminated non-blocking lists. They are based on *compare and swap* to update pointers. You can carry out concurrent RMW (Read/Modify/Write) operations on a `next` pointer, without the need to use a lock.

The last 2 bits of the next pointer are always `00`. This way, you can "logically" delete a node by setting the last bit to 1. In the meanwhile, other threads can traverse the list as if the logically deleted node didn't exist. After having marked a node as logically deleted, you need to perform an additional "compare and swap" to `kfree` that node. You can't know for sure if the node is still being used by some threads though (and that is why the "delete" operation is still not in the current vanilla kernel tree).

----
#### Queues

Queues (called `kfifo` in `/include/linux/kfifo.h`) are used to implement a producer-consumer model. The operations used are `kfifo_in()` (enqueue) to insert an element and `kfifo_out()` (dequeue) to pop an element.

Queues are created with `kfifo_alloc(struct kfifo *fifo, unsigned int size, gfp_t gfp_mask)` and deleted with `kfifo_free(struct kfifo *fifo)`.

----
#### Red-Black Tree

An RB Tree is a self balancing binary tree. Nodes in this tree have a color, which is related usually to the level of the node in the tree.

All leaves (which are null pointers) are **black**. Each red node has two black children.

The tree needs rebalancing if paths from the root to the leaves have different number of black nodes.

<img src=".\images\rb-tree.png" alt="rb-tree" style="zoom: 80%;" />

RB Trees are defined in `/include/linux.rbtree.h`.

Initialized with
```C
struct rb_root root = RB_ROOT
```

The API for RB Trees allows you to:
- get the payload of a node `rb_entry()`
- insert a node: `rb_link_node()`
- you can set the color of a node: (triggers rebalancing): `rb_insert_color()`
- remove a node: `rb_erase()`

This implementation of an RB tree doesn't have a default "traversal" operation because it cannot be known a priori what should the default implementation of `compare` be. It therefore needs to be implemented by hand.

----
#### Radix Tree

Used to allocate process ids. It's a compact prefix tree, in which each node represents a partial aspect of all of its children.

<img src=".\images\radix-tree.png" alt="rb-tree" style="zoom: 60%;" />

Each node is associated with a key. Some bits of this keys represent a pointer to reach the next level of the tree. The leaves of a radix tree are objects: this means that a radix tree is a way of mapping integers to objects.

They are therefore used to map a pid (integer) to a task-struct (object).

There are 2 different implementations of radix trees in the kernel:
- `/include/linux/radix-tree.h` (the one in the pictura above)
- `/include/linux/idr.h` (simpler, based on radix-tree.h)

Both provide a mapping from `unsigned long` to a pointer `void *`.