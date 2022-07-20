# InsideUE文章的记录
https://www.zhihu.com/column/insideue4
## UObject
在UE4中，几乎所有的类都继承自UObject，UObject为他们提供了基础的垃圾回收，反射，元数据，序列化等等。
![UObject](InsideImg/UObject.png)
## Actor
Actor继承自UObject，能在场景中放置的都是Actor，同时它也多了额外的属性，如Replication(网络复制) Spwan(生成) Tick(每帧执行的函数)
一个对象在3D世界中表示，必需要有Transform合Matrix。但是有些对象它是Actor，但是不需要表示其位置，如Ainfo(AWorldSetting AGameMode AGameSession APlayerState AGameState) AHUD APlayerCameraManager等，他们代表了世界的某种信息，状态，规则。但是他们不需要在场景中显示出来。
在UE中，万物皆是Actor。
![AActor](InsideImg/AActor.png)
UE把Transform封装进了SceneComponent，要显示位置的Actor，就有这个RootComponent，即物体加组件的模式。
所以Actor获取位置或者设置位置，都是dispatch到这个RootComponent的。
同理，要获取输入的话，就需要InputComponent。
UE也是参考了Unity的，采用了这种ECS架构，Actor只是个基础，而功能都是由component实现的。

	TSet<UActorComponent*> OwnedComponents，保存着这个Actor所拥有的所有components。一般会有一个SceneComponent作为其RootComponent。
	TArray<UActorComponent*> InstanceComponents 保存着其实例化的components，一个Actor如果想放进Level里，就必须实例化USceneComponent* RootComponent
### ActorComponent
components的类图如下。
![AActor](InsideImg/ActorComponent.jpg)
可以看到，SceneComponent有Transform，另外的AttachParent和AttachChildren告诉我们，component可以作为另一个component的父类，即components可以嵌套。
一个component可以有多个component子类。但是注意限制，只有SceneComponent才可以互相嵌套。而最上层的ActorComponent是不行的。
可以说，只有位置Transform的component才能互相嵌套。
![SceneComponentCombine](InsideImg/SceneComponetCombine.jpg)
但是也要注意SceneComponent嵌套过深的问题。
### Actor之间的父子关系
Actor之间的父子关系是通过Component确定的，UE里通过Child:AttachToActor或Child:AttachToComponent来创建父子连接的。

	void AActor::AttachToActor(AActor* ParentActor, const FAttachmentTransformRules& AttachmentRules, FName SocketName)
	{
		if (RootComponent && ParentActor)
		{
			USceneComponent* ParentDefaultAttachComponent = ParentActor->GetDefaultAttachComponent();
			if (ParentDefaultAttachComponent)
			{
				RootComponent->AttachToComponent(ParentDefaultAttachComponent, AttachmentRules, SocketName);
			}
		}
	}
	void AActor::AttachToComponent(USceneComponent* Parent, const FAttachmentTransformRules& AttachmentRules, FName SocketName)
	{
		if (RootComponent && Parent)
		{
			RootComponent->AttachToComponent(Parent, AttachmentRules, SocketName);
		}
	}
可以看到，它实际上是拿自身的RootComponent附加Attach到父节点的RootComponent上或者某个component上的。所以本质上来讲，还是SceneComponent的互相嵌套。
可以绑定到不同的component上，意味着就是有多个不同的锚点。比如mesh上的武器socket槽。那么运动的时候，父节点component的Transform就会影响到子component上。
Actor其实更像一个容器，只提供基本的创建销毁，网络复制，事件触发等一些逻辑性的功能，而父子的关系维护就委托给了具体的SceneComponent。
### ChildActorComponent
ChildActorComponent担负着Actor之间互相组合的胶水。它的ChildActorClass表示它实例化时要创建的Actor。
## Level
一个Level组织所有的Actor，多个Level组成World。
![AActor](InsideImg/ULevel.jpg)
AInfo，记录着本Level的各种规则属性。WorldSettings记录Level的规则。
## World
当一个World有多个level的时候，这些Level在什么位置，是在一开始加载进来的，还是Streaming运行时加载，每个World支持一个PersistentLevel和多个其它level。
Persistent即一开始加载进world，streaming时后续动态加载的意思，levels里保存有所有的当前已经加载的level，streaminglevels保存整个world的levels配置列表。
![AActor](InsideImg/UWorld.png)
玩家切换场景，就是在world种加载和释放不同的Level。
## WorldContext
World可以有多个，主要是编辑器一个world，编辑器里的game一个world，worldcontext负责这些world的切换，而同时也保存着Level切换的上下文。TravelUrl和TravelType就是负责设定下一个level的目标和转换过程。
level的上下文由worldcontext来保存，而不是由world。world和level的切换是在下一帧完成的。
![FWorldContext](InsideImg/FWorldContext.png)
## GameInstance
GameInstance管理着当前worldcontext和其它对象，它在游戏的运行中一直存在，所以那些独立于level的逻辑或数据，可以放在gameinstance里。
![FWorldContext](InsideImg/UGameInstance.jpg)
## Engine
它保存着所有的WorldContext，编辑器也是一个WorldContext，其实编辑器也是一个游戏，我们在游戏中开发游戏。
![FWorldContext](InsideImg/UEngine.jpg)
## GamePlayStatics
这个类是暴露给蓝图用的静态类，是一个蓝图函数库，如GetPlayerController, SpawnActor等等方法，其实它就是封装了操作world和level的能力。
Object->Actor+Component->Level->World->WorldContext->GameInstance->Engine
## Pawn
Actor可以说是由component组成的，即Actor的功能实现是由component实现的，在UE里，component表达的是功能的概念。你的component是可以随便迁移到下一个游戏中的，actor没有你的component也是可以运行的。
当你发现违背了这两点，那么你的component就是失败的。耦合度太高了，你应该重新设计。blueprintActor其实就是unity中的预制体prefab。
Pawn可以被controller控制，有3块基本的接口
1：可被controller控制 可以响应输入
2：PhysicsCollision表示 表达自身的存在，物理表示
3：MovementInput的基本接口 可以移动。
![FWorldContext](InsideImg/APwan.jpg)
pawn想表达的最关键的点是它可以被controller操纵的能力，这是它与普通actor的区别。
输入事件处理流程：
![FWorldContext](InsideImg/InputProcess.jpg)
从上面的图可以看出，响应输入的首先是actor检测是否可以接收输入，接着是PlayerController，接着是Level BluePrint，最后才是Pawn。
输入的处理功能被实现为InputComponent，输入的种类就很多了，按键 摇杆 触摸等等。
响应了按键之后，响应逻辑是在MovementInput里处理的。
### DefaultPawn
![DefaultPawn](InsideImg/ACharacter.jpg)
DefaultPawn默认带了一个DefaultPawnMovementComponent spherical CollisionComponent和StaticMeshComponent
### SpectatorPawn
这个是观战得pawn，继承自default pawn，拥有摄像机得漫游能力，拥有USpectatorPawnMovement(该组件是不带重力得漫游)。并关闭了StaticMesh显示，碰撞也设置到了Spectator通道。
### Character
Character是人形得pawn，它有个CharacterMovementComponent，表示玩家得所有移动，游泳，飞翔等等。
CapsuleComponent是一个贴近人形得胶囊。skeletalMesh是有骨骼蒙皮得mesh。并且可以IK。
当你有人形要表示，那就用character，否则就pawn就足够了。
## Controller
Pawn是可以供Controller控制得，而其控制逻辑就是放在Controller里，即业务逻辑是放在controller里得，如玩家按A键，角色自动找一个最近得敌人并攻击。这个寻找目标并攻击得逻辑过程，就是控制pawn得过程。
在游戏中，程序=数据+逻辑+显示
显示就是UI 如屏幕上得3D画面 手柄上得输入震动，VR头盔等等
数据就是Actor Mesh，Matieral，level等等
逻辑就是各种渲染算法，物理模拟，AI寻路等等。
在ue中，逻辑可以说就是我们得controller。
### AController
![AController](InsideImg/AController.jpg)
AController继承自Actor，并添加了控制Pawn得接口。Possess和UnPossess，因为继承自Actor，所以有SceneComponent作为rootcomponent，可以放置在世界中。
在UE中，pawn和controller是一对一得关系。如开车时，则需要从人物得controller切换到车辆得控制。
因为是1对1得关系，所以从pawn中提供了GetController方法，同理在controller中提供了GetPawn方法。
Controller本身有位置信息，所以可以利用该信息更好得控制pawn得位置和移动。
Controller的Rotation，因为想要让我的pawn和controller保持旋转朝向一致，因为controller作为控制pawn，所以controller需要维护自己的rotation。
再说位置，为了让pawn在respawn的时候，可以选择当前位置创建，所以controller就有了自己的位置。因此，为了自动更新controller的位置，ue提供了bAttachToPawn的开关选项，默认是关闭的，即默认不会自动更新controller的位置。如果打开，那么controller就会附加到pawn的子节点上，从而更新位置和转向。
当然，如果controller确实只是逻辑控制，如AIController的话，那位置确实不需要更新。
在上面的图中，可以看到，pawn也是能接收到玩家输入的，同时controller也是可以的，并且controller算是第一个接收到输入的。
从概念上，pawn是能动的物体，重点在于动。controller则是动到哪里，重点的是决策，如方向等等。所以，pawn本身固有的能力逻辑，如前进后退，播放动画，碰撞检测之类的，应该放在pawn内实现，而对于一些可替换的逻辑，或者只能决策的，应该放在controller实现。
从对应上来说，如果某个逻辑只属于某类pawn，那可以在pawn内实现(如坦克开炮)，而如果在多个pawn中，如自动寻找攻击目标(如战车和坦克)，那么应该放在controller中。
从存在周期来讲，pawn销毁了就没了，而controller是一直存在的，所以一些逻辑和状态要放在controller里。
### APlayerState
上面说到一些状态可以放在controller里，但是ue实现了APlayerState这个类，来存放玩家状态。controller只作为逻辑的实现。APlayerState继承自AInfo，而AInfo继承自AACtor，因为PlayerState也需要进行网络复制的功能，所以就继承了AInfo。
![AController](InsideImg/AplayerState.png)
AInfo是不爱表现，纯数据的大本营，继承自Actor，有网络复制等功能。
controller和网络的结合很紧密。可以理解成controller也可以当作玩家在服务器上的代理对象，当玩家偶尔掉线的时候，因为连接不在，所以controller失效被释放了，服务器可以把对应的该PlayerState暂存起来，等玩家再重连上的时候，可以利用该PlayerState重新挂载上controller。有一个顺畅的体验。AIcontroller运行再server上，client上并没有。
应当注意的是，当关卡切换的时候，APlayerState也会被释放掉。所以跨关卡数据不应该放进来。playerstate只表示玩家的游玩数据。而关卡数据应该放在GameState中。
Component->Actor->Pawn->Controller的结构。
![AController](InsideImg/AController_Pawn.jpg)


