能力(Capability)系统
===================

能力(Capability)使特性更动态和灵活地展现(Expose)，而不必直接实现很多接口(Interface)。

总的来说，每个能力提供了一个接口形式的特性，一个可被调用的默认实现，和一个至少对于默认实现的储存处理器(Storage Handler)。储存管理器可以支持其它的实现，但是这个是取决于能力的实现者，所以你应该先看一下它们的文档之后再试着对非默认实现使用默认储存。

Forge对TileEntity、实体、和ItemStack添加了能力支持。它们可以通过事件附加上去，也可以通过在你的实现中重写能力的方法来展现。这将在后面的小节中进行更详细地解释。

Forge提供的能力
--------------

在本文写作时，Forge只提供了一种能力：IItemHandler。

这一能力展现了一个处理背包格子的接口。它可以被应用到TileEntity(箱子，机器)、实体(玩家更多的背包格子，生物的背包)、或ItemStack(便携背包等)。它通过一个自动化友好的系统代替了以前的 `IInventory` 和 `ISidedInventory`。

使用现有能力
-----------

正如之前提到的那样，TileEntity、实体、和ItemStack通过 `ICapabilityProvider` 实现了能力Provider特性。这一接口添加了两个方法：`hasCapability` 和 `getCapability`，它们可以用来获取对象中存在的能力。

要想获取能力，你首先要用其唯一的实例(Instance)引用它。在这里是Item Handler，这个能力一般储存在 `CapabilityItemHandler.ITEM_HANDLER_CAPABILITY`，但是我们也可以用 `@CapabilityInject` 这个注解(Annotation)获取其它的实例。

```java
@CapabilityInject(IItemHandler.class)
static Capability<IItemHandler> ITEM_HANDLER_CAPABILITY = null;
```

这个注解可以被应用在字段(Field)和方法(Method)上。当应用到字段上的时候，它会在能力注册时将能力的实例(同一个实例赋值到所有字段中)赋值到字段上，如果该能力没有被注册，它将保持现有值(null)。因为本地静态字段访问会更快，所以您最好保留一份能力对象的引用。这个注解也可以同样应用到方法中，应用的方法将在能力注册时被调用，以便某些特性的应用。

`hasCapability` 和 `getCapability` 这两个方法都有第二个参数，类型为 `EnumFacing`，他们可以用来对特定面请求特定的实例。如果传入的是 `null`，那么就可认定请求是从方块内或者从一个面无意义的地方来的，比如说另一个维度(Dimension)。这时候一个不考虑面的通用能力实例将会被请求。`getCapability` 的返回类型将会对应能力声明时的类型。对于Item Handler这个能力，它是 `IItemHandler`。

展现一个能力
-----------

要想展现(Expose)一个能力，你需要先获得一个潜在能力类型的实例。注意你需要对每个存有能力的对象赋值不同的实例，因为能力很可能会和包含的对象联系起来。

有两种方式能获得实例，第一种是通过 `Capability` 本身，第二种是显式实例化一个它的实现。第一种方法是为默认实现设计的，如果那些默认的值对你很有用的话。Item Handler这个能力的默认实现只会展现一个单格背包(Inventory)，可能不是你想要的结果。

第二种方法可以用自定义的实现。在 `IItemHandler` 这个例子中，默认实现使用的是 `ItemStackHandler` 类，它有一个可选的带参数的构造器，参数为格子的个数。然而，我们不应认定默认实现是一直存在的，因为能力系统设计初衷就是为了防止能力不存在时的加载错误。所以实例化之前我们需要检查能力是否被注册了(见上面关于 `@CapabilityInject` 的内容)

一旦你有了自己的能力接口实例，你会希望在你展现能力的时候通知能力系统的用户。这个可以通过重写 `hasCapability` 方法进行实现，并将实例和你展示的能力进行对比。如果你的机器根据面的不同有不同数量的格子，你可以通过 `facing` 参数进行判定。对于实体和ItemStack，这个参数可以忽略掉，但是你仍然可以自己对面进行定义以使用这一参数，比如说玩家不同的装备格(顶面 => 头部格子?)，或者背包中四周的格子(西面 => 左边的格子?)。不要忘记回溯到 `super` 中，否则附加的能力将会停止工作。

```java
@Override
public boolean hasCapability(Capability<?> capability, EnumFacing facing) {
  if (capability == CapabilityItemHandler.ITEM_HANDLER_CAPABILITY) {
    return true;
  }
  return super.hasCapability(capability, facing);
}
```

同样，当接收到请求的时候，你需要提供能力实例的接口引用。一样，不要忘记回溯到 `super`。

```java
@Override
public <T> T getCapability(Capability<T> capability, EnumFacing facing) {
  if (capability == CapabilityItemHandler.ITEM_HANDLER_CAPABILITY) {
    return (T) inventory;
  }
  return super.getCapability(capability, facing);
}
```

我们强烈建议您使用直接检查来判定能力而不是尝试依靠于地图和其他数据结构，因为能力判定每个tick可能对多个对象进行，它们需要以最快速度完成以避免游戏卡顿。

附加能力
-------

之前说过，对实体和ItemStack附加能力可以通过 `AttachCapabilityEvent` 来完成。这个事件包括三个更细的子事件：

- `AttachCapabilityEvent.Entity`: 仅对实体触发
- `AttachCapabilityEvent.TileEntity`: 仅对TileEntity触发
- `AttachCapabilityEvent.Item`: 仅对ItemStack触发

每个事件都有一个方法 `addCapacity`，它可以用来附加能力到目标对象上。你在能力列表中添加的是能力Provider，而不是能力本身，它可以从特定面返回相应的能力。Provider只需要实现 `ICapabilityProvider`，如果能力需要持久储存数据，你需要实现 `ICapabilitySerializable<T extends NBTBase>`，它不仅能返回能力，还能提供NBT存储与读取函数。

要想获得实现 `ICapabilityProvider` 的更多信息，请看上面的展现能力小节。