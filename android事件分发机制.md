#Android事件分发机制
##主要关注三个方法：

- dispatchTouchEvent(MotionEvent ev);
- onInterceptTouchEvent(MotionEvent ev);
- onTouchEvent(MotionEvent ev);


View和ViewGroup调用方法的区别：

- 对于ViewGroup来说，主要会调用以上三个方法进行分发和拦截。
- 对于View来说，会调用dispatchTouchEvent、onTouchEvent方法进行分发和处理。View的onTouchEvent默认返回true，除非该View是不可点击的(即clickable和longClickable同时为false)

Activity.dispatchTouchEvent -> Window -> DecorView -> ViewGroup -> View

上述流程的一些说明:

- Window的实现类为PhoneWindow，通过调用superDispatchTouchEvent方法，调用到了DecorView的superDispatchTouchEvent方法，这样就把事件传递到了DecorView。
- DecorView是顶级View，是一个FrameLayout, 包含一个竖直方向的LinearLayout，线性布局分两部分，上面是标题栏，下面是内容栏，setContentView方法设置的正是这个内容栏的界面。
- 事件分发和事件处理是两个不同的过程，虽然过程很相似，但是调用的方法和处理的过程不一样，注意区分。事件分发是从顶级View向下传递，事件处理是某个View决定要拦截时要做的处理流程，如果处理不了则向上传递。

事件类型主要分三类，ACTION_DOWN、ACTION_MOVE、ACTION_UP，一个事件序列中的每个事件都从按照上面的传递顺序，传递到DecorView后按照View的事件分发和拦截机制进行传递。

##在ViewGroup中

在不设置onTouchListener时，其调用顺序为

dispatchTouchEvent -> onInterceptTouchEvent -> onTouchEvent

设置了onTouchListener时，会重写onTouch()方法，其调用顺序为：

dispatchTouchEvent -> onInterceptTouchEvent -> onTouch(返回false) -> onTouchEvent
dispatchTouchEvent -> onInterceptTouchEvent -> onTouch(返回true)

如果设置了OnClickListener，会在onTouchEvent方法中调用onClick方法。上述调用顺序变成：

- dispatchTouchEvent -> onInterceptTouchEvent -> onTouchEvent -> onClick  
- dispatchTouchEvent -> onInterceptTouchEvent -> onTouch(返回false) -> onTouchEvent -> onClick

##在View中

相对ViewGroup简单，没有调用onInterceptTouchEvent方法，在调用dispatchTouchEvent方法之后，直接交给onTouchEvent方法做处理。onTouchEvent执行的过程和ViewGroup一样，前提是该View是enable的，并且OnTouchListener的onTouch方法返回true(如果有设置的话)。

##事件传递过程

- 当事件到达顶级View时，就开始了View的事件分发和拦截机制。一般从一个ViewGroup开始，先调用dispatchToucEvent方法对事件进行分发，如果这个ViewGroup决定拦截，则onInterceptTouchEvent方法返回true，并调用onTouchEvent方法对事件做处理。如果不拦截，onInterceptTouchEvent方法返回false，交由到下一级子View做处理。

- 如果下一级子View是一个ViewGroup，则处理流程和它的父ViewGroup一样。如果子View是一个View，则该View会调用dispatchTouchEvent方法对事件进行分发。此处不会像ViewGroup那样调用onInterceptTouchEvent方法，因为它没有子View可传递，因此会调用onTouchEvent方法对事件做处理。如果设置了OnTouchListener和OnClickListener中的接口方法，则会按照上述在ViewGroup中设置了接口方法时的调用顺序那样进行调用。

- View的onTouchEvent方法默认返回true。如果返回了true，表示该事件被消费了，那么View不会再往上传递到其父ViewGroup的onTouchEvent方法中做处理。如果返回了false，则表示该事件未被消费，该View会把该事件处理结果传递到父View，交由父View处理，父View也会在onTouchEvent中做处理。如果父View处理不了，则继续向上传递，如果所有的元素都没有处理该事件，则最后会调用Activity的onTouchEvent方法做处理。

##注意点

- 某个View一旦决定拦截，那么这个事件序列都只能由它处理，如果事件能传递到它的话。并且该View的onInterceptTouchEvent方法不会再被调用。
- 正常情况下，一个事件序列只能被一个View拦截且消耗。一旦一个元素拦截了某次事件，同一事件序列内所有事件都将直接交给它处理。
- 事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View。通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但ACTION_DOWN事件除外。