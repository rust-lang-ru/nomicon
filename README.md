# Rustonomicon

Перевод "Rustonomicon"

# Оригинал

https://github.com/rust-lang/rust/tree/master/src/doc/nomicon

# Чёрная магия программирования на Небезопасном Rust

Примечание: это чёрновик книги, и он может содержать серьёзные ошибки.

> Instead of the programs I had hoped for, there came only a shuddering
blackness and ineffable loneliness; and I saw at last a fearful truth which no
one had ever dared to breathe before — the unwhisperable secret of secrets — The
fact that this language of stone and stridor is not a sentient perpetuation of
Rust as London is of Old London and Paris of Old Paris, but that it is in fact
quite unsafe, its sprawling body imperfectly embalmed and infested with queer
animate things which have nothing to do with it as it was in compilation.

(*Пассаж в стиле Лавкрафта, который переводчик тронуть не рискнул*)

Эта книга покажет вам все ужасные подробности, которые нужно знать, чтобы
правильно писать на Небезопасном Rust. Естественно, это может привести к
высвобождению несказанных ужасов, которые разобьют вашу душу на неисчислимое
количество бесконечно малых осколков безысходности.

Если вы хотите долго и счастливо программировать на Rust, вам стоит развернуться
и забыть, что вы когда-либо видели эту книгу. Она не нужна. Однако, если вы
собираетесь писать небезопасный код - или просто хотите залезть в потроха
языка - эта книга содержит бесценную информацию.

В отличие от [Книги][trpl], для чтения этих писаний потребуется заметная
подготовка. В частности, вы должны хорошо представлять себе основы системного
программирования и Rust. Если это не так, вам следует сначала ознакомиться с
[Книгой][trpl]. Хотя, мы всё же не будем предполагать, что вы сделали это, и
постараемся напоминать о всех необходимых моментах. Вы можете сразу читать эту
книгу; просто не ожидайте, что мы будем объяснять всё с самого начала.

Поясним: эта книга крайне подробна. Мы будем вникать в безопасность с точки
зрения исключений (exception safety), совпадение указателей (pointer aliasing),
модели памяти, и даже теорию типов. Также мы будем много говорить о разных видах
безопасности и гарантий.

[trpl]: https://github.com/ruRust/rust_book_ru
