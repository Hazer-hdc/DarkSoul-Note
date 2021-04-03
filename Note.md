[toc]

# 1、用户输入模块

## 1、输入信号

将w和s的输入转化为-1到1的信号量，a和d也是。
再通过mathf.smoothDamp让信号量平滑过渡.

通过设置一个flag"inputEnable"来关闭输入.
```
public string keyUp = "w";
    public string keyDown = "s";
    public string keyLeft = "a";
    public string keyRight = "d";

    public float Dup;
    public float Dright;

    public bool inputEnable = true;

    private float targetDup;
    private float targetDright;
    private float velocityDup;
    private float velocityDright;

    void Update()
    {
        //将wasd输入转化微signal。
        targetDup = (Input.GetKey(keyUp) ? 1.0f : 0) - (Input.GetKey(keyDown) ? 1.0f : 0);
        targetDright = (Input.GetKey(keyRight) ? 1.0f : 0) - (Input.GetKey(keyLeft) ? 1.0f : 0);

        //关闭输入
        if(!inputEnable)
        {
            targetDup = 0;
            targetDright = 0;
        }
        //通过smoothDamp让singal平滑过渡
        Dup = Mathf.SmoothDamp(Dup, targetDup, ref velocityDup, 0.1f);
        Dright = Mathf.SmoothDamp(Dright, targetDright, ref velocityDright, 0.1f);
    }
```

## 2、动画控制器

### 1、animatorController
新建一个animatorController，将其放入角色的animator组件中。
![](2021-04-02-18-19-20.png)
![](2021-04-02-18-19-42.png)

将需要的动画拖入animController中。

### 2、混合树

[混合树的使用](https://docs.unity3d.com/cn/2019.4/Manual/class-BlendTree.html)

混合树(Blend Tree)：用于允许通过按不同程度组合所有动画的各个部分来平滑混合多个动画。各个运动参与形成最终效果的量使用混合参数进行控制，该参数只是与动画器控制器关联的数值动画参数之一。

要使混合运动有意义，混合的运动必须具有相似性质和时间。混合树是动画器控制器中的特殊状态类型。

**混合树可以让不同动画之间进行混合，让切换平滑过渡。切换动画的时候不会显得僵硬。**


1、创建一个名叫ground的混合树，所有和踩在地面相关的都放在里面。
![](2021-04-02-18-29-16.png)

2、双击混合树，在混合树的inspector中添加motion。
![](2021-04-02-18-38-32.png)

![](2021-04-02-20-27-19.png)
第一个红圈表示动画播放速度，第二个是镜像

### 3.串连玩家控制与角色动画

**各种getComponent操作最好放在awake函数中，因为在start阶段都要开始操作别人身上的组件了，所以最好是大家约好在awake中把所有需要的组件抓好，那么在start阶段彼此交互就不会缺失某些组件**


```
public class actorController : MonoBehaviour
{
    public GameObject model;

    public PlayerInput pi;
    [SerializeField]
    private Animator anim;
    // Start is called before the first frame update
    void Awake()
    {
        pi = GetComponent<PlayerInput>();
        anim = model.GetComponent<Animator>();
    }

    // Update is called once per frame
    void Update()
    {
        anim.SetFloat("forward", pi.Dup);
    }
}
```

## 3.角色行走


**再利用transform.forward 和 trasnform.right 来旋转角色。通过变换角色本地坐标系中的z，来让角色旋转**。

Dmag 是 Dup 和 Dright 各自的平方相加，再开方,用来控制角色的移动动画与行走速度。

forwardDirection = Dup * transform.forward + Dright * transform.right;


```
public class PlayerInput : MonoBehaviour
{
    //Dup和Dright各自的平方相加，再开方,用来控制角色的移动动画与行走速度。
    private float Dmag;
    //改变角色的朝向，用来旋转juese
    private Vector3 forwardDirection;
    void Update()
    {
        Dmag = Mathf.Sqrt(Dup * Dup + Dright * Dright);
        forwardDirection = Dup * transform.forward + Dright * transform.right;
    }
}

public class actorController : MonoBehaviour
{
    public GameObject model;

    public PlayerInput pi;
    [SerializeField]
    private Animator anim;

    // Update is called once per frame
    void Update()
    {
        //切换行走动画
        anim.SetFloat("forward", pi.Dmag);
        //旋转角色模型。当Dup和Dright都为0，就不旋转角色了，否则会把0赋值给forward
        if(pi.Dmag > 0.1f)
        {
            model.transform.forward = pi.forwardDirection;
        }

    }
}
```

#### 让角色真正移动

**方案1： 利用Rigidbody组件**。rigidbody不太好控制。rigidbody想爬楼梯会很累，需要自己写代码，要判断这个楼梯是多高，先试踩一脚，如果发现可以踩，那就把坐标往上，推诸如此类逻辑。但rigidbody爬坡会比较好做。


**方案2： 利用CharacterController组件**。这是unity自己制作的，它的运算比较快。如果需要有重力的效果，需要明白move函数和simpleMove函数的区别。
如果要旋转重力的方向，那么characterController可以很快地利用simpleMove搭配另外方向的重力。
它比较好的地方是可以很容易地爬楼梯。

**这里采用rigidbody**



给角色添加rigidbody组件，再Freeze Rotation x、y、z轴。

**第一种方法：在fixedUpdate中更改rigidBody的position。**
```
public class actorController : MonoBehaviour
{
    public float walkSpeed = 2.0f;
    //角色位移量
    private Vector3 movingVec;

    void Update()
    {
        movingVec = pi.Dmag * model.transform.forward * walkSpeed;
    }
    private void FixedUpdate()
    {
        //位移 = 速度 * 时间
        rigid.position += movingVec * Time.fixedDeltaTime;
    }
}
```
**第二种方法：更改rigidbody的velocity**

需要注意的是：
```
        //直接指派速度，不需要乘时间
        rigid.velocity = movingVec;
```
与
```
        //位移 = 速度 * 时间
        rigid.position += movingVec * Time.fixedDeltaTime;
```
是等价的。但velocity不需要乘时间。

**但此时velocity是有问题的，** 因为movingVec是由forward值乘出来的，所以y分量为0。如果直接将movingVec赋给velocity，就会把原来rigidbody的y分量的值给覆写了。这样就不会有地心引力，而且也跳不起来。

**所以正确的写法是：**
```
rigid.velocity = new Vector3(movingVec.x, rigid.velocity.y, movingVec.z);
```

**做完这些后都要做爬楼梯或爬坡的测试。**

#### 还存在的问题： 

1.当角色斜45度角走时，速度会变快，这是因为Dmag的值此时变成了根号2，而不是1。

2.在斜坡上时，角色的脚掌还不能很好地踩地。这地方要通过逐步IK的设置。

## 4.跑步功能

将跑步动画添加进混合树中，设置Threshold为2.

![](2021-04-03-15-35-11.png)

playerInput代码中新加一个string变量KeyRun用来存储奔跑键，bool变量run标识奔跑键是否被按下。
run = Input.GetKey(keyRun);

然后在actorController中控制角色奔跑动画与速度。
```
public class actorController : MonoBehaviour
{
    //控制奔跑速度
    public float runMultiplier = 2.0f;

    void Update()
    {
        //在后面乘上
        anim.SetFloat("forward", pi.Dmag * (pi.run? 2.0f : 1.0f));
        
        //角色移动速度
        movingVec = pi.Dmag * model.transform.forward * walkSpeed * (pi.run ? runMultiplier : 1.0f);
    }
```

## 5.线性插值和球形线性插值（lerp和slerp）

Vector3.Lerp是在两个点之间进行线性插值。

Vector.slerp是在两个向量之间进行线性插值，返回的向量的方向通过角度进行插值

**通过使用Slerp让角色的转身不会转得过快。**

下面修改角色转身代码：
```
model.transform.forward = Vector3.Slerp(model.transform.forward, pi.forwardDirection, 0.3f);
```

**通过使用Mathf.lerp让角色从行走切换到跑步的动画不要太突兀。**
```
    float targetRunMulti = pi.run ? 2.0f : 1.0f;
    anim.SetFloat("forward", pi.Dmag * Mathf.Lerp(anim.GetFloat("forward"),targetRunMulti,0.5f));
```

## 6.椭圆映射法

[论文地址](https://arxiv.org/ftp/arxiv/papers/1509/1509.06344.pdf)

 前面遗留的问题，当让角色斜向运动时，运动速度会变快。这是因为以Dup和Dright为直角边的三角形的斜边为根号2，不是1。

 此时可以通过椭圆映射法将正方形的坐标映射到圆的坐标上，当Dup和Dright同时为1时，通过映射后，得到的斜边值为1。

![](2021-04-03-19-16-40.png)

修改代码

```

void Update()
{
    //将水平输入和垂直输入映射到圆上
        Vector2 tempDAxis = sphereToCircle(new Vector2(Dright, Dup));
        float newDup = tempDAxis.y;
        float newDright = tempDAxis.x;

        Dmag = Mathf.Sqrt(newDup * newDup + newDright * newDright);
        forwardDirection = newDup * transform.forward + newDright * transform.right;
}
private Vector2 sphereToCircle(Vector2 input)
    {
        Vector2 output = Vector2.zero;

        output.x = input.x * Mathf.Sqrt(1 - (input.y * input.y) / 2);
        output.y = input.y * Mathf.Sqrt(1 - (input.x * input.x) / 2);

        return output;
    }
```










