% IntoIter

Продвинемся к написанию итераторов. `iter` и `iter_mut` уже написаны для нас,
спасибо Магии Deref. Однако еще два интересных итератора предоставляются Vec,
которые не могут предоставить срезы: `into_iter` и `drain`.

IntoIter потребляет Vec по значению, и, следовательно, может пройтись по его
элементам по значению. Для этого IntoIter должен взять контроль над размещением
в памяти Vec.

IntoIter к тому же должен быть двусторонним, чтобы уметь читать с обоих
концов. Чтение с конца можно реализовать вызовом `pop`, чтение с начала
гораздо труднее. Мы могли бы вызывать `remove(0)`, но это было бы чрезвычайно
дорого. Вместо этого мы просто используем `ptr::read`, чтобы скопировать значение
с любого конца Vec, вообще не изменяя буфер.

Для этого используем очень популярную идиому Си по итерации массива. Сделаем два
указателя; один, указывающий на начало массива, и один, указывающий на элемент
после конца массива. Если нам нужен элемент с одной стороны, мы читаем
значение указателя и сдвигаем указатель на единицу. Когда два указателя
эквивалентны, мы знаем, что закончили.

Заметьте, что порядок чтения и сдвига противоположны для `next` и `next_back`.
Для `next_back` указатель всегда указывает на элемент после того, который ему
нужно прочитать, а для `next` - всегда на элемент, который он хочет следующим
прочитать. Чтобы понять почему это так, предположим случай, в котором каждый
элемент кроме одного был пройден.

Массив выглядит так:

```text
          S  E
[X, X, X, O, X, X, X]
```

Если E указывал бы напрямую на элемент, который надо пройти следующим, то этот
случай был бы не отличим от случая, когда элементов больше нет.

Несмотря на то, что мы на самом деле не волнуемся о расположении Vec в памяти во
время итерации, нам также надо владеть информацией об этом, чтобы освободить
память во время освобождения IntoIter.

Итак, используем следующую структуру:

```rust,ignore
struct IntoIter<T> {
    buf: Unique<T>,
    cap: usize,
    start: *const T,
    end: *const T,
}
```

И вот с чем мы заканчиваем инициализацию:

```rust,ignore
impl<T> Vec<T> {
    fn into_iter(self) -> IntoIter<T> {
        // Нельзя деструктурировать Vec из-за того, что он Drop
        let ptr = self.ptr;
        let cap = self.cap;
        let len = self.len;

        // Убеждаемся, что не освобождаем Vec, из-за того что он освободит буфер
        mem::forget(self);

        unsafe {
            IntoIter {
                buf: ptr,
                cap: cap,
                start: *ptr,
                end: if cap == 0 {
                    // нельзя сместить этот указатель, он не расположен в памяти!
                    *ptr
                } else {
                    ptr.offset(len as isize)
                }
            }
        }
    }
}
```

Вот итератор с начала:

```rust,ignore
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                let result = ptr::read(self.start);
                self.start = self.start.offset(1);
                Some(result)
            }
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let len = (self.end as usize - self.start as usize)
                  / mem::size_of::<T>();
        (len, Some(len))
    }
}
```

А вот с конца.

```rust,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                self.end = self.end.offset(-1);
                Some(ptr::read(self.end))
            }
        }
    }
}
```

Из-за того, что IntoIter забирает владение своего места расположения, необходимо
реализовать Drop для его освобождения. Но также надо реализовать Drop,
чтобы освободить те элементы, которые еще не были пройдены.


```rust,ignore
impl<T> Drop for IntoIter<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            // освобождаем все оставшиеся элементы
            for _ in &mut *self {}

            let align = mem::align_of::<T>();
            let elem_size = mem::size_of::<T>();
            let num_bytes = elem_size * self.cap;
            unsafe {
                heap::deallocate(*self.buf as *mut _, num_bytes, align);
            }
        }
    }
}
```
