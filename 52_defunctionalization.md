### 1

Недавно у нас появился первый сервис с гексангональной архитектурой. 
Огромное спасибо Вам за гайд "Функциональные архитектуры"! Я предложила сделать новый сервис в архитектуре портов и адаптеров, и тимлид, и команда поддержали!
Теперь это единственный сервис, где в слое бизнес логики нет ORM.
Модуль dao сделан как адаптер, он использует Spring Data JDBC, а модуль app никак не зависит от него, а вызывает через порты. 

Чтобы сохранить объекты в базе данных транзакционно, есть отдельный TransactionPort. 
Рассмотрю сценарий, когда в сервис приходит сообщение, а в результате его обработки надо создать объект Operation, а также исходящее соообщение Outbox. Также надо обновить статус у самого входящего Inbox.

```
Inbox inbox = updateInbox(...);
Operation op = new Operation(...);
Outbox o = new Outbox(...);

Runnable r = () -> {
    inboxPort.save(inbox);
    operationPort.save(op);
    outboxPort.save(o);
};

// ...

transactionPort.save(r);
```

```Java

public class TransactionAdapter implements TransactionPort {

    @Transactional
    public void save(Runnable toSave) {
        toSave.run();
    }
}
```


Но проблема в том, что в Runnable можно передать все, что угодно. 
В других сервисах у нас есть такая проблема, что в границах транзакций есть Http вызовы.
Я хотела, чтобы в новом сервисе вообще невозможно было так сделать. Но в нашем текущем варианте ничго не помешает выполнить http-запрос в транзакции.

Как вариант, можно просто определить транзакционный метод, который принимает нужные нам объекты. Без использования функций высшего порядка. 
```java
public class TransactionAdapter implements TransactionPort {


    @Transactional
    public void save(Inbox inbox, Operation op, Outbox outbox) {
        inboxPort .save(inbox);
        operationPort.save(op);
        outboxPort.save(outbox);
    }
}

// в слое домена будет просто :
transactionPort.save(inbox, op, outbox);

```
Но тут потенциально возможно множество методов, и возможно большое количество принимаемых параметров, если надо сохранить несколько объектов.

Попробовала переделать с использованием дефункционализации.
У нас Java 21, и поэтому доступны sealed классы (алгебраический тип данных) и pattern matching для таких классов в switch. 


```java
public sealed interface TransactionalCommand permits OperationSave {}
public record OperationSave (Inbox inbox, Operation op, Outbox outbox) implements TransactionalCommand {}

public class TransactionAdapter implements TransactionPort {

    //метод, который вызывается из слоя домена
    public void save(TransactionalCommand tc) {
        switch (tc) {
            case OperationSave c: save(operationSave(c));
            // другие типы
        }
    }
    
    private void save(Consumer<TransactionStatus> consumer) {
        transactionTemplate.executeWithoutResult(consumer);
    }

    
    public Consumer<TransactionStatus> operationSave(OperationSave command) {
        return status -> {
            // сохранение объектов
        };
    }
}

```
Теперь просто невозможно передать в транзакцию вызов http-метода. 
Для этого надо добавить в pom.xml модуля dao (адаптер для БД) зависимость на адаптер, где делается http-запрос. А такое намного проще отследить на ревью.

И  мне кажется, это больше соответствует принципу DIP. 
В данном случае у нас простой пример, объекты Inbox, Operation, Outbox друг от друга не зависят. 
Но допустим, сценарий другой: operation имеет foreingKey на таблицу wallet.
И надо в одной транзакции сохранить operation и wallet. 
Это значит, в коде надо сначала сохранить wallet, у полученного объекта получить id, прописать его в объект operation, и только потом сохранить operation. 
Бизнес-логика не должна знать об этом. Там у нас wallet - просто поле у класса Operation. 
А вот слой dao уже правильно сохранит все объекты, чтобы они были корректно связаны в БД. 


### 2

В одном из сервисов обработка исходящих сообщней поисходит в помощью иерархии классов-процессоров. 
На каждый тип входящего сообщения есть свой процессор. 
Но все они наследуются от родительского класса AbstractTaskProcessor. 
А AbstractTaskProcessor содержит поле-функцию (функциональный интерфейс), которая выполняется до основной обработки.

```java
public abstract class AbstractTaskProcessor<T extends Outbox> {

    private final BeforeOperation<T> beforeOperation;
    
    public final void process(T t) {
        t = beforeOperation.apply(t);
        Function<T, ?> processResult = this.processConcrete(t);
       //...
    }
}
public interface BeforeOperation<T extends Outbox> extends Function<T, T> {
    @Override
    default T apply(T t) {return t;}
}

//Пример обработки ДО: увеличиваем количество предпринятых попыток отправки.
public class EncreaseAttempts<T extends Task> implements BeforeOperation<T> {

    private final JpaRepository<T, Long> repository;

    @Override
    public T apply(T t) {
        int current = t.getAttempts() + 1;
        if (current >= getMaxAttempts()) {
            throw AttemptsExpiredException();
        }
        t.setNextExecutionTime(getNextExecutionTime(current));
        t.setAttempts(current);
        return repository.save(t);
    }
}
```
На деле такой класс, который имплементирует BeforeOperation, всего один. 
Он передается в конструктор каждого процессора. 

Но если бы мы хотели добавить несколько классов, например, для разных типов Outbox по-разному рассчитывать nextExecutionTime, можно было бы применить дефункционализацию. 
Пусть для некоторых Outbox время следующей отправки увеличивается линейно, а для других - экспоненциально. 
Тогда BeforeOperation станет sealed-классом с двумя потомками:

```java
public sealed interface BeforeOperation permits LinearIncrease, ExponentialIncrease {}
public record LinearIncrease implements BeforeOperation {
  Function<Int, Instant> getNextExecTime = t -> { ... }; //
}
public record ExponentialIncrease implements BeforeOperation {
  Function<Int, Instant> getNextExecTime = t -> { ... };
}

public abstract class AbstractTaskProcessor<T extends Outbox> {

    private final BeforeOperation beforeOperation;
    
    public final void process(T t) {
        
        switch (beforeType) {
            case LinearIncease l: updateAttempt(t, l.getNextExecTime());
            case ExponentialIncrease e: updateAttempt(t, e.getNextExecTime());
        }
        
        //...
    }

    public T updateAttempt(T t, Funсtion<Int, Instant> f) {
        int current = t.getAttempts() + 1;
        if (current >= getMaxAttempts()) {
            throw AttemptsExpiredException();
        }
        t.setNextExecutionTime(f.apply(current));
        t.setAttempts(current);
        return repository.save(t);
    }    
}
```


