---
title: Android StateMachine分析
date: 2018/4/22
comments: true
categories: architecture
tags: architecture

---

# Android StateMachine分析

## 状态模式

### 简介
- 例子：按钮控制电梯状态：电梯有开门、关门、运行、停止、上升、下降、选择楼层等状态。每种状态改变，都有可能根据其他状态来更新处理，如在上升中不能开门。在实际编码中，考虑遍历完各种情况状态，需要引入大量的if...else if 或者 switch case，对可读性、维护性、扩展性带来挑战。
- 问题：对象如何在每一种状态下表现出不同的行为？
- 解决：状态模式：允许一个对象在其内部状态改变时改变它的行为。对象看起来似乎修改了它的类。（第一句话解释：行为会随内部变化而变化，如在电梯停止和上升时，点击选择，将会得到不一样的行为。第二句话解释：从外部角度来看，如果使用的对象能够完全改变它的行为，那么这个对象会是其他类的实例化。但实际上它只是引用了不同状态对象来造成类改变假象）
- 适用：一个对象的行为取决于它的状态；代码中包含大量与对象状态有关的条件语句。
- 优缺点：优点：通过将特定状态相关的行为局部化，并且将不同状态的行为分割开来来实现解耦；使得状态转换显式化。缺点：增加系统类和对象的个数，实现都较为复杂，如果使用不当将导致程序结构和代码的混乱。
- 
<!-- more -->

### 实现
- 类图：![](http://y.photo.qq.com/img?s=5w9WDsIEz&l=y.jpg)
- Context：持有一个ConcreteState子类的实例，这个实例代表当前状态。当调用request的时候，会所谓到状态类来实现。在例子中就是整个电梯。
- State：定义了一个所有具体状态的共同接口
- ConcreteState：处理来自Context的请求，提供自己对于请求的实现。所以，的那个Context改变状态时，行为也跟着改变。

## Android StateMachine
Android系统的StateMachine（https://android.googlesource.com/platform/frameworks/base/+/master/core/java/com/android/internal/util/StateMachine.java）是一个状态模式的应用，StateMachine是一个分层处理消息的状态机，并且是能够有分层排列状态。

### 实现架构
- 类图：![](http://y.photo.qq.com/img?s=m9cNCmYFm&l=y.jpg)
- StateMachine：构造函数为Protected，不能实例化，需要子类进行构造。transitionTo用于改变StateMachine的状态
- IState、State、StateInfo：State继承IState接口，StateInfo是State更丰富信息封装，表示一个State的是否正在被状态机使用和该State的父状态。StateMachine进入状态时调用enter，结束状态时调用exit，处理行为processMeesage。
- SmHandler：是消息处理派发和状态控制切换的核心，运行在单独的线程上。其中mStateStack和mTempStateStack是一个数组栈，用于保存状态机中的链式状态关系。mInitialState保存初始状态，mDestState保存切换的目的状态。

### 实现细节
以下会以“状态添加”、“启动状态机”、“状态切换”、“消息处理”四部分进行分析。
#### 一、状态添加
以执行以下状态为例：
> addState(mP1);//P1没有父状态
> 
> addState(mS1, mP1);//S1父状态为P1
> 
> addState(mS2, mP1);//S2父状态为P1
> 
> addState(mP2);//P2没有父状态

addState最终会进入SmHandler中

    private final StateInfo addState(State state, State parent) {  
	   	//确认State的父状态的添加情况，如果需要会进行递归添加
	    StateInfo parentStateInfo = null;  
	    if (parent != null) {  
		    parentStateInfo = mStateInfo.get(parent);  
		    if (parentStateInfo == null) {  
			    // Recursively add our parent as it's not been added yet. 
				//当前状态父状态未加入到StateMachine中，递归先加入其父State   
			    parentStateInfo = addState(parent, null);  
		    }  
	    }  
	    StateInfo stateInfo = mStateInfo.get(state); 
		//构造StateInfo，保存到mStateInfo中 
	    if (stateInfo == null) {  
		    stateInfo = new StateInfo();  
		    mStateInfo.put(state, stateInfo);  
	    }  
	      
	    // Validate that we aren't adding the same state in two different hierarchies.  
	    if ((stateInfo.parentStateInfo != null) &&  
	    (stateInfo.parentStateInfo != parentStateInfo)) {  
	  	  throw new RuntimeException("state already added");  
	    }  
	    stateInfo.state = state;  
		//在stateInfo中关联父状态
	    stateInfo.parentStateInfo = parentStateInfo;  
	    stateInfo.active = false;  
	    
	    return stateInfo;  
    }


根据上述代码，可以发现流程为：

1. 确认父状态情况：判断是否需要添加父状态，如果需要，将会递归加入
2. 若首次加入新状态，则构造StateInfo，并保存到mStateInfo的HashMap<State, StateInfo>中，若非首次，则沿用以前的状态。
3. 在StateInfo中记录状态和父状态的关系。
4.  图示：![](http://y.photo.qq.com/img?s=0CnNuz8xW&l=y.jpg)

#### 二、启动状态机
完成状态添加后，需要调用start（）来启动状态机，start函数最终会进入到SmHandler的completeConstruction函数

    private final void completeConstruction() {  
    	    //查找状态树的深度  
	    int maxDepth = 0;  
	    for (StateInfo si : mStateInfo.values()) {  
	    int depth = 0;  
	    //根据父子关系计算树枝层次数  
	    	for (StateInfo i = si; i != null; depth++) {  
	    		i = i.parentStateInfo;  
	    	}  
		    if (maxDepth < depth) {  
		    	maxDepth = depth;  
		    }  
	    }  
	    
	    //创建mStateStack，mTempStateStack状态栈  
	    mStateStack = new StateInfo[maxDepth];  
	    mTempStateStack = new StateInfo[maxDepth];  
	    //设置状态堆栈  
	    setupInitialStateStack();  
	    /** Sending SM_INIT_CMD message to invoke enter methods asynchronously */  
	    sendMessageAtFrontOfQueue(obtainMessage(SM_INIT_CMD, mSmHandlerObj));  
    }  
根据上述代码，可以发现计算状态树的最大深度方法：

1. 遍历状态树中的所有节点
2. 以每一个节点为起始点，根据节点父子关系查找到根节点
3. 累计起始节点到根节点之间的节点个数；查找最大的节点个数。
4. 根据查找到的树的最大节点个数来创建两个状态堆栈，并调用函数setupInitialStateStack

setupInitialStateStack中主要工作为填充mStateStack和mTempStateStack两个状态栈

    private final void setupInitialStateStack() {  
	    //在mStateInfo中取得初始状态mInitialState对应的StateInfo  
	    StateInfo curStateInfo = mStateInfo.get(mInitialState);  
	    //从初始状态mInitialState开始根据父子关系填充mTempStateStack堆栈  
	    for (mTempStateStackCount = 0; curStateInfo != null; mTempStateStackCount++) {  
		    mTempStateStack[mTempStateStackCount] = curStateInfo;  
		    curStateInfo = curStateInfo.parentStateInfo;  
	    }  
	    // Empty the StateStack  
	    mStateStackTopIndex = -1;  
	    //将mTempStateStack中的状态按反序方式移动到mStateStack栈中  
	    moveTempStateStackToStateStack();  
    }
    private final int moveTempStateStackToStateStack() {  
	    //startingIndex= 0  
	    int startingIndex = mStateStackTopIndex + 1;  
	    int i = mTempStateStackCount - 1;  
	    int j = startingIndex;  
	    while (i >= 0) {  
	  	    mStateStack[j] = mTempStateStack[i];  
	    	j += 1;  
	    	i -= 1;  
	    }  
	    mStateStackTopIndex = j - 1;  
	    return startingIndex;  
    } 

根据上述代码，可以发现两个堆栈会被初始化为：（假设当初始状态为S1时）

1. mTempStateStack的节点为：{S1,P1}
2. mStateStack={P1,S1}
3. 两者互为反序

初始化完状态栈后，SmHandler将向消息循环中发送一个SM_INIT_CMD消息

    sendMessageAtFrontOfQueue(obtainMessage(SM_INIT_CMD, mSmHandlerObj)) 
在SmHandler的handleMessage中：

    else if (!mIsConstructionCompleted &&(mMsg.what == SM_INIT_CMD) 
		&& (mMsg.obj == mSmHandlerObj)) {  
	    mIsConstructionCompleted = true;  
	    invokeEnterMethods(0);  //mStateStack栈中的所有状态设置为激活状态,
							      同时调用每一个状态的enter()函数
    }  
    performTransitions();
 
    private final void invokeEnterMethods(int stateStackEnteringIndex) {  
	    for (int i = stateStackEnteringIndex; i <= mStateStackTopIndex; i++) {   
		    mStateStack[i].state.enter();  
		    mStateStack[i].active = true;  
	    }  
    } 

根据上述代码，可以发现最后执行SM_INIT_CMD消息流程如下：

1. invokeEnterMethods函数将mStateStack栈中的所有状态设置为激活状态，同时调用每一个状态的enter()函数
2. performTransitions函数来切换状态，同时设置mIsConstructionCompleted为true
3. SmHandler在以后的消息处理过程中不再重启状态机

#### 三、切换状态
![](http://y.photo.qq.com/img?s=GiYRhzUmx&l=y.jpg)以由S1切换到P2为例。

1. mStateStack={P0,P1,S1},mTempStateStack={S1,P1,P0}
2. 以P2为起始节点，同样遍历状态节点树，查找未激活的所有节点，并保存到mTempStateStack数组中,mTempStateStack={P2,P0}
3. 接着调用mStateStack中除P0节点外的其他所有节点的exit()，并且将每个状态节点设置为未激活状态
4. 将切换后的状态节点链表mTempStateStack移动到mStateStack,mStateStack={P0,P2}
5. 调用节点P2的enter()，同时设置为激活状态。


SmHandler中handleMessage在处理每个消息时，都会使用performTransitions来检查状态切换。
    
    private synchronized void performTransitions() {  
    　　while (mDestState != null){  
    　　　　//当前状态切换了 存在于mStateStack中的State需要改变  
    　　　　//仍然按照链式父子关系来存储  
    　　　　//先从当前状态找到 最近的被激活的parent状态    　　　　
    　　　　StateInfo commonStateInfo = setupTempStateStackWithStatesToEnter(destState);  
    　　　　//将mStateStack中退出这个stateInfo(执行exit方法)  
    　　　　invokeExitMethods(commonStateInfo);  
    　　　　//新建立的状态节点链表保存到mStateStack栈中
    　　　　int stateStackEnteringIndex = moveTempStateStackToStateStack();  
    　　　　//将新加入到mStateStack中 激活State 
    　　　　invokeEnterMethods(stateStackEnteringIndex);  
    　　　　//将延迟的消息移动到消息队列的前面，以便快速得到处理 
    　　　　moveDeferredMessageAtFrontOfQueue();  
    　　}  
    }

发起切换状态：SmHandler的transitionTo

    private final void transitionTo(IState destState) {  
    	mDestState = (State) destState;       
    } 

1. transitionTo是单纯设置mDestState变量 ，并未真正更新状态栈mStateStack
2. SmHandler在每次处理消息时都会自动更新一次mStateStack，无论mDestState变量值是否改变
3. 状态的设置与状态栈的更新是异步的。


#### 四、消息处理
StateMachine提供了sendMessage、sendMessageDelayed多个发送接口将消息发送到SmHandler中。在SmHandler中的processMessage进行消息处理

    private final void processMsg(Message msg) {  
	    StateInfo curStateInfo = mStateStack[mStateStackTopIndex];  
	    //如果当前状态未处理该消息  
	    while (!curStateInfo.state.processMessage(msg)) {  
		    //将消息传给当前状态的父状态处理  
		    curStateInfo = curStateInfo.parentStateInfo;  
		    if (curStateInfo == null) {  
		     //当前状态无父状态，则unhandledMessage  
			    mSm.unhandledMessage(msg);  
			    if (isQuit(msg)) {  
			   	   transitionTo(mQuittingState);  
			    }  
			    break;  
			    }  
		    }  
		    //记录处理过的消息  
		    if (mSm.recordProcessedMessage(msg)) {  
		    if (curStateInfo != null) {  
			    State orgState = mStateStack[mStateStackTopIndex].state;  
			    mProcessedMessages.add(msg, mSm.getMessageInfo(msg), curStateInfo.state,orgState);  
		    } else {  
		   	    mProcessedMessages.add(msg, mSm.getMessageInfo(msg), null, null);  
		    }  
	    }  
    } 

根据上述代码，可以发现：

1. 消息处理过程是从mStateStack栈顶派发到栈底，直到该消息被处理
2. 当所有状态的processMessage返回false时，会调用unhandleMessage

#### 五、日志输出
整个框架通过LogRec封装日志信息，LogRecords来进行日志记录，收归了整个框架的流程日志

    public static class LogRec {
	    private StateMachine mSm;
	    private long mTime;
	    private int mWhat;
	    private String mInfo;
	    private IState mState;
	    private IState mOrgState;
	    private IState mDstState;
    } 
    private static class LogRecords {
	    private static final int DEFAULT_SIZE = 20;
	    private Vector<LogRec> mLogRecVector = new Vector<LogRec>();
	    private int mMaxSize = DEFAULT_SIZE;
	    private int mOldestIndex = 0;
	    private int mCount = 0;
	    private boolean mLogOnlyTransitions = false;
    }

## 附：
1. 对[StateMachine]([https://android.googlesource.com/platform/frameworks/base/+/master/core/java/com/android/internal/util/StateMachine.java)中的注释例子进行了流程梳理，以下为图示:
![](http://y.photo.qq.com/img?s=RDDiVuAkQ&l=y.jpg)
2. Android FrameWork中使用到StateMachine的模块有：MediaPlayer、WifiStateMachine、BluetoothAdapterStateMachine、DataConnection、DhcpStateMachine，RilMessageDecoder等

## 参考：
- https://android.googlesource.com/platform/frameworks/base/+/master/core/java/com/android/internal/util/StateMachine.java
- Head First设计模式
- http://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/state.html
- https://blog.csdn.net/yanbober/article/details/45502665
- https://blog.csdn.net/hguisu/article/details/7557252
- https://blog.csdn.net/yangwen123/article/details/10591451
- http://www.cnblogs.com/bastard/archive/2012/06/05/2536258.html
- https://www.jianshu.com/p/a5435a05ae48

## 致敬：
[Wink Saville](https://github.com/winksaville)是Android StateMachine的作者，在退休之后依然在repositories上有持续的commit，向Wink Saville和这份热爱致敬。