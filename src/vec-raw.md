% RawVec

Мы дошли до интересного момента: у нас дублируется логика по размещению буфера и
освобождения его памяти в Vec и IntoIter. Теперь после реализации и обнаружения
*действительного* дублирования логики пришло время выполнить ее сжатие.

Мы абстрагируем пару `(ptr, cap)` и опишем для нее логику размещения в памяти,
возрастания и освобождения:

```rust,ignore
struct RawVec<T> {
    ptr: Unique<T>,
    cap: usize,
}

impl<T> RawVec<T> {
    fn new() -> Self {
        assert!(mem::size_of::<T>() != 0, "TODO: реализовать поддержку ТНР");
        unsafe {
            RawVec { ptr: Unique::new(heap::EMPTY as *mut T), cap: 0 }
        }
    }

    // неизмененная из Vec
    fn grow(&mut self) {
        unsafe {
            let align = mem::align_of::<T>();
            let elem_size = mem::size_of::<T>();

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
        if self.cap != 0 {
            let align = mem::align_of::<T>();
            let elem_size = mem::size_of::<T>();
            let num_bytes = elem_size * self.cap;
            unsafe {
                heap::deallocate(*self.ptr as *mut _, num_bytes, align);
            }
        }
    }
}
```

И изменим Vec:

```rust,ignore
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
}

impl<T> Vec<T> {
    fn ptr(&self) -> *mut T { *self.buf.ptr }

    fn cap(&self) -> usize { self.buf.cap }

    pub fn new() -> Self {
        Vec { buf: RawVec::new(), len: 0 }
    }

    // push/pop/insert/remove lв основном без изменений:
    // * `self.ptr -> self.ptr()`
    // * `self.cap -> self.cap()`
    // * `self.grow -> self.buf.grow()`
}

impl<T> Drop for Vec<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() {}
        // освобождение обрабатывается RawVec
    }
}
```

И наконец можем сильно упростить IntoIter:

```rust,ignore
struct IntoIter<T> {
    _buf: RawVec<T>, // нам не нужно волноваться об этом. Просто нужно, чтобы это жило.
    start: *const T,
    end: *const T,
}

// next и next_back, буквально, не поменялись, потом что не ссылаются на buf

impl<T> Drop for IntoIter<T> {
    fn drop(&mut self) {
        // только убедимся что все элементы прочтены;
        // буфер сам почистит себя после этого.
        for _ in &mut *self {}
    }
}

impl<T> Vec<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        unsafe {
            // нужно использовать ptr::read чтобы небезопасно передать buf, потому что
            // он не Copy, а Vec реализует Drop (поэтому мы не можем деструктурировать его).
            let buf = ptr::read(&self.buf);
            let len = self.len;
            mem::forget(self);

            IntoIter {
                start: *buf.ptr,
                end: buf.ptr.offset(len as isize),
                _buf: buf,
            }
        }
    }
}
```

Гораздо лучше.
