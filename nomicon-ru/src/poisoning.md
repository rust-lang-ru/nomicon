# Отравление

Несмотря на то что весь небезопасный код *должен* гарантировать минимальную безопасность исключений, не все типы гарантируют *максимальную* безопасность исключений. Даже если тип гарантирует её, ваш код может приписать ему дополнительное значение. Например, integer точно безопасен от исключений, но никакой семантики этого у него самого нет. Возможно, код, вызывающий панику, не сможет корректно изменить integer, создав несогласованное состояние программы.

*Обычно* это нормально, потому что всё, что наблюдает возникновение исключения должно быть уничтожено. Например, если вы посылаете Vec в другой поток и тот поток вызывает панику, неважно в каком состоянии находится Vec. Он будет уничтожен и отброшен навсегда. Но есть некоторые типы, которые особенно хороши в контрабанде своих значений через границы паники.

Эти типы могут явно *отравить* себя, если становятся свидетелями паники. Отравление ничего не влечёт за собой. Обычно оно просто означает препятствие нормальной работе. Самым заметным примером этого является Mutex из стандартной библиотеки. Mutex отравится, если один из его MutexGuards (то, что он возвращает когда получает захват) удалится во время паники. Все будущие попытки захватить Mutex вернут `Err` или панику.

Mutex отравляется не для настоящей безопасности в том смысле, как обычно это понимает Rust. Он отравляется как охранник безопасности слепого использования пришедших во время паники данных пока Mutex был захвачен. Данные в таком Mutex были в каком-то промежуточном состоянии, и поэтому, возможно, являются несогласованными или незаконченными. Необходимо отметить, что нельзя нарушить безопасность памяти таким типом, если он корректно написан. В конце концов он должен быть в минимальной безопасности исключений!

Однако, если Mutex содержит, скажем, BinaryHeap, у которого на самом деле нет свойств кучи, не похоже, что любой код, использующий его, делает то, что задумывал автор. Поэтому программа не будет работать правильно. Итак, если вы дважды убедились, что вы можете сделать *что-то* со значением, Mutex предоставляет метод для получения захвата в любом случае. Это *безопасно* в конце концов. Просто может получиться чушь.
