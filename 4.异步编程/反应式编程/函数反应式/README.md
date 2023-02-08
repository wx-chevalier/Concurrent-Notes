# Functional Reactive Programming

何谓函数响应式编程(Functional Reactive Programming)，笔者觉得 Heinrich Apfelmus 的两段描述较为形象：

- declarative programming with data that changes over time: 包含了随时间变化的数据流的响应式编程
- The essence of functional reactive programming is to specify the dynamic behavior of a value completely at the time of declaration. : 函数响应式编程的核心在于声明时即完全确定某个值的动态行为

复杂用户交互处理，譬如[Composing Reactive Animations](http://conal.net/fran/tutorial.htm)

目前 FRP 主要有两个流派：

- Event Stream Based FRP: Event stream based libraries, focus at manipulating streams and are very good at that. Joining, splitting, merging, mapping, sampling etc. These libs are useful if you want to reason about multiple event streams at the same time. Or when throttling plays an important role, like with network traffic. For solving simple problems though, these libraries are quite obtrusive (imho). They need a different mindset. Also, representing every variable in your state as stream might become quite unwieldy. Examples: RxJs and BaconJS.

- Transparent Functional Reactive programming: Transparent Reactive programming on the other hand tries to hide reasoning about time. It just applies reactive programming in the background. Like with event based, TFRP updates views whenever needed. The difference is that in TFRP you don't define how and when stuff updates. While with streams you (have to) define that explicitly.Tracker gives you much of the power of a full-blown Functional Reactive Programming (FRP) system without requiring you to rewrite your program as a FRP data flow graph.Examples of frameworks that apply TFRP libs are knockoutJS, EmberJS and Meteor. Mobservable is stand-alone, although it ships with a small ReactJS binding.

# Event Stream

# TFRP
