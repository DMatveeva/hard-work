Примеры из моего проекта ООП "Три в ряд".
Там я создала класс Game, содержащий поля 
- enterCoordinatesStatus : флаг, показывающий, правильно ли пользователем введены координаты клеток, которые надо поменять местами (например, что клетки соседние)
- executeRoundStatus : флаг, показывающий, что после введения координат получилась хоть одна комбинация. Если нет, то игра оканчивается.

Здесь возможен неправильный порядок вызова методов. Например, если сначала вызвать executeRound(), без вызова enterCoordinates(), то игра сразу окончится.


```java
class Game extends AbstractGame {

    private final Matrix matrix;
    private final CommandHistory commandHistory;
    private Figures figures = Figures.empty();
    private Scanner input = new Scanner(System.in);
    private int executeRoundStatus;
    private int enterCoordinatesStatus;

    public void enterCoordinates() { /*...*/ }
    public void executeRound()  { /*...*/ }

    public int getExecuteRoundStatus() { /*...*/ }
    public int getEnterCoordinatesStatus() { /*...*/ }
}


public static void main(String[] args) {
    Game game = new Game(/*..*/);

    while (game.getExecuteRoundStatus() == EXECUTE_ROUND_OK) {
        while (game.getEnterCoordinatesStatus() != ENTER_COORDINATES_OK) {
            game.enterCoordinates();
        }
        game.executeRound();
    }
    System.out.println("Game over!");
}

```

Попробуем переделать, чтобы вместо статуса enterCoordinatesStatus было два класса.
- IActiveGame - класс, соответствующий флагу enterCoordinatesStatus == ENTER_COORDINATES_OK, т.е. координаты клеток введены корректно. 
- IPendingGame - класс, соответствующий флагу enterCoordinatesStatus == ENTER_COORDINATES_ERR или ENTER_COORDINATES_NIL, т.е. коодинаты пока не были введены, либо введены некорректно. 
```java

class IActiveGame extends Game { // координаты введены корректно

    private final Matrix matrix;
    private final CommandHistory commandHistory;
    private Figures figures = Figures.empty();
    private int executeRoundStatus;
    //input нам здесь уже не нужен

    public Game executeRound() {
        //...
        return new PendingGame(matrix, commandHistory, figures);
    }
}

class IPendingGame extends Game { // игра ожидает координаты 

    private final Matrix matrix;
    private final CommandHistory commandHistory;
    private Figures figures = Figures.empty();
    private Scanner input = new Scanner(System.in); 
    private int executeRoundStatus;

    public Game enterCoordinates() {
        //...
        //if input incorrect -> return IActiveGame, else - new IPendingGame
        return new ActiveGame(matrix, commandHistory, figures);;
    }
}
public static void main(String[] args) {
    IPendingGame newGame = new IPendingGame(/*..*/);

    Game activeGame = startRound(newGame);
    
    while (activeGame.getExecuteRoundStatus() == EXECUTE_ROUND_OK) {
        activeGame = startRound(newGame);
    }
    System.out.println("Game over!");
}


public static Game startRound(Game game) {
    if (game instanceof IPendingGame pendingGame) {
        startRound(pendingGame.enterCoordinates());
    }
    if (game instanceof IActiveGame activeGame) {
        return activeGame.executeRound();
    }
    throw new IllegalArgumentException("No such game type");
}
```

Теперь не получиться вызвать метод executeRound(), пока пользователь не ввел координаты. 
Но появилась проверка типов в рантайме. 

Попробую пойти дальше и сделать 2 класса вместо флага executeRoundStatus.
- IActiveGame - этот уже существующий класс теперь будет соответствовать флагу executeRoundStatus = EXECUTE_ROUND_OK, когда раунд прошел успешно.
- IFailedGame - новый класс, будет соответствовать флагу executeRoundStatus = EXECUTE_ROUND_ERR. По смыслу это означает, что после перестановки клеток не образовалось ни одной фигуры "3 в ряд".

Если мы сделаем ошибку и вызовем executeRound() у игры со статусом executeRoundStatus = EXECUTE_ROUND_ERR, то игра продолжится дальше, что нарушит наши требования - заканчивать игру, если у игрока не получилось сделать комбинации. 

```java

public class IFailedGame extends Game {

    private final Matrix matrix;
    private final CommandHistory commandHistory;
    private Figures figures;
    
    public Game exit() {
        //...
        // например, вывод очков и истории ходов
        System.out.println("Game over!");
        return new IFailedGame(matrix, commandHistory, figures);
    }

}
public class IActiveGame extends Game {

    private final Matrix matrix;
    private final CommandHistory commandHistory;
    private Figures figures;

    public Game executeRound() {
        //...

        if(/*если раунд прошел успешно, возвращаем IPendingGame для ввода координат*/) {
            return new IPendingGame(matrix, commandHistory, figures);
        }
        // в случае неудачи IFailedGame, т.к. игра окончена
        return new IFailedGame(); 
    }
}

public static void main(String[] args) {
    IPendingGame newGame = new IPendingGame();

    Game activeGame = startRound(newGame);
    executeRound(activeGame);
    
}

public static Game executeRound(Game game) {
    if (game instanceof IFailedGame failedGame) {
        failedGame.exit();
    }
    if (game instanceof IActiveGame activeGame) {
        executeRound(startRound(activeGame));
    }
    throw new IllegalArgumentException("No such active game type");
}


public static Game startRound(Game game) {
    if (game instanceof IPendingGame pendingGame) {
        startRound(pendingGame.enterCoordinates());
    }
    if (game instanceof IActiveGame activeGame) {
        return activeGame.executeRound();
    }
    throw new IllegalArgumentException("No such game type;");
}

```
Плюсы: 
- можем лучше контролировать поведение объектов, не получится вызвать методы в неправильном порядке и получить класс с неправильным состоянием
- классы стали компактнее, за счет отсутствия полей-флагов и частично методов, которые теперь разделены между всеми классами-потомками 

Минусы:
- для перехода между статусами приходится проверять тип в рантайме (не придумала пока, как от этого избавиться)
- на первый взгляд, код может выглядеть запутаннее, особенно рекурсия
