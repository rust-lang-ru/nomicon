# Insert и Remove

Вот, что срез *не* предоставляет, так это `insert` и `remove`, поэтому давайте сделаем их следующими.

Insert нужно сдвинуть все элементы от целевого направо на единицу. Для этого используем `ptr::copy`, являющийся нашей версией `memmove` в Си. Он копирует кусок памяти из одного места в другое, корректно обрабатывая случаи, если источник и назначение пересекаются (что точно произойдёт здесь).

Если мы вставим по индексу `i`, мы должны сдвинуть `[i .. len]` в `[i+1 .. len+1]`, используя старую длину.

<!-- ignore: simplified code -->

```rust,ignore
pub fn insert(&mut self, index: usize, elem: T) {
    // Внимание: `<=` потому что считается привильным вставлять после любого элемента
    // что было бы эквивалентно push.
    assert!(index <= self.len, "index out of bounds");
    if self.cap == self.len { self.grow(); }

    unsafe {
        if index < self.len {
            // ptr::copy(src, dest, len): "копировать из источника в назначение len элементов"
            ptr::copy(self.ptr.offset(index as isize),
                      self.ptr.offset(index as isize + 1),
                      self.len - index);
        }
        ptr::write(self.ptr.offset(index as isize), elem);
        self.len += 1;
    }
}
```

Remove ведёт себя наоборот. Нужно сдвинуть все элементы из `[i+1 .. len + 1]` в `[i .. len]`, используя *новую* длину.

<!-- ignore: simplified code -->

```rust,ignore
pub fn remove(&mut self, index: usize) -> T {
    // Внимание: `<` потому что *не* правильно удалять после всего
    assert!(index < self.len, "index out of bounds");
    unsafe {
        self.len -= 1;
        let result = ptr::read(self.ptr.offset(index as isize));
        ptr::copy(self.ptr.offset(index as isize + 1),
                  self.ptr.offset(index as isize),
                  self.len - index);
        result
    }
}
```
