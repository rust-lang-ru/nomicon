# Пример: Реализация Vec

Объединив все вместе, напишем `std::Vec` с самого начала. Данный проект будет работать только на стабильной сборке. Мы не будем использовать разные внутренние инструменты, которые смогу сделать наш код более красивым или эффективным из-за того что они перманентно нестабильны. Хотя многие встроенные функции в других местах действительно *стали* стабильными (`std::ptr` и `str::mem` состоят из множества<br>встроенных функций).

В общем случае это означает, что наша реализация не будет обладать преимуществами всех возможных оптимизаций, хотя и без сомнений не будет *наивной*. Мы погрузимся во все самые мелкие детали, даже если вопросы не будут стоить выеденного яйца.

Вы хотели продвинутого программирования. Будет вам продвинутое.
