### Важно
Возможны ошибки, связанные с txt файлами (вернее их отсутствием). Я не нашёл нигде инструкцию, как заставить докер создавать файлики, поэтому закинул их ручками в репо. Если будут вопросы (или вы вдруг решите мне объяснить как это грамотно делать) - пишите @black_chick 
Также, я не уверен, что скрипт бота и parse.py корректно работают параллельно, ибо я залил на облако в день дедлайна и дальше локального тестирования не уходил (тоже хотел бы услышать коммент по этому поводу). Вижу, что на облаке возникают конфликты при очередном парсинге, но вроде это не мешает работе. Если в терминале в разных вкладках запускать, то норм

**ps** если решите сами протестить бота, то там после выбора первого размера будет доступна в самом верху над размерами кнопка пропустить (не знаю, как поставить её куда-то на видное место)

**pps** я не уверен, что на момент проверки бот будет жив, ибо могут возникнуть какие-то оч странные ошибки во время парсинга (вернее работы с библиотеками для этого дела)

### Смысл бота

Зачастую, люди хотят купить вещь на скидках, но им лень заходить периодически на сайт и мониторить интересные предложения, подходящие им. Я создал бота, который самостоятельно отслеживает новые скидки и уведомляет заинтересованных в ней юзеров. 

### @SneakerSales_Bot
Бот доступен по адресу **@SneakerSales_Bot** в telegram

Используется тематическая картинка и описание для бота

Код написан с использованием библиотеки telebot. Первое сообщение от пользователя '/start' появляется при запуске бота.
Бот приветствует и сразу предлогает выбрать параметры (пол, размер), либо информирует пользователя о его текущих параметрах. Реализовывается стандартный многоступенчатый диалог бота с юзером при помощи запоминания user_step (на каком уровне диалога пользователь) в словарь {id: step}. Пользователь сообщает тип модели и размеры. 

Бот записывает id пользователя в текстовый документ по адресу "model type"/"model type""size".txt в папке sought_for_items. Также запоминает параметры пользователя в creating_users {id: user_parameters}, где user_parameters - объект соответствующего простого класса для удобной работы с параметрами юзеров. 

**Доступны команды**

/report - пользователь сообщает свои замечания боту, он пишет их в report.txt

/edit - пользователь меняет свои параметры (почти также, как при первом запуске, только бот переспрашивает). Тут, к сожалению, никакой эвристики нет. Бот за линию удаляет id пользователя из соответствующих текстовых документов.

/help - информирует о доступных командах

### Scrapping

**Подготовка текстовых документов перед записью новых данных**

Функция **prepare_txt_files** принимает расположение текстового документа, в который записан последний скрап (new) и 
расположение текстового документа, в котором предыдущий скрап (old).

Соответственно, функция готовит эти два файла для записи: 

    1) очищает old
    2) переносит данные из new в old
    3) очищает new, чтобы туда можно было записать скрап. 
    
**Настойчиво посылаем запросы**

Так как сайты имеют защиту от подобного рода занятий, приходится посылать запросы несколько раз. 
Функция **get_response** принимет ссылку и пытается получить response с фейковым user agent и моими cookies, вытащенными при загрузке brandshop. При возникновении ошибки 403 - посылает запрос заново через 2 -  10 секунды. При ошибке 404 - возвращает None.

**Сбор данных**

Функции 'shop name'_scrap скрапят данные из раздела "SALES" со всех страниц (логика говорит парсить только первую страницу и смотреть обновления на ней, но сайт ведёт себя не культурно и постоянно перемешивает товары).
Сначала происходит подготовка соответствующих тесктовых документов для записи скрапа prepare_txt_files.
Далее настойчиво запрашиваем страницу скидок.
Функция вытаскивает название модели, ссылку на неё, ссылку на фото в приемлемом качестве, старую и новую цены, тип модели и доступные размеры. Запускает **add_sneakers_scrap**, которая добавляет данные в соответствующий текстовый документ. 


### Поиск новых скидок, уведомление пользователей

Функция **find\_new\_items\_"shop name"** пробегает по старому и новому скрапу магазина и переносит вещи, которые встречаются только в новом скрапе в файл new\_items\_'shop name'.txt. Также, если в новом и старом скрапе присутствует одинаковая вещь, но в новом больше её размеров, то вещь перенесётся только с новыми размерами.

Функция **notify\_about\_new\_items\_"shop name"** просто считывает соответствующий документ new_items, извлекает из него параметры вещи и запускает  **notify\_about\_item** - функция уведомлеяет юзеров (отправляет доступный размер и ссылку) по id из текстового документа, соответствующего типу модели и размеру. Некоторые производители делают половинные размеры, например 42.5 EU. При появлении такого товара будут уведомленны пользователи, ожидающие размер 42 и 43. Если сообщение отправить не удалось (пользователь выключил бот), то его id будет удалён из текстового документа, чтобы не тратить на него время в следующий раз (конечно, юзер сможет заново запустить бот и передать данные)

Так хочется проверять новые скидки и уведомлять пользователей с какой-то периодичностью, я делаю функцию **regular\_update\_and\_notification**, которая выполняет сценарий скрапинга, поиска новых вещей и уведомления пользователей. С помощью модуля schedule реализую периодичное выполнение regular\_update\_and\_notificatio с частотой в день
