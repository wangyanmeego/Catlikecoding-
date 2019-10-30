# 03 - Reusing Objects(对象池)

这是关于对象管理的系列教程中的第三篇。它增加了破坏形状的能力，并提供了重用它们的方法。
本教程是用Unity 2017.4.4f1制作的。

![一个循环利用多种形状的机会](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/tutorial-image.jpg)

# 销毁对象

如果我们只能创造形状，那么他们的数量只能增加，直到我们开始一个新的游戏。但几乎总是当游戏中创造了某些东西时，它也可能被销毁。所以让我们有可能销毁形状。

## 销毁对象的按键

已经有一个创建形状的键，所以添加一个键来销毁一个形状是有意义的。为游戏添加一个关键变量。虽然D看起来是一个合理的默认值，但它是通用WASD移动键配置的一部分。让我们使用X代替它，它是取消或终止的常见符号，在大多数键盘上与C相邻。

```cs
public KeyCode createKey = KeyCode.C;
	public KeyCode destroyKey = KeyCode.X;
```

##  销毁一个随机形状

在Game中加入一个`DestroyShape`的方法来处理形态的销毁。就像我们创造随机形状一样，我们也销毁随机形状。这是通过为形状列表选择一个随机索引并使用Destroy方法销毁相应的对象来实现的。

```cs
void DestroyShape () {
		int index = Random.Range(0, shapes.Count);
		Destroy(shapes[index]);
	}
```

但这仅在当前存在形状时有效。可能不是这样，因为还没有创建或加载，或者所有现有的形状都已经被销毁了。因此，我们只能在列表至少包含一个形状时销毁该形状。否则，destroy命令将什么也不做。

```cs
void DestroyShape () {
		if (shapes.Count > 0) {
			int index = Random.Range(0, shapes.Count);
			Destroy(shapes[index]);
		}
	}
```

Destroy可以用于游戏对象、组件或资产。为了摆脱整个形状对象而不仅仅是它的Shape组件，我们必须明确地摧毁游戏对象，该组件是游戏对象的一部分。我们可以通过组件的gameObject属性来访问它。

```cs
Destroy(shapes[index].gameObject);
```

现在我们的DestroyShape方法是有效的，当玩家按下destroy键时，在Update中调用它。

```cs
void Update () {
		if (Input.GetKeyDown(createKey)) {
			CreateShape();
		}
		else if (Input.GetKeyDown(destroyKey)) {
			DestroyShape();
		}
		…
	}
```

## 保持List的正确

我们现在可以创建和销毁对象。然而，当尝试破坏多个形状时，您可能会得到一个错误。类型'Shape'的对象已被销毁，但您仍试图访问它。
发生此错误的原因是，虽然我们已经销毁了形状，但尚未将其从形状列表中删除。因此，列表仍然包含对被破坏游戏对象组件的引用。它们仍然以类僵尸的状态存在于内存中。当试图销毁这样一个对象第二次，Unity报告一个错误。
解决方案是适当地去掉对我们刚刚销毁的形状的引用。所以在销毁一个形状之后，把它从列表中删除。这可以通过调用列表的RemoveAt方法来实现，将元素的索引作为参数删除。

```cs
void DestroyShape () {
		if (shapes.Count > 0) {
			int index = Random.Range(0, shapes.Count);
			Destroy(shapes[index].gameObject);
			shapes.RemoveAt(index);
		}
	}
```

##  高效地删除

虽然这种方法有效，但它不是从列表中删除元素的最有效方法。因为列表是有序的，删除一个元素会在列表中留下一个空白。从概念上讲，这种差距很容易消除。被移除元素的相邻元素彼此成为邻居。

![从概念上删除元素D](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/destroying-objects/list-conceptual.png)

但是，List类是用数组实现的，因此不能直接操作相邻关系。相反，通过将下一个元素移动到这个间隙中来消除间隙，因此它直接在被删除的元素之前到达。这使得差距向列表的末尾移动了一步。重复这个过程，直到间隙从列表的末尾消失。

![缓慢移动，保持秩序](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/destroying-objects/list-with-order.png)

但是我们不关心我们追踪的图形的顺序。因此，所有这些元素的转移都是不必要的。虽然我们不能在技术上避免它，但我们可以通过手动获取最后一个元素并将其放在被破坏的元素的位置上，从而有效地将空白传送到列表的末尾，从而跳过几乎所有的工作。然后移除最后一个元素。

![快速去除，不保留秩序](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/destroying-objects/list-without-order.png)

```cs
void DestroyShape () {
		if (shapes.Count > 0) {
			int index = Random.Range(0, shapes.Count);
			Destroy(shapes[index].gameObject);
			int lastIndex = shapes.Count - 1;
			shapes[index] = shapes[lastIndex];
			shapes.RemoveAt(lastIndex);
		}
	}
```

# 持续的创建和销毁

一次创建和销毁一个形状并不是填充或减少游戏对象的快速方法。如果我们想要不断地创造和摧毁它们呢?我们可以一遍又一遍地快速按下按键，但很快就会厌倦。让我们自动化它。
应该以什么速度创建形状?我们把它变成可配置的。这次我们不打算通过检视面板来控制它。相反，我们将使它成为游戏本身的一部分，所以玩家可以根据自己的喜好改变速度。

## GUI

为了控制创建速度，我们将在场景中添加一个图形用户界面(gui)。GUI需要一个Canvas，可以通过GameObject / UI / canvas创建。这将向场景添加两个新的游戏对象。首先是Canvas本身，然后是一个EventSystem，使得与它交互成为可能。

![Canvas和EventSystem](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/canvas-event-system.png)

这两个对象都有多个组件，但是我们不需要关心它们的细节。我们可以在不改变任何东西的情况下使用它们。默认情况下，画布作为一个覆盖层，呈现在游戏窗口的顶部，在屏幕空间中。
虽然屏幕空间画布在逻辑上并不存在于3D空间中，但它仍然显示在场景窗口中。这允许我们编辑它，但是当场景窗口处于3D模式时，这很难做到。GUI没有与场景摄像机对齐，它的比例是每像素一个单位，因此它最终就像场景中的一个巨大的平面。在编辑GUI时，通常会将场景窗口切换到2D模式，可以通过工具栏左侧的2D按钮进行切换。

![2D模式的场景窗口](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/scene-window-2d.png)

## 创建速度标签

在添加创建速度控制之前，我们将添加一个标签告诉玩家它是关于什么的。我们通过添加一个文本对象，通过GameObject / UI / text，命名为创建速度标签。它自动成为画布的一个子元素。事实上，如果我们没有画布，在创建text对象时就会自动创建一个。

![创建速度的标签](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/creation-speed-label-hierarchy.png)

GUI对象的功能与所有其他游戏对象一样，只是它们有一个Rect Transform，扩展了常规的Transform组件。它不仅控制对象的位置、旋转和比例，还控制对象的矩形大小、轴心点和锚点

锚点（Anchor）控制GUI对象相对于其父容器的定位方式，以及它如何对父容器的大小变化做出反应。让我们把标签放在游戏窗口的左上角。不管我们最终得到的窗口大小如何，都要保持它不变，将它的锚点设置为左上角。您可以通过单击锚点方框并选择弹出的适当选项来实现这一点。还可以将显示的文本更改为Creation Speed。

![锚点设置为左上角](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/creation-speed-label-inspector.png)

将标签放置在画布的左上角，在它和游戏窗口之间留一点空白。

![位于画布的左上角](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/creation-speed-label-scene.png)

## 创建速度滑动条

我们将使用滑块来控制创建速度。通过GameObject / UI / Slider添加一个。这将创建多个对象的层次结构，这些对象共同构成一个GUI滑块小部件。命名它的本地根对象为_Creation Speed Slider_。

![Slider hierarchy for creation speed.](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/creation-speed-slider-hierarchy.png)

将滑块直接置于标签下方。默认情况下，它们具有相同的宽度，并且标签在文本下面有大量的空白空间。你可以将滑块向上拖动到标签的底边，它会与标签相邻。

![定位滑动条](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/creation-speed-slider-scene.png)

滑动条的本地根对象的slider组件有一堆设置，我们将保留它们的默认值。我们唯一要更改的是它的Max值，它定义了最大创建速度，以每秒创建的形状表示。设为10。

![最大值设置为10](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/creation-speed-slider-inspector.png)

## 设置创建速度

滑动条已经工作了，你可以在运行模式下调整它。但它还没有影响到任何东西。我们必须先在游戏中加入创造速度，所以有些东西需要改变。我们会给它一个默认的公共CreationSpeed属性。

```cs
public float CreationSpeed { get; set; }
```

滑动条的检视面板在其底部有一个_On Value Changed (Single)_框。这表示在滑动条的值更改后调用的方法或属性列表。在_On Value Changed_ i后面的(single)值表示被更改的值是一个浮点数。当前列表是空的。通过单击框底部的+按钮来更改它。

![Slider without connection.](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/creation-speed-slider-not-connected.png)

事件列表现在包含一个条目。它有三个配置选项。第一个设置控制何时激活该条目。它默认设置为运行时，这是我们想要的。在它下面是一个设置游戏对象的字段。将游戏对象的引用拖放到上面。这允许我们选择附加到目标对象的组件的方法或属性。现在我们可以使用第三个下拉列表，选择Game，然后在顶部的CreationSpeed，在动态浮动标题下。

![连接到属性的滑块](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/creation-speed-slider-connected.png)

## 持续创建形状

为了使持续的创建成为可能，我们必须跟踪创建的进程。为此添加一个浮动字段到游戏中。当该值达到1时，应该创建一个新形状。

```cs
float creationProgress;
```


通过添加自上一帧以来的时间，可以通过time . deltatime增加更新进度。进度的快慢是由时间增量乘以创建速度来控制的。

```cs
void Update () {
		…

		creationProgress += Time.deltaTime * CreationSpeed;
	}
```

每当creationProgress达到1时，我们必须将其重置为0并创建一个形状。

```cs
creationProgress += Time.deltaTime * CreationSpeed;
		if (creationProgress == 1f) {
			creationProgress = 0f;
			CreateShape();
		}
```

但是，我们不太可能得到正好为1的进展值。相反，我们会多做一些。所以我们应该检查是否至少有1个。然后我们减少1的进度，节省额外的进度。所以时间并不精确，但我们不会放弃额外的进度。

```cs
creationProgress += Time.deltaTime * CreationSpeed;
		if (creationProgress >= 1f) {
			creationProgress -= 1f;
			CreateShape();
		}
```

但是，自上一帧以来，可能已经取得了很大的进度，最后得到的值可能是2、3甚至更多。这可能发生在帧速率下降期间，同时还有很高的创建速度。为了确保尽可能快地赶上进度，将if语句更改为while语句。

```cs
creationProgress += Time.deltaTime * CreationSpeed;
		while (creationProgress >= 1f) {
			creationProgress -= 1f;
			CreateShape();
		}
```

你现在可以让游戏创建一个规则的新形状流，在一个理想的速度下高达每秒10个形状。如果您想关闭自动创建过程，只需将滑块设置回零。

## 连续地销毁形状

接下来，重复我们对创建滑块所做的所有工作，现在是对销毁滑动条所做的工作。创建另一个标签和滑块，通过复制现有的标签和滑块，将它们向下移动，并重新命名它们，这是最快的方法。

![创建和销毁滑动条](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/both-sliders.png)

然后添加一个DestructionSpeed属性，并将销毁滑动条连接到它。如果你复制了创建滑块，你只需要改变它的目标属性。

```cs
	public float DestructionSpeed { get; set; }
```


![连接到属性的销毁滑动条](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/continuous-creation-and-destruction/destruction-speed-slider-connected.png)

最后，添加跟踪销毁过程的代码。

```cs
float creationProgress, destructionProgress;

	…

	void Update () {
		…

		creationProgress += Time.deltaTime * CreationSpeed;
		while (creationProgress >= 1f) {
			creationProgress -= 1f;
			CreateShape();
		}

		destructionProgress += Time.deltaTime * DestructionSpeed;
		while (destructionProgress >= 1f) {
			destructionProgress -= 1f;
			DestroyShape();
		}
	}
```

游戏现在可以同时自动创建和销毁形状。如果两者的速度相同，形状的数量大致保持不变。为了让创建和销毁以一种令人愉快的方式同步，您可以稍微调整其中一个的速度，直到它们的进度对齐或交替。

# 对象池

每次实例化一个对象时，都必须分配内存。每当一个对象被销毁时，它使用的内存就会被回收。但回收不会马上进行。有一个垃圾收集过程，它偶尔运行来清理所有东西。这是一个代价高昂的过程，因为它必须根据任何对象是否仍然持有对它的引用，来确定哪些对象实际上不再有效地活着。因此，所使用的内存的数量会增长一段时间，直到它被认为是很多，然后不可到达的内存被识别出来并再次可用。如果涉及大量内存块，这可能会导致游戏中的帧率显著下降。
虽然重用底层内存很困难，但是在更高级别上重用对象要容易得多。如果我们永远不会破坏游戏对象，而是回收它们，那么垃圾收集过程就永远不需要运行。

## Profiling

为了了解有多少内存分配发生，以及什么时候发生，你可以使用Unity的profiler窗口，你可以通过window / profiler打开它。它可以在运行模式下记录大量信息，包括CPU和内存的使用情况。
在积累了一些形态之后，让游戏以最大的创造和毁灭速度运行一段时间。然后在分析器的数据图上选择一个点，它将暂停游戏。当选择CPU部分时，所选帧的所有顶层调用都显示在图的下方。您可以通过内存分配(如GC Alloc列中所示)对调用进行排序。
在大多数帧中，总分配为零。但是，当在该帧中实例化一个形状时，您将在顶部看到一个分配内存的条目。您可以展开该条目以查看Game.[Update](http://docs.unity3d.com/Documentation/ScriptReference/MonoBehaviour.Update.html)负责实例化的更新。

![用于创建形状的概要数据](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/object-pools/allocation-editor.png)
在运行之间，编辑器中分配的字节数可能会有所不同。游戏没有像独立版本那样优化，编辑器本身也会影响配置。通过创建standalone Build，并将其自动连接到编辑器以进行分析，可以获得更好的数据。

![使用概要分析的开发构建的构建设置](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/object-pools/build-settings.png)

创建build，运行一段时间，然后在编辑器中检查分析器数据

![分析独立构建版本](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/object-pools/allocation-standalone.png)

这个概要数据不受编辑器的影响，尽管我们仍然在使用一个必须收集和发送概要数据的开发构建版本。

## 回收

因为我们的图形是简单的游戏对象，所以它们不需要太多的内存。尽管如此，新实例化的恒定创建过程最终将触发垃圾收集过程。为了防止这种情况，我们必须重复使用形状，而不是销毁它们。所以每次游戏会销毁一个形状时，我们应该把它们送回工厂循环利用。
回收形状是可行的，因为它们在使用时不会改变太多。它们得到一个随机的变换，材料和颜色。如果做了更复杂的调整，比如添加或删除组件，或添加子对象，那么回收就不可行了。为了支持这两种情况，让我们向ShapeFactory添加一个切换来控制它是否回收。对于我们当前的游戏来说，回收是可行的，因此在检视面板激活它。

```cs
[SerializeField]
	bool recycle;
```


![带有回收功能的Factory](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/object-pools/recycle.png)

## 把形状对象池化

当一个形状被回收，我们把它放在一个备用池。然后，当需要一个新形状时，我们可以从这个池中获取一个现有的形状，而不是默认创建一个新形状。只有当池为空时，我们才需要实例化一个新形状。我们需要一个单独的池为每个形状类型的工厂可以生成，所以给它一个数组的形状列表。

```cs
using System.Collections.Generic;
using UnityEngine;

[CreateAssetMenu]
public class ShapeFactory : ScriptableObject {

	…

	List<Shape>[] pools;

	…
}
```

添加一个创建池的方法，只需为prefabs数组中的每个条目添加一个空列表。

```cs
void CreatePools () {
		pools = new List<Shape>[prefabs.Length];
		for (int i = 0; i < pools.Length; i++) {
			pools[i] = new List<Shape>();
		}
	}
```

在Get方法开始时，检查是否启用了回收。如果有，检查池是否存在。如果没有，那么此时创建池。

```cs
public Shape Get (int shapeId = 0, int materialId = 0) {
		if (recycle) {
			if (pools == null) {
				CreatePools();
			}
		}
		Shape instance = Instantiate(prefabs[shapeId]);
		instance.ShapeId = shapeId;
		instance.SetMaterial(materials[materialId], materialId);
		return instance;
	}
```

##  从池中检索对象

实例化形状并设置其ID的现有代码现在应该只在不进行回收时使用。否则，应该从池中检索实例。为此，必须在决定如何获取实例之前声明实例变量。

```cs
Shape instance;
		if (recycle) {
			if (pools == null) {
				CreatePools();
			}
		}
		else {
			instance = Instantiate(prefabs[shapeId]);
			instance.ShapeId = shapeId;
		}
		
		instance.SetMaterial(materials[materialId], materialId);
```

当启用回收时，我们必须从正确的池中提取实例。我们可以使用形状ID作为池索引。然后从池中获取一个元素并激活它。这是通过调用其游戏对象上的SetActive方法来实现的，true作为参数。然后把它从池中取出。因为我们不关心池中元素的顺序，我们可以只获取最后一个元素，这是最有效的。

```cs
Shape instance;
		if (recycle) {
			if (pools == null) {
				CreatePools();
			}
			List<Shape> pool = pools[shapeId];
			int lastIndex = pool.Count - 1;
			instance = pool[lastIndex];
			instance.gameObject.SetActive(true);
			pool.RemoveAt(lastIndex);
		}
		else {
			instance = Instantiate(prefabs[shapeId]);
		}
```

但这只有在池中有东西时才有可能，所以检查一下。

```cs
List<Shape> pool = pools[shapeId];
			int lastIndex = pool.Count - 1;
			if (lastIndex >= 0) {
				instance = pool[lastIndex];
				pool.RemoveAt(lastIndex);
			}
```

如果没有，我们别无选择，只能创建一个新的shape实例。

```cs
if (lastIndex >= 0) {
				instance = pool[lastIndex];
				pool.RemoveAt(lastIndex);
			}
			else {
				instance = Instantiate(prefabs[shapeId]);
				instance.ShapeId = shapeId;
			}
```

##  回收一个对象

为了利用池，工厂必须有一种方法来回收不再需要的形状。这是通过添加一个带有形状参数的公共`Reclaim`方法来实现的。此方法还应该首先检查是否启用了回收，如果启用，则在执行任何其他操作之前确保池是存在的

```cs
public void Reclaim (Shape shapeToRecycle) {
		if (recycle) {
			if (pools == null) {
				CreatePools();
			}
		}
	}
```

现在我们确定了池的存在，通过使用其形状ID作为池索引，可以将回收的形状添加到正确的池中。

```cs
public void Reclaim (Shape shapeToRecycle) {
		if (recycle) {
			if (pools == null) {
				CreatePools();
			}
			pools[shapeToRecycle.ShapeId].Add(shapeToRecycle);
		}
	}
```

此外，回收的形状必须被停用，现在代表的意思是销毁。

```cs
pools[shapeToRecycle.ShapeId].Add(shapeToRecycle);
			shapeToRecycle.gameObject.SetActive(false);
```

但是，如果不允许回收利用，那么就应该将其销毁。

```cs
		if (recycle) {
			…
		}
		else {
			Destroy(shapeToRecycle.gameObject);
		}
```

## 回收而不是销毁

工厂不能强制将形状返回给它。这取决于Game，使回收成为可能，通过调用回收，而不是摧毁摧毁的形状。

```cs
void DestroyShape () {
		if (shapes.Count > 0) {
			int index = Random.Range(0, shapes.Count);
			//Destroy(shapes[index].gameObject);
			shapeFactory.Reclaim(shapes[index]);
			int lastIndex = shapes.Count - 1;
			shapes[index] = shapes[lastIndex];
			shapes.RemoveAt(lastIndex);
		}
	}
```

当你开始一个新游戏的时候的一样。

```cs
void BeginNewGame () {
		for (int i = 0; i < shapes.Count; i++) {
			//Destroy(shapes[i].gameObject);
			shapeFactory.Reclaim(shapes[i]);
		}
		shapes.Clear();
	}
```

确保Game运行得很好，并且在归还后不会销毁形状。这会导致错误。所以这不是一种万无一失的技术，程序员必须表现出良好的行为。只有从工厂获得的形状应该返回给它，而不需要进行重大更改。并且有可能破坏这些形状，但这将使回收成为不可能。

## 启用了回收

尽管不管是否启用了回收，游戏的玩法都是一样的，但是您可以通过观察hierarchy窗口来看到区别。当创建和销毁以相同的速度发生时，您将看到形状将变成活动和不活动状态，而不是被创建和销毁。游戏对象的总数将在一段时间后变得稳定。只有当特定形状类型的池为空时，才会创建新实例。游戏运行的时间越长，这种情况发生的频率就越低，除非创建速度高于销毁速度。

![活动和非活动对象的混合](https://catlikecoding.com/unity/tutorials/object-management/reusing-objects/object-pools/active-inactive-objects.png)

您还可以使用profiler来验证内存分配发生的频率要低得多。它们还没有被完全消除，因为有时还需要创建新的形状。此外，有时在回收对象时分配内存。这种情况的发生有两个原因。首先，有时需要增加池列表。第二，要停用一个对象，我们必须访问gameObject属性。当属性第一次检索到游戏对象的引用时，会分配一点内存。所以这种情况只在每种形状第一次被循环利用的时候才会发生。
