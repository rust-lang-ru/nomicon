% Drain

Перейдем к Drain. Drain очень похож на IntoIter, за исключением того, что он не
потребляет Vec, а заимствует Vec и оставляет его расположение в памяти
нетронутым. Для начала реализуем только "базовую" полноразмерную версию.

```rust,ignore
use std::marker::PhantomData;

struct Drain<'a, T: 'a> {
    // Нужно ограничить время жизни, поэтому делаем это с помощью `&'a mut Vec<T>`
    // потому что семантически именно это и содержится. Мы "просто" вызываем
    // `pop()` и `remove(0)`.
    vec: PhantomData<&'a mut Vec<T>>
    start: *const T,
    end: *const T,
}

impl<'a, T> Iterator for Drain<'a, T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
```

-- постойте, кажется это уже было. Выполним еще сжатие логики. И IntoIter и 
Drain имеют одну и ту же структуру, просто вынесем ее.

```rust
struct RawValIter<T> {
    start: *const T,
    end: *const T,
}

impl<T> RawValIter<T> {
    // небезопасно создавать, потому что нет связанных времен жизни.
    // Важно хранить RawValIter в той же структуре, что и ее настоящее 
    // размещение. Это допустимо, поскольку это скрытые детали нашей реализации.
    unsafe fn new(slice: &[T]) -> Self {
        RawValIter {
            start: slice.as_ptr(),
            end: if slice.len() == 0 {
                // если `len = 0`, то это не настоящее место размещения.
                // Нужно избежать сдвига, потому что это даст неправильную
                // информацию LLVM через GEP.
                slice.as_ptr()
            } else {
                slice.as_ptr().offset(slice.len() as isize)
            }
        }
    }
}

// Iterator и DoubleEndedIterator реализуются аналогично IntoIter.
```

А IntoIter станет следующим:

```rust,ignore
pub struct IntoIter<T> {
    _buf: RawVec<T>, // нам не нужно волноваться об этом. Просто нужно, чтобы это жило.
    iter: RawValIter<T>,
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> { self.iter.next() }
    fn size_hint(&self) -> (usize, Option<usize>) { self.iter.size_hint() }
}

impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> { self.iter.next_back() }
}

impl<T> Drop for IntoIter<T> {
    fn drop(&mut self) {
        for _ in &mut self.iter {}
    }
}

impl<T> Vec<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        unsafe {
            let iter = RawValIter::new(&self);

            let buf = ptr::read(&self.buf);
            mem::forget(self);

            IntoIter {
                iter: iter,
                _buf: buf,
            }
        }
    }
}
```

Заметьте, что я оставил несколько причудливых мест в проекте, чтобы сделать
модернизацию Drain по работе с произвольными поддиапазонами немного проще. В
частности мы *могли бы* сделать, чтобы RawValIter выполнял опустошение самого
себя при освобождении, но это не будет работать правильно для более сложного
Drain. Также возьмем срез для упрощения инициализации Drain.

Итак, теперь Drain по-настоящему прост:

```rust,ignore
use std::marker::PhantomData;

pub struct Drain<'a, T: 'a> {
    vec: PhantomData<&'a mut Vec<T>>,
    iter: RawValIter<T>,
}

impl<'a, T> Iterator for Drain<'a, T> {
    type Item = T;
    fn next(&mut self) -> Option<T> { self.iter.next() }
    fn size_hint(&self) -> (usize, Option<usize>) { self.iter.size_hint() }
}

impl<'a, T> DoubleEndedIterator for Drain<'a, T> {
    fn next_back(&mut self) -> Option<T> { self.iter.next_back() }
}

impl<'a, T> Drop for Drain<'a, T> {
    fn drop(&mut self) {
        for _ in &mut self.iter {}
    }
}

impl<T> Vec<T> {
    pub fn drain(&mut self) -> Drain<T> {
        unsafe {
            let iter = RawValIter::new(&self);

            // это безопасный mem::forget. Если Drain забыт, у нас просто утечет
            // все содержимое Vec. К тому же нам нужно сделать это со *временем*
            // в любом случае, так почему не сделать это сейчас?
            self.len = 0;

            Drain {
                iter: iter,
                vec: PhantomData,
            }
        }
    }
}
```

Для деталей по проблеме `mem::forget`, смотри [раздел по утечкам][leaks].

[leaks]: leaking.html
