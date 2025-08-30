
### Future 是什么？

`Future` 接口在 Java 5 中引入，代表了异步计算的结果。当您将一个任务（通常是 `Callable` 或 `Runnable`）提交给 `ExecutorService` 后，它会立即返回一个 `Future` 对象。这个 `Future` 对象是一个占位符，它承诺在将来的某个时间点提供异步操作的最终结果。

**`Future` 的主要特点和局限性：**

*   **异步执行：** 允许在后台线程中执行耗时操作，避免阻塞主线程。
*   **结果检索：** 提供了 `get()` 方法来获取计算结果。然而，`get()` 方法是**阻塞的**，意味着调用线程会一直等待，直到异步任务完成并返回结果。
*   **任务状态：** 提供了 `isDone()` 方法来检查任务是否完成，以及 `isCancelled()` 来检查任务是否被取消。
*   **取消操作：** 提供了 `cancel()` 方法来尝试取消任务的执行。
*   **缺乏回调机制：** `Future` 不允许直接附加回调函数，即在任务完成后自动执行某些操作。您必须阻塞线程并手动处理结果。
*   **难以组合和链式操作：** 很难将多个 `Future` 操作串联起来，或者组合它们的 T 结果。
*   **异常处理笨拙：** `Future` 没有内置的异常处理机制。如果任务失败，您需要在调用 `get()` 时手动捕获 `ExecutionException`。

### CompletableFuture 是什么？

`CompletableFuture` 是在 Java 8 中引入的一个类，它实现了 `Future` 接口和 `CompletionStage` 接口。它代表了异步计算的未来结果，并且是 `Future` 的一个强大扩展。`CompletableFuture` 提供了一套丰富的 API，用于组合、链接和执行异步计算步骤，以及处理可能发生的错误。

**`CompletableFuture` 的主要特点：**

*   **非阻塞和响应式：** `CompletableFuture` 鼓励非阻塞编程。它允许您通过方法如 `thenApply()`、`thenAccept()` 和 `thenRun()` 定义当计算完成时应该发生什么，而无需阻塞主线程。
*   **支持回调和链式操作：** 这是 `CompletableFuture` 最强大的功能之一。它允许您将多个异步操作组合成一个链式流程，一个操作的完成可以触发下一个操作的执行。这种设计模式在函数式编程中很常见，被称为单子设计模式。
*   **灵活的异常处理：** 提供了 `exceptionally()` 和 `handle()` 等方法来更优雅地处理异步计算中的异常，允许您定义回退动作或恢复机制。
*   **手动完成：** 可以通过 `complete()` 方法手动完成 `CompletableFuture`，并设置其结果。
*   **丰富的组合方法：** 提供了 `thenCombine()`、`thenCompose()`、`allOf()`、`anyOf()` 等多种方法来组合多个 `CompletableFuture`，以应对复杂的异步工作流。

### CompletableFuture 与 Future 的区别

| 特性           | Future                                             | CompletableFuture                                          |
| :------------- | :------------------------------------------------- | :--------------------------------------------------------- |
| **阻塞 vs. 非阻塞** | `get()` 方法是阻塞的，会暂停当前线程直到结果可用。 | 鼓励非阻塞编程，通过回调方法处理结果，不会阻塞调用线程。 |
| **回调支持**   | 不支持回调，需要手动 `get()` 结果。           | 全面支持回调，可以通过 `thenApply()`、`thenAccept()` 等方法附加任务。 |
| **链式/组合操作** | 难以组合多个异步操作或对结果进行链式处理。  | 提供了强大的 API (`thenCompose()`, `thenCombine()`, `allOf()` 等) 来轻松组合和链式操作多个异步任务。 |
| **异常处理**   | 没有内置的异常处理机制，需要在 `get()` 调用时捕获 `ExecutionException`。 | 提供了 `exceptionally()` 和 `handle()` 等方法来优雅地处理异常。 |
| **完成方式**   | 只能由提交给 `ExecutorService` 的任务隐式完成。 | 可以手动完成 (`complete()` / `completeExceptionally()`) 或通过异步任务完成。 |
| **功能**       | 提供基本的状态检查、等待和结果获取功能。       | 功能更强大、更灵活，适用于复杂的异步工作流。       |
| **引入版本**   | Java 5。                                     | Java 8。                                          |
| **使用场景**   | 适用于简单的异步操作，只需在另一个线程中执行任务并阻塞获取结果。 | 适用于复杂的异步工作流，需要非阻塞行为、错误处理和任务组合的场景。 |

### CompletableFuture 常用的两个方法及区别

#### 1. `runAsync()` 与 `supplyAsync()`

这两个是创建 `CompletableFuture` 的静态工厂方法，用于启动一个异步任务。它们的主要区别在于所执行的任务是否会返回结果。

*   **`CompletableFuture.runAsync(Runnable runnable)`**
    *   **作用：** 执行一个没有返回值的异步计算。
    *   **参数：** 接受一个 `Runnable` 函数式接口作为参数。`Runnable` 表示一个执行动作但不产生结果的任务。
    *   **返回值：** 返回一个 `CompletableFuture<Void>`，表示任务的完成状态，但不包含任何结果值。
    *   **使用场景：** 适用于那些只关心任务是否完成，而不关心具体返回值的后台操作，例如记录日志、发送通知等。
    *   **示例：**
        ```java
        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
            System.out.println("这是一个没有返回值的异步任务。");
            // 模拟耗时操作
            try { Thread.sleep(1000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            System.out.println("异步任务执行完毕。");
        });
        // 可以使用 thenRun, thenAccept 等进行链式操作，但无法获取返回值
        future.thenRun(() -> System.out.println("任务完成后执行这个动作。"));
        ```

*   **`CompletableFuture.supplyAsync(Supplier<T> supplier)`**
    *   **作用：** 执行一个有返回值的异步计算。
    *   **参数：** 接受一个 `Supplier<T>` 函数式接口作为参数。`Supplier` 表示一个提供结果但不需要输入参数的任务。
    *   **返回值：** 返回一个 `CompletableFuture<T>`，其中 `T` 是 `Supplier` 提供结果的类型。
    *   **使用场景：** 适用于需要从异步任务中获取并进一步处理结果的场景，例如从数据库加载数据、调用远程服务等。
    *   **示例：**
        ```java
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("这是一个有返回值的异步任务。");
            // 模拟耗时操作并返回结果
            try { Thread.sleep(2000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            return "Hello from supplyAsync!";
        });
        // 可以使用 thenApply, thenAccept 等进行链式操作并获取返回值
        future.thenAccept(result -> System.out.println("获取到的结果是: " + result));
        ```

**总结：** `runAsync()` 关注执行动作，不关心结果；`supplyAsync()` 关注执行动作并返回结果。

#### 2. `thenApply()` 与 `thenCompose()`

这两个方法都用于在 `CompletableFuture` 完成后执行后续操作并返回一个新的 `CompletableFuture`，但它们处理返回类型的方式不同，尤其是在处理嵌套 `CompletableFuture` 时。

*   **`thenApply(Function<T, R> fn)`**
    *   **作用：** 当当前的 `CompletableFuture` 正常完成时，使用其结果作为输入，执行一个**同步的**转换函数 `fn`，并返回一个新的 `CompletableFuture`，其中包含转换后的结果。
    *   **参数：** 接受一个 `Function` 函数式接口。该 `Function` 会接收上一个 `CompletableFuture` 的结果 `T`，并返回一个 `R` 类型的值.
    *   **返回值：** 返回 `CompletableFuture<R>`。如果 `fn` 返回的是另一个 `CompletableFuture`，那么 `thenApply` 会将其包装成一个嵌套的 `CompletableFuture`，例如 `CompletableFuture<CompletableFuture<R>>`。
    *   **使用场景：** 当您想要**同步地**转换上一个 `CompletableFuture` 的结果时使用。
    *   **类比：** 类似于 Java Stream API 中的 `map()` 操作。
    *   **示例：**
        ```java
        CompletableFuture<Integer> initialFuture = CompletableFuture.supplyAsync(() -> 10);

        CompletableFuture<String> resultFuture = initialFuture.thenApply(result -> {
            // 同步地将 Integer 转换为 String
            return "Number: " + result;
        });

        // 如果 thenApply 返回的是 CompletableFuture，会得到嵌套的 Future
        // CompletableFuture<CompletableFuture<String>> nestedFuture = initialFuture.thenApply(result -> CompletableFuture.supplyAsync(() -> "Nested Number: " + result));
        ```

*   **`thenCompose(Function<T, CompletableFuture<R>> fn)`**
    *   **作用：** 当当前的 `CompletableFuture` 正常完成时，使用其结果作为输入，执行一个**异步的**转换函数 `fn`（该函数本身返回一个 `CompletableFuture`），然后将这个内部的 `CompletableFuture` 扁平化，返回一个单一的 `CompletableFuture`。
    *   **参数：** 接受一个 `Function` 函数式接口。该 `Function` 会接收上一个 `CompletableFuture` 的结果 `T`，并返回一个 `CompletableFuture<R>`。
    *   **返回值：** 返回 `CompletableFuture<R>`。它会避免创建嵌套的 `CompletableFuture`，使得链式调用更加简洁。
    *   **使用场景：** 当您需要将两个**依赖的异步任务**串联起来，其中第二个任务的启动依赖于第一个任务的结果时使用。它用于解决 `thenApply` 可能导致的 `CompletableFuture<CompletableFuture<R>>` 这种“回调地狱”问题。
    *   **类比：** 类似于 Java Stream API 中的 `flatMap()` 操作。
    *   **示例：**
        ```java
        CompletableFuture<Integer> initialFuture = CompletableFuture.supplyAsync(() -> 10);

        CompletableFuture<String> composedFuture = initialFuture.thenCompose(result -> {
            // 异步地进行下一个操作，返回一个新的 CompletableFuture
            return CompletableFuture.supplyAsync(() -> "Composed Number: " + result);
        });
        // composedFuture 的类型是 CompletableFuture<String>，而不是 CompletableFuture<CompletableFuture<String>>
        ```

**总结：**
*   `thenApply()` 用于**同步地**转换 `CompletableFuture` 的结果，返回一个包含转换后结果的新 `CompletableFuture`。如果转换函数返回 `CompletableFuture`，则会导致嵌套 `Future`。
*   `thenCompose()` 用于**链式连接**依赖的异步任务，其中前一个任务的结果用于启动后一个任务。它会扁平化 `CompletableFuture` 嵌套，保持链的扁平结构。

简单来说，如果你的转换函数返回的不是 `CompletableFuture`，或者你不需要处理内部的 `CompletableFuture`，使用 `thenApply()`。如果你的转换函数会返回另一个 `CompletableFuture`，并且你希望避免嵌套，让异步任务的链条保持扁平，那么应该使用 `thenCompose()`。