Как можно снять/ослабить ограничения ORM-ов. И как в целом можно улучшить модели данных в ваших проектах.

### 1

Так совпало, что как раз сейчас я работаю над задачей, где необходимо разрабатывать с нуля новый микросервис. И это задание оказалось особенно актуально, потому что в самом начале можно внести какие-то новые идеи, которые в новый микросервис вносить дорого.

Во всех наших микросервисах повсеместно используется Spring Data JPA. 
Это обертка Spring над Hibernate, которая позволяет автоматически генерировать реализацию метода в репозиториях по имени метода.
Также, у нас везде анемичная доменная модель. 
Т.е. есть класс Entity, который через ORM маппится на таблицу в БД. 
В таком классе есть только поля, которые соответствуют стоблцам таблицы в БД, но нет никакой бизнес логики. Вся логика расположена в слое сервисов.
```java
@Entity
@Table(name = "task")
public class TaskEntity {
    
    @Column(name = "id")
    Long id;

    @Column(name = "description")
    String description;
    
    @Column(name = "is_completed")
    Boolean isCompleted;
    
    @ManyToOne
    @JoinColumn(
            name = "user_id",
            referencedColumnName = "id"
    )
    User user;
}

@Service
public class TaskService {
    
    public cancelTask(Long id) {
        TaskEntity task = taskRepository.findById(id);
        task.setIsCompleted(false);
        task.setUser(null);
        taskRepository.save(task);
    }
}
```

Но когда я разбиралась с Hibernate для данного задания, я выяснила интересную вещь: из-за того, что объект Entity не содержит бизнес-логики, мы не используем многих фичей Hibernate, таких как dirty checking, lazy loading. 

И в нашем случае вместо Spring Data JPA можно использовать Spring Data JDBC. Он вообще не использует Hibernate. Вот его уровни:

`Spring Data JDBC -> JdbcTemplate (надстройка Spring над plain JDBC)-> JDBC -> Postgres`  

В то время как Spring Data JPA:

`Spring Data JPA -> EntityManager -> Hibernate -> JDBC -> Postgres`

Не будет накладных расходов на прокси, n+1. При этом, это будет тот же удобный CRUD Repository, к которому привыкла команда, но без накладных расходов ORM.


### 2
В тех проектах, где уже активно используется Sping Data JPA (с Hibernate под капотом), можно частично внедрить принцип CQRS.

Сейчас для всех запросов, даже для простых GET, ORM загружает из БД полноценный объект. Допустим, пользователь хочет увидеть список задач на UI + имя исполнителя задачи. Фронтенд делает запрос GET /tasks.

На бэкенде Hibernate генерирует запрос по извлечению всех задач.
В `task` есть foreign key `user_id`, а в таблице `user` есть foreign key на `user_params`. 
И hibernate загружает все эти связанные сущности, хотя для отображения на фронденде нам нужно только несколько полей из таблицы `task` + поле из таблицы `user` (тут можно также словить n+1).
А дальше результат преобразовывает в список прокси-объектов `TaskEntity`, загружает в контекст, начинает отслеживать.

Оказывается, в Spring Data есть фича, которая называется record projection (и в Spring Data JPA, и в Spring Data JDBC). (Record - это тип данных, появившийся в Java 16, запись immutable)

Его суть в том, что можно сделать возвращаемый тип метода в репозитории - не Entity (который управляется ORM), а простой класс record, с нужными нам полями.
В этом случае не будут созданы Entity в контексте Hibernate.
Будут созданы просто неизменяемые записи с нужными полями, которые можно отдать на фронтенд.  

```java
public record TaskWithUser (
        Long id, 
        String description,
        Long userId
) {}
@Repository
public interface TaskRepository extends JpaRepository<Task, Long> {
    @Query(value = """
            select 
                t.id as task_id,
                t.description as task_description,
                u.id as user_id
            from task t
            join user u on t.user_id = u.id
            """)
    List<TaskWithUser> findAllTasks();
} 
```

Таким образом можно разделить репозитории на read only, которые только возвращают неизменяемые проекции, и на write, которые могут модицировать данные.
