### 1.
Дана иерархия классов, которые обрабатывают outbox сообщение в микросервисе.

Все наследуются от абстрактного класса OutboxProcessor - он содержит абстрактный метод processEvent.

Вот такая иерархия:
- OutboxProcessor
  - AdapterRequestingProcessor
    - AbstractDebtRequestingProcessor
      - PlannedDebtRequestingProcessor
      - EarlyDebtRequestingProcessor  
    - AdapterRequestingAsIsProcessor
      - ...


Мне кажется, здесь "абстрактные" классы - AdapterRequestingAsIsProcessor и AbstractDebtRequestingProcessor.
В комментариях опишу свои рассуждения.

```java

// processEvent - основной метод для обработки сообщения
public abstract class OutboxProcessor {
    abstract void processEvent(Outbox outbox);
}
// Этот класс реализует processEvent как REST-запрос к сервису-адаптеру.
abstract class AdapterRequestingProcessor extends OutboxProcessor {
    @Override
    public void processEvent(Outbox outbox) {
        R response = requestAdapter(outbox); // делает запрос к адаптеру
        //...
    }
    protected R requestAdapter(Outbox outbox) {
        T data = getDataFromEvent(outbox);
        R request = extractRequest(data);
        S response = performRequest(request);
        return responce;
    }
    protected abstract R extractRequest(T data);
    protected abstract S performRequest(R query);
}

// Этот класс предназначен для работы с операциями получения долговых обязательств.
// Jн переопределяет метод processEvent из AdapterRequestingProcessor, добавляет логику для проверки суммы долга и другое.
// Относительно операции processEvent потомки этого класса PlannedDebtRequestingProcessor и EarlyDebtRequestingProcessor ведут себя одинаково,
// они отличаются только реализацией методов performRequest и extractRequest.
abstract class AbstractDebtRequestingProcessor extends AdapterRequestingProcessor {
    @Override
    public void processEvent(Outbox outbox) {
        var response = requestAdapter(outbox);
        //check debt sum, update related events
        //...
    }
}
// Этот класс переиспользует метод processEvent из родительского класса AdapterRequestingProcessor.
// По сути, он нужен только чтобы абстрагировать классы, которые вызывают адаптер as is, не добавляя дополнительной логики.
abstract class AdapterRequestingAsIsProcessor extends AdapterRequestingProcessor { /**/ }
```

AbstractDebtRequestingProcessor - наглядно показывает, что его запрашивают данные именно по долговым обязательствам.

AdapterRequestingAsIsEventProcessor также сделан для наглядности, что его потоки вызывают адаптер as is, не делая доп. логики как AbstractDebtRequestingProcessor.

Эти классы можно было бы вовсе опустить, сделав так, что потомки AdapterRequestingAsIsProcessor наследуются напрямую от AdapterRequestingProcessor.

### 2. 
Рассмотрим иерархию классов, которые занимаются отправкой сообщения о действиях с финансовыми активами.


- CommonAssetSender
    - AbstractIcoAssetSender
        - AbstractIcoCouponAssetSender
            - IcoFixedCouponAssetSender
            - IcoFloatCouponAssetSender
        - IcoDiscountAssetSender
            - ...

Корень иерархии - класс CommonAssetSender, содержит методы, общие для все типов отправщиков.

Далее от него наследуется AbstractIcoAssetSender, который отправляет сообщения о событиях с облигациями.
Содержит abstract метод createMetaAttrs, который по-своему реализуется в каждом потомке.

Можно рассмотреть относительно этого метода два подтипа - AbstractIcoCouponAssetSender и IcoDiscountAssetSender.
Каждый из них задает эту операцию одинаково для своих потомков. А вот сам AbstractIcoAssetSender служит просто для наглядности, говоря о том, что все его потоки работают только с облигациями.

Теоретически, мы могли бы сделать интерфейс (например HasMetaAttributes), с методом createMetaAttrs, чтобы AbstractIcoCouponAssetSender и IcoDiscountAssetSender его реализовывали. 
Избавившись от промежуточного класса AbstractIcoAssetSender.

