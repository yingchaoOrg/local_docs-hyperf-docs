# 模型

模型组件衍生于 [Eloquent ORM](https://laravel.com/docs/5.8/eloquent)，相关操作均可参考 Eloquent ORM 的文档。

## 创建模型

Hyperf 提供了创建模型的命令，您可以很方便的根据数据表创建对应模型。命令通过 `AST` 生成模型，所以当您增加了某些方法后，也可以使用脚本方便的重置模型。

```
$ php bin/hyperf.php db:model table_name
```

创建的模型如下
```php
<?php

declare(strict_types=1);

namespace App\Models;

use Hyperf\DbConnection\Model\Model;

/**
 * @property $id
 * @property $name
 * @property $gender
 * @property $created_at
 * @property $updated_at
 */
class User extends Model
{
    /**
     * The table associated with the model.
     *
     * @var string
     */
    protected $table = 'user';

    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected $fillable = ['id', 'name', 'gender', 'created_at', 'updated_at'];

    /**
     * The attributes that should be cast to native types.
     *
     * @var array
     */
    protected $casts = ['id' => 'integer', 'gender' => 'integer'];
}
```

## 模型参数

|    参数    |  类型  | 默认值  |         备注         |
|:----------:|:------:|:-------:|:--------------------:|
| connection | string | default |      数据库连接      |
|   table    | string |   无    |      数据表名称      |
| primaryKey | string |   id    |       模型主键       |
|  keyType   | string |   int   |       主键类型       |
|  fillable  | array  |   []    | 允许被批量复制的属性 |
|   casts    | string |   无    |    数据格式化配置    |
| timestamps |  bool  |  true   |  是否自动维护时间戳  |

### 数据表名称

如果我们没有指定模型对应的 table，它将使用类的复数形式「蛇形命名」来作为表名。因此，在这种情况下，Hyperf 将假设 User 模型存储的是 users 数据表中的数据。你可以通过在模型上定义 table 属性来指定自定义数据表：

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Hyperf\DbConnection\Model\Model;

class User extends Model
{
    protected $table = 'user';
}
```

### 主键

Hyperf 会假设每个数据表都有一个名为 id 的主键列。你可以定义一个受保护的 $primaryKey 属性来重写约定。

此外，Hyperf 假设主键是一个自增的整数值，这意味着默认情况下主键会自动转换为 int 类型。如果您希望使用非递增或非数字的主键则需要设置公共的 $incrementing 属性设置为 false。如果你的主键不是一个整数，你需要将模型上受保护的 $keyType 属性设置为 string。

### 时间戳

默认情况下，Hyperf 预期你的数据表中存在 `created_at` 和 `updated_at` 。如果你不想让 Hyperf 自动管理这两个列， 请将模型中的 `$timestamps` 属性设置为 `false`：

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Hyperf\DbConnection\Model\Model;

class User extends Model
{
    public $timestamps = false;
}
```

如果需要自定义时间戳的格式，在你的模型中设置 `$dateFormat` 属性。这个属性决定日期属性在数据库的存储方式，以及模型序列化为数组或者 JSON 的格式：

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Hyperf\DbConnection\Model\Model;

class User extends Model
{
    protected $dateFormat = 'U';
}
```

如果您需要不希望保持 `datetime` 格式的储存，或者希望对时间做进一步的处理，您可以通过在模型内重写 `fromDateTime($value)` 方法实现。   

如果你需要自定义存储时间戳的字段名，可以在模型中设置 `CREATED_AT` 和 `UPDATED_AT` 常量的值来实现，其中一个为 `null`，则表明不希望 ORM 处理该字段：

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Hyperf\DbConnection\Model\Model;

class User extends Model
{
    const CREATED_AT = 'creation_date';

    const UPDATED_AT = 'last_update';
}
```

### 数据库连接

默认情况下，Hyperf 模型将使用你的应用程序配置的默认数据库连接 `default`。如果你想为模型指定一个不同的连接，设置 `$connection` 属性：当然，`connection-name` 作为 `key`，必须在 `databases.php` 配置文件中存在。

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Hyperf\DbConnection\Model\Model;

class User extends Model
{
    protected $connection = 'connection-name';
}
```

### 默认属性值

如果要为模型的某些属性定义默认值，可以在模型上定义 `$attributes` 属性：

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Hyperf\DbConnection\Model\Model;

class User extends Model
{
    protected $attributes = [
        'delayed' => false,
    ];
}
```

## 模型查询

```php
use App\Models\User;

/** @var User $user */
$user = User::query()->where('id', 1)->first();
$user->name = 'Hyperf';
$user->save();

```

### 重新加载模型

你可以使用 `fresh` 和 `refresh` 方法重新加载模型。 `fresh` 方法会重新从数据库中检索模型。现有的模型实例不受影响：

```php
use App\Models\User;

/** @var User $user */
$user = User::query()->find(1);

$freshUser = $user->fresh();
```

`refresh` 方法使用数据库中的新数据重新赋值现有模型。此外，已经加载的关系会被重新加载：

```php
use App\Models\User;

/** @var User $user */
$user = User::query()->where('name','Hyperf')->first();

$user->name = 'Hyperf2';

$user->refresh();

echo $user->name; // Hyperf
```

### 集合

对于模型中的 `all` 和 `get` 方法可以查询多个结果，返回一个 `Hyperf\Database\Model\Collection` 实例。 `Collection` 类提供了很多辅助函数来处理查询结果：

```php
$users = $users->reject(function ($user) {
    // 排除所有已删除的用户
    return $user->deleted;
});
```

### 检索单个模型

除了从指定的数据表检索所有记录外，你可以使用 `find` 或 `first` 方法来检索单条记录。这些方法返回单个模型实例，而不是返回模型集合：

```php
use App\Models\User;

$user = User::query()->where('id', 1)->first();

$user = User::query()->find(1);
```

### 检索多个模型

当然 `find` 的方法不止支持单个模型。

```php
use App\Models\User;

$users = User::query()->find([1, 2, 3]);
```

### 聚合函数

你还可以使用 查询构造器 提供的 `count`，`sum`, `max`, 和其他的聚合函数。这些方法只会返回适当的标量值而不是一个模型实例：

```php
use App\Models\User;

$count = User::query()->where('gender', 1)->count();
```

## 插入&更新模型

### 插入

要往数据库新增一条记录，先创建新模型实例，给实例设置属性，然后调用 `save` 方法：

```php
use App\Models\User;

/** @var User $user */
$user = new User();

$user->name = 'Hi Hyperf';

$user->save();
```

在这个示例中，我们赋值给了 `App\Models\User` 模型实例的 `name` 属性。当调用 `save` 方法时，将会插入一条新记录。 `created_at` 和 `updated_at` 时间戳将会自动设置，不需要手动赋值。

### 更新

`save` 方法也可以用来更新数据库已经存在的模型。更新模型，你需要先检索出来，设置要更新的属性，然后调用 `save` 方法。同样， `updated_at` 时间戳会自动更新，所以也不需要手动赋值：

```php
use App\Models\User;

/** @var User $user */
$user = User::query()->find(1);

$user->name = 'Hi Hyperf';

$user->save();
```

### 批量更新

也可以更新匹配查询条件的多个模型。在这个示例中，所有的 `gender` 为1的用户，修改 `gender_show` 为 男性：

```php
use App\Models\User;

User::query()->where('gender', 1)->update(['gender_show' => '男性']);
```

> 批量更新时， 更新的模型不会触发 saved 和 updated 事件。因为在批量更新时，从不会去检索模型。

### 批量赋值

你也可以使用 `create` 方法来保存新模型，此方法会返回模型实例。不过，在使用之前，你需要在模型上指定 `fillable` 或 `guarded` 属性，因为所有的模型都默认不可进行批量赋值。

当用户通过 HTTP 请求传入一个意外的参数，并且该参数更改了数据库中你不需要更改的字段时。比如：恶意用户可能会通过 HTTP 请求传入 `is_admin` 参数，然后将其传给 `create` 方法，此操作能让用户将自己升级成管理员。

所以，在开始之前，你应该定义好模型上的哪些属性是可以被批量赋值的。你可以通过模型上的 `$fillable` 属性来实现。 例如：让 `User` 模型的 `name` 属性可以被批量赋值：

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Hyperf\DbConnection\Model\Model;

class User extends Model
{
    protected $fillable = ['name'];
}
```

一旦我们设置好了可以批量赋值的属性，就可以通过 `create` 方法插入新数据到数据库中了。 `create` 方法将返回保存的模型实例：

```php
use App\Models\User;

$user = User::create(['name' => 'Hyperf']);
```

如果你已经有一个模型实例，你可以传递一个数组给 fill 方法来赋值：

```php
$user->fill(['name' => 'Hyperf']);
```

### 保护属性

`$fillable` 可以看作批量赋值的「白名单」, 你也可以使用 `$guarded` 属性来实现。 `$guarded` 属性包含的是不允许批量赋值的数组。也就是说， `$guarded` 从功能上将更像是一个「黑名单」。注意：你只能使用 `$fillable` 或 `$guarded` 二者中的一个，不可同时使用。下面这个例子中，除了 `gender_show` 属性，其他的属性都可以批量赋值：

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Hyperf\DbConnection\Model\Model;

class User extends Model
{
    protected $guarded = ['gender_show'];
}
```

### 删除模型

可以在模型实例上调用 `delete` 方法来删除实例：

```php
use App\Models\User;

$user = User::query()->find(1);

$user->delete();
```

### 通过查询删除模型

您可通过在查询上调用 `delete` 方法来删除模型数据，在这个例子中，我们将删除所有 `gender` 为 `1` 的用户。与批量更新一样，批量删除不会为删除的模型启动任何模型事件：

```php
use App\Models\User;

// 注意使用 delete 方法时必须建立在某些查询条件基础之上才能安全删除数据，不存在 where 条件，会导致删除整个数据表
User::query()->where('gender', 1)->delete(); 
```

### 通过主键直接删除数据

在上面的例子中，在调用 `delete` 之前需要先去数据库中查找对应的模型。事实上，如果你知道了模型的主键，您可以直接通过 `destroy` 静态方法来删除模型数据，而不用先去数据库中查找。 `destroy` 方法除了接受单个主键作为参数之外，还接受多个主键，或者使用数组，集合来保存多个主键：

```php
use App\Models\User;

User::destroy(1);

User::destroy([1,2,3]);
```

### 软删除

除了真实删除数据库记录，`Hyperf` 也可以「软删除」模型。软删除的模型并不是真的从数据库中删除了。事实上，是在模型上设置了 `deleted_at` 属性并将其值写入数据库。如果 `deleted_at` 值非空，代表这个模型已被软删除。如果要开启模型软删除功能，你需要在模型上使用 `Hyperf\Database\Model\SoftDeletes` trait

> `SoftDeletes` trait 会自动将 `deleted_at` 属性转换成 `DateTime / Carbon` 实例

```php
<?php

namespace App;

use Hyperf\Database\Model\Model;
use Hyperf\Database\Model\SoftDeletes;

class Flight extends Model
{
    use SoftDeletes;
}

```
