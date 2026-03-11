В проекте (Java) есть класс EventEntity, который представляет собой событие в блоке блокчейна.
Мы бы хотели сопоставить событие с транзакцией, в которой оно появилось (TransactionEntity).

Map<EventEntity, TransactionEntity> eventToTransaction = new HashMap<>();

Рассмотрю ситуацию в случае, когда мы используем Map и в случае, если бы добавили поле TransactionEntity transaction напрямую в EventEntity.

Класс EventEntity -- мутабельный . Тут есть аннотация @Data (библиотека lombok), благодаря которой будет сгенерирован метод equals и hashcode, куда включены все поля класса.

```java
@Data
@FieldDefaults(level = AccessLevel.PRIVATE)
public class EventEntity {
    @Id
    String id;

    ProductEntity product;

    BlockEntity block;

    String transactionId;

    // other fields
    }
```


Обе модели - со словарем и добавлением напрямую поля в класс EventEntity -- будут эквивалентны в данном случае.

Но т.к. мы работаем с Hibernate, может возникнуть ситуация, когда мы поместили в HashMap новый объект EventEntity с id = null, потом сохранили его в БД, в результате id уже не null, и при поиске в HashMap мы такой объект уже не найдем, т.к hashcode будет выдавать другое значение.
Я не сталкивалась с такой ситуацией, но читала, что чтобы этого избежать, рекомендуется применять бизнес-ключ, т.е. уникальные поля для каждой сущности. В нашем случае это может быть transactionId + product.

Но, если бы наш equals учитывал только бизнес-ключ, то две модели стали бы неэквивалентны.
Т.к. 2 объекта EventEntity с одинаковым бизнес-ключом, но отличающиеся другими полями, давали бы один и тот же hashcode в HashMap. В этом случае, с использованием словаря получилось бы упрощение модели. 
