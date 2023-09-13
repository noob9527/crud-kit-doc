# cascade API (experimental)

## "声明时" VS "使用时"
日常 CRUD 工作流中，常常有一些＂级联＂操作，举例来说，假如我们有 Employee 与 EmployeeContact 两张表，分别用于保存雇员与其联系方式。我们希望在保存 Employee 时，自动更新其对应的联系方式。熟悉 JPA 的人可能会见过如下 API：
```kotlin
@Entity
class Employee{
    @Column
    var name: String = ""

    @OneToMany(cascade = [CascadeType.ALL], orphanRemoval = true, mappedBy = "employee")
    var contacts: MutableList<EmployeeContact> = mutableListOf()
}
```
简单来说，这里 `CascadeType.ALL` 意味着 "级联新增", "级联更新" 和 "级联删除"。我们分别对这些概念做一个简单的介绍.
- 级联新增
  保存 Employee 时，会自动保存所有 contacts 集合中的 EmployeeContact
- 级联更新
  更新 Employee 时，会自动更新所有 contacts 集合中的 EmployeeContact
- 级联删除
  删除 Employee 时，会自动删除所有 contacts 集合中的 EmployeeContact
- orphanRemoval
  更新 Employee 时，会自动删除数据库中该 Employee 关联的，却不存在于 contacts 集合中的 EmployeeContact
> 由于 JPA 中有 Object State 概念，因此实际 `CascadeType.ALL` 表达的含义会更加复杂一些。

以我个人的使用经验来看，我并不喜欢这个 API。因为它需要开发者在定义时，就指明需要怎样的级联关系，导致没办法灵活根据实际场景来控制是否需要级联。举例来说，假设我们 Employee 管理系统同时存在 WEB, MOBILE 两个客户端，其中 WEB 客户端在同一个页面维护 Employee 和 EmployeeContact。也就是说它在修改 Employee 时发起如下请求
```bash
curl --location --request PUT 'host/employees/1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "John Snow",
    "contacts": [
        "type": "EMAIL",
        "contact": "john.snow@gmail.com",
    ]
}'
```
在这个场景中，我们需要级联操作。而假设在 MOBILE 客户端中，我们使用不同的页面分别维护 Employee 和 EmployeeContact，换句话说，在修改 Employee 和 EmployeeContact 时，我们分别调用不同的接口。在修改 Employee 时，发起如下请求
```bash
curl --location --request PUT 'host/employees/1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "John Snow",
}'
```
注意请求内容不包含 contacts，所以我们不再需要级联操作。但由于我们已经在声明时指明了级联，这将导致不熟悉 JPA 的用户可能会犯下致命的错误。考虑下面的更新接口实现：
```kotlin
@PutMapping("/{id}")
fun update(
        @PathVariable id: Int,
        @RequestBody newEmployee: Employee
): Employee {
    val existedEmployee = entityManager.find(id, Employee::class.java)
    BeanUtil.copy(newEmployee, existedEmployee)
    existedEmployee.id = id
    entityManager.merge(existedEmployee)
    entityManager.flush()
    return existedEmployee
}
```
在我看来，上面的代码思路并没有什么问题，由于 JPA 的更新机制，我们需要先查询已存在的 employee(我们暂且称之为 existedEmployee)。接下来，我们将新的 employee 属性拷贝至已存在的 employee（只有 existedEmployee 是 managed 状态），之后完成更新。如果我没有记错的话，上面的代码会直接报错，因为 BeanUtil.copy 可能会执行如下操作
```kotlin
existedEmployee.id = newEmployee.id
existedEmployee.name = newEmployee.name
existedEmployee.contacts = newEmployee.contacts
```
问题就出在 `existedEmployee.contacts = newEmployee.contacts`，JPA 不允许直接修改一个 orphanRemoval 集合的引用。针对这个问题，一些程序员可以会推荐使用如下的方式来定义 orphanRemoval 集合：
```kotlin
@OneToMany(cascade = [CascadeType.ALL], orphanRemoval = true, mappedBy = "employee")
var contacts: MutableList<EmployeeContact> = mutableListOf()
        set(value) {
            field.clear()
            field.addAll(value)
        }
```
如果恰巧你的类也是这样定义的，**那么情况会更加糟糕。上面的代码不会报错，但每次有人从 MOBILE 端调用更新 Employee 接口，就会自动删除该 employee 的所有的联系方式。（因为关联的 contacts 被清空了）** \
由于我已经很长时间没用过 JPA，我不保证上面这个例子跟我描述的执行结果一致。这里我只是想表达我的观点，即：如果我们需要提供 cascade API，我个人更倾向于在调用保存/更新/删除方法时，指定需要 cascade 的属性，而不是在定义 Entity 时指定。

## Cascade in crud-kit
现在的问题是，我们如何修改现有的 API（如下），使其支持级联操作呢？
```kotlin
fun save(entity: T): T
fun save(entities: List<T>): List<T>

fun update(entity: T): T
fun update(entities: List<T>): List<T>

fun patch(patch: Patch<T>)
fun patch(patches: List<Patch<T>>)
fun patchThenGet(patch: Patch<T>, include: Set<String> = emptySet()): T

fun delete(id: ID)
fun delete(entity: T)
```
我的想法是，新增如下 API
```kotlin
fun save(entity: T, cascades: Set<Cascade<T, *>>): T
fun update(entity: T, cascades: Set<Cascade<T, *>>): T
fun patch(patch: Patch<T>, cascades: Set<Cascade<T, *>>)
fun patchThenGet(patch: Patch<T>, cascades: Set<Cascade<T, *>>, include: Set<String> = emptySet()): T
fun delete(id: ID, cascades: Set<Cascade<T, *>>)
fun delete(entity: T, cascades: Set<Cascade<T, *>>)
```
> 之所以新增重载方法，而不是在原有方法上添加 `cascades: Set<Cascade<T, *>> = emptySet()` 默认参数，是因为在我们目前的使用场景中，经常需要重写 `save(entity: T)`, `update(entity: T)` 等方法，目前我不想增加重写方法的负担。

回顾上面的例子，如果我们不希望级联更新，则继续使用老的 API。如果需要级联更新 `Employee.contacts` 则按照如下方式调用
```kotlin
Employee.update(employee, setOf(
    Cascade.oneToMany(Employee::contacts)
))
```
> 对比 JPA，这样调用的一个好处是，我们可以明显的看出来，这次调用会级联的修改该 employee 对应的 contacts。

在上面的调用中，我们会自动使用 "join" 机制查询数据库中该 `employee` 所关联的 `contacts`（这里我们暂时称为"oldContacts"），然后根据 id 对比传入的 `employee.contacts` （这里我们称为"newContacts"），根据对比结果，删除不需要的 "oldContacts"(对应 JPA 的 orphanRemoval)，保存或更新对应的 "newContacts"。\
这里的 `Cascade.oneToMany` 只是其中一个工厂方法，目前完整的工厂方法列表如下：
```kotlin
interface Cascade<T : Entity<*>, P : Entity<*>> {
    /**
     * This works the same way as Cascade.oneToMany
     * except that methods in this class has fewer parameters.
     * because when you perform "cascade save" instead of "cascade update", you don't need those param.
     */
    object Save {
        fun <T : Entity<*>, P : Entity<*>> oneToOne(
                property: KProperty1<T, P?>,
                callback: Callback<P, T> = Callback.noOp(),
                nestedCascades: Set<Cascade<P, *>> = emptySet(),
        ): Cascade<T, P> {
            return SimpleCascade(
                    property,
                    Action.SAVE.toSet(),
                    patchFields = null,
                    callback = callback,
                    nestedCascades = nestedCascades,
            )
        }

        fun <T : Entity<*>, P : Entity<*>> oneToMany(
                property: KProperty1<T, Collection<P>?>,
                callback: Callback<P, T> = Callback.noOp(),
                nestedCascades: Set<Cascade<P, *>> = emptySet(),
        ): Cascade<T, P> {
            return SimpleCascade(
                    property,
                    Action.SAVE.toSet(),
                    patchFields = null,
                    callback = callback,
                    nestedCascades = nestedCascades,
            )
        }
    }

    object Delete {
        fun <T : Entity<*>, P : Entity<*>> oneToOne(
                property: KProperty1<T, P?>,
                callback: Callback<P, T> = Callback.noOp(),
                nestedCascades: Set<Cascade<P, *>> = emptySet(),
        ): Cascade<T, P> {
            return SimpleCascade(
                    property,
                    Action.DELETE.toSet(),
                    patchFields = null,
                    callback = callback,
                    nestedCascades = nestedCascades,
            )
        }

        fun <T : Entity<*>, P : Entity<*>> oneToMany(
                property: KProperty1<T, Collection<P>?>,
                callback: Callback<P, T> = Callback.noOp(),
                nestedCascades: Set<Cascade<P, *>> = emptySet(),
        ): Cascade<T, P> {
            return SimpleCascade(
                    property,
                    Action.DELETE.toSet(),
                    patchFields = null,
                    callback = callback,
                    nestedCascades = nestedCascades,
            )
        }
    }

    companion object {
        fun <T : Entity<*>, P : Entity<*>> oneToOne(
                property: KProperty1<T, P?>,
                actions: Set<Action> = Action.SAVE_UPDATE,
                patchFields: Set<String>? = null,
                callback: Callback<P, T> = Callback.noOp(),
                nestedCascades: Set<Cascade<P, *>> = emptySet(),
        ): Cascade<T, P> {
            return SimpleCascade(
                    property,
                    actions,
                    patchFields,
                    callback,
                    nestedCascades,
            )
        }

        fun <T : Entity<*>, P : Entity<*>> oneToMany(
                property: KProperty1<T, Collection<P>?>,
                actions: Set<Action> = Action.ALL,
                patchFields: Set<String>? = null,
                callback: Callback<P, T> = Callback.noOp(),
                nestedCascades: Set<Cascade<P, *>> = emptySet(),
        ): Cascade<T, P> {
            return SimpleCascade(
                    property,
                    actions,
                    patchFields,
                    callback,
                    nestedCascades,
            )
        }
    }
}
```
下面分别说明各参数的用法。

### actions 参数
对于新增(`save`)接口，可能的级联操作只有级联新增。但对于更新接口，我们可能会根据情况对“级联”的对象执行新增，更新与删除操作。有时，我们可能希望只进行一部分操作。比如说，我们可能只想新增或删除 contacts，但不允许更新 contacts，我们可以指明 actions 参数，调用如下:
```kotlin
Employee.update(employee, setOf(
    Cascade.oneToMany(
        Employee::contacts, 
        actions = setOf(Cascade.Action.Save, Cascade.Action.DELETE)
    )
))
```
下面是 `Action` 类的定义
```kotlin
enum class Action {
    SAVE,
    PATCH,

    /**
     * acts just like orphanRemoval in JPA
     */
    DELETE,
    ;

    fun toSet(): Set<Action> {
        return setOf(this)
    }

    companion object {
        val SAVE_PATCH = setOf(
                SAVE,
                PATCH,
        )
        val SAVE_DELETE = setOf(
                SAVE,
                DELETE,
        )
        val ALL = values().toSet()
    }
}
```

### patchFields 参数
由于我们在方法内部使用 `patch` 而不是 `update` 来执行级联更新操作（理由参见[丢失更新问题与 patch API](articles/lost-update-and-patch-api.md)) **，我们可以通过 `patchFields` 来指定具体允许修改的字段，比如我们可以用下面的方式限时级联更新时，只允许修改 `EmployeeContact` 的 `email` 属性
```kotlin
Employee.update(employee, setOf(
    Cascade.oneToMany(Employee::contacts, patchFields = setOf(EmployeeContact.email.name))
))
```

### callback 参数
有时，我们需要在执行具体的添加，修改动作前执行一些逻辑，这时我们可以使用 callback 参数
```kotlin
Employee.update(employee, setOf(
    Cascade.oneToMany(Employee::contacts, callback = object: Cascade.Callback<EmployeeContact, Employee> {
            override fun preSave(entity: EmployeeContact, parent: Employee) {
                println("preSave: $entity")
            }

            override fun preUpdate(entity: EmployeeContact, parent: Employee) {
                println("preSave: $entity")
            }

            override fun preDelete(entity: EmployeeContact, parent: Employee) {
                println("preSave: $entity")
            }
    })
))
```
`Callback` 接口也有对应的工厂方法定义如下：
```kotlin
interface Callback<A : Entity<*>, B: Entity<*>> {
    companion object {
        fun <A : Entity<*>, B : Entity<*>> noOp(): Callback<A, B> {
            return object : Callback<A, B> {}
        }

        fun <A : Entity<*>, B : Entity<*>> prePersist(callback: (entity: A, parent: B) -> Unit): Callback<A, B> {
            return object : Callback<A, B> {
                override fun preSave(entity: A, parent: B) {
                    callback.invoke(entity, parent)
                }

                override fun preUpdate(entity: A, parent: B) {
                    callback.invoke(entity, parent)
                }
            }
        }
    }

    fun preSave(entity: A, parent: B) {
    }

    fun preUpdate(entity: A, parent: B) {
    }

    fun preDelete(entity: A, parent: B) {
    }

    fun postSave(entity: A, parent: B) {
    }

    fun postUpdate(entity: A, parent: B) {
    }

    fun postDelete(entity: A, parent: B) {
    }
}
```

### nestedCascades 参数
TBD

## Conclusion
由于 cascade API 目前还处于实验阶段，未来上述 API 可能会有修改，但大致的思路应该不会变。
