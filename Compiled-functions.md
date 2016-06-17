compiled repository的数据处理流程可以做到数据类型无关的(除了封装的`Result`哈，ps:就是说泛型呗)。 在实践中, 很常见的list数据处理(比如 RecyclerView中)。尤其像下面这种典型的从网络下载list数据的处理流:

1. 下载数据:byte[]类型;
2. 将byte[]解析成数据对象;
3. 从数据对象中读取内容;
4. 数据给UI层(比如 Adapter)展示前，对数据可以做一些转换工作：对list进行and、or、sort等任何集合筛选操作；
5. 将结果集(list)设置为Adapter的数据源。

开发者也许想把前4步封装成一个函数`Function`，这个叫做：a compiled repository，然后用一个 `Updatable` 作为repository的响应实现第5步, 为UI提供list数据。如果大多数子程序都单独编写UI模型转换方法，势必影响代码可读性 (比如：将领域模型数据转换成UI模型数据)。

Agera 为compiled repository提供了实用工具：编译规模较小，可重用操作符，相同风格:

```java
// For type clarity only, the following are smaller, reused operators:
Function<String, DataBlob> urlToBlob = …;
Function<DataBlob, List<ItemBlob>> blobToItemBlobs = …;
Predicate<ItemBlob> activeOnly = …;
Function<ItemBlob, UiModel> itemBlobToUiModel = …;
Function<List<UiModel>, List<UiModel>> sortByDateDesc = …;
    
Function<String, List<UiModel>> urlToUiModels =
    Functions.functionFrom(String.class)
        .apply(urlToBlob)
        .unpack(blobToItemBlobs)
        .filter(activeOnly)
        .map(itemBlobToUiModel)
        .morph(sortByDateDesc)
        .thenLimit(5);
```

所谓_reused_是指 operator背后的逻辑部分在别的地方需要使用到, 并且只需要一小点工作就可以包装成有用的operator接口。此外，如果超过`Function`/`Predicate`的函数定义(ps:相对复杂的函数)，更要使用function compiler, 要比简单写一个自定义的`Function`好的多，已经发生的开销 (编译时间：编译附加的类的时间；运行时时间：加载、创建、链接这些类的时间)。使用function compiler可以有效减少代码行。

function编译器对应的编译器状态，定义在`FunctionCompilerStates` 接口中。 就像使用 [[repository compiler|Compiled-repositories]], 编译function的表达式是不能中间中断的.