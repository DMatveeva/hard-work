### 1

Для обработки входящих сообщений используется паттерн стратегия - есть параметризованный интерфейс TaskHandler, его реализуют классы-обработчики конкретного сообщения.
Конкретный обработчик получаем из HashMap по ключу type (тип задачи).

Было: проверяем, что среди наших обработчиков есть обработчик для данного типа задач. Если нет - бросаем исключение.
```java
//handlers - классы-обработчики, автоматически собираются через Spring
private final Map<Type, TaskHandler<?>> handlerMap = handlers.stream()
                .collect(Collectors.toMap(TaskHandler::Type, Function.identity()));

public void createTask(TaskRequest request) {
        Type type = request.getType();
        
        if(!handlerMap.contains(type)) {
            throw new IllegalArgumentException("Unsupposted task type");
        }
        
        TaskHandler<?> handler = handlerMap.get(type);
        ((TaskHandler<TaskParameters>) handler).handle(request.getParameters());
    }
    
    
```
Применяем принцип #2. Изменяем ментальную модель логики, которую реализует код с условиями, так что они больше не нужны. 

Стало: используем паттерн Null-object. 
Создаем универсальный класс-обработчик, для всех неподдерживаемых типов задач. 

```java
public final class UnknownTaskHandler implements TaskHandler<TaskParameters> {
    public static final UnknownTaskHandler INSTANCE = new UnknownTaskHandler();

    private UnknownTaskHandler() {}

    @Override
    public void handle(TaskParameters parameters) {
        throw new IllegalArgumentException("Unsupported task type");
    }
}

public void createTask(TaskRequest request) {
        Type type = request.getType();
        
        TaskHandler<?> handler = handlerMap.getOrDefault(Type, UnhandledTaskHandler.INSTANCE);
        
        //Теперь не нужна проверка. В случае, если тип задачи не поддерживается, 
        //мы вызовем handle у UnknownTaskHandler, который выбросит исключение
        ((TaskHandler<TaskParameters>) handler).handle(request.getParameters());
    }
```

### 2

Сервис обрабатывает запросы на создание задач. 
Для создания задачи каждого типа требуется своя роль. Список ролей передается в заголовке roles запроса.

Было:
Первая проверка происходит в контроллере - проверяем, что заголовок не пустой.
Вторая - уже в сервисе, который создает задачу, проверяем наличие нужной роли. 

```java
@RestController
public class TaskController {
    
    @PostMapping
    public void createTask(
            @RequestHeader(name = "Roles") String roles,
            @Valid @RequestBody TaskRequest request) {

        List<String> roles = parseRoles(roles);
        
        //1-я проверка
        if (roles.isEmpty()) { throw new MissingHeaderException(); }

        taskService.createTask(roles, request);
    }
}
//TaskService
public void createTask(TaskRequest request) {
    //В этом сервисе получаем нужный обработчик в зависимости от типа задачи. 
    //Допустим, задача была с типом VALIDATION, тогда вызовется обработчик ValidationTaskService
}

public class ValidationTaskService {
    
    public void createTask(List<String> roles, TaskRequest request) {
        
        //2-я проверка. В конкретном обработчике проверяем наличие нужно роли, в данном случае ADMIN
        if (roles.contains(ADMIN_ROLE)) {
            throw new ForbiddenException("Operation forbidden.");
        }
        // create task
    }
}
```
Стало: применим принцип 1. "Переместите проверку некой неизвестной туда, где она будет известна."

Создадим класс UserContext.
В конструкторе сразу делаем проверку, что роли есть.
И для каждой роли делаем метод, проверяющий ее наличие, который затем вызываем из соответствующего обработчика. 

```java

public record UserContext(List<String> roles) {
    private static final String ADMIN_ROLE = "ADMIN";
    // other roles

    public UserContext(String rolesStr) {
        if(rolesStr == null || rolesStr.isBlank()) {
            throw new MissingHeaderException();
        }
        roles = parse(rolesStr);
    }
    
    public void validateAdminRole() {
        if (!roles.contains(ADMIN_ROLE)) {
            throw new ForbiddenException();
        }
    }
}

@RestController
public class TaskController {
    
    @PostMapping
    public void createTask(
            @RequestHeader(name = "Roles", required = false) String roles,
            @Valid @RequestBody TaskRequest request) {

        UserContext context = new UserContext(roles);
        taskService.createTask(context, request);
    }
}

public class ValidationTaskService {
    public void createTask(UserContext context, TaskRequest request) {
        context.validateAdminRole();
    }
}

```


### 3
Сервис обрабатывает входящее сообщение на совершение покупки какого-то финансового актива. 
Во входящем сообщении есть поле parameters, а у него - поле dealValue.

```json
{
  "parameters": {
    "dealValue": 100.0869
  }
}
```

Есть условие, что dealValue != null, больше 0 и имеет не более 4-х знаков после запятой. 

Было: входящее сообщение десериализуется в DTO DealRequest с помощью библиотеки Jackson. 
Далее в сервисе DealService вызывается метод createDeal, где перед сохранением проверяются все условия dealValue.

```java
public void createDeal(DealRequest request) {
    validateParameters(request.getParameters());
    //...
}

private void validateParameters(DealParameters params) {
        if (params == null || params.getDealValue() == null) {
            throw new IllegalArgumentException("dealValue must be >= 0");
        }
        BigDecimal dealValue = params.getDealValue();
        if (dealValue.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("dealValue must be >= 0");
        }
        if (newLimit.scale() > 4) {
            throw new IllegalArgumentException("dealValue must have not more than 4 decimal places");
        }
    }
```

Применяем принцип №1. Перенесите каждую проверку некой неизвестной туда, где она уже известна.

Мы знаем значение dealValue сразу же при создании DealParameters. 
Jackson имеет аннотации, которые позволяют провалидировать поля объекта.
А валидацию количества знаков можно вынести в конструктор. 
А в сервисе вообще не нужно валидации - в случае некорректного значения dealValue, мы получим ошибку сразу при десериализации. 

```java
public class DealParameters {

    @NotNull(message = "dealValue must not be null")
    @DecimalMin(value = "0.0", message = "dealValue must be >= 0")
    private final BigDecimal dealValue;

    @JsonCreator
    public DealParameters(@JsonProperty("dealValue") BigDecimal dealValue) {
        if (dealValue.scale() > 4) {
            throw new IllegalArgumentException("dealValue must have not more than 4 decimal places");
        }
        this.dealValue = dealValue;
    }
}
```

### 4

Дано: при создании задачи нужно выполнить проверку, что для данного типа задач существует шаблон в базе данных.
Если да, то надо извлечь из шаблона поле Limit, а если нет - выбросить исключение.

Было:

```java
Optional<TaskTemplateEntity> taskTemplateOpt = taskTemplateRepository.findByTaskType(taskType);
int limit = taskTemplateOpt
        .map(TaskTemplateEntity::getLimit)
        .orElseThrows(() -> new IllegalArgumentException("Task template does not exist."));
```

Хоть это записано в функциональном стиле, но это все равно проверка объекта TaskTemplateEntity на null. И ее можно избежать, переместив проверку непосредственно в базу данных (принцип №1).

```java
int threshold = taskTemplateRepository.findLimitByTaskType(taskType)
        .orElseThrow(() -> new IllegalArgumentException("Task template does not exist."));
```

### 5
В сервисе происходит создание новых продуктов. При создании продукта должно отправляться письмо, при условии, что включен режим мониторинга. 
В методе для отправки есть проверка - если режим включен, отпрвляем.

```java
public class NewProductAlertingService {

    @Value("${new-product-alerting.enabled}") 
    boolean alertingEnabled;

    public void alertAboutProduct(ProductEntity product) {
        if(!alertingEnabled) {
            log.info("Skipping sending mail because alerting is disabled.");
            return;
        }
        alertAboutNewProduct(product);
    }
    
}
```

Используем принцип 2 и паттерн "Null object".

Вместо проверки используем реализацию ProductAlertingService с переопределенным методом alertAboutProduct, который просто логирует. 
Spring Boot позволяет в классе Configuration создавать нужный бин в зависимости от значения проперти new-product-alerting.enabled.


```java
@Configuration
public class Configuration {

    @Bean
    @ConditionalOnProperty(name = "new-product-alerting.enabled", havingValue = "false")
    public ProductAlertingService productAlertingServiceDisabled() {
        return new ProductAlertingService() {
            @Override
            public void alertAboutProduct(ProductEntity product) {
                log.info("Skipping sending mail because alerting is disabled.");
            }
        };
    }
}
```
