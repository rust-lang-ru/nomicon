% Опасности управления ресурсами на основе владения

Управление ресурсами на основе владения (англ. OBRM, Ownership 
Based Resource Management) - RAII: Resource Acquisition Is Initialization
- это то, с чем вы будете много сталкиваться в Rust. Особенно если будете
использовать стандартную библиотеку.

Грубо говоря, правило следующее: для получения ресурса, вы создаете объект,
управляющий им. Для освобождения ресурса просто уничтожаете объект, а он сам
чистит ресурсы за вас. Самым частым "ресурсом", который управляется этим
правилом, является просто *память*. `Box`, `Rc` и почти все в `std::collections`
- это удобство, созданное для правильного управления памятью. Это особенно важно
в Rust, потому что у нас нет всепроникающего GC, на который можно было бы
возложить управление памятью. Важно понимать: Rust - это контроль. В то же время
мы не ограничены только памятью. Почти каждый ресурс системы - поток, файл или
сокет - проходит через этот API.