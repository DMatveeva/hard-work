### 1
Есть приложение, которое принимает сообщения с котировками финансовых данных от других систем, а затем пересылает их в другую единую систему. 

Оно работает по приницпу адаптеров. 

Есть главный класс, InboxHandler, который обрабатывает сообщение от любого источника, независимо от его структуры.
Для этого InboxHandler параметризован и использует параметризованные интерфейсы.


```java
// Упрощенно, он содержит такую логику:

public class InboxHandler<M, D, E extends ItemEntity>  {
    
    private final Extractor<M, D> extractor;
    private final Mapper<M, D, E> mapper;

    @Override
    public void handle(M message) {
        InboxEntity inbox = inboxService.register(/**/);

        List<D> dtos = extractor.extractItems(message);
        for (D dto : dtos) {
            validate(dto);
            E item = mapper.map(messag, dto);
        }
    }
}

public interface Extractor<M, D> {
    boolean isPossibleToExtract(M message);
    default void validate(D dto) {}
    List<D> extractItems(M message);
}
public interface Mapper<M, D, E extends ItemEntity> {
    E map(M message, D dto);
}
```
Для каждого источника данных есть отдельный адаптер &ndash; мавен модуль, где объявлены реализации этих интерфейсов. 
За счет этого расширяются возможности по добавлению новых источников. 

`<M>` &ndash; десериализованное сообщение от  источника.
Изначально сырое сообщение может прийти в любом формате: String в формате Json или String в формате XML.
В конкретном адаптере мы реализуем логику преобразования сырых данных к промежуточному типу `M`.
Например, если сырое сообщение от системы X - это Json, то в конкретном адаптере мы десериализуем в тип XMessage.
Но не обязательно нам нужны все поля этого сообщения. Поэтому с помощью `Extractor<M, D>` мы преобразуем в удобный нам Dto (`<D>`) &ndash; XDto. 

А затем, с помощью Mapper, в `<E extends ItemEntity>`, для сохранения в базу данных. 

Тут есть некоторая избыточность. 
Например, для некоторых источников не трубуется валидировать Dto, следовательно, метод validate(M message) не будет переопределен.
Но за счет этого расширяются возможности по добавлению новых источников. 
Не нужно писать обработчик для каждой котировки, нужно лишь реализовать понятные интерфейсы. 

### 2

Из разных источников котировки приходят в разном формате.
- У котого-то в 1-м сообщении только 1 инструмент.
- У кого-то несколько инструментов.
- А у кого-то может быть и несколько инструментов и несколько котировок для одного инструмента.

Чтобы сделать обработку сообщений универсальной, в InboxHandler мы как раз считаем, что в результате обработки сообщения мы можем получить Map, где ключ - id инструмента, а значение - список новых котировок для него. 

```java
public class InboxHandler<M, D, E extends ItemEntity>  {
    
    @Override
    public void handle(M message) {
        //...
        Map<String, List<E>> instrumentsMap = new HashMap<>();
        List<D> dtos = extractor.extractItems(message);
        for (D dto : dtos) {
            E item = mapper.map(messag, dto);
            String instrumentId = extractor.getInstrumentId(dto);
            List<E> values = instrumentsMap.computeIfAbsent(instrumentId, k -> new ArrayList<>());
            values.add(item);
        }
        saveValues(instrumentsMap);
    }
}
```

Это расширяет возможности по обработке котировок любой структуры. 
Но с другой стороны, если источник, например, обязан присылать только 1 котировку для только 1-го инструмента, мы должны явно это проверить в extractItems.

### 3

После того, как мы получили для каждого датафида список объектов `List<E> (E extends ItemEntity)`, мы сохраняем их в БД.
`ItemEntity` содержит поле value &ndash; значение котировки для инструмента. Это объект сохраняется в таблицу items.
В таблице items есть колонка value типа `numeric (19,2)`. 

Все работало, пока источники присылали value c двумя знаками после запятой. 
Но в какой-то момент добавилось 3 новых источника, которые присылали value с 0, 4-мя и 12-ю знаками после запятой соответственно. 

Изначально рассматривали вариант сделать 3 новые таблицы, и в каждой колонку value с соответствующим количеством знаком после запятой. 

Но в итоге реализовали более универсальной решение &ndash; единая таблица extended_item, где будет 12 знаков после запятой. 
А валидацию, что в сообщении корректная точность, перенесли в `extractor.validate(D dto)`.

Это также расширяет возможности по обработке новых типов сообщений. 
Но теперь для каждого источника все валидации нужно прописывать явно. 
А если бы мы сделали отдельные таблицы для каждого источника, мы бы получили эти валидации автоматически. 


### 4

В приложении надо было реализовать новый источник котировок.

Но логика обработки нового сообщения не согласовалась с логикой существующего InboxHandler.
Согласно дизайну, при получении сообщения надо сделать http-запрос в сторонюю систему.
А при получении ошибки, сделать до 5-ти повторных запросов с интервалом 5 секунд.

Самая простая реализация (которую, кстати, мне предложил ИИ-агент), это сделать цикл с Thread.sleep(5_000) :)
Но мне показалось, что это плохая мысль - блокировать поток на 25 секунд. 
Это хрупко, сложно для откладки, можно получить таймаут от Kafka.

Поэтому я решила сделать для входящего сообщения механизм ретраев.
В таблицу inbox добавила колонку attempt и реализовала новый RetryInboundHandler.
И его можно переиспользовать для других источников, если им потребуется повторная обработка входящего сообщения. 

### 5 

Каждое сообщение из системы-источника имеет лейбл, который обозначает финансовый инструмент. 
Для отдельной системы лейбл чаще всего соответствует какому-то простому паттерну.
Например, для системы ABC все лейблы начинаются с префикса "ABC.". Далее идет название инструмента, например, "ABC.SNP500".
B коде, чтобы извлечь название инструмента, просто парсилась строка, и отбрасывался префикс. 
Но в какой-то момент добавился еще один паттерн "ABC.DE.***" . 

Чтобы поддержать его, я добавила класс ABCLabelMasks.

```java
public enum ABCLabelMasks {
    ABC_DE(ABC_DE_REGEXP, s -> extract(ABC_DE_REGEXP, s)),   //"^ABC\\.DE\\.(.*)$"
    ABC(ABC_REGEXP, s -> extract(ABC_REGEXP, s));            //"^ABC\\.(.*)$"

    private final String mask;
    private final UnaryOperator<String> instrumentExtractor;

    private static String extract(String regexp, String value) {
        Pattern pattern = Pattern.compile(regexp);
        Matcher matcher = pattern.matcher(value);
        return matcher.find() ? matcher.group(1) : null;
    }

    public static Optional<String> getInstrument(String label) {
        return from(label).map(m -> m.instrumentExtractor.apply(label));
    }
}
```

Теперь в коде можно просто вызывать ABCLabelMasks.getInstrument("ABC.SNP500").

Поначалу, возможно, это выглядело слишком усложненно для всего 2-х масок. Но вскоре начали добавляться новые. 
А потом потребовалось различать случаи, когда одна маска соответсвует другой. 
Например, "ABC.DE.SNP500" также подходит под "ABC.SNP500". И тогда оказалось достаточно просто добавить карту соответсвтвий в ABCLabelMasks. 
Вместо того, чтобы сравнивать строки. 

```java
public enum ABCLabelMasks {
    ABC_DE(ABC_DE_REGEXP, s -> extract(ABC_DE_REGEXP, s)),
    ABC(ABC_REGEXP, s -> extract(ABC_REGEXP, s));

    static final EnumMap<ABCLabelMasks, List<ABCLabelMasks>> maskMatches= new EnumMap<>(Map.of(
            ABC, List.of(ABC, ABC_DE)
    ));  

    public boolean matches(ABCLabelMasks maskToMatch) {
        return maskMatches.containsKey(maskToMatch) && maskMatches.get(maskToMatch).contains(this);
    }
}
```
