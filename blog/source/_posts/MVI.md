---
title: Thinking in MVI
date: 2018/3/21
comments: true
categories: architecture
tags: architecture

---

## 一、业务问题
个人负责维护过的几个业务，都以MVC、MVP模式来设计。随着业务发展，出现以下问题：

- 改动一处Model牵连数处View，难以测试
- View的更新时序不一，难以调试
- View、Model调用源头过多，难以跟踪

## 二、问题分析
对整体业务进行了熟悉和梳理后，发现架构情况（单以MVC为例）如图：

<!-- more -->

![](http://m.qpic.cn/psb?/V10Sw18W1Je6qM/98yuLBH7e4bFca6OuslJp0cpBbwTGAuSlBWUjAscuYM!/b/dEABAAAAAAAA&bo=XQHIAF0ByAADFzI!&rf=viewer_4&t=5)
![](http://m.qpic.cn/psb?/V10Sw18W1Je6qM/GBy37MY70nI7bnfTY4QOugvyOl*rlxMtduo5GoZIiv0!/b/dEUBAAAAAAAA&bo=9ADIAPQAyAADFzI!&rf=viewer_4&t=5)


1. 当业务复杂后，修改Model或者View后，他们有可能会引起多个其他MV的变化。这些“暗箱操作”缺乏统一管理。
2. Model、View变化来源有可能是异步情景（IO、UI交互、系统生命周期），对这些情景下的异步、同步代码缺乏有效组织。

以上问题导致出现了难以“预测”结果，难以Scale Up，难以测试，难以维护的问题。

## 三、解决方式
- 方式目的：让代码流程变得可预测
- 解决思路：数据逻辑单向流动，程序响应数据变化
- 方案学习：
学习了前端中的单向数据流、响应式的思想和框架，包括Flux、Redux、Cycle.js。

### 1、 Flux
#### 图述：
![](http://m.qpic.cn/psb?/V10Sw18W1Je6qM/RTsxjtcfw96eh*VuOG0UrrazCAHEUAOmSvR7D4mgOUw!/b/dEUBAAAAAAAA&bo=lALIAJQCyAADFzI!&rf=viewer_4&t=5)

-  核心：
- Stores:存放业务数据和应用状态，一个Flux中可能存在多个Stores
- View:视图层
- Actions:用户输入之后触发View发出的事件
- Dispatcher:负责分发Actions

#### Android实现：https://github.com/lgvalle/android-flux-todo-app

### 2、Cycle.js
#### 图述：
![](http://m.qpic.cn/psb?/V10Sw18W1Je6qM/zuBoEqyrxNgrDR4nQX6CBAr2.WVVt.arzCrtpDIBhiY!/b/dFYBAAAAAAAA&bo=4gHIAOIByAADFzI!&rf=viewer_4&t=5)
-  核心：View(Model(Intent))--MVI架构 和 采用RxJs实现响应式

#### Android实现：https://github.com/sockeqwe/mosby

### 3、Managing State with RxJava
#### 图述：
![](http://m.qpic.cn/psb?/V10Sw18W1Je6qM/mJkKUupvtQWMs57vjeeHNnftb0ze8Gctqtb0iHnpfYE!/b/dPMAAAAAAAAA&bo=qwHIAKsByAADFzI!&rf=viewer_4&t=5)

- 核心：类似Cycle.js，安卓实现的另一版本。

- 方案沉淀：采用MVI架构设计、采用RxJava实现响应。（基于2和3）

## 四、实现细节：
### （一）MVI实现：
![](http://m.qpic.cn/psb?/V10Sw18W1Je6qM/xzYY37GAMtFpO8OJQz6u2qDZw2epd7gcKR9iQyH28oA!/b/dEIBAAAAAAAA&bo=YgHIAGIByAADFzI!&rf=viewer_4&t=5)

- Action：包含业务逻辑处理所需的参数数据
- Intent：代表View或其他事件的触发
- Result：包含业务逻辑返回的数据
- ViewState：包含View层所需绘制的数据
- IView：是View层核心职责抽象：对ViewState的绘制、提供Intent事件源
- IViewModel：是业务层抽象：对Intent的业务处理、对业务返回转化为ViewState


```java
public interface Action {}
public interface Intent {}
public interface Result {}
public interface ViewState {}

public interface IView<I extends Intent, VS extends ViewState> {
    Observable<I> getIntents();//提供Intent事件源
    void render(VS vs);//对ViewState的绘制
}
public interface IViewModel<I extends Intent, VS extends ViewState> {
    void process(Observable<I> intent); //处理Intent
    Observable<VS> getViewState();//获取ViewState
}
```



### （二）数据流实现：
数据流的起始为Intent，结束为ViewState。下面分为几个部分描述：
#### 1、Intent to Action：
A、通过Observable.merge（）把IView中的各个intent数据流合并，收归处理。
B、通过ofType（）  +  map （）来过滤各个Action将要进行的业务逻辑。

```java
intentSelector = new Func1<Observable<SampleIntent>, Observable<SampleIntent>>() {
            @Override
            public Observable<SampleIntent> call(Observable<SampleIntent> shareIntent) {
                return Observable.merge(
                        shareIntent.ofType(SampleIntent.AIntent.class),
                        shareIntent.ofType(SampleIntent.BIntent.class)
                );
            }
        };
 intentToAction = new Func1<SampleIntent, SampleAction>() {
            @Override
            public SampleAction call(SampleIntent sampleIntent) {
                if (sampleIntent instanceof SampleIntent.AIntent) {
                    return new SampleAction.AAction();
                } else if (sampleIntent instanceof SampleIntent.BIntent) {
                    return new SampleAction.BAction();
                } else {
                    return null;
                }
            }
        };
```


#### 2、Action to Result：
通过compose（） 和 ObservableTransformer 组织业务逻辑，方便代码梳理和业务逻辑隔离

```java
bizProcessor = new Observable.Transformer<SampleAction, SampleResult>() {
            @Override
            public Observable<SampleResult> call(final Observable<SampleAction> shareAction) {
                return shareAction.publish(new Func1<Observable<SampleAction>, Observable<SampleResult>>() {
                    @Override
                    public Observable<SampleResult> call(Observable<SampleAction> sampleActionObservable) {
                        return Observable.merge(
                                shareAction.ofType(SampleAction.AAction.class).compose(aActionToAResultProcessor),
                                shareAction.ofType(SampleAction.BAction.class).compose(bActionToBResultProcessor));

                    }
                });
            }
        };
```

#### 3、Result to ViewState：
通过Scan（）来遍历每个业务返回的Result，并根据每个业务Result的字段来返回新的ViewState。（Scan会缓存上次的Result，因此也可以用于做增量更新）

```java
reducer = new Func2<SampleViewState, SampleResult, SampleViewState>() {
            @Override
            public SampleViewState call(SampleViewState previousState, SampleResult sampleResult) {
                if (sampleResult instanceof SampleResult.AResult) {
                    return new SampleViewState.ASampleViewState();
                } else if (sampleResult instanceof SampleResult.BResult) {
                    return new SampleViewState.BSampleViewState();
                } else {
                    return null;
                }
            }
        };
```


#### 4、Summary：

```java
intents.publish(intentSelector)
                .map(intentToAction)
                .compose(bizProcessor)
                .scan(new SampleViewState(), reducer);
```


### （三）实现问题：



#### 问题Ⅰ：前后相同的ViewState，会触发UI造成不必要的刷新。
解决：通过distinctUntilChanged（）来过滤前后两次的scan（）结果，不相同的ViewState才进行刷新。

#### 问题Ⅱ：当在onConfigurationChanged时，数据流无法保持，会被重新创建。无法应用在屏幕旋转但需要保持最近展示数据的情景。（数据流在onCreate时订阅subscribe，onDestory时unSubscribe）
解决：
A、对于流的结果ViewState，需要进行缓存，可以通过replay(） +  autoConnect（）或者BehaviorSubject实现
B、对于流的起始输入，需要保持在内存，不受生命周期影响。可以通过使用一个静态的PublishSubject来中转View层的Intent输入。

## 五、使用效果
（1）所有异步逻辑收归到同一条流，View层只进行响应，因此会

- 便于代码跟踪和调试
- 便于解决时序问题
- 便于组织异步代码

（2）流是单向的，因此会

- 消除额外的“暗箱操作”
- 更直观更容易理解业务逻辑（都有唯一和相同的输入输出）

## 六、问题思考
这套开发模式（框架）并不是解决问题的“银弹”，会有以下问题

- 小粒度修改，修改成本大：因为需要统一的收归处理，因此小粒度的对View的增删修改，都需要全流程修改，没有像之前MVC的灵活方便。
- 业务复杂后，收归成本大：因为统一收归里需要处理每种Action和Result。业务复杂后，在收归处会有大量的if-else if-else 以及switch 处理。
- RxJava的学习成本

因此需要针对核心业务或业务的核心数据，灵活采用能解决主要矛盾的框架或模式，才是“银弹”。

## 七、相关参考
1. https://facebook.github.io/flux/
2. https://cycle.js.org/
3. https://www.infoq.com/news/2014/05/facebook-mvc-flux
4. http://jakewharton.com/managing-the-reactive-world-with-rxjava/
5. http://hannesdorfmann.com/android/mosby3-mvi-1
6. https://github.com/oldergod/android-architecture/tree/todo-mvi-rxjava
7. https://www.jianshu.com/p/42d77c577ff4
8. https://www.cnblogs.com/gujf2016/p/5780086.html



