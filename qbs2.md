Примеры, когда код работал не так, как я предполагала:

Пример 1. 
У нас есть сущность Hibernate Task с полем user, которое загружается лениво. Методы toString, equals, hashCode реализованы через аннотации Lombok.

```java
import lombok.*;
import jakarta.persistence.*;

@Entity
@ToSting
@EqualsAndhashCode
class Task {
    
    @ManyToOne(fetchType = FetchType.LAZY)
    User user;
    
    //...
}
@Slf4j
public getTask(long id) {
    // Пример 1
    Task taskOpt = taskRepository.findById(id).get();
    log.info("Task: {}", task); // LazyInitializationException
}

public interface TaskRepository extends JpaRepository<Task, Long> {
    // Пример 2 
    Set<Task> findAll(); //Could not initialize proxy Task - no session
}
```
В такой реализации я получила две неожиданные ошибки.

Первая - в методе getTask(), где по id находим в базе Task и хотим залогировать.
Здесь получаем LazyInitializationException, т.к. по умолчанию Lombok использует все поля для реализации toString, но сессия Hibernate уже закрылась, а поле user не инициализировано. 

Вторая - в методе TaskRepository.findAll.
Реализацию этого метода автоматически генерирует библиотека Spring Data, где в коде сначала извлекается из БД список List<Task>, 
а затем конвертируется в HashSet, для чего вызывается метод hashCode у Task, для чего lombok требует поле user, но сессия Hibernate уже закрыта.
```
Exception: Could not initialize proxy Task - no session
org.springframework.core.convert.ConversionFailedException: Failed to convert from type [java.util.ArrayList<?>] to type [java.util.Set<?>] for value [[...]]
at org.springframework.core.convert.support.ConversionUtils.invokeConverter(ConversionUtils.java:47)
...
at org.springframework.data.jpa.repository.query.AbstractJpaQuery.execute(AbstractJpaQuery.java:148)
```
Эти ошибки - моя ответственность, т.к. я не ознакомилась со спецификацией библиотек, полагаясь на то, что все будет само работать из коробки. 


Пример 2. 

В проекте используется библиотека quickfix, для интеграции по протоколу FIX с другим сервисом. Во время работы сервисы устанавливают сессию, и FIX-сервер присылает котировки нашему FIX-клиенту. 
И вдруг спустя несколько месяцев нормальной работы вдруг сессия стала периодически разрываться. Восстановление сессии занимает до пары минут, что критично для системы, т.к. клиент пересылает котировки в третью систему, где есть требования к частоте получения котировок.

Оказалось, что в сообщении с котировкой, которую посылает FIX-сервер, есть время обновления котировки. А когда сообщение обрабатывается на клиенте, библиотека quickfix по умолчанию проверяет, что время, указанное в сообщении, и время, в которое выполняется обработка на клиенте, различаются не более, чем на 2 минуты. Иначе сессия обрывается. А ошибки стали появляться, когда третья система стала испытывать проблемы с нагрузкой, и стала обрабатывать запросы от FIX-клиента слишко долго (а мы отсылали туда котировки синхронно), и latency накапливалась. 

Тут ответственность на мне, т.к. в документации упоминается параметр, который отвечает за это поведение.
```
CheckLatency = If set to Y, messages must be received from the counterparty within a defined number of seconds (see MaxLatency). 
It is useful to turn this off if a system uses localtime for its timestamps instead of GMT.
```
Но честно говоря, сложно предугадать такое, учитывая большое количество различных параметров. А ошибка никак не проявляла себя, пока третья система не стала испытывать проблемы с нагрузкой. 
https://www.quickfixj.org/usermanual/2.3.0/usage/configuration.html


Пример 3.

Ошибка инфаструктуры.

Наш сервис A вызывает API другого сервиса B. В какой-то момент он стал получать 500-ю ошибку по непонятной причине, хотя сервис B работал и был в порядке. 

Оказалось, что команда devOps сделала новую балансировку кластера, где развернут сервис B. Работоспособность всего кластера стала определяться по другому сервису С, который в тот момент был недоступен. 

Т.к. другая команда настроила эту балансировку, не уведомив нас, ответственность за ошибку была на них. 


-----------------
Примеры, где с помощью продуманного типа результата метода можно явно исключить нежелательные формы обработки этого результата:

Пример 1.

Использовала метод репозитория, который возвращал List, и в свой код необдуманно написала из расчета, что значения будут упорядочены по created_at. При тестах не отловила ошибку, т.к. в БД были только значения, созданные автоматически, и соответственно значения id были по возрастанию, как и created_at. Но ошибка проявилась, когда создали вручную записи вручную. 

Ошибки можно было бы избежать, используя Set как возвращаемое значение.

Пример 2. 

Мы получаем подписи, вызывая API другого сервиса, и преобразуем в объект нашей модели.
По бизнес-логике может быть только одна подпись, но API возвращает список. 

```java

List<Signature> certificateList = certService.callSignatures(id);
Signature signature = certificateList.get(0);
if (signature == null) {
    // ...
}
verify(signature);

class CertificateService {
    List<Signature> callSignatures(long id) {
        return map(api.getSignatures(id));
    }
    
}
```


Мы могли бы возвращать Optional<Signature>, и обрабатывать возможные ошибки внутри метода CertificateService.callSignatures.
