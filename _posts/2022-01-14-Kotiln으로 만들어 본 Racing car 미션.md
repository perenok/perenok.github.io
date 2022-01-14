---
title: "Kotiln으로 만들어 본 Racing Car 미션"
date: 2022-01-14
tags:
- Kotlin
toc: true
toc_sticky: true
toc_label: "Kotiln으로 만들어 본 Racing Car 미션"
---

Kotln 공식 문서 정리를 통해 상승한 자신감을 등에 업고 호기롭게 미션으로 뛰어들어 보았다. 깨지고 부서지며 작성한 코드 후기를 통해 혹시라도 이 미션을 도전할 사람에게 도움이 되길 바란다.

## 미션 설명

미션 깃허브 주소는 [여기](https://github.com/woowacourse/kotlin-racingcar) 로 들어가면 된다.

내가 짠 코드는 [여기](https://github.com/perenok/kotlin-racingcar/tree/step1)

### 기능 요구사항

- 주어진 횟수 동안 n대의 자동차는 전진 또는 멈출 수 있다.
- 각 자동차에 이름을 부여할 수 있다. 전진하는 자동차를 출력할 때 자동차 이름을 같이 출력한다.
- 자동차 이름은 쉼표(,)를 기준으로 구분하며 이름은 5자 이하만 가능하다.
- 사용자는 몇 번의 이동을 할 것인지를 입력할 수 있어야 한다.
- 전진하는 조건은 0에서 9 사이에서 random 값을 구한 후 random 값이 4 이상일 경우 전진하고, 3 이하의 값이면 멈춘다.
- 자동차 경주 게임을 완료한 후 누가 우승했는지를 알려준다. 우승자는 한 명 이상일 수 있다.

**실행 결과**

- 위 요구사항에 따라 3대의 자동차가 5번 움직였을 경우 프로그램을 실행한 결과는 다음과 같다.

```
경주할 자동차 이름을 입력하세요(이름은 쉼표(,)를 기준으로 구분).
pobi,crong,honux
시도할 회수는 몇회인가요?
5

실행 결과
pobi : -
crong : -
honux : -

pobi : --
crong : -
honux : --

pobi : ---
crong : --
honux : ---

pobi : ----
crong : ---
honux : ----

pobi : -----
crong : ----
honux : -----

pobi : -----
crong : ----
honux : -----

pobi, honux가 최종 우승했습니다.
```

## 미션 시작

### README 작성

먼저 README에 요구사항을 정리했다.

```markdown
# kotlin으로 구현하는 racing-car 미션
우아한테크코스 첫번째 미션이었던 racing car를 kotlin으로 구현해보자.

# 자동차 경주 게임
## 기능 요구사항
### 1. 경주할 자동차 이름 입력

- 이름들을 쉼표(,) 구분으로 받아 나눠서 각각의 Car 객체에 삽입
- 이름이 5자가 넘는다면 예외처리

### 2. 이동 시도 횟수 입력

- 전진 시도 횟수를 입력한다.
- 숫자가 아닌 입력을 받으면 예외처리

### 3. 랜덤값을 통한 position 설정

- 각각의 Car 객체에 대해 랜덤 조건 실행
- 랜덤값이 4 이상일 경우 한칸 전진

### 4. 결과 출력
- 모든 Car 객체가 한번씩 랜덤 조건을 실행했다면 그 결과를 출력
- "실행 결과" 출력 후 줄바꿈
- 이름 + " : " + 위치의 형태로 출력
- position값만큼 '-'를 출력

### 5. 반복

- 시도횟수에 도달할 때까지 3~4과정 반복

### 6. 최종 우승자 출력

- Car.position 값을 비교하여 최대값을 가진 객체의 name을 출력
- 동점자들이 있으면 해당 객체들의 name을 출력
- 이때 name들 사이에 쉼표(,)로 구분해야 함
```

readme에 정리한 기능을 따라 하나씩 만들어보려고 한다.

흐름은 main → controller → service → domain으로 간다. 콘솔에 출력하는 코드라 controller가 필요없지만 MVC 패턴에 가깝게 만들어보고자 넣었다.

```kotlin
fun main() {
    val racingCarController = RacingCarController()
    racingCarController.run()
}
```

mani 함수이다. 컨트롤러를 선언하고 run 메서드를 실행한다.

### 1. 경주할 자동차 이름 입력

먼저 입력을 받는 InputView 클래스를 만든다. 상태를 가지고 있지 않기 때문에 매번 객체를 생성할 필요 없어 object로 싱글턴 객체로 만들어주었다.

```kotlin
object InputView {
    tailrec fun inputCarNames(): List<String> {
        println("경주할 자동차 이름을 입력하세요(이름은 쉼표(,)를 기준으로 구분)")
        return readLine()?.replace(" ", "")?.split(",") ?: inputCarNames()
    }
}
```

`tailrec` 함수로 선언했다. 꼬리재귀라는 의미로 추가적인 연산 없이 스스로 재귀호출하다 특정 값을 리턴하는 함수를 뜻한다. 실제로 컴파일 할 때 자원 소비가 적은 루프 코드로 변경된다. readLine()부터 replace(), split()에 null이 들어오면 재귀함수를 통해 다시 inputCarNames 함수를 호출한다.

```kotlin
class RacingCarController { 
    
    private val inputView = InputView

    fun run() {
        val carNames = inputView.inputCarNames()
        val carService = RacingCarService(carNames)
    }
}
```

```kotlin
class RacingCarService(carNames: List<String>) {
    
    private val cars: Cars = Cars()

    init {
        cars.addAll(createCars(carNames))
    }

    // 추후 로직 추가

    private fun createCars(carNames: List<String>): MutableList<Car> {
        return carNames.map { Car(CarName(it)) as MutableList<Car> }
    }
}
```

```kotlin
class Cars {
    
    private val cars: MutableList<Car> = mutableListOf()

    ... // 일급컬렉션. 추후 로직 추가
}
```

```kotlin
class Car(
    
    private val carName: CarName,
    val position: Int = 0,
) {
    // 추후 로직 추가
}
```

자동차 클래스이다. carName이라는 원시값 포장클래스와 position을 필드로 가지고 있다.

```kotlin
private const val MAX_NAME_LENGTH = 5

class CarName(
    val name: String,
) {
    init {
        if (name.length > MAX_NAME_LENGTH) {
            throw CarNameException()
        }
    }
}
```

원시값 포장 클래스. String인 name을 CarName이라는 클래스로 포장해 생성 시 조건을 체크한다.

```kotlin
class CarNameException(
    message: String = "자동차 이름은 5자 이하여야 합니다.",
) : RuntimeException(message)
```

커스텀 예외 클래스. RuntimeException을 상속받았다.

### 2. 이동 시도 횟수 입력

```kotlin
object InputView {
    tailrec fun inputCarNames(): List<String> {
        println("경주할 자동차 이름을 입력하세요(이름은 쉼표(,)를 기준으로 구분)")
        return readLine()?.replace(" ", "")?.split(",") ?: inputCarNames()
    }

    tailrec fun inputTryNumber(): Int {
        println("시도할 회수는 몇회인가요?")
        return readLine()?.toIntOrNull() ?: inputTryNumber()
    }
}
```

inputView object에 inputTryNumber() 함수 추가. toIntOrNull()을 통해 숫자가 아닌 값이 들어오면 null을 반환한다. null이 들어오면 꼬리재귀에 의해 다시 입력 함수가 실행된다.

### 3. 랜덤값을 통한 position 변경

```kotlin
class Car(
    private val carName: CarName,
    val position: Int = 0,
) {
    fun move(moveStrategy: MoveStrategy): Car {
        if (moveStrategy.canMove()) {
            return Car(carName, position + 1)
        }
        return this
    }
}
```

전략 패턴을 사용해 움직이는 조건을 정했다. Car에선 어떤 전략이 오는지는 모르고 움직이는 조건만 만족하면 position + 1 된 객체를 반환한다. 이를 통해 Car 객체는 불변 객체로 유지된다.

```kotlin
interface MoveStrategy {
    fun canMove(): Boolean
}
```

```kotlin
private const val MOVE_CONDITION = 4
private const val RANDOM_NUMBER_MAX_BOUNDARY = 10

class RandomMoveStrategy : MoveStrategy {
    
    override fun canMove(): Boolean {
        return generateRandomNumber() >= MOVE_CONDITION
    }

    private fun generateRandomNumber(): Int {
        return Random.nextInt(RANDOM_NUMBER_MAX_BOUNDARY)
    }
}
```

Car의 앞으로 이동하는 로직은 완성됐다. 이를 일급 컬렉션인 Cars도 적용해보자.

```kotlin
class Cars(private val cars: MutableList<Car>) {

    fun moveAll(moveStrategy: MoveStrategy) {
        cars.replaceAll { it.move(moveStrategy) }
    }
}
```

car의 move가 position이 변경된 Car 객체를 반환하므로 cars에 있는 인자들을 각각 랜덤한 move 결과로 생긴 새로운 car객체로 바꿔치기한다.

### 4. 결과 출력 + 5.반복

```kotlin
object OutputView {

    fun printResultMessage() {
        println("실행 결과")
    }

    fun printRaceResult(cars: List<CarDto>) {
        cars.forEach {
            println("${it.name} : ${"-".repeat(it.position)}")
        }
        println()
    }
}
```

결과 출력을 위한 OutputView object다.

```kotlin
class RacingCarController {

    private val inputView = InputView
    private val outputView = OutputView

    fun run() {
        val carNames = inputView.inputCarNames()
        val tryNumber = inputView.inputTryNumber()
        val carService = RacingCarService(carNames)

        outputView.printResultMessage()
        repeat(tryNumber) {
            outputView.printRaceResult(carService.race())
        }
    }
}
```

tryNumber만큼 반복해서 race를 하고 그 결과를 출력한다.

```kotlin
class RacingCarService(
    carNames: List<String>,
) {
    private val cars: Cars = Cars()

    init {
        cars.addAll(createCars(carNames))
    }

    private fun createCars(carNames: List<String>): List<Car> {
        return carNames.map { Car(CarName(it)) }
    }

    fun race(): List<CarDto> {
        cars.moveAll(RandomMoveStrategy())
        return cars.getCars()
    }
}
```

```kotlin
class Cars {
    private val cars: MutableList<Car> = mutableListOf()

    fun addAll(cars: List<Car>) {
        this.cars.addAll(cars)
    }

    fun moveAll(moveStrategy: MoveStrategy) {
        cars.replaceAll { it.move(moveStrategy) }
    }

    fun getCars(): List<CarDto> {
        return cars.map { CarDto(it.getName(), it.position) }
    }
}
```

CarDto는 car와 동일한 property를 가지는 data class이다. 이를 Dto로 이용했다.

### 6. 최종 우승자 출력

```kotlin
object OutputView {

    fun printResultMessage() {
        println("실행 결과")
    }

    fun printRaceResult(cars: List<CarDto>) {
        cars.forEach {
            println("${it.name} : ${"-".repeat(it.position)}")
        }
        println()
    }

    fun printWinners(winnerNames: List<String>) {
        println("${winnerNames.joinToString(",")}가 최종 우승했습니다.")
    }
}
```

printWinners() 로직이 추가되었다. list로 받은 우승자들의 이름을 joinToString()을 통해 하나의 String으로 만들었다.

```kotlin
class RacingCarController {

    private val inputView = InputView
    private val outputView = OutputView

    fun run() {
        ... // 기존 코드
        outputView.printWinners(carService.getWinners())
    }
}
```

컨트롤러에 승자 출력 코드를 추가한다.

```kotlin
class RacingCarService(
    carNames: List<String>,
) {
    ... // 기존 로직

    fun getWinners(): List<String> {
        val maxPosition = cars.calculateMaxPosition()
        return cars.findCarsBySamePosition(maxPosition)
            .map { it.getName() }
    }
}
```

cars에서 승자를 찾기 위해 먼저 가장 멀리 간 차의 포지션을 찾는다. 그 후 해당 위치와 동일한 자동차들을 name으로 mapping해 List<String>으로 반환한다.

```kotlin
class Cars {
    ... // 기존 로직

    fun calculateMaxPosition(): Int {
        return cars.maxByOrNull { it.position }?.position ?: 0
    }

    fun findCarsBySamePosition(position: Int): List<Car> {
        return cars.filter { it.isSamePosition(position) }
    }
}
```

cars에서 포지션 중 최대를 찾는다. 만약 없으면 null이 들어오고 디폴트 값으로 0이 된다.

findCarsBySamePosition(position: Int) 함수에서는 Car의 isSamePosition(position)가 참인 경우의 car들만 filter()를 통해 남긴다.

```kotlin
class Car(
    private val carName: CarName,
    val position: Int = 0,
) {
    fun move(moveStrategy: MoveStrategy): Car {
        if (moveStrategy.canMove()) {
            return Car(carName, position + 1)
        }
        return this
    }

    fun isSamePosition(position: Int): Boolean {
        return this.position == position
    }

    fun getName(): String {
        return carName.name
    }
}
```

### 출력 화면

![스크린샷 2022-01-13 오후 11.37.25.png](/assets/image/racingcar/racingcar1.png)

원하는 결과대로 나왔다. 테스트는 따로 정리하여 말해보려고 한다.

## 정리

이렇게 첫 번째 미션이었던 racing-car를 코틀린으로 구현해보았다. 문법 자체에 익숙해지기 위해 아키텍쳐나 불변, 예외처리에 대해서는  넘어간 부분들이 있어 아쉬움이 남는다. 추후 리팩토링을 진행하며 하나씩 보강을 하겠다. 공식 문서를 통해 문법을 봤지만 막상 코딩을 하려고 하니 막막하고 자꾸 자바를 하던 습관대로 코드를 작성하여 코틀린을 제대로 활용하지 못하는 느낌이었다.

필요한 함수와 문법에 대해 찾아가며 하나하나씩 채웠다. 줄어드는 코드의 양과 간편하게 표현식을 사용해서 기능을 완성할 때 코틀린의 장점을 어렴풋하게나마 느낀 것 같다. 다만 내가 짠 코드가 맞냐고 물으면 그건 아니라고 생각한다. 각각의 코드에 대해 대체할 수 있는 다양한 문법도 있을 것이고 하나의 문법에 대해 명확한 이유를 가지고 선택한 것이 아니다보니 내 코드를 보면서 다시 자세하게 공부를 해보려고 한다.

## 키워드

이 부분은 코드를 짜면서 고민했지만 아직 명확하게 답을 찾지 못한 부분이다. 추후 학습을 통해 답을 추가하겠다.

- private const val로 상수 선언과 companion object로 내부 싱글턴 객체로 상수 선언의 차이점
- mutableList와 List는 무조건 분리해서 사용해야 하나?
- List.toMutableList()와 List as MutableList<>() 의 차이
