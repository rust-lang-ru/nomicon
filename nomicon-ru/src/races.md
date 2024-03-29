# Гонки данных и их условия

Безопасный Rust гарантирует отсутствие гонок данных, определяемых так:

- два или больше потока одновременно получают доступ к участку памяти
- один из них пишет
- один из них не синхронизирован

У гонок данных Неопределённое Поведение, и, соответственно, их невозможно выполнить в безопасном Rust. Гонки данных *в большинстве своём* предотвращаются системой владения Rust: изменяемые ссылки не могут совпадать, поэтому и невозможно получить гонки данных. Внутренняя изменяемость делает все сложней, именно поэтому у нас есть типажи Send и Sync (смотри ниже).

**Однако Rust не предотвращает общие условия для гонок.**

Это принципиально невозможно, и, честно говоря, нежелательно. Ваше железо может вызывать гонки, ваша ОС может вызывать гонки, другие программы на вашем компьютере могут вызывать гонки и мир, в котором все это выполняется, тоже этому подвержен. Всеми системами, искренне утверждающими, что предотвращают *все* состояния гонок, будет довольно ужасно пользоваться, если просто не невозможно.

Поэтому программы на безопасном Rust абсолютно "спокойно" могут зайти в тупик или сделать что-нибудь невероятно глупое в случае некорректной синхронизации. Очевидно, такие программы не очень хороши сами по себе, но Rust не может предотвратить всё. Итак, состояния гонок сами по себе не могут нарушить безопасность памяти в Rust. Они могут это сделать только вместе с другим небезопасным кодом. Например:

```rust,no_run
use std::thread;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

let data = vec![1, 2, 3, 4];
// Arc нужен затем, чтобы память, в которой хранится AtomicUsize, всё ещё существовала
// на момент, когда другой поток попытается увеличить этот AtomicUsize, даже если
// исполнение полностью завершится к этому моменту. Rust не компилирует программу без этого,
// из-за требований времен жизни для thread::spawn!
let idx = Arc::new(AtomicUsize::new(0));
let other_idx = idx.clone();

// `move` захватывает other_idx по-значению, передавая его в этот поток
thread::spawn(move || {
    // Нормально изменять idx, потому что это значение атомарно,
    // тем самым оно не может вызвать гонку данных.
    other_idx.fetch_add(10, Ordering::SeqCst);
});

// Индексируем значением, полученным из атомарного. Это безопасно, потому что мы
// читаем атомарную память только один раз, и затем передаем копию этого
// значения в реализацию индексирования Vec. Это индексирование проверит
// корректность границ и шанс, что значение поменяется в середине выполнения,
// равен нулю. Но наша программа может вызвать панику если поток, который мы
// создали выполнит инкремент перед этим запуском. Условия для гонки во время
// корректного выполнения программы (паника очень редко правильна) зависит от
// порядка вызова потоков.
println!("{}", data[idx.load(Ordering::SeqCst)]);
```

```rust,no_run
use std::thread;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

let data = vec![1, 2, 3, 4];

let idx = Arc::new(AtomicUsize::new(0));
let other_idx = idx.clone();

// `move` захватывает other_idx по-значению, передавая его в этот поток
thread::spawn(move || {
    // Нормально изменять idx, потому что это значение атомарно,
    // тем самым оно не может вызвать гонку данных.
    other_idx.fetch_add(10, Ordering::SeqCst);
});

if idx.load(Ordering::SeqCst) < data.len() {
    unsafe {
        // Некорректная загрузка idx после выполнения проверки границ.
        // Оно может поменяться. Это условие для гонки, *и это опасно*,
        // потому что мы решили сделать `get_unchecked`, который `unsafe`.
        println!("{}", data.get_unchecked(idx.load(Ordering::SeqCst)));
    }
}
```
