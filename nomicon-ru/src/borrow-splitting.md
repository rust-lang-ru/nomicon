# Деление заимствований

Свойство эксклюзивного изменения у изменяемых ссылок может сильно ограничивать работу с составными структурами. Анализатор заимствований понимает какие-то базовые вещи, но может поломаться довольно просто. Он достаточно понимает структуры, чтобы знать, что можно одновременно заимствовать непересекающиеся поля структуры. Вот так работать будет:

```rust
struct Foo {
    a: i32,
    b: i32,
    c: i32,
}

let mut x = Foo {a: 0, b: 0, c: 0};
let a = &mut x.a;
let b = &mut x.b;
let c = &x.c;
*b += 1;
let c2 = &x.c;
*a += 10;
println!("{} {} {} {}", a, b, c, c2);
```

Однако анализатор заимствований вообще не понимает массивы или срезы, поэтому так работать не будет:

```rust,compile_fail
let mut x = [1, 2, 3];
let a = &mut x[0];
let b = &mut x[1];
println!("{} {}", a, b);
```

```text
error[E0499]: cannot borrow `x[..]` as mutable more than once at a time
 --> src/lib.rs:4:18
  |
3 |     let a = &mut x[0];
  |                  ---- first mutable borrow occurs here
4 |     let b = &mut x[1];
  |                  ^^^^ second mutable borrow occurs here
5 |     println!("{} {}", a, b);
6 | }
  | - first borrow ends here

error: aborting due to previous error
```

Хоть и кажется, что анализатор заимствований мог бы понимать этот простой случай, очевидно, безнадёжным кажется, что он мог бы понимать различия в общих типах контейнеров, таких как деревья, особенно, если разные ключи *скрывают* одно и то же значения.

Для того чтобы "объяснить" анализатору заимствований, что мы делаем правильно, нам нужно перейти к небезопасному коду. Например, изменяемые срезы подвергаются действию функции `split_at_mut`, которая берет срез и возвращает два изменяемых среза. В один попадает то, что находится слева от индекса, в другой - справа. Интуитивно мы понимаем, что это безопасно, потому что срезы не пересекаются и, следовательно, не совпадают ссылки на них. Однако для реализации потребуется щепотка небезопасного кода:

```rust
# use std::slice::from_raw_parts_mut;
# struct FakeSlice<T>(T);
# impl<T> FakeSlice<T> {
# fn len(&self) -> usize { unimplemented!() }
# fn as_mut_ptr(&mut self) -> *mut T { unimplemented!() }
pub fn split_at_mut(&mut self, mid: usize) -> (&mut [T], &mut [T]) {
    let len = self.len();
    let ptr = self.as_mut_ptr();

    unsafe {
        assert!(mid <= len);

        (from_raw_parts_mut(ptr, mid),
         from_raw_parts_mut(ptr.add(mid), len - mid))
    }
}
# }
```

Все здесь тонко. Поэтому, чтобы избежать создания двух `&mut` к одному значению, мы явно конструируем два абсолютно новых среза через сырые указатели.

Ещё сложнее работают итераторы, перебирающие изменяемые ссылки. Типаж итератора описывается так:

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

Глядя на определение, видим, что у Self::Item *нет* связи с `self`. Это означает, что мы можем вызвать `next` несколько раз подряд, и получим все результаты *одновременно*. Для итераторов по значению, у которых именно такая семантика, все по-другому. Для общих ссылок все нормально, потому что они допускают сколь угодно много ссылок на одно и то же (хотя итератор и общий объект должны быть разными объектами).

Но с изменяемыми ссылками все превращается в кашу. На первый взгляд кажется, что они полностью несовместимы с этим API, из-за того, что создадут много изменяемых ссылок на один и тот же объект!

Однако, *на самом деле* все работает, именно, потому что итераторы - это одноразовые объекты. Все, по чему пройдётся IterMut, будет пройдено только один раз, поэтому на самом деле мы никогда не создадим много изменяемых ссылок на один кусок данных.

Возможно, это удивительно, но изменяемым итератором не нужно использовать небезопасный код для реализации разных типов!

Например, вот пример однонаправленного списка:

```rust
# fn main() {}
type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

pub struct LinkedList<T> {
    head: Link<T>,
}

pub struct IterMut<'a, T: 'a>(Option<&'a mut Node<T>>);

impl<T> LinkedList<T> {
    fn iter_mut(&mut self) -> IterMut<T> {
        IterMut(self.head.as_mut().map(|node| &mut **node))
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node| {
            self.0 = node.next.as_mut().map(|node| &mut **node);
            &mut node.elem
        })
    }
}
```

Вот изменяемый срез:

```rust
# fn main() {}
use std::mem;

pub struct IterMut<'a, T: 'a>(&'a mut[T]);

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        let slice = mem::replace(&mut self.0, &mut []);
        if slice.is_empty() { return None; }

        let (l, r) = slice.split_at_mut(1);
        self.0 = r;
        l.get_mut(0)
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        let slice = mem::replace(&mut self.0, &mut []);
        if slice.is_empty() { return None; }

        let new_len = slice.len() - 1;
        let (l, r) = slice.split_at_mut(new_len);
        self.0 = l;
        r.get_mut(0)
    }
}
```

Двоичное дерево:

```rust
# fn main() {}
use std::collections::VecDeque;

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    left: Link<T>,
    right: Link<T>,
}

pub struct Tree<T> {
    root: Link<T>,
}

struct NodeIterMut<'a, T: 'a> {
    elem: Option<&'a mut T>,
    left: Option<&'a mut Node<T>>,
    right: Option<&'a mut Node<T>>,
}

enum State<'a, T: 'a> {
    Elem(&'a mut T),
    Node(&'a mut Node<T>),
}

pub struct IterMut<'a, T: 'a>(VecDeque<NodeIterMut<'a, T>>);

impl<T> Tree<T> {
    pub fn iter_mut(&mut self) -> IterMut<T> {
        let mut deque = VecDeque::new();
        self.root.as_mut().map(|root| deque.push_front(root.iter_mut()));
        IterMut(deque)
    }
}

impl<T> Node<T> {
    pub fn iter_mut(&mut self) -> NodeIterMut<T> {
        NodeIterMut {
            elem: Some(&mut self.elem),
            left: self.left.as_mut().map(|node| &mut **node),
            right: self.right.as_mut().map(|node| &mut **node),
        }
    }
}


impl<'a, T> Iterator for NodeIterMut<'a, T> {
    type Item = State<'a, T>;

    fn next(&mut self) -> Option<Self::Item> {
        match self.left.take() {
            Some(node) => Some(State::Node(node)),
            None => match self.elem.take() {
                Some(elem) => Some(State::Elem(elem)),
                None => match self.right.take() {
                    Some(node) => Some(State::Node(node)),
                    None => None,
                }
            }
        }
    }
}

impl<'a, T> DoubleEndedIterator for NodeIterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        match self.right.take() {
            Some(node) => Some(State::Node(node)),
            None => match self.elem.take() {
                Some(elem) => Some(State::Elem(elem)),
                None => match self.left.take() {
                    Some(node) => Some(State::Node(node)),
                    None => None,
                }
            }
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        loop {
            match self.0.front_mut().and_then(|node_it| node_it.next()) {
                Some(State::Elem(elem)) => return Some(elem),
                Some(State::Node(node)) => self.0.push_front(node.iter_mut()),
                None => if let None = self.0.pop_front() { return None },
            }
        }
    }
}

impl<'a, T> DoubleEndedIterator for IterMut<'a, T> {
    fn next_back(&mut self) -> Option<Self::Item> {
        loop {
            match self.0.back_mut().and_then(|node_it| node_it.next_back()) {
                Some(State::Elem(elem)) => return Some(elem),
                Some(State::Node(node)) => self.0.push_back(node.iter_mut()),
                None => if let None = self.0.pop_back() { return None },
            }
        }
    }
}
```

Все это абсолютно безопасно и работает на стабильном Rust! На самом деле, это вытекает из случая с простой структурой, который мы видели выше: Rust понимает, что вы можете делить изменяемую ссылку на ссылки на под-поля. Мы можем возвращать постоянно употребляемую ссылку по средствам Options (или в случае срезов, заменить на пустой срез).
