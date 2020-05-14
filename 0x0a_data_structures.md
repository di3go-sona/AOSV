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