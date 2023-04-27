---
title: "Bean Validation в Spring на практике"
author: ""
type: ""
date: 2023-04-24T12:00:00+03:00
subtitle: ""
image: ""
tags: []
private: false
---
Статья демонстрирует основные возможности [Bean Validation](https://beanvalidation.org/) интегрированные в Spring на практических примерах. Статья подразумевает, что вы знакомы с теорией по `Bean Validation` и хотите лучше понять библиотеку на практике.
<!--more-->

В статье примеры реализованы на `Spring Boot 3` и `Spring 6`, поэтому все Jakarta EE (Java EE) классы импортируются из пакета `jakarta.validation.*`. Примеры совместимы с предыдущей версией `Spring` в который классы импортируются из пакета `javax.validation.*`.

Я пропустил документацию по аннотациям ограничений (`jakarta.validation.constraints.*`), так как аннотаций достаточно много и они подробно описаны в [официальной документации](https://jakarta.ee/specifications/bean-validation/3.0/jakarta-bean-validation-spec-3.0.html#builtinconstraints) и многих других статьях.

# Типы валидации
## Валидация входных данных контроллера
Главной аннотацией в [Bean Validation](https://beanvalidation.org/) является аннотация `@Valid`.
С помощью нее активируется автоматическая валидация бинов.

Для валидации входных данных контроллера аннотация `@Valid` ставится перед входными аргументами методов в контроллере.
Если валидируемый бин в качестве поля содержит другой бин, который тоже требуют валидировать, то поле с бином так же необходимо пометить аннотацией `@Valid`.

[CreateInputRequest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/controller/request/CreateInputRequest.java):
```java
@Data
public class CreateInputRequest {
    @NotNull
    @Size(max = 20)
    private String name;
    @NotNull
    @Min(0)
    private Integer inputId;
    @NotNull
    @Valid
    private AttachmentDto attachment;
}
```

[AttachmentDto](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/dto/AttachmentDto.java):
```java
@Data
public class AttachmentDto {
    @NotNull
    @Max(255)
    private String content;
}
```

[InputController](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/controller/InputController.java):
```java
@RestController
@RequestMapping("/v1/inputs")
public class InputController {
    @PostMapping
    public void create(@Valid @RequestBody CreateInputRequest request) {
        log.info("create request: {}", request);
    }
}
```

Преобразование ошибок `Bean Validator` в HTTP ошибки `Spring REST` выполняется с помощью кастомного обработчика исключений.

[GlobalExceptionHandler](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/controller/advice/GlobalExceptionHandler.java):
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(MethodArgumentNotValidException ex) {
        Stream<String> fieldErrors = ex.getFieldErrors().stream()
                .map(e -> String.format("%s: %s", e.getField(), e.getDefaultMessage()));

        Stream<String> globalErrors = ex.getGlobalErrors().stream()
                .map(e -> String.format("%s: %s", e.getObjectName(), e.getDefaultMessage()));

        String error = Stream.concat(fieldErrors, globalErrors)
                .collect(Collectors.joining(", "));

        ErrorResponse response = new ErrorResponse();
        response.setCode(HttpStatus.BAD_REQUEST.value());
        response.setError(error);
        return new ResponseEntity<>(response, new HttpHeaders(), HttpStatus.BAD_REQUEST);
    }
}
```

[InputControllerTest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/test/java/pro/jaitl/spring/examples/validation/controller/InputControllerTest.java):
```java
class InputControllerTest extends BaseApiTest {

    @Test
    public void testInput_Valid() throws Exception {
        AttachmentDto attachmentDto = new AttachmentDto();
        attachmentDto.setContent("some_bite_content");

        CreateInputRequest request = new CreateInputRequest();
        request.setInputId(1);
        request.setName("newInputValidTest");
        request.setAttachment(attachmentDto);


        mockMvc.perform(doPost("/v1/inputs", request))
                .andExpect(status().isOk());
    }

    @Test
    public void testInput_Invalid_InputId() throws Exception {
        AttachmentDto attachmentDto = new AttachmentDto();
        attachmentDto.setContent("some_bite_content");

        CreateInputRequest request = new CreateInputRequest();
        request.setInputId(-1);
        request.setName("newInputValidTest");
        request.setAttachment(attachmentDto);


        mockMvc.perform(doPost("/v1/inputs", request))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.code", equalTo(400)))
                .andExpect(jsonPath("$.error", equalTo("inputId: must be greater than or equal to 0")));
    }

    @Test
    public void testInput_Invalid_Attachment() throws Exception {
        AttachmentDto attachmentDto = new AttachmentDto();
        attachmentDto.setContent(null);

        CreateInputRequest request = new CreateInputRequest();
        request.setInputId(1);
        request.setName("newInputValidTest");
        request.setAttachment(attachmentDto);

        mockMvc.perform(doPost("/v1/inputs", request))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.code", equalTo(400)))
                .andExpect(jsonPath("$.error", equalTo("attachment.content: must not be null")));
    }
}
```

## Валидация Spring Data Entity
`Bean Validation` умеет валидировать `Spring Data Entity` перед вставкой в БД.

[InputEntity](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/repository/InputEntity.java):
```java
@Entity
@Data
public class InputEntity {
    @Id
    @GeneratedValue(strategy = IDENTITY)
    private Integer id;
    @NotNull
    @Size(max = 20)
    private String name;
    @NotNull
    @Min(0)
    private Integer inputId;
}
```
[InputRepository](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/repository/InputRepository.java):
```java
@Repository
public interface InputRepository extends JpaRepository<InputEntity, Integer> {
}
```
[InputRepositoryTest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/test/java/pro/jaitl/spring/examples/validation/repository/InputRepositoryTest.java):
```java
@SpringBootTest
class InputRepositoryTest {
    @Autowired
    private InputRepository inputRepository;

    @Test
    public void testSave_Success() {
        InputEntity entity = new InputEntity();
        entity.setInputId(1);
        entity.setName("test");

        assertDoesNotThrow(() -> inputRepository.save(entity));
    }

    @Test
    public void testSave_Failed() {
        InputEntity entity = new InputEntity();
        entity.setInputId(-1);
        entity.setName(null);

        ConstraintViolationException e = assertThrows(ConstraintViolationException.class,
                () -> inputRepository.save(entity));

        Set<String> constraintViolations = e.getConstraintViolations().stream()
                .map(v -> String.format("%s: %s", v.getPropertyPath().toString(), v.getMessage()))
                .collect(Collectors.toSet());

        assertTrue(constraintViolations.contains("name: must not be null"));
        assertTrue(constraintViolations.contains("inputId: must be greater than or equal to 0"));
    }
}
```

## Валидация входных данных сервиса
`Bean Validation` умеет валидировать входные аргументы методов сервиса. 
Для этого над сервисом ставится аннотация `@Validated`, а перед аргументами аннотация `@Valid`.

[InputService](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/service/InputService.java):
```java
@Slf4j
@Service
@Validated
public class InputService {
    public void handle(@Valid InputEntity inputEntity) {
        log.info("entity: {}", inputEntity);
    }
}
```
[InputServiceTest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/test/java/pro/jaitl/spring/examples/validation/service/InputServiceTest.java):
```java
@SpringBootTest
class InputServiceTest {
    @Autowired
    private InputService inputService;

    @Test
    public void testHandle_Success() {
        InputEntity entity = new InputEntity();
        entity.setId(1);
        entity.setInputId(1);
        entity.setName("test");

        assertDoesNotThrow(() -> inputService.handle(entity));
    }

    @Test
    public void testHandle_Failed_InvalidInput() {
        InputEntity entity = new InputEntity();
        entity.setId(2);
        entity.setInputId(-1);
        entity.setName(null);

        ConstraintViolationException e = assertThrows(ConstraintViolationException.class,
                () -> inputService.handle(entity));

        Set<String> constraintViolations = e.getConstraintViolations().stream()
                .map(v -> String.format("%s: %s", v.getPropertyPath(), v.getMessage()))
                .collect(Collectors.toSet());

        assertTrue(constraintViolations.contains("handle.inputEntity.name: must not be null"));
        assertTrue(constraintViolations.contains("handle.inputEntity.inputId: must be greater than or equal to 0"));
    }
}
```

## Явная валидация
Для явной валидации используется класс `jakarta.validation.Validator`.
Явная валидация позволяет разработать кастомную логику обработки ошибок валидации бинов.

[ValidationTest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/test/java/pro/jaitl/spring/examples/validation/ValidationTest.java):
```java
@SpringBootTest
class ValidationTest {
    @Autowired
    private Validator validator;

    @Test
    public void testValidate_Success() {
        AttachmentDto attachmentDto = new AttachmentDto();
        attachmentDto.setContent("some_bite_content");

        CreateInputRequest request = new CreateInputRequest();
        request.setInputId(1);
        request.setName("newInputValidTest");
        request.setAttachment(attachmentDto);

        Set<ConstraintViolation<CreateInputRequest>> result = validator.validate(request);
        assertTrue(result.isEmpty());
    }

    @Test
    public void testValidate_Invalid_InputId() {
        AttachmentDto attachmentDto = new AttachmentDto();
        attachmentDto.setContent("some_bite_content");

        CreateInputRequest request = new CreateInputRequest();
        request.setInputId(-1);
        request.setName("newInputValidTest");
        request.setAttachment(attachmentDto);

        List<ConstraintViolation<CreateInputRequest>> result = List.copyOf(validator.validate(request));
        assertEquals(1, result.size());

        ConstraintViolation<CreateInputRequest> validationResult = result.get(0);

        assertEquals("inputId", validationResult.getPropertyPath().toString());
        assertEquals("must be greater than or equal to 0", validationResult.getMessage());
    }
}
```

# Группы валидации
`Bean Validation` позволяет делить валидаторы полей на группы под разные юзкейсы. 
Например, для использования одного и того же DTO для создания и обновления сущностей в контроллере, можно маркировать валидаторы которые выполняются при создании сущностей одной группой, а валидаторы которые выполняются при изменении сущностей другой группой.

Группы валидации разделяются через пустые интерфейсы:
```java
interface OnCreateGroup {}
interface OnUpdateGroup {}
```

Затем интерфейсы используются как маркеры для групп.

[OutputDto](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/dto/OutputDto.java):
```java
@Data
public class OutputDto {
    @NotNull(groups = OnCreateGroup.class)
    @Size(max = 20, groups = {OnCreateGroup.class, OnUpdateGroup.class})
    public String name;
    @NotNull(groups = OnCreateGroup.class)
    @Min(value = 0, groups = {OnCreateGroup.class, OnUpdateGroup.class})
    public Integer outputId;
}
```

## Валидация входных данных контроллера
Валидация полей по группам выполняется с помощью аннотации `@Validated` с указанием группы.

[OutputController](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/controller/OutputController.java):
```java
@Slf4j
@RestController
@RequestMapping("/v1/outputs")
public class OutputController {

    @PostMapping
    public void create(@Validated(OnCreateGroup.class) @RequestBody OutputDto outputDto) {
        log.info("create request: {}", outputDto);
    }

    @PutMapping
    public void update(@Validated(OnUpdateGroup.class) @RequestBody OutputDto outputDto) {
        log.info("update request: {}", outputDto);
    }
}
```

[OutputControllerTest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/test/java/pro/jaitl/spring/examples/validation/controller/OutputControllerTest.java):
```java
class OutputControllerTest extends BaseApiTest {
    @Test
    public void testOutput_Create_Valid() throws Exception {
        OutputDto outputDto = new OutputDto();
        outputDto.setName("test");
        outputDto.setOutputId(1);

        mockMvc.perform(doPost("/v1/outputs", outputDto))
                .andExpect(status().isOk());
    }

    @Test
    public void testOutput_Create_Invalid() throws Exception {
        OutputDto outputDto = new OutputDto();
        outputDto.setName(null);
        outputDto.setOutputId(null);

        mockMvc.perform(doPost("/v1/outputs", outputDto))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.code", equalTo(400)))
                .andExpect(jsonPath("$.error", containsString("outputId: must not be null")))
                .andExpect(jsonPath("$.error", containsString("name: must not be null")));
    }

    @Test
    public void testOutput_Update_Valid() throws Exception {
        OutputDto outputDto = new OutputDto();
        outputDto.setName(null);
        outputDto.setOutputId(null);

        mockMvc.perform(doPut("/v1/outputs", outputDto))
                .andExpect(status().isOk());
    }

    @Test
    public void testOutput_Update_Invalid() throws Exception {
        OutputDto outputDto = new OutputDto();
        outputDto.setName("it's longer than 20 characters");
        outputDto.setOutputId(-1);

        mockMvc.perform(doPut("/v1/outputs", outputDto))
                .andExpect(jsonPath("$.code", equalTo(400)))
                .andExpect(jsonPath("$.error", containsString("outputId: must be greater than or equal to 0")))
                .andExpect(jsonPath("$.error", containsString("name: size must be between 0 and 20")));
    }
}
```

## Валидация входных данных сервиса
Валидация входных аргументов методов сервиса выполняется несколько иначе.

[OutputService](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/service/OutputService.java):
```java
@Validated
@Slf4j
@Service
public class OutputService {
    @Validated(OnCreateGroup.class)
    public void handleCreate(@Valid OutputDto outputDto) {
        log.info("create request: {}", outputDto);
    }

    @Validated(OnUpdateGroup.class)
    public void handleUpdate(@Valid OutputDto outputDto) {
        log.info("update request: {}", outputDto);
    }
}
```

[OutputControllerTest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/test/java/pro/jaitl/spring/examples/validation/service/OutputServiceTest.java):
```java
@SpringBootTest
class OutputServiceTest {
    @Autowired
    private OutputService outputService;

    @Test
    public void testOutput_Create_Valid() {
        OutputDto outputDto = new OutputDto();
        outputDto.setName("test");
        outputDto.setOutputId(1);

        assertDoesNotThrow(() -> outputService.handleCreate(outputDto));
    }

    @Test
    public void testOutput_Create_Invalid() {
        OutputDto outputDto = new OutputDto();
        outputDto.setName(null);
        outputDto.setOutputId(-1);

        ConstraintViolationException e = assertThrows(ConstraintViolationException.class,
                () -> outputService.handleCreate(outputDto));

        Set<String> constraintViolations = e.getConstraintViolations().stream()
                .map(v -> String.format("%s: %s", v.getPropertyPath(), v.getMessage()))
                .collect(Collectors.toSet());

        assertTrue(constraintViolations.contains("handleCreate.outputDto.name: must not be null"));
        assertTrue(constraintViolations.contains("handleCreate.outputDto.outputId: must be greater than or equal to 0"));
    }

    @Test
    public void testOutput_Update_Valid() {
        OutputDto outputDto = new OutputDto();
        outputDto.setName(null);
        outputDto.setOutputId(null);

        assertDoesNotThrow(() -> outputService.handleUpdate(outputDto));
    }

    @Test
    public void testOutput_Update_Invalid() {
        OutputDto outputDto = new OutputDto();
        outputDto.setName("it's longer than 20 characters");
        outputDto.setOutputId(-1);

        ConstraintViolationException e = assertThrows(ConstraintViolationException.class,
                () -> outputService.handleCreate(outputDto));

        Set<String> constraintViolations = e.getConstraintViolations().stream()
                .map(v -> String.format("%s: %s", v.getPropertyPath(), v.getMessage()))
                .collect(Collectors.toSet());

        assertTrue(constraintViolations.contains("handleCreate.outputDto.name: size must be between 0 and 20"));
        assertTrue(constraintViolations.contains("handleCreate.outputDto.outputId: must be greater than or equal to 0"));
    }
}
```

# Кастомный валидатор
## Кастомная аннотация
Кастомный валидатор состоит из кастомной аннотации и реализации интерфейса для валидации. 

Кастомный валидатор позволяет валидировать отдельные поля бина или весь бин целиком. Для валидации всего бина кастомная аннотация ставится над классом этого бина.

Ниже приведен пример реализации кастомного валидатора, валидирующего бин целиком.

Кастомная аннотация [FacadeHasPattern](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/custom/FacadeHasPattern.java):
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Constraint(validatedBy= FacadeHasPatternValidator.class)
@Documented
public @interface FacadeHasPattern {
    String message() default "Facade fields don't have the string pattern";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String stringPattern();
}
```

Реализация кастомного валидатора [FacadeHasPatternValidator](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/custom/FacadeHasPatternValidator.java):
```java
public class FacadeHasPatternValidator implements ConstraintValidator<FacadeHasPattern, FacadeDto> {
    private String stringPattern;

    @Override
    public void initialize(FacadeHasPattern constraintAnnotation) {
        stringPattern = constraintAnnotation.stringPattern();
    }

    @Override
    public boolean isValid(FacadeDto value, ConstraintValidatorContext context) {
        if (value == null || value.getDescription() == null || value.getName() == null) {
            return false;
        }
        if (value.getDescription().contains(stringPattern) && value.getName().contains(stringPattern)) {
            return true;
        }
        return false;
    }
}
```

[FacadeDto](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/dto/FacadeDto.java):
```java
@FacadeHasPattern(stringPattern = "aaa")
@Data
public class FacadeDto {
    private String name;
    private String description;
}
```

[FacadeController](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/controller/FacadeController.java):
```java
@Slf4j
@RestController
@RequestMapping("/v1/facades")
public class FacadeController {
    @PostMapping
    public void create(@Valid @RequestBody FacadeDto facadeDto) {
        log.info("create request: {}", facadeDto);
    }
}
```

[FacadeControllerTest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/test/java/pro/jaitl/spring/examples/validation/controller/FacadeControllerTest.java):
```java
class FacadeControllerTest extends BaseApiTest {
    @Test
    public void testFacadeCreate_Valid() throws Exception {
        FacadeDto facadeDto = new FacadeDto();
        facadeDto.setDescription("test aaa");
        facadeDto.setName("test aaa");


        mockMvc.perform(doPost("/v1/facades", facadeDto))
                .andExpect(status().isOk());
    }

    @Test
    public void testFacadeCreate_Invalid() throws Exception {
        FacadeDto facadeDto = new FacadeDto();
        facadeDto.setName("test");
        facadeDto.setDescription("test");


        mockMvc.perform(doPost("/v1/facades", facadeDto))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.code", equalTo(400)))
                .andExpect(jsonPath("$.error", equalTo("facadeDto: Facade fields don't have the string pattern")));
    }
}
```
## Композиция ограничивающих аннотаций
Несколько ограничивающих аннотаций (`jakarta.validation.constraints.*`) можно объединить в одну аннотацию. Это полезно когда несколько аннотаций составляют одну логическую проверку.

Базовое поведение такое: при нарушении констрейнов будет отдан список ошибок, как будто констрейны определены на поле напрямую, без использования кастомной аннотации. В этом случае ошибка (поле `message()`) из кастомной аннотации отдаваться не будет.
Аннотация `@ReportAsSingleViolation` меняет поведение на следующее: проверка выполняется до нарушения первого констрейна, после нарушения будет отдана ошибка (поле `message()`) из кастомной аннотации.

[RusPhoneNumber](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/custom/RusPhoneNumber.java):
```java
@Pattern(regexp = "\\+7\\(\\d{3}\\)\\d{3}\\-\\d{2}\\-\\d{2}")
@Size(min = 16, max = 16)
@Constraint(validatedBy = { })
@ReportAsSingleViolation
@Documented
@Target(FIELD)
@Retention(RUNTIME)
public @interface RusPhoneNumber {
    String message() default "Wrong phone number";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

[UserDto](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/dto/UserDto.java):
```java
@Data
public class UserDto {
    @NotNull
    @Size(min = 1, max = 20)
    private String name;
    @NotNull
    @RusPhoneNumber
    private String phone;
}
```

[RusPhoneNumberTest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/test/java/pro/jaitl/spring/examples/validation/custom/RusPhoneNumberTest.java):
```java
@SpringBootTest
public class RusPhoneNumberTest {
    @Autowired
    private Validator validator;

    @Test
    public void test_Success() {
        UserDto userDto = new UserDto();
        userDto.setName("test");
        userDto.setPhone("+7(222)333-22-11");

        Set<ConstraintViolation<UserDto>> result = validator.validate(userDto);
        assertTrue(result.isEmpty());
    }

    @Test
    public void test_Failed() {
        UserDto userDto = new UserDto();
        userDto.setName("test");
        userDto.setPhone("not phone");

        List<ConstraintViolation<UserDto>> result = List.copyOf(validator.validate(userDto));
        assertEquals(1, result.size());

        ConstraintViolation<UserDto> validationResult = result.get(0);

        assertEquals("phone", validationResult.getPropertyPath().toString());
        assertEquals("Wrong phone number", validationResult.getMessage());
    }
}
```

## Аннотация @AssertTrue
Костыльнй метод валидации, которым не стоит пользоваться.
Аннотация `@AssertTrue` ставится над методом в классе, который валидирует весь бин.
Существует зеркальная аннотация - `@AssertFalse`.

Имя метода должно начинаться с `is`. Например: `isValid()`

[DoorDto](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/dto/DoorDto.java):
```java
@Data
public class DoorDto {
    private String materialType;
    private String lockType;

    @AssertTrue(message = "Unreliable door")
    public boolean isValidDoor() {
        return materialType.contains("steel") && lockType.contains("digital");
    }
}
```

[DoorController](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/main/java/pro/jaitl/spring/examples/validation/controller/DoorController.java):
```java
@Slf4j
@RestController
@RequestMapping("/v1/doors")
public class DoorController {
    @PostMapping
    public void create(@Valid @RequestBody DoorDto doorDto) {
        log.info("create request: {}", doorDto);
    }
}
```

[DoorControllerTest](https://github.com/jaitl/spring-examples/blob/main/spring-validation/src/test/java/pro/jaitl/spring/examples/validation/controller/DoorControllerTest.java):
```java
class DoorControllerTest extends BaseApiTest {
    @Test
    public void testFacadeCreate_Valid() throws Exception {
        DoorDto doorDto = new DoorDto();
        doorDto.setMaterialType("steel");
        doorDto.setLockType("digital");


        mockMvc.perform(doPost("/v1/doors", doorDto))
                .andExpect(status().isOk());
    }

    @Test
    public void testFacadeCreate_Invalid() throws Exception {
        DoorDto doorDto = new DoorDto();
        doorDto.setMaterialType("wood");
        doorDto.setLockType("key");

        mockMvc.perform(doPost("/v1/doors", doorDto))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.code", equalTo(400)))
                .andExpect(jsonPath("$.error", equalTo("validDoor: Unreliable door")));
    }
}
```

# Источники:
* [Pro Spring 5](https://www.amazon.com/Pro-Spring-Depth-Guide-Framework/dp/1484228073)
* [Validation with Spring Boot - the Complete Guide](https://reflectoring.io/bean-validation-with-spring-boot/)
* [Hibernate Validator 8.0.0.Final - Jakarta Bean Validation Reference Implementation: Reference Guide](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/?v=8.0)