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
      photo (5 url, размер каждой 150 byte)
      description (1 char == 3 byte => 150 символов - 450 byte)
      geo (GPS coordinates 8 byte)
  
      traffic write = rps * avg_request_size = 470 * (5*150 + 450 + 8) = 567_760 byte/s = 0.6 MB/s
      traffic write for season = rps * avg_request_size = 940 * (5*150 + 450 + 8)  = 1.2 MB/s
  ```
  
* Публикация вложений для постов:

    ```
    RPS аналогичен публикации постов
    ```
  
    ```
    Traffic write
      5 вложений каждое по 0.5 MB
      traffic write = rps * avg_request_size = 470 * 5 * 0.5 = 1.1 GB/s
      traffic write for season = rps * avg_request_size = 940 * 5 * 3  = 2.2 GB/s
    ```

* Просмотр постов\ленты. Считаем, что за один запрос мы отдаем пользователю 10 постов. Так как в среднем ожидаем, что пользователь просматривает 100 постов, то получается 10 запросов в сутки. В сезон - 200 постов в сутки => 20 запросов в сутки. Плюс к каждому посту нужно вернуть несколько комментариев, пусть будет 5 штук
  ```
  RPS = dau * avg_posts_load_per_day_by_user / 86_400
  RPS read = 20_000_000 * 10 / 86_400 == 2400
  RPS read for season = 20_000_000 * 20 / 86_400 == 4800
  ```

  ```
  Traffic read
      photo (5 url, размер каждой 150 byte)
      description (1 char == 3 byte => 150 символов - 450 byte)
      geo (GPS coordinates 8 byte)

      traffic read = rps * avg_request_size = 2400 * (5*150 + 450 + 8) = 3 MB/s
      traffic read for season = rps * avg_request_size = 4800 * (5*150 + 450 + 8) = 6 MB/s
  ```

* Получение вложений для постов:

    ```
    RPS аналогичен просмотру постов постов
    ```
  
    ```
    Traffic write
      5 вложений каждое по 0.5 MB
      traffic read = rps * avg_request_size = 2400 * 5 * 3MB = 6 GB/s
      traffic read for season = rps * avg_request_size = 4800 * 5 * 0.5MB  = 12 GB/s
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
   
  ```
  Traffic read
      comment (1 char == 3 byte => 150 символов - 450 byte)
  
      traffic read = rps * avg_request_size = 2400 * 450 * 5 = 5.4 MB/S
      traffic write for season = rps * avg_request_size = 4800 * 450 * 5 = 10.8 MB/S
  ```
  
## Требуемая память

### Посты

```bash
traffic = traffic_write + traffic_read = 0.6 MB/s + 3 MB/s = 3.6 MB/s
iops = rps_write + rps_read = 940 + 4800 = 5740

capacity = traffic * 86400 * 365 = 3.6 MB/s * 86400 * 365 = 113 TB

// Посчитаем для SSD размером 30 TB, 1000 iops, пропускная способность 500 MB/s
disks_for_capacity = capacity / disk_capacity = 113 / 30 = 3.8
disks_for_throughput = traffic_per_second / disk_throughput = 3.6 MB/s / 500 MB/s = 0.0072 
disks_for_iops = iops / disk_iops = 5740 / 1000 = 5.74
disks = max(ceil(Disks_for_capacity), ceil(Disks_for_throughput), ceil(Disks_for_iops)) = 6

Итого 6 дисков по 30TB
```

### Комментарии и Реакции (Хранятся в одной БД)

```bash
traffic = traffic_write + traffic_read = 6 MB/s
iops = rps_write + rps_read = 28_000 + 1300 = 30_000

capacity = traffic * 86400 * 365 = 6 MB/s * 86400 * 365 = 190 TB 

// Посчитаем для SSD размером 30 TB, 1000 iops, пропускная способность 500 MB/s
disks_for_capacity = capacity / disk_capacity = 190 / 30 = 6.3
disks_for_throughput = traffic_per_second / disk_throughput = 6 MB/s / 500 MB/s = 0.012
disks_for_iops = iops / disk_iops = 30_000 / 1000 = 30
disks = max(ceil(Disks_for_capacity), ceil(Disks_for_throughput), ceil(Disks_for_iops)) = 30

Итого 30 дисков по 30 TB
```

### Вложения

```bash
traffic = traffic_write + traffic_read = 12 GB/s + 2 GB/s = 14 GB/s
iops = rps_write + rps_read = 940 + 4800 = 5740

// У меня такое ощущение, что я очень сильно просчитался тут, раз до Петабайт добрался. Но вроде картинки гонять в таком объеме - это и правда дорого? Тут без CDN никак вообще.
capacity = traffic * 86400 * 365 = 14 GB/s * 86400 * 365 = 441 PB 

// Посчитаем для SSD размером 100 TB, 1000 iops, пропускная способность 500 MB/s
disks_for_capacity = capacity / disk_capacity = 441 PB / 0.1 = 4410
disks_for_throughput = traffic_per_second / disk_throughput = 86000 MB/s / 500 MB/s = 172 
disks_for_iops = iops / disk_iops = 5740 / 1000 = 5.74
disks = max(ceil(Disks_for_capacity), ceil(Disks_for_throughput), ceil(Disks_for_iops)) = 4410

Итого 4410 дисков по 100TB
```