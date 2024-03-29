# Деструкторы

Что язык *действительно* даёт, так это полноценные автоматические деструкторы в виде типажа `Drop`, который предоставляет следующий метод:

<!-- ignore: function header -->

```rust,ignore
fn drop(&mut self);
```

Этот метод даёт типу время каким-то образом завершить то, что он делал.

**После запуска `drop`, Rust рекурсивно попытается удалить все поля `self`.**

Это удобная возможность, позволяющая не писать *рутинный деструктор* для удаления всех полей. Если у структуры нет никакой дополнительной логики при удалении кроме удаления всех своих полей, то `Drop` не надо реализовывать вовсе!

**Нельзя запретить это поведение в Rust 1.0.**

Заметьте, обладание `&mut self` означает, что даже если вы сможете отменить рекурсивный вызов деструктора, Rust запретит вам, например, передавать владение полями из self. Для большинства типов это абсолютно нормально.

Например, в собственной реализации `Box` можно написать `Drop` так:

```rust
#![feature(ptr_internals, allocator_api)]

use std::alloc::{Allocator, Global, GlobalAlloc, Layout};
use std::mem;
use std::ptr::{drop_in_place, NonNull, Unique};

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>())
        }
    }
}
# fn main() {}
```

и это нормально работает, потому что, когда Rust собирается удалить поле `ptr`, он видит [Unique], у которого нет реализации `Drop` в данном примере. Аналогично, ничто не сможет использовать-после-освобождения `ptr`, потому что это поле станет недостижимым, когда закончится выполнение деструктора.

В то же время так работать не будет:

```rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Allocator, Global, GlobalAlloc, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Box<T> }

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        unsafe {
            // Гипер-оптимизация: освобождаем содержимое box вместо него,
            // не вызывая `drop` у его содержимого
            let c: NonNull<T> = self.my_box.ptr.into();
            Global.deallocate(c.cast::<u8>(), Layout::new::<T>());
        }
    }
}
# fn main() {}
```

После освобождения ptr из `box` в деструкторе SuperBox Rust с радостью приступит к вызову деструктора у самого box, и все сломается из-за использования-после-освобождения и двойного-освобождения.

Заметьте, что поведение рекурсивного вызова деструктора применяется ко всем структурам и перечислениям, независимо от того, реализуют они Drop или нет. Поэтому что-то вроде

```rust
struct Boxy<T> {
    data1: Box<T>,
    data2: Box<T>,
    info: u32,
}
```

будет вызывать деструкторы полей data1 и data2 всякий раз, когда они "должны" быть удалены, даже при том, что они сами не реализуют Drop. Мы говорим, что такому типу *нужен Drop*, хотя он сам не реализует Drop.

Аналогично,

```rust
enum Link {
    Next(Box<Link>),
    None,
}
```

удалит внутреннее поле Box тогда и только тогда, когда экземпляр будет содержать вариант Next.

В большинстве случаев это отлично работает, потому что вам не нужно волноваться о добавлении/удалении деструкторов во время рефакторинга ваших данных. Тем не менее, есть, конечно, ещё много случаев, в которых действительно нужно делать с деструкторами вещи посложнее.

Классическим безопасным решением для того, чтобы отменить рекурсивный вызов деструктора и позволить передать владение из Self во время `drop`, является использование Option:

```rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Allocator, GlobalAlloc, Global, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Option<Box<T>> }

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        unsafe {
            // Гипер-оптимизация: освобождаем содержимое box вместо него,
            // не вызывая `drop` у содержимого. Необходимо установить поля `box`
            // как `None` для того, чтобы Rust не пытался вызвать Drop у них.
            let my_box = self.my_box.take().unwrap();
            let c: NonNull<T> = my_box.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
            mem::forget(my_box);
        }
    }
}
# fn main() {}
```

В то же время здесь довольно странная семантика: вы говорите, что поле, которое всегда *должно* быть Some, *может* быть None, только потому что это произошло в деструкторе. Конечно, в этом, наоборот, большой смысл: вы можете вызывать произвольные методы у self во время вызова деструктора, а после деинициализации поля это будет делать запрещено. Хотя это и не запретит вам создавать любые другие произвольные недопустимые состояния.

В конечном счёте это нормальное решение. Безусловно, так вы добьётесь отмены вызова деструктора. В то же время мы надеемся, что найдётся в будущем первоклассный способ сообщить, что у поля не должен автоматически вызываться деструктор.


[Unique]: phantom-data.html