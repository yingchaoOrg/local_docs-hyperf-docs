# 註解

註解是 Hyperf 非常強大的一項功能，可以通過註解的形式減少很多的配置，以及實現很多非常方便的功能。

## 概念

### 什麼是註解什麼是註釋？

在解釋註解之前我們需要先定義一下 `註解` 與 `註釋` 的區別：   
- 註釋：給程式設計師看，幫助理解程式碼，對程式碼起到解釋、說明的作用。
- 註解：給應用程式看，用於元資料的定義，單獨使用時沒有任何作用，需配合應用程式對其元資料進行利用才有作用。

### 註解解析如何實現？

Hyperf 使用了 [doctrine/annotations](https://github.com/doctrine/annotations) 包來對程式碼內的註解進行解析，註解必須寫在下面示例的標準註釋塊才能被正確解析，其它格式均不能被正確解析。
註釋塊示例：
```php
/**
 * @AnnotationClass()
 */
```
在標準註釋塊內通過書寫 `@AnnotationClass()` 這樣的語法即表明對當前註釋塊所在位置的物件(類、類方法、類屬性)進行了註解的定義， `AnnotationClass` 對應的是一個 `註解類` 的類名，可寫全類的名稱空間，亦可只寫類名，但需要在當前類 `use` 該註解類以確保能夠根據名稱空間找到正確的註解類。

### 註解是如何發揮作用的？

我們有說到註解只是元資料的定義，需配合應用程式才能發揮作用，在 Hyperf 裡，註解內的資料會被收集到 `Hyperf\Di\Annotation\AnnotationCollector` 類供應用程式使用，當然根據您的實際情況，也可以收集到您自定義的類去，隨後在這些註解本身希望發揮作用的地方對已收集的註解元資料進行讀取和利用，以達到期望的功能實現。

### 忽略某些註解

在一些情況下我們可能希望忽略某些 註解，比如我們在接入一些自動生成文件的工具時，有不少工具都是通過註解的形式去定義文件的相關結構內容的，而這些註解可能並不符合 Hyperf 的使用方式，我們可以通過在 `config/autoload/annotations.php` 內將相關注解設定為忽略。

```php
return [
    'scan' => [
        // ignore_annotations 陣列內的註解都會被註解掃描器忽略
        'ignore_annotations' => [
            'mixin',
        ],
    ],
];
```

## 使用註解

註解一共有 3 種應用物件，分別是 `類`、`類方法` 和 `類屬性`。

### 使用類註解

類註解定義是在 `class` 關鍵詞上方的註釋塊內，比如常用的 `@Controller` 和 `@AutoController` 就是類註解的使用典範，下面的程式碼示例則為一個正確使用類註解的示例，表明 `@ClassAnnotation` 註解應用於 `Foo` 類。   
```php
/**
 * @ClassAnnotation()
 */
class Foo {}
```

### 使用類方法註解

類方法註解定義是在方法上方的註釋塊內，比如常用的 `@RequestMapping` 就是類方法註解的使用典範，下面的程式碼示例則為一個正確使用類方法註解的示例，表明 `@MethodAnnotation` 註解應用於 `Foo::bar()` 方法。   
```php
class Foo
{
    /**
     * @MethodAnnotation()
     */
    public function bar()
    {
        // some code
    }
}
```

### 使用類屬性註解

類屬性註解定義是在屬性上方的註釋塊內，比如常用的 `@Value` 和 `@Inject` 就是類屬性註解的使用典範，下面的程式碼示例則為一個正確使用類屬性註解的示例，表明 `@PropertyAnnotation` 註解應用於 `Foo` 類的 `$bar` 屬性。   
```php
class Foo
{
    /**
     * @PropertyAnnotation()
     */
    private $bar;
}
```

### 註解引數傳遞

- 傳遞主要的單個引數 `@DemoAnnotation("value")`
- 傳遞字串引數 `@DemoAnnotation(key1="value1", key2="value2")`
- 傳遞陣列引數 `@DemoAnnotation(key={"value1", "value2"})`

## 自定義註解

### 建立一個註解類

在任意地方建立註解類，如下程式碼示例：    

```php
namespace App\Annotation;

use Hyperf\Di\Annotation\AbstractAnnotation;

/**
 * @Annotation
 * @Target({"METHOD","PROPERTY"})
 */
class Bar extends AbstractAnnotation
{
    // some code
}

/**
 * @Annotation
 * @Target("CLASS")
 */
class Foo extends AbstractAnnotation
{
    // some code
}
```

> 注意註解類的 `@Annotation` 和 `@Target` 註解為全域性註解，無需 `use` 

其中 `@Target` 有如下引數：   
- `METHOD` 註解允許定義在類方法上
- `PROPERTY` 註解允許定義在類屬性上
- `CLASS` 註解允許定義在類上
- `ALL` 註解允許定義在任何地方

我們注意一下在上面的示例程式碼中，註解類都繼承了 `Hyperf\Di\Annotation\AbstractAnnotation` 抽象類，對於註解類來說，這個不是必須的，但對於 Hyperf 的註解類來說，繼承 `Hyperf\Di\Annotation\AnnotationInterface` 介面類是必須的，那麼抽象類在這裡的作用是提供極簡的定義方式，該抽象類已經為您實現了`註解引數自動分配到類屬性`、`根據註解使用位置自動按照規則收集到 AnnotationCollector` 這樣非常便捷的功能。

### 自定義註解收集器

註解的收集時具體的執行流程也是在註解類內實現的，相關的方法由 `Hyperf\Di\Annotation\AnnotationInterface` 約束著，該介面類要求了下面 3 個方法的實現，您可以根據自己的需求實現對應的邏輯：

- `public function collectClass(string $className): void;` 當註解定義在類時被掃描時會觸發該方法
- `public function collectMethod(string $className, ?string $target): void;` 當註解定義在類方法時被掃描時會觸發該方法
- `public function collectProperty(string $className, ?string $target): void` 當註解定義在類屬性時被掃描時會觸發該方法

### 利用註解資料

在沒有自定義註解收集方法時，預設會將註解的元資料統一收集在 `Hyperf\Di\Annotation\AnnotationCollector` 類內，通過該類的靜態方法可以方便的獲取對應的元資料用於邏輯判斷或實現。

## IDE 註解外掛

因為 `PHP` 並不是原生支援 `註解`，所以 `IDE` 不會預設增加註解支援。但我們可以新增第三方外掛，來讓 `IDE` 支援 `註解`。

### PhpStorm

我們到 `Plugins` 中搜索 `PHP Annotations`，就可以找到對應的元件 [PHP Annotations](https://github.com/Haehnchen/idea-php-annotation-plugin)。然後安裝元件，重啟 `PhpStorm`，就可以愉快的使用註解功能了，主要提供了為註解類增加自動跳轉和程式碼提醒支援，使用註解時自動引用註解對應的名稱空間等非常便捷有用的功能。
