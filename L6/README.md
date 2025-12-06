# Исследование вредоносной активности в домене Windows


## Цель работы

1 Закрепить навыки исследования данных журнала Windows Active Directory
2 Изучить структуру журнала системы Windows Active Directory 3 Зекрепить
практические навыки использования языка программирования R для обработки
данных 4 Закрепить знания основных функций обработки данных экосистемы
tidyverse языка R

### Ход выполнения

1 Импортируйте данные в R. Это можно выполнить с помощью
`jsonlite::stream_in(file())` . Датасет находится по адресу
[ссылка](https://storage.yandexcloud.net/iamcth-data/dataset.tar.gz).

``` r
library(jsonlite)
library(tidyverse)
```

    Warning: пакет 'tidyverse' был собран под R версии 4.5.2

    Warning: пакет 'ggplot2' был собран под R версии 4.5.2

    Warning: пакет 'tidyr' был собран под R версии 4.5.2

    Warning: пакет 'purrr' был собран под R версии 4.5.2

    Warning: пакет 'forcats' был собран под R версии 4.5.2

    Warning: пакет 'lubridate' был собран под R версии 4.5.2

    ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ✔ dplyr     1.1.4     ✔ readr     2.1.5
    ✔ forcats   1.0.1     ✔ stringr   1.5.2
    ✔ ggplot2   4.0.1     ✔ tibble    3.3.0
    ✔ lubridate 1.9.4     ✔ tidyr     1.3.1
    ✔ purrr     1.2.0     
    ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ✖ dplyr::filter()  masks stats::filter()
    ✖ purrr::flatten() masks jsonlite::flatten()
    ✖ dplyr::lag()     masks stats::lag()
    ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

``` r
library(xml2)
```

    Warning: пакет 'xml2' был собран под R версии 4.5.2

``` r
library(rvest)
```

    Warning: пакет 'rvest' был собран под R версии 4.5.2


    Присоединяю пакет: 'rvest'

    Следующий объект скрыт от 'package:readr':

        guess_encoding

``` r
untar(tarfile = "dataset.tar.gz", exdir = "./")
data <- jsonlite::stream_in(file("caldera_attack_evals_round1_day1_2019-10-20201108.json"), verbose = FALSE)
```

1 1 Импортируем справочник по кодам журнала Windows

``` r
webpage_url <- "https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/appendix-l--events-to-monitor"
webpage <- xml2::read_html(webpage_url)
event_df <- rvest::html_table(webpage)[[1]]
```

1 2 Приведем справочник к tidy формату

``` r
clean_event_df <- event_df %>%
  rename(., 
         Current_Windows_Event_ID=1,
         Legacy_Windows_Event_ID=2,
         Potential_Criticality=3,
         Event_Summary=4
         ) %>%
  mutate(.,
         Current_Windows_Event_ID = as.integer(Current_Windows_Event_ID),
         Legacy_Windows_Event_ID = as.integer(Legacy_Windows_Event_ID)
         )
```

    Warning: There were 2 warnings in `mutate()`.
    The first warning was:
    ℹ In argument: `Current_Windows_Event_ID =
      as.integer(Current_Windows_Event_ID)`.
    Caused by warning:
    ! в результате преобразования созданы NA
    ℹ Run `dplyr::last_dplyr_warnings()` to see the 1 remaining warning.

``` r
glimpse(clean_event_df)
```

    Rows: 381
    Columns: 4
    $ Current_Windows_Event_ID <int> 4618, 4649, 4719, 4765, 4766, 4794, 4897, 496…
    $ Legacy_Windows_Event_ID  <int> NA, NA, 612, NA, NA, NA, 801, NA, NA, 550, 51…
    $ Potential_Criticality    <chr> "High", "High", "High", "High", "High", "High…
    $ Event_Summary            <chr> "A monitored security event pattern has occur…

2 Привести датасеты в вид “аккуратных данных”, преобразовать типы
столбцов в соответствии с типом данных

``` r
data <- data %>%
  rename(., 
         metadata = `@metadata`, 
         timestamp = `@timestamp`
  ) %>%
  mutate(., timestamp = as.POSIXct(timestamp, format = "%Y-%m-%dT%H:%M:%OSZ", tz = "UTC"))
```

3 Просмотрим общую структуру данных с помощью функции glimpse()

``` r
glimpse(data)
```

    Rows: 101,904
    Columns: 9
    $ timestamp <dttm> 2019-10-20 20:11:06, 2019-10-20 20:11:07, 2019-10-20 20:11:…
    $ metadata  <df[,4]> <data.frame[26 x 4]>
    $ event     <df[,4]> <data.frame[26 x 4]>
    $ log       <df[,1]> <data.frame[26 x 1]>
    $ message   <chr> "A token right was adjusted.\n\nSubject:\n\tSecurity ID:\…
    $ winlog    <df[,16]> <data.frame[26 x 16]>
    $ ecs       <df[,1]> <data.frame[26 x 1]>
    $ host      <df[,1]> <data.frame[26 x 1]>
    $ agent     <df[,5]> <data.frame[26 x 5]>

4 Раскройте датафрейм избавившись от вложенных датафреймов. Для
обнаружения таких можно использовать функцию `dplyr::glimpse()` , а для
раскрытия вложенности – `tidyr::unnest()` . Обратите внимание, что при
раскрытии теряются внешние названия колонок – это можно предотвратить
если использовать параметр `tidyr::unnest(..., names_sep = )`

``` r
data_clean <- data %>%
  unnest(
    cols = c(metadata, event, log, host, agent, ecs),
    names_sep = "_",
    keep_empty = TRUE
  ) %>%
  unnest(
    cols = winlog,
    names_sep = "_",
    keep_empty = TRUE
  ) %>%
  unnest(
    cols = c(winlog_event_data, winlog_user, winlog_user_data, winlog_process),
    names_sep = "_",
    keep_empty = TRUE
  ) %>%
  mutate(., 
        event_created = as.POSIXct(event_created, format = "%Y-%m-%dT%H:%M:%OSZ", tz = "UTC")
  )
```

5 Минимизируйте количество колонок в датафрейме – уберите колоки с
единственным значением параметра

``` r
data_clean_min <- data_clean %>%
  select(where(~ n_distinct(.x) > 1))
```

6 Какое количество хостов представлено в данном датасете?

``` r
print(data_clean_min %>%
  pull(winlog_computer_name) %>%
  na.omit() %>%
  unique() %>%
  length())
```

    [1] 5

7 Подготовьте датафрейм с расшифровкой Windows Event_ID, приведите типы
данных к типу их значений

`✅✅✅`

8 Есть ли в логе события с высоким и средним уровнем значимости? Сколько
их?

``` r
result <- data_clean_min %>%
  left_join(., clean_event_df, by=c("event_code"="Current_Windows_Event_ID")) %>%
  filter(., Potential_Criticality == "High" | Potential_Criticality == "Medium" | Potential_Criticality == "Medium to High")

result
```

    # A tibble: 0 × 290
    # ℹ 290 variables: timestamp <dttm>, event_created <dttm>, event_code <int>,
    #   event_action <chr>, log_level <chr>, message <chr>,
    #   winlog_event_data_SubjectDomainName <chr>,
    #   winlog_event_data_TargetDomainName <chr>,
    #   winlog_event_data_SubjectUserSid <chr>,
    #   winlog_event_data_SubjectUserName <chr>,
    #   winlog_event_data_TargetUserName <chr>, …

## Оценка результата

В результате анализа журнала Windows оказалось, что потенциально опасных
событий нет.

## Вывод

Таким образом, мы: - закрепили навыки исследования данных журнала
Windows Active Directory - изучили структуру журнала системы Windows
Active Directory - закрепили практические навыки использования языка
программирования R для обработки данных - Закрепили знания основных
функций обработки данных экосистемы tidyverse языка R
