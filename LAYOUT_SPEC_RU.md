# HobDrive Layout Specification (Russian Translation)

Этот документ собирает теги, атрибуты и опции, используемые в файлах макетов (layout) интерфейса HobDrive.

Цель: дать краткую и практичную справку для создания и редактирования UI-раскладок (панели, экраны ЭБУ, гибрид/GPS панели и т.д.).

## Контракт макета
- Вход: XML-подобные файлы layout (теги: `ui`, `section`, `grid`, `item`, `union`, `stack-vertical`, `stack-horizontal` и т.д.)
- Выход: отображаемые UI-экраны в HobDrive; условная видимость и декораторы влияют на поведение во время выполнения
- Режимы ошибок: неверные атрибуты игнорируются или используются значения по умолчанию; отсутствующие датчики отображаются пустыми/недоступными
- Критерии успеха: макет рендерится и учитывает условные атрибуты `if`, размещение в сетке, декораторы и атрибуты виджетов

## Верхнеуровневые теги
- `ui` — корневой элемент файла layout. Содержит элементы `section` и другие конструкции макета.

- `include` — позволяет встраивать другой файл layout или фрагмент. Атрибуты:
  - `file` — относительный путь к включаемому файлу (например, `file="default-tripcomp.layout"`).
  - `if` — опциональное условие, контролирующее, обрабатывается ли включаемый файл (та же синтаксис выражений, что и `section` `if`).
  - Инклюды обрабатываются при загрузке файла и обычно используются для составления главных макетов из переиспользуемых фрагментов (trip computer, HUD, dashkits).

Пример:
```xml
<include file="default-tripcomp.layout" if='${!sexists("GasTrigger")}'/>
```

## `section`
Определяет логический блок экрана/страницы. Общие атрибуты:
- `name` — человекочитаемое имя секции (используется в списках UI).
- `fixed` — `true/false` (зафиксирована ли секция в навигации).
- `if` — условное выражение (имя датчика или выражение). Секция показывается только если условие истинно. Примеры: `if="Oxygen_b1s1"`, `if="PortraitLayout"`, `if="Renault_01_11103.ControlModuleVoltage"`.
- `class` — подобие CSS-класса, используемого системой для применения предопределённого поведения (например, `SensorList`).
- `ns` — namespace для фильтрации датчиков при использовании `SensorList`.
- `exclude` — regex для исключения ID датчиков в автогенерируемых списках сенсоров.
- `maximize-focus` — (из changelog) `true/false` убирает дополнительное управление при фокусе.
- `controls` — `autohide` или другие значения для управления видимостью кнопок UI.

Примеры:
```xml
<section name="GPS">
  ...
</section>
```

```xml
<section name="Fuel Trims" if="STFT1">
  ...
</section>
```


### `config` элементы
- `config` — объявляет конфигурируемые в UI опции, которые появляются в диалогах настроек. Типичные атрибуты:
  - `id` — уникальный идентификатор, доступный в выражениях макета (например, `${Layout_ECUFuel}`).
  - `type` — `bool`, `int`, `string` и т.д.
  - `default` — значение по умолчанию.
  - `title`, `descr` — текст, отображаемый в UI; поддерживаются локализованные варианты вроде `title-ru`, `descr-ru`.

Пример:
```xml
<config id="Layout_ECUFuel" type="bool" default="false"
        title="Use Fuel Level data from ECU"
        descr="If enabled - fuel level will be shown from ECU. Otherwise - estimated from fuel consumption"
        title-ru="Показывать уровень топлива из ЭБУ"
        descr-ru="Если включено - уровень топлива будет браться из ЭБУ. Иначе - показывается оценочный уровень топлива"/>
```

## Контейнеры макета
 - `grid` — размещает дочерние `item` в табличной сетке.
  - Атрибуты: `rows`, `cols` — значения, разделённые запятыми, задающие подсказки по размерам. Пример: `rows=",," cols=",35"`.
  - `rows` и `cols` могут включать численные высоты/ширины или пустые элементы для гибкой размерности.
  - Сетки можно вкладывать.

- `stack-vertical` / `stack-horizontal` — укладывают элементы вертикально/горизонтально; может использоваться `columns`.
  - Пример: `<stack-vertical columns="${LandscapeLayout ? 2 : 1}">`

- `union` — позволяет композировать несколько альтернативных макетов в одной области (используется в TPMS и т.д.).
  - Применяйте `union`, когда несколько взаимоисключающих подмакетов занимают одну и ту же область; каждый ребёнок обычно содержит свою `grid`.

 - `optional` / `volatile` (атрибуты секции) — лёгкие подсказки, используемые в некоторых default-layout: `optional="true"` отмечает экраны, которые можно скрыть, `volatile` отмечает временные экраны.

- `switch` — выбирает один из дочерних элементов для показа на основе выражения `index`. Полезно для циклических списков или условных блоков.
  - Атрибуты:
    - `index` — выражение или целое число, выбирающее, какой ребёнок показать (0-based). Может ссылаться на значения сенсоров через `svalue(...)` или любые TinyExe-выражения.
    - `interval` — при наличии `index` может периодически пересчитываться с указанным интервалом (в миллисекундах) для циклирования детей.
    - `if` — опциональное условие для включения/выключения `switch` целиком.
  - Дети обычно `item` (или вложенные `grid`); когда `index` даёт номер ребёнка, показывается соответствующий элемент.

Пример (циклим температуры каждые 4 секунды):
```xml
<switch if='${Layout_RollTemp = true}' index='$${ svalue("DateTime") / 4000 }'>
  <item id="ATFTemp" .../>
  <item id="CVTTemp" .../>
  <item id="EngineOilTemp" .../>
</switch>
```

## `item`
Представляет виджет в `grid` или `stack`. Общие атрибуты и примеры использования:

- `id` — идентификатор датчика или специального управления. Примеры: `RPM`, `Latitude`, `ControlModuleVoltage`.
- `type` — `button`, `static` и пр. По умолчанию — виджет для отображения значения датчика.
- `text` — для кнопок или статических блоков — видимый текст.
- `action` — для `button`: строка действия, например `invoke(FAN_ON)` или `run(...)`.
- `size` — подсказка визуального размера: `micro`, `small`, `normal`, `large`, `huge`, `extralarge` или числовое значение.
- `precision` — количество знаков после запятой или специальные теги вроде `6fixed`.
- `inherit` — ссылается на имя декоратора/стиля, например `inherit="_ColoredOnOff"`.
- `colspan` / `rowspan` — раcширение элемента на несколько ячеек сетки.
- `description` — дополнительная метка/описание (например `description="aside"`).
- `period` — период обновления в миллисекундах (например, `period='5000'`).
- `wrap` — фильтр для сглаживания/предсказания: примеры из changelog:
  - `wrap="predict(50,0.7)"`
  - `wrap="smooth(400,100,5)"`
  - `wrap="average(30)"`
- доп. атрибуты: `text-values`, `custom-units`, `colored-descr`, `text-evaluator`

- `onclick` — обработчик клика на элементе (навигация или запуск действия). Примеры: `onclick="NewFueling"`, `onclick="Go(Efficiency)"`.
- `actions` — список действий через запятую, доступных в диалоге информации о сенсоре. Пример: `actions="SetupOdometer"`.
- `units` — расположение текста единиц: `units="below"`, `units="aside"` или `units="hidden"`.
- `interval` — интервал обновления для динамики UI или внутренних таймеров (отличается от `period` датчика).

Примеры (взяты из `default-tripcomp.layout`):
```xml
<item id="FuelLevel" if='${Layout_ECUFuel != true}' inherit="_FuelIndicator, _EditAction" onclick="NewFueling"/>
<item id="Odometer" if="${Layout_ECU_Odometer != true}" actions="SetupOdometer" size="large" inherit="_EditAction"/>
```

- Специфика типов и атрибуты графиков:
  - `type="chart"` — рисует график по сенсору; общие атрибуты: `chart='line'`, `time-scale`, `min-limit`, `max-limit`, `cover-description`.
  - `border-opacity` — контроль непрозрачности рамки (используется в dashkits).
  - `custom-description` / `description` / `custom-units` — кастомные подписи и единицы.

- Доступ к extradata сенсора:
  - `id="sensor[extradata]"` — отображает дополнительные данные, связанные с сенсором (например `TripTime[fillup]`).

Примеры:
```xml
<item id='Latitude' precision='6fixed' size="normal"/>
<item type="button" text="Fan On" action="invoke(FAN_ON)" size='extralarge' />
<item id='Lambda' colspan="2"/>
```

## Специальные типы item
- `type="button"` — кликабельная кнопка. Используйте `action="invoke(NAME)"` для вызова ECU/сервиса.
- `type="static"` — статический текстовый блок. Пример: `type="static" size="micro"`
- gauge / chart / image decorators — могут быть объявлены как вложенные декораторы внутри `item` или в файлах тем/гауджей; принимают атрибуты `image-path`, `image-scale`, `image-width`, `image-height`, `image-rotate`.

## Декораторы и вложенные элементы
- `item` может содержать вложенные декораторы. Пример:
```xml
<item id="Speed">
  <image image-zorder="bottom" image-path="images/dashboard-svgrepo-com.svg" image-scale="0.5"/>
  <image image-zorder="bottom" image-path="images/arrow.png" image-scale="0.5"/>
</item>
```

- Общие атрибуты декораторов: `image-zorder`, `image-path`, `image-scale`, `image-width`, `image-height`, `image-rotate` (можно подставлять динамические выражения).

Другие декораторы используются для progress/tile/padding и пр.; многие принимают динамические выражения, например `image-rotate='$${ Sensor_Value - 45 }'`.

## Уровневые атрибуты и функции (из changelog)
- `if` — условное отображение для `section` или `item`.
- `ignore-gauges` в `section` — игнорировать глобальные дескрипторы gauges для всех элементов внутри секции.
- `gauge` локальные определения могут объявляться внутри `section`.
- `swap-direction` на gauges: `swap-direction="true"` меняет направление бара или roundbar.
- `horizontal-offset` для roundbar.
- декоратор `break` — `<break/>` разделяет рендеринг на независимые этапы.
- `maximize-focus`, `controls="autohide"` — полезны для dash панелей.
- `time-scale` — используется для изменения шкалы времени в графиках.
- `text-evaluator` — расширенный атрибут для gauges, возвращающий динамический текст.

Дополнительные наблюдаемые атрибуты и поведения:
- `optional` / `volatile` на `section`.
- `period` на `item`.
- `precision`.
- `colspan` / `rowspan`.

Пример графика с фильтрами и кастомным описанием:
```xml
<item id="Speed" wrap="predict(50,0.7)" type="chart" chart='line' period="100" precision="0" custom-description="Predictive speed" border-opacity="1"/>
```

## Сенсоры и выражения
- Сенсоры могут ссылаться напрямую по `id` в `if` атрибутах или как `item`.
- TinyExe хелперы и `eval` выражения доступны в ecuxml. Доступные глобалы: Pi, E, PositiveInfinity, NegativeInfinity и пр.
- Фильтры сенсоров (через `wrap`) как выше.

Пример TinyExe / eval:

`image-rotate='$${ Sensor_Value - 45 }'`

Пример доступа к extradata:

`<item id="TripTime[fillup]"/>`

## `SensorList` и автогенерируемые панели
- `section` с `class="SensorList"` и `ns="<namespace>"` генерирует экран со списком датчиков для namespace. Используйте `exclude` regex для фильтрации шумных датчиков.
  Пример:
  ```xml
  <section name="Ford" class="SensorList" ns="Ford_MondeoDuratorq2003" if="Ford_MondeoDuratorq2003.Speed" exclude="\\bCACT_|...">
  </section>
  ```

## Примеры (взято из реальных layout)
- Простой GPS экран (из `ecu-gps.layout`):

```xml
<section name="GPS">
  <grid rows=",," cols=",,">
    <item id='Latitude' precision='6fixed' size="normal"/>
    <item id='Longitude' precision='6fixed' size="normal"/>
    <item id='Satellites'/>
    <item id='Altitude' precision='1'/>
    <item id='GPSSpeed' precision='1'/>
    <item id='ActiveSatellites'/>
    <item id='Direction'/>
    <item id='GPSOBD2SpeedDiff'/>
    <item id='FixedPosition'/>
  </grid>
</section>
```

- Fan control (кнопки + inherit decorator):

```xml
<section name="FAN" fixed="false" if="PortraitLayout">
  <grid rows=",," cols="">
    <item id="FAN_Status" inherit="_ColoredOnOff" size='large'/>
    <item type="button" text="Fan On" action="invoke(FAN_ON)" size='extralarge' />
    <item type="button" text="Fan Off" action="invoke(FAN_OFF)" size='extralarge' />
  </grid>
</section>
```

- Fuel trims:

```xml
<section name="Fuel Trims" if="STFT1">
  <grid rows="," cols=",">
    <item id="STFT1" size='large' precision="1" />
    <item id="STFT2" size='large' precision="1" />
    <item id="LTFT1" size='large' precision="1" />
    <item id="LTFT2" size='large' precision="1" />
  </grid>
</section>
```

## Условные выражения
- Атрибут `if` может быть простым тестом присутствия сенсора (например `if="STFT1"`) или полным выражением, ссылающимся на переменные темы, флаги день/ночь, флаги макета и значения сенсоров.

Примечание: `if` может содержать через запятую несколько идентификаторов, которые в практике работают как OR (множество вариантов).

## Интерполяция в выражениях

Кроме простых выражений доступны два стиля интерполяции:

- `${ }` интерполяция: вычисляется один раз при сборке макета. Используйте для статических вычислений.

- `$${ }` интерполяция: вычисляется динамически — подходит для значений, которые меняются во времени (анимации, периодические обновления сенсоров). Применимо к выбранным атрибутам: `text-evaluator`, `switch[index]`, `visibility`, `image-rotate`, `image-path`, `crop`.

Эти формы позволяют вставлять TinyExe-выражения в значения атрибутов для статического или динамического вычисления.

## Подсказки по размерам сетки
- `rows` и `cols` принимают значения, разделённые запятыми. Пустые поля — гибкие ячейки, числа — фиксированные подсказки. Пример: `rows="30,,," cols="30,,30"`.

Можно вкладывать `grid` для создания сложных адаптивных областей.

## Дополнительные полезные атрибуты и поведения
- `period` на `item` — устанавливает период обновления виджета.
- `precision` — точность отображения.
- `description="aside"` — размещает описательный текст сбоку.
- `inherit` — повторно использует стиль/цвета из темы.
- `text-values` и `custom-units` — показывают текстовые значения или альтернативные единицы.
- `min-limit` / `max-limit` — ограничивают значения на графиках.

## Widget и атрибуты декораторов (из `user.layout`)

- `type` (примеры): `bar`, `roundbar`, `chart`, `text`, `static`.

- `cache-interval` — подсказка кэширования/обновления UI. Пример: `cache-interval="6"`.

- `red-green` — список порогов цвета, значения через запятую. Примеры: `red-green="100,5"`.

- `bars-count` — число сегментов в bar/roundbar. Пример: `bars-count="10"`.

- `transparent` — `true/false`.

- `border-width` / `border-opacity` — визуальные параметры рамки.

- `colored-value` — применять окраску значений по порогам.

- `fade-coeff` — коэффициент затухания трейла в графиках.

- `width-compression` — сжатие по горизонтали.

- `item-size` / `item-width` / `item-height` / `item-spacing` — параметры размера сегментов.

- `small-radius-delta` / `small-radis-delta` — варианты опечатки встречались в исходниках; означают смещение малого радиуса.

- `start-angle` / `end-angle` / `radius` / `vertical-offset` — геометрические параметры roundbar.

- `corn-angle` / `middle-angle` — дополнительные угловые параметры.

- `decorator-p` (padding) и `p-padding` — декоратор отступов. Пример: `decorator-p="padding" p-padding="10 10 10 10"`.

- `decorator-i` / `decorator-iN` — нумерованные декораторы изображений (`decorator-i1="image"`, `i1-image-path`, `i1-image-rotate`, и т.д.). Альтернативно поддерживается вложенная форма `<image>`.

Примеры:
```xml
<item id="Speed" decorator-i1="image" i1-image-zorder="bottom" i1-image-path="images/gauges/without_arrow_circle.png" i1-image-rotate='$${...}' i1-image-rotate-align="center"/>

<item id="Speed">
  <image zorder="bottom" path="images/gauges/without_arrow_circle.png"/>
  <image zorder="bottom" path="images/gauges/arrow.png" rotate='$${...}' scale="1"/>
</item>
```

- `rotate` / `scale` на вложенных `<image>` — `rotate` может содержать динамические выражения, `scale` задаёт масштаб изображения.

- `custom-units` — используется для `DateTime` и пр.: `custom-units="time12"`, `time`, `timesec`, `datetime`, `seconds`.

- `border-opacity` — применялось в тестах для просмотра рамок.

- числовой `size` на `item` — можно задавать числа (например, `size="72"`) помимо именованных размеров.

## Частые `type` в dashkits
- `type="text"` — текстовое представление датчика (`text-values`).
- `type="chart"` — временной ряд.
- `type="button"` — интерактивная кнопка.

Пример использования `text-values`:

`<item id="RPM" wrap="predict(50, 0.7)" text-values="0:" type="text" description="hidden">`

## Advanced: ecuxml и определения сенсоров
- определения сенсоров и `eval` выражения находятся в ecuxml файлах. Примеры возможностей:
  - `<eval>` выражения для вычисляемых сенсоров
  - `base-data` / `base-raw` для сырых байтов
  - функции `get(offset)`, `get_word()` и т.д.
  - тег `text` для строковых сенсоров

## Практические рекомендации
- Используйте `if` для модульности файлов макетов; макеты загружаются только для релевантных ЭБУ, но `if` дополнительно предотвращает пустые секции.
- Применяйте `SensorList` для генерации экранов со списками сенсоров и `exclude` для отбора.
- Для сложной визуализации предпочтительнее вложенные декораторы — они локальны и проще в управлении.
- Применяйте `wrap` фильтры для шумных сенсоров (GPS, speed, MAF) для плавного отображения.
- По возможности держите gauges локальными для секции, используя `ignore-gauges` и локальные `gauge` объявления.

## Appendix — быстрый справочник по атрибутам
- section: `name`, `fixed`, `if`, `class`, `ns`, `exclude`, `maximize-focus`, `controls`
- grid: `rows`, `cols`
- item: `id`, `type`, `text`, `action`, `size`, `precision`, `inherit`, `colspan`, `rowspan`, `description`, `period`, `wrap`
- decorators: `image` (`image-path`, `image-scale`, `image-zorder`, `image-rotate`, `image-width`/`image-height`)

## Try it — готовые шаблоны

GPS panel (минимальный)
```xml
<ui>
  <section name="GPS">
    <grid rows=",," cols=",,">
      <item id='Latitude' precision='6fixed' size="normal"/>
      <item id='Longitude' precision='6fixed' size="normal"/>
      <item id='GPSSpeed' precision='1' />
    </grid>
  </section>
</ui>
```

Fan control (кнопки + inherit)
```xml
<ui>
  <section name="FAN" fixed="false" if="PortraitLayout">
    <grid rows=",," cols="">
      <item id="FAN_Status" inherit="_ColoredOnOff" size='large'/>
      <item type="button" text="Fan On" action="invoke(FAN_ON)" size='extralarge' />
      <item type="button" text="Fan Off" action="invoke(FAN_OFF)" size='extralarge' />
    </grid>
  </section>
</ui>
```

Hybrid summary example
```xml
<ui>
  <section name="Hybrid">
    <grid rows="30,,," cols="30,,30">
      <item id="Efficiency_Battery" size='huge' />
      <item id="FuelEconomy_NoBattery" size='huge' />
      <item id="BatteryCurrent" />
      <item id="SoC" />
    </grid>
  </section>
</ui>
```

---

Файл переведён с сохранением терминов, имен атрибутов и примеров в исходном английском виде.
