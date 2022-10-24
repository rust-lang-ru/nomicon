% Обработка типов нулевого размера

Пришло время. Начнем бороться с чудищем, называемым типами нулевого размера.
Безопасному Rust *никогда* не нужно волноваться об этом, а вот Vec очень
интенсивно использует сырые указатели и сырое выделение места, именно которым и
надо заботиться о ТНР. Надо быть осторожным в двух вещах:

* API сырого распределения места вызовет неопределенное поведение при передаче 
0 в качестве размера выделяемого места.
* Сдвиги сырых указателей являются пустыми операциями для ТНР, что сломает наш 
Си-подобный итератор указателей.

К счастью, мы абстрагировали наши итераторы по указателям и обработку
распределения места в RawValIter и RawVec заранее. Как неожиданно удобно
получилось.




## Размещение типов нулевого размера

Итак, если API аллокатора не поддерживает выделение памяти нулевого размера, что
же нам хранить в нашей выделенной памяти? Ну что ж, `heap::EMPTY`, конечно!
Почти любая операция с ТНР является пустой операцией из-за того, что у ТНР
только одно значение, и, следовательно, не нужно учитывать никакое состояние ни 
при чтении, ни при записи значений таких типов. Это на самом деле распространяется на
`ptr::read` и `ptr::write`: они вообще не смотрят на указатель. По сути, нам 
никогда не придётся менять указатель.

Заметим, однако, что мы больше не можем надеяться на возникновение нехватки
памяти до переполнения в случае ТНР. Мы должны явно защититься от переполнения
емкости для ТНР.

В нашей текущей архитектуре это означает написание 3 охраняющих условий, по
одному в каждый метод RawVec.

```rust,ignore
impl<T> RawVec<T> {
    fn new() -> Self {
        unsafe {
            // !0 это usize::MAX. Эта ветка удалится во время компиляции.
            let cap = if mem::size_of::<T>() == 0 { !0 } else { 0 };

            // heap::EMPTY служит как для "невыделения", так и для "выделения нулевого размера"
            RawVec { ptr: Unique::new(heap::EMPTY as *mut T), cap: cap }
        }
    }

    fn grow(&mut self) {
        unsafe {
            let elem_size = mem::size_of::<T>();

            // из-за того, что мы установили емкость в usize::MAX если elem_size равен
            // 0, то попадание сюда обозначает, что Vec переполнен.
            assert!(elem_size != 0, "capacity overflow");

            let align = mem::align_of::<T>();

            let (new_cap, ptr) = if self.cap == 0 {
                let ptr = heap::allocate(elem_size, align);
                (1, ptr)
            } else {
                let new_cap = 2 * self.cap;
                let ptr = heap::reallocate(*self.ptr as *mut _,
                                            self.cap * elem_size,
                                            new_cap * elem_size,
                                            align);
                (new_cap, ptr)
            };

            // Если выделение или перераспределение памяти возвращается с ошибкой, получим `null`
            if ptr.is_null() { oom() }

            self.ptr = Unique::new(ptr as *mut _);
            self.cap = new_cap;
        }
    }
}

impl<T> Drop for RawVec<T> {
    fn drop(&mut self) {
        let elem_size = mem::size_of::<T>();

        // не освобождаем выделения нулевого размера, потому что выделение никогда не происходило.
        if self.cap != 0 && elem_size != 0 {
            let align = mem::align_of::<T>();

            let num_bytes = elem_size * self.cap;
            unsafe {
                heap::deallocate(*self.ptr as *mut _, num_bytes, align);
            }
        }
    }
}
```

Вот и все. Теперь мы добавили поддержку push и pop для ТНР. Хотя наши итераторы 
(не предоствляемые срезом Deref) все еще не работают.




## Итерирование по типам нулевого размера

Смещения нулевого размера являются пустыми операциями. Это означает, что в нашей 
текущей архитектуре мы всегда будем инициализировать `start` и `end` одним и тем же
значением, и наши итераторы ничего не вернут. Хорошим решением будет явно
привести указатели к целым, увеличивать их, и затем явно приводить их обратно:

```rust,ignore
impl<T> RawValIter<T> {
    unsafe fn new(slice: &[T]) -> Self {
        RawValIter {
            start: slice.as_ptr(),
            end: if mem::size_of::<T>() == 0 {
                ((slice.as_ptr() as usize) + slice.len()) as *const _
            } else if slice.len() == 0 {
                slice.as_ptr()
            } else {
                slice.as_ptr().offset(slice.len() as isize)
            }
        }
    }
}
```

Теперь у нас другая ошибка. Раньше наши итераторы вообще не
запускались, теперь они выполняются *вечно*. Необходимо сделать тот же трюк в
реализации итераторов. Также, наш код вычисления size_hint будет вызывать
деление на 0 в случае ТНР. Мы считаем, что два указателя ссылаются на байты,
поэтому просто подставим деление на 1 в случае нулевого размера.

```rust,ignore
impl<T> Iterator for RawValIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                let result = ptr::read(self.start);
                self.start = if mem::size_of::<T>() == 0 {
                    (self.start as usize + 1) as *const _
                } else {
                    self.start.offset(1)
                };
                Some(result)
            }
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let elem_size = mem::size_of::<T>();
        let len = (self.end as usize - self.start as usize)
                  / if elem_size == 0 { 1 } else { elem_size };
        (len, Some(len))
    }
}

impl<T> DoubleEndedIterator for RawValIter<T> {
    fn next_back(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                self.end = if mem::size_of::<T>() == 0 {
                    (self.end as usize - 1) as *const _
                } else {
                    self.end.offset(-1)
                };
                Some(ptr::read(self.end))
            }
        }
    }
}
```

И все. Итерация работает!
