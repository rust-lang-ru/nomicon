# Разыменование

Отлично! Мы реализовали приличный минимальный стек. Можем делать push, можем делать pop и можем освобождать ресурсы за собой. Однако, нам нужен ещё приличный объем функциональности. В частности, у нас есть функциональность массива, но никакой функциональности среза. Это довольно просто решить: можем реализовать `Deref<Target=[T]>`. Он магическим образом заставит наш Vec неявно приводиться и вести себя как срез в любых условиях.

Все, что нам нужно - `slice::from_raw_parts`. Он будет корректно обрабатывать пустые срезы для нас. А также, раз мы добавили поддержку ТНР, он будет Просто Работать и для них.

<!-- ignore: simplified code -->

```rust,ignore
use std::ops::Deref;

impl<T> Deref for Vec<T> {
    type Target = [T];
    fn deref(&self) -> &[T] {
        unsafe {
            std::slice::from_raw_parts(self.ptr.as_ptr(), self.len)
        }
    }
}
```

И сделаем DerefMut тоже:

<!-- ignore: simplified code -->

```rust,ignore
use std::ops::DerefMut;

impl<T> DerefMut for Vec<T> {
    fn deref_mut(&mut self) -> &mut [T] {
        unsafe {
            std::slice::from_raw_parts_mut(self.ptr.as_ptr(), self.len)
        }
    }
}
```

Теперь у нас есть `len`, `first`, `last`, индексирование, нарезка, сортировка, `iter`, `iter_mut` и все другие примочки среза. Мило!
