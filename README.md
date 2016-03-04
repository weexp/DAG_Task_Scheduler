# DAG_Task_Scheduler
A Java library for defining tasks that have directed acyclic dependencies and executing them with various scheduling algorithms. 

This is supposed to be a library that will allow a developer to quickly define executable tasks, define the dependencies between tasks. The library takes care of passing arguments between the tasks.

The library also gives a way to easily define a resource that is supposed to be shared between multiple threads, providing the user with a mechanism to lock, modify and unlock the resource (See example 3)

## Current state
1. Only one scheduling algorithm is implemented, selects the first of available tasks
2. Performance of the scheduler has not been taken into consideration

## Examples
### Example 1: 
A sample workflow in an application server
* **Task CheckUserCredentials** - go to DB and check if credentials are ok
* **Task PrepareTemplate** - read some file that contains a web site template
* **Task DisplayResult** - if credentials are ok, put username in template, else put error
 
Dependencies DAG
```
CheckUserCredentials---→DisplayResult
PrepareTemplate--------↗
```
```java
public class CheckCredentialsForUser extends Executable {

    public CheckCredentialsForUser(String id) {
        super(id);
    }

    @Override
    public void execute() {
        List<? extends Object> input = this.get(LogInUserSchedule.Username);
        if (input == null || input.size() != 1) {
            error("wrong input");
        }

        //Simulate checking of credentials in database
        Thread.sleep(1000);
        produce(LogInUserSchedule.CheckCredentialsResult, true);
        produce(LogInUserSchedule.Username, input.get(0));
    }
}

public class PrepareTemplate extends Executable {
    public PrepareTemplate(String id) {
        super(id);
    }

    @Override
    public void execute() {
        //Simulate workload of reading file
        Thread.sleep(500);
        produce(LogInUserSchedule.Template, "<html>something something {insert_result_here}</html>");
    }
}

public class DisplayResult extends Executable {
    public DisplayResult(String id) {
        super(id);
    }

    @Override
    public void execute() {
        List<? extends Object> credOk = get(LogInUserSchedule.CheckCredentialsResult);
        if (true == (Boolean) credOk.get(0)) {
            String username = (String) get(LogInUserSchedule.Username).get(0);
            String template = (String) get(LogInUserSchedule.Template).get(0);
            produce(LogInUserSchedule.Result, template.replaceAll("\\{insert_result_here\\}", username));
        } else {
            String template = (String) get(LogInUserSchedule.Template).get(0);
            produce(LogInUserSchedule.Result, template.replaceAll("\\{insert_result_here\\}", "Wrong credentials!!!"));
        }
    }
}

public class LogInUserSchedule extends Schedule {
    public static final String CheckCredentialsResult = "credentials";
    public static final String Template = "template";
    public static final String Username = "username";
    public static final String Result = "result";

    public LogInUserSchedule(String userName) {
        Executable cred = new CheckCredentialsForUser("check credentials")
                .addInput(Username, userName);
        Executable prep = new PrepareTemplate("prepare template");
        this.add(cred)
            .add(prep)
            .add(new DisplayResult("display result"), cred, prep);
    }
}
```
### Example 2: 
Task of type Square = takes a list of integers, squares them

Task of type Sum = takes a list of integers, reduces them by addition
* **Task Square1** - take numbers from 1 to 5, square them
* **Task Square2** - take numbers from 6 to 10, square them
* **Task Sum** - take results from Square1 and Square2, apply reduction by addition and produce result
 
Dependencies DAG
```
Square1---→Sum
Square2---↗
```
```java
public class SquareTheInputExecutable extends Executable {
    public SquareTheInputExecutable(String id) {
        super(id);
    }

    @Override
    public void execute() {
        List<? extends Object> a = this.get(SampleSchedule.Input_Square);
        a.stream().forEach(k -> {
            int aa = ((Integer) k).intValue();
            aa = aa * aa;
            produce(SampleSchedule.Result_Square, aa);
        });
    }
}

public class SumTheInputExecutable extends Executable {
    public SumTheInputExecutable(String id) {
        super(id);
    }

    @Override
    public void execute() {
        List<? extends Object> inputParams = this.get(SampleSchedule.Result_Square);
        int result = inputParams.stream()
            .map(element -> ((Integer) element).intValue())
            .reduce(0, (k, l) -> k.intValue() + l.intValue());
        produce(SampleSchedule.Final_Result, result);
    }
}

public class SampleSchedule extends Schedule {
    public static final String Input_Square = "input_square";
    public static final String Result_Square = "result_square";
    public static final String Final_Result = "final_result";

    public SampleSchedule(List<Integer> input1, List<Integer> input2) {
        Executable sq1 = new SquareTheInputExecutable("Square1").addInput(Input_Square, input1);
        Executable sq2 = new SquareTheInputExecutable("Square2").addInput(Input_Square, input2);
        this.add(sq1)
            .add(sq2)
            .add(new SumTheInputExecutable("Sum"), sq1, sq2);
    }
}

public static void main(String... args){
    Scheduler s = new Scheduler(new DummySchedulingAlgorithm());
    Schedule schedule = new SampleSchedule(Arrays.asList(1, 2, 3, 4, 5), Arrays.asList(6, 7, 8, 9, 10));
    s.execute(schedule);
    System.out.println("results:" + schedule.getResults());
}
```
### Example - 3 Using a shared resource
Two parallel tasks use the same id generator
```java

class A extends Executable {
    private final IdGenerator generator;

    protected A(IdGenerator generator, String id) {
        super(id);
        this.generator = generator;
    }

    @Override
    public void execute() {
        try {
            int newId = generator.generate();
            produce(ScheduleWithSharedResource.Result, String.format("%s: id %d", Thread.currentThread().getName(), newId));
        } catch (IllegalAccessException | InterruptedException e) {
            if (e instanceof IllegalAccessException) {
                error("resource not locked");
            } else {
                error("thread interrupted while requesting lock");
            }
        }
    }
}

class IdGenerator {
    SharedResource<Integer> serial = new SharedResource<>(0);

    public int generate() throws IllegalAccessException, InterruptedException {
        SharedResource<Integer>.ResourceOperation<Integer> getter = serial.createOperation(Function.identity());
        serial.lock();
        int newId = getter.getResult();
        serial.set(newId + 1).unlock();
        return newId;
    }
}

public class ScheduleWithSharedResource extends Schedule {
    public static final String Result = "result";

    public ScheduleWithSharedResource() {
        IdGenerator g = new IdGenerator();
        this.add(new A(g, "A1")).add(new A(g, "A2"));
    }
}

public static void main(String... args) throws InterruptedException {
    Scheduler s = new Scheduler(new DummySchedulingAlgorithm());
    Schedule schedule = new ScheduleWithSharedResource();
    s.execute(schedule);
}
```
### Example - 4 Using different scheduling algorithms
