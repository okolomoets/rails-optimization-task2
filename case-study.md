# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она потребляла слишком много памяти `~576 MB` (измерено с помощью команды `ps`).

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на используемую программой память я придумал использовать такую метрику: затраченая память на файле меньшего размера

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений за 1-2 секунды


Вот как я построил `feedback_loop`: 
- Создал набор тестовых файлов меньшего размера
- Нашел Главную Точку Роста
- Исправил ее
- Проверил тест

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался: `memory_profiler`, `ruby-prof`

Приняв во внимание ошибки допущенные при оптимизации по `CPU`, я решил выполнять оптимизации более маленькими шагами.  
Вот какие проблемы удалось найти и решить

### Ваша находка №1
Для того, чтобы найти "точку роста" на этом шаге я воспользовался `memory_profiler`. Использовался файл объемом `~5 mb` тестовых данных. 

Тест указал, что главная точка роста находится в файле `task.rb:16` (`~71.08 MB`), а так же то что основное время уходит на алокауию памяти под объекты класса `String`(`~111.38 MB`).
В этой строке, есть только одна строка, которая могла вызвать такой эфект -- это сепаратор, который передается в функцию `split`.

Для того что бы улучшить работоспособность, я добавил заморозку строковых литералов (`# frozen_string_literal: true`) в начало документа.

Это немного улучшило показатели, но не слишком существенно, стало: 
- `task.rb:18` (`~65.08 MB`) 
- алокация под `String`(`~99.38 MB`)
- Потребляемая память `~523 MB`
 
Точка роста при этом не изменилась :(

### Ваша находка №2

Проведя дальнейший анализ, понял, что необходимо переделать выполнение программы в потоковый режим. 
Без этого никакие изменения не дадут существенного результата.

Поэтому я полностью переделал выполение на поток и обновил спеки

После проделаных изменений 
- Потребляемая память на продакшн файлу составила `15 MB` 
- Время выполнения программы скратилось до `~15 sec`

[Скриншот с результатами](http://joxi.ru/gmvlbJNUvBRVvA)
[Скриншот с результатами Visualizer](http://joxi.ru/bmoJ0ZXH9oqb0r)


## Результаты
Удалось улучшить потребление памяти с `~576 MB` до `15 MB` и уложиться в заданный бюджет (`70 MB`).


## Защита от регрессии производительности
Для того что бы защитить выполнение от регресии был добавлен тест на выполнение, который проверяет, что файл объемом `~1mb` обрабатывается меньше чем за 0.5 секунды.
