# System design социальной сети для путешественников

## Функциональные требования
- личный кабинет
- публикация постов с дополнительным контентом:
    * фотографии (до 5 штук)
    * описание (до 150 символов)
    * привязка геолокации
- реакции на посты (стандартные лайки и дизлайки)
- комментарии к постам (до 150 символов)
- система подписок
- поиск по тексту и по геолокации
- просмотр ленты по заданным фильтрам:
    * только подписки
    * по геолокации
    * все подряд

## Нефункциональные требования
- Аудитория приложения
    * DAU - закладываемся на 20_000_000 пользователей, так как планируется постоянный линейный рост
    * Среднестатистическая User story:
        1) 2 поста в день (утро, вечер)
        2) 30 реакций в день
        3) 3 комментария в день
        4) просматривает 100 постов в день
    * Регионы использования - только СНГ
- Особенности приложения
    * Сезонность есть. Так как говорим о странах СНГ, то нужно ожидать повышенную пользовательскую активность с мая по сентябрь и в январе (длительные праздники, летний период отпусков)
        1) 4 поста в день
        2) 60 реакций в день
        3) 6 комментариев в день
        4) просмотр 200 постов в день
    * Данные храним всегда
    * Максимльное количество подписок - 5_000. Максимальное количество подписчиков - 1_000_000.
    * Доступность приложения 99.95
    * Временные ограничения:
        1) Публикация поста не более 1 секунды
        2) Публикация комментария не более 0.5 секунд
        3) Подгрузка ленты не более 2-3 секунд
    * Приложение для смартфонов, web-приложение для ПК

## Оценка нагрузки

* Публикация постов:
  
  ```
  RPS = dau * avg_post_publishing_per_day_by_user / 86_400
  RPS write = 20_000_000 * 2 / 86_400 == 470
  RPS write for season = 20_000_000 * 4 / 86_400 == 940
  ```

  ```
  Traffic write
      photo (Будем подгружать картинки 1080 x 1350 px = 1_458_000 px => 1_458_000 bits => 1_458 byte на одну фотографию. Их 5 штук)
      description (1 char == 3 byte => 150 символов - 450 byte)
      geo (GPS coordinates 8 byte)
  
      traffic write = rps * avg_request_size = 470 * (5*1_458 + 450 + 8) = 3_700_000 byte/s = 3_700 kB/s = 3,7 mB/s
      traffic write for season = rps * avg_request_size = 940 * (5*1_458 + 450 + 8)  = 7,4 mB/s
  ```

* Просмотр постов\ленты. Считаем, что за один запрос мы отдаем пользователю 10 постов. Так как в среднем ожидаем, что пользователь просматривает 100 постов, то получается 10 запросов в сутки. В сезон - 200 постов в сутки => 20 запросов в сутки. Плюс к каждому посту нужно вернуть несколько комментариев, пусть будет 5 штук
  ```
  RPS = dau * avg_posts_load_per_day_by_user / 86_400
  RPS read = 20_000_000 * 10 / 86_400 == 2400
  RPS read for season = 20_000_000 * 20 / 86_400 == 4800
  ```

  ```
  Traffic read
      photo (Будем подгружать картинки 1080 x 1350 px = 1_458_000 px => 1_458_000 bits => 1_458 byte на одну фотографию. Их 5 штук)
      description (1 char == 3 byte => 150 символов - 450 byte)
      geo (GPS coordinates 8 byte)
      comments (5 comments по 450 byte каждый => 2_250 byte)

      Подгружаем 100 постов в сутки
      traffic read = rps * avg_request_size = 2400 * 100 * (5*1_458 + 450 + 8 + 2_250) = 2,4 gB/s
      traffic read for season = rps * avg_request_size = 2400 * 100 * (5*1_458 + 450 + 8 + 2_250) = 4,8 gB/s
  ```

  
* Реакции:
  ```
  RPS = dau * avg_reactions_per_day_by_user / 86_400
  RPS write = 20_000_000 * 30 / 86_400 == 14_000
  RPS write for season = 20_000_000 * 60 / 86_400 == 28_000
  ```

  ```
  Traffic write
      reaction (bool - 1 byte)
      
      traffic write = rps * avg_request_size = 14_000 * 1 = 14_000 byte/s = 14 kB/s
      traffic write for season = rps * avg_request_size = 28_000 * 1 = 28_000 byte/s = 28 kB/s
  ```
  
* Комментарии:
  ```
  RPS = dau * avg_comments_publishing_per_day_by_user / 86_400
  RPS write = 20_000_000 * 3 / 86_400 == 700
  RPS write for season = 20_000_000 * 6 / 86_400 == 1300
  ```

  ```
  Traffic write
      comment (1 char == 3 byte => 150 символов - 450 byte)
  
      traffic write = rps * avg_request_size = 700 * 450 = 315_000 byte/s = 315 kB/s
      traffic write for season = rps * avg_request_size = 1300 * 450 = 630 kB/s
  ```