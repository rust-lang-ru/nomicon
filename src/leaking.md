% Утечка

Основанное на владении управление ресурсами предназначено для упрощения
композиции. Вы получаете ресурсы, создавая объект, и отпускаете ресурсы, удаляя
его. Из-за того что удаление производится за вас, вы не можете забыть отпустить
ресурсы, и это произойдет настолько быстро, насколько это возможно! Конечно, все
замечательно, и все наши проблемы решены.

На самом деле все ужасно и мы должны попытаться решить новые и экзотичные
проблемы.

Многие люди полагают, что Rust устраняет утечку ресурсов. На практике, это в
основном правда. Вы бы очень удивились, увидев, что у программы на Безопасном
Rust утекают ресурсы в неконтролируемом направлении.

Однако с теоретической точки зрения все абсолютно не так, независимо от того,
как вы смотрите на это. В самом строгом смысле, "утечка" настолько абстрактна,
насколько и неизбежна. Довольно просто инициализировать коллекцию вначале
программы, наполнить ее кучей объектов с деструкторами и затем войти в
бесконечный цикл, который никогда не обращается к ней. Коллекция будет
бесполезно сидеть в памяти, удерживая ее драгоценные ресурсы до окончания
программы (в этот момент все эти ресурсы все равно будут собраны сборщиком ОС).

Можем ограничить форму утечки: ошибочная попытка удаления значения, которое уже
недоступно. Rust также не предотвращает это. На самом деле у Rust даже есть
функция *для этого*: `mem::forget`. Эта функция съедает полученное значение *и
не вызывает его деструктор*.

Раньше `mem::forget` помечалась unsafe в качестве статической проверки против
нее, из-за того, что ошибка при вызове деструктора это чаще всего неправильная
вещь (хотя и полезная в некотором особом небезопасном коде). Однако в целом это
принимали как непригодную позицию: есть много способов получить ошибки при
вызове деструктора в безопасном коде. Самым известным примером является создание
цикла из указателей подсчета-ссылок (RC), использующих внутреннюю изменяемость.

Разумно для безопасного кода предполагать, что утечка в деструкторе не
происходит, потому что любая программа с такой утечкой неправильна. Однако
*небезопасный* код не может полагаться, на то, что вызываемые деструкторы
являются безопасными. Для большинства типов это не играет роли: если утечка в
деструкторе, то тип по определению недоступен, поэтому это не важно, не так ли?
Например, если утечка в `Box<u8>`, то вы тратите память впустую, но это вряд ли
нарушит безопасность памяти.

Но, вот где мы должны быть осторожны с утечкой в деструкторах, это в *прокси*
типах. Это типы, управляющие доступом к определенному объекту, но на самом деле
не владеющие им. Прокси объекты довольно редки. Прокси объекты, о которых вы
должны волноваться, еще более редки. Однако, рассмотрим три интересных примера
из стандартной библиотеки:

* `vec::Drain`
* `Rc`
* `thread::scoped::JoinGuard`



## Drain

`drain` это API коллекций, который передает владение данными из контейнера, не 
уничтожая сам контейнер. Это позволяет нам заново использовать место 
расположения `Vec` после передачи владения всего содержимого. Он создает 
итератор (Drain), который возвращает содержимое Vec по-значению.

Теперь, представьте Drain в середине итерации: некоторые значения уже переданы,
некоторые еще нет. Это означает, что часть Vec - это полностью
неинициализированные данные! Мы могли бы сохранять все элементы Vec каждый раз
перед удалением значения, но это будет иметь довольно катастрофические
последствия по производительности.

Вместо этого, мы хотим, чтобы Drain восстанавливал то, что осталось в Vec после
своего удаления. Он должен закончить выполнение, восстановить любые элементы,
которые еще не были удалены (процедить поддиапазоны поддержки), и изменить `len`
у Vec. Он даже безопасен при размотке! Элементарно!

Теперь представим следующее:

```rust,ignore
let mut vec = vec![Box::new(0); 4];

{
    // начало процеживания, vec больше не доступен
    let mut drainer = vec.drain(..);

    // вытаскиваем два элемента и тут же их уничтожаем
    drainer.next();
    drainer.next();

    // избавляемся от drainer, но не вызываем его деструктор
    mem::forget(drainer);
}

// Упсс, vec[0] удален, мы читаем указатель на освобожденную память!
println!("{}", vec[0]);
```

Это определенно Не Хорошо. К сожалению, мы застряли между молотом и наковальней:
поддержка согласованного состояния имеет неподъемную цену (и обесценит любые
преимущества API). Ошибка при поддержке несогласованного состояния дает нам
Неопределенное Поведение в безопасном коде (делает API несостоятельным).

Так что нам делать? Ну, мы можем выбрать обычное согласованное состояние:
установить длину Vec в 0 вначале итерации, и поменять ее при необходимости в
деструкторе. Таким образом, если все выполняется нормально, мы получим
предсказуемое поведение с небольшими накладными расходами. Но если у кого-то
*появится наглость* выполнить mem::forget в середине итерации, все *еще больше
утечет* (и возможно оставит Vec в неожиданном, но при этом согласованном
состоянии). Из-за того, что mem::forget безопасен, все остальное тоже
определенно безопасно. Мы называем утечки, вызывающие еще большие утечки,
*усилением утечки*.




## Rc

Rc is an interesting case because at first glance it doesn't appear to be a
proxy value at all. After all, it manages the data it points to, and dropping
all the Rcs for a value will drop that value. Leaking an Rc doesn't seem like it
would be particularly dangerous. It will leave the refcount permanently
incremented and prevent the data from being freed or dropped, but that seems
just like Box, right?

Nope.

Let's consider a simplified implementation of Rc:

```rust,ignore
struct Rc<T> {
    ptr: *mut RcBox<T>,
}

struct RcBox<T> {
    data: T,
    ref_count: usize,
}

impl<T> Rc<T> {
    fn new(data: T) -> Self {
        unsafe {
            // Wouldn't it be nice if heap::allocate worked like this?
            let ptr = heap::allocate::<RcBox<T>>();
            ptr::write(ptr, RcBox {
                data: data,
                ref_count: 1,
            });
            Rc { ptr: ptr }
        }
    }

    fn clone(&self) -> Self {
        unsafe {
            (*self.ptr).ref_count += 1;
        }
        Rc { ptr: self.ptr }
    }
}

impl<T> Drop for Rc<T> {
    fn drop(&mut self) {
        unsafe {
            (*self.ptr).ref_count -= 1;
            if (*self.ptr).ref_count == 0 {
                // drop the data and then free it
                ptr::read(self.ptr);
                heap::deallocate(self.ptr);
            }
        }
    }
}
```

This code contains an implicit and subtle assumption: `ref_count` can fit in a
`usize`, because there can't be more than `usize::MAX` Rcs in memory. However
this itself assumes that the `ref_count` accurately reflects the number of Rcs
in memory, which we know is false with `mem::forget`. Using `mem::forget` we can
overflow the `ref_count`, and then get it down to 0 with outstanding Rcs. Then
we can happily use-after-free the inner data. Bad Bad Not Good.

This can be solved by just checking the `ref_count` and doing *something*. The
standard library's stance is to just abort, because your program has become
horribly degenerate. Also *oh my gosh* it's such a ridiculous corner case.




## thread::scoped::JoinGuard

The thread::scoped API intends to allow threads to be spawned that reference
data on their parent's stack without any synchronization over that data by
ensuring the parent joins the thread before any of the shared data goes out
of scope.

```rust,ignore
pub fn scoped<'a, F>(f: F) -> JoinGuard<'a>
    where F: FnOnce() + Send + 'a
```

Here `f` is some closure for the other thread to execute. Saying that
`F: Send +'a` is saying that it closes over data that lives for `'a`, and it
either owns that data or the data was Sync (implying `&data` is Send).

Because JoinGuard has a lifetime, it keeps all the data it closes over
borrowed in the parent thread. This means the JoinGuard can't outlive
the data that the other thread is working on. When the JoinGuard *does* get
dropped it blocks the parent thread, ensuring the child terminates before any
of the closed-over data goes out of scope in the parent.

Usage looked like:

```rust,ignore
let mut data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
{
    let guards = vec![];
    for x in &mut data {
        // Move the mutable reference into the closure, and execute
        // it on a different thread. The closure has a lifetime bound
        // by the lifetime of the mutable reference `x` we store in it.
        // The guard that is returned is in turn assigned the lifetime
        // of the closure, so it also mutably borrows `data` as `x` did.
        // This means we cannot access `data` until the guard goes away.
        let guard = thread::scoped(move || {
            *x *= 2;
        });
        // store the thread's guard for later
        guards.push(guard);
    }
    // All guards are dropped here, forcing the threads to join
    // (this thread blocks here until the others terminate).
    // Once the threads join, the borrow expires and the data becomes
    // accessible again in this thread.
}
// data is definitely mutated here.
```

In principle, this totally works! Rust's ownership system perfectly ensures it!
...except it relies on a destructor being called to be safe.

```rust,ignore
let mut data = Box::new(0);
{
    let guard = thread::scoped(|| {
        // This is at best a data race. At worst, it's also a use-after-free.
        *data += 1;
    });
    // Because the guard is forgotten, expiring the loan without blocking this
    // thread.
    mem::forget(guard);
}
// So the Box is dropped here while the scoped thread may or may not be trying
// to access it.
```

Dang. Here the destructor running was pretty fundamental to the API, and it had
to be scrapped in favor of a completely different design.
