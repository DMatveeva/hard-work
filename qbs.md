#### Задача № 1.
Бизнес-смысл задачи:
Сервис (А) отправляет задачу по подписанию документа в другой сервис (B). Если сервис Б долго не отвечает, мы не хотим ждать, а хотим отменить задачу и закрыть документ. 

Псевдокод для таблиц в БД:
```sql
table task {
    int8 id;
    int8 document_id; ...
}
table document {
    int8 id; ...
}
```

```java
@Entity
class TaskEntity {
    @ManyToOne
    @JoinColumn(name = "document_id")
    private DocumentEntity document;
    //...
}
@Entity
class DocumentEntity {  
    //...
}
```


С помощью Hibernate это было реализовано так:
```java
@Scheduled
public void cancelAllExpiredTasksAndDocuments() {
    List<TaskEntity> tasks = taskRepository.findAllByStatusInAndCreatedAtLessThan(List<TaskStatus> statuses, Instant expirationTime);
    tasks.stream().forEach(cancelService::cancelExpiredTasksAndDocuments);
}

class TaskRepository extends JpaRepository<TaskEntity, Long> {
    List<TaskEntity> findAllByStatusInAndCreatedAtBefore(List<TaskStatus> statuses, Instant expirationTime);
}

class CancelService {
    @Transactional
    public void cancelExpiredTasksAndDocuments(TaskEntity task) {
        task.setStatus(EXPIRED);
        taskRepository.save(task);
        
        Document document = task.getDocument();
        documentRepository.save(CANCELLED);
    }
}
```
Когда я изучала как работает Hibernate, я вспомнила про проблему n+1. И проверив, увидела, что тут она появляется. 

TaskEntity.document загружается жадно, и при вызове findAllByStatusInAndCreatedAtBefore мы получаем 1 запрос для получения всех tasks, и n запросов для получения document для каждой записи task.

Чтобы переделать задачу без использования Hibernate, я использовала JDBC.
Получилось так:
```java
private static String SELECT_EXPIRED_TASKS = '''
        select t.id as task_id, d.id as doc_id 
        from tasks t 
        left join documents d on t.document_id = d.id; 
        where t.status = 'ON_SIGNING' and t.created_at < ?;
        ''';

@Scheduled
public void cancelAllExpiredTasksAndDocuments() {
    List<TaskEntity> tasks = jdbcTemplate.query(SELECT_EXPIRED_TASKS, new CancelTaskViewMapper(), expirationDate);
    tasks.stream().forEach(cancelService::cancelExpiredTasksAndDocuments);
}

private static String UPDATE_TASK = 'update tasts set status = ? where id = ?';
private static String UPDATE_DOCUMENTS = 'update documents set status = ? where id = ?';

class CancelService {
    @Transactional
    public void cancelExpiredTasksAndDocuments(CancelTaskView view) {
        long taskId = view.getTaskId();
        long docId = view.getDocId();
        
        jdbcTemplate.query(UPDATE_TASK, EXPIRED, taskId);
        jdbcTemplate.query(UPDATE_TASK, CANCELLED, docId);
    }
}
```

В БД у меня было 6 записей в task, и 6 записей в document (это соответствует реальным значениям, у нас вряд ли будет больше 10 просроченных задач).

Несколько раз прогнала тесты. В таблице усредненные значения.  
1-й раз при старте - это первый вызов метода после старта приложения. 
Повторно - это уже во время работы приложения.

|                    | Row Sql | Hibernate  |
|--------------------|---------|------------|
| 1-й раз при старте | ~400 ms | ~1200 ms   |
| Повторно           | ~350 ms | ~610 ms    |


Почему, как я думаю, есть отличия при первом запуске и при повторных, и для Hibernate и для row sql.
Как я увидела в логах, запросы и для Hibernate и для row sql формируются одинаковые, что при первом, что при повторных запусках.
Но, в логах я также обнаружила, что и для row sql и для Hibernate есть задержка перед выполнением самого первого запроса в БД при старте, если сравнивать с последующими. 
Для Hibernate она порядка 200-300 мс. Думаю, это может быть из-за инициализации подключения к БД или каких-то объектов самого Hibernate. Для row sql разница меньше, но тоже есть - порядка 50 ms.

При повторном запуске разница для Hibernate и для Sql оказалась ~ 260 ms в пользу row sql.

Из чего она складывается:

В hibernate имеем дополнительно 6 select-запросов (из-за n+1) к document, каждый порядка 20 ms, итого 120 ms.

Также, при запуске cancelExpiredTasksAndDocuments для каждой задачи имеем:
- для row sql 3 обращения в БД в среднем по 20 ms: update task, update document, commit  = 60 ms.
- для hibernate 4 обращения в среднем по 20 ms: select task join document (перед сохранением, чтобы проверить состояние объектов), update task, update document, commit.

Итого суммарно разница в среднем 20*6 + 120 = 240 ms.
Экспериментально так и получили -> 610 - 350 = 260. 

#### Задача № 2.

Бизнес-задача состоит в том, чтобы вывести на экран список продуктов с их типом (product_type) в определенном порядке. 
```
table product {
    int8 id;
    int8 product_type_id;
    int8 owner_type_id;
    int8 code;
    int8 status; ...
}

table product_type {
    int8 id; ...
}

table owner {
    long id;
    varchar type; ...
}
```
C использованием Hibernate было так (указаны только поля, которые нас интересуют в этом сценарии):
```java
@Entity
class ProductEntity {
    long id;
    String code;
    String status;
    
    @ManyToOne
    ProductTypeEntity productType;

    @ManyToOne
    OwnerEntity ownerType; 
    //other fields
}
@Entity
class ProductTypeEntity {
    long id;
    String name;
    // other fields
}
@Entity
class OwnerEntity {
    long id;
    OwnerType type;
    //other fields
}

class ProductService {
    
    public List<ProductDto> getAllProducts(String category, String ownerTypeStr,
                                           Integer limit, Integer offset, String sort) {

        Pageable pageable = getPageble(offset, limit, sort); // создаем экземпляр класса Pageable из библиотеки Spring
        OwnerType ownerType = getOwnerType(ownerTypeStr); // создаем enum из строки
        List<ProductEntity> products = productRepository.findAllByCategoryAndOwnerEntityTypeIn(category, ownerType, pageable); //Spring Data автоматически создает реализацию используя под капотом Hibernate
        
        return List<ProductDto> productDtoList = page.getContent()
                .stream()
                .map(productMapper::convert) //в маппере используются не все поля Product
                .toList();
    }
}
```
Здесь мы опять получаем проблему n+1, т.к. поле productType загружается жадно.
Но даже если бы оно загружалось лениво, это бы не помогло, потому что оно требуется в нашем запросе. 

Вот что получилось с использованием row sql:

```java
import java.sql.SQLException;
import java.util.ArrayList;

class ProductService {
    public static String SELECT_PRODUCTS = """
            select p.id as product_id, p.code as code, p.status as status,
                pt.id as product_type_id, pt.name as product_type_name
            from product p 
            left join product_type pt on p.product_type_id = pt.id
            left join owner o on p.owner_id = o.id
            where p.category = ?
            and o.type = ?
            order by ?
            limit ? offset ?
            """;

    public List<ProductDto> getAllProducts(String category,
                                           String ownerTypeStr,
                                           Integer limit,
                                           Integer offset,
                                           String sort) {


        List<SelectProductRow> products = findProducts(category, ownerTypeStr, sort, limit, offset);

        return List <ProductDto> productDtoList = page.getContent()
                .stream()
                .map(this::convert) //новый метод
                .toList();
    }

    private List<SelectProductRow> findProducts(String category, String ownerTypeStr, String sort, Integer limit, Integer offset) {

        List<Object> params = new ArrayList<>();
         //добавляем все параметры

        return jdbcTemplate.query(SELECT_PRODUCTS, new SelectProductRowMapper(), params.toString());
    }

    private class SelectProductRow {
        //здесь только поля, нужные нам для отображения
    }

    private class SelectProductRowMapper implements RowMapper<SelectProductRow> {
        @Override
        public SelectProductRow mapRow(ResultSet rs, int num) throws SQLException {
            //копируем в новый объект SelectProductRow данные из rs
            return selectProductRow;
        }
    }
    
    private ProductDto convert(SelectProductRow product) {
        //копируем поля из SelectProductRow в ProductDto
    }
}


```
У меня было 3 тыс. записей в таблице product. 
Если limit = 10, offset = 0, получились в среднем следующие значения:



|                    | Row Sql | Hibernate |
|--------------------|---------|-----------|
| Повторно           | ~45 ms  | ~480 ms   |

Выигрыш в основном произошел за счет того, что избавились от проблемы n+1.

-----------------------
Во время выполнения этого здания я обнаружила много скрытых моментов и подводных камней Hibernate, о которых раньше не знала и даже не задумывалась. Записала их тут:
1. Если мы для методов toString, hash, equals используем Lombok, то получается, что реализация этих методов для класса зависит от вложенных объектов. И если эти объекты с ленивой загрузкой, то мы можем получить ошибки вроде "LazyInitializationException".
2. Пункт 1 также означает, что если у нас вложенные объекты с ленивой загрузкой + toString из Lombock, мы можем неожиданно получить n+1 при логировании, если Lombock реализует toString так, что вызывает toString для всех вложенных объектов. Для меня это было совершенно неожиданно. 
3. Перед сохранением объекта Hibernate делает select-запрос в БД, чтобы проверить, что объект не изменился.
4. Какой тип загрузки будет у вложенного объекта - lazy/eager - не всегда ясно из кода. В спецификации JPA написано, что для @ManyToOne - eager, но это может меняться в зависимости от версии Hibernate. Я прочитала, что для последних версий Hibernate для всех типов отношений будет Lazy, но у нас последняя версия, и у нас Eager. (А что если в какой-то версии Hibernate она станет ленивой? Наверное было бы лучше, если бы мы как разработчики явно прописывали для каждого поля способ загрузки в нашем проекте.)
5. Если мы используем @EntityGraph, чтобы избежать n+1, то запрос подгружает сущности, которые мы указали в аннотации, но более глубокие уровни вложенности нет. И можно получить LazyInitializationException.
6. Стандартные методы JpaRepository не умеют исправлять n+1.
7. Я прочитала такую рекомендацию, что нельзя передавать managed-объекты из одной сессии в другую, особенно если они содержат lazy-связи. Но тогда мы должны писать код, подчиняясь Hibernate-у, что выглядит неправильным.
8. Если мы расширяем интерфейс JpaRepository (из Spring Boot Data, он позволяет вызывать методы вроде findAll и не писать их реализацию, ее автоматически генерирует Spring Boot), и хотим чтобы findAll возвращал Set, a не List, то может возникнуть проблема. Spring сначала получает List, а потом пытается конвертировать его в Set. При этом он внутри себя вызывает метод hash() у объектов из листа, и если hash() реализован через Lombok, то получаем ошибку LazyInitializationException для вложенных объектов с ленивой инициализацией.
9. В Hibernate если мы обновляем поле у объекта, а в БД уже такое же значение, то Hibernate не будет делать update в БД. 

У нас в проекте ORM выглядит просто и красиво. 
Но теперь у меня вопросы, сколько под капотом у нас скрывается n+1? Если один запрос это примерно 20 ms (как показала библиотека логирования), т.е. 100 лишних запросов это 2 лишние секунды?













