# Houdini + UE5布料解算
![](https://github.com/ban-kexuan/ClothSimulation/blob/main/overview.png)
在houdini中制作服装＋解算，使用ABC格式导入UE做渲染。
走路的模型来自mixamo，这里我发现如果导入的模型不是T-pose模型，那么在腋下处拉伸会很明显，于是乎，最好还是T-pose模型效果最好，由于这次做的是长袍外套所以看不到内部模型。  houdini的初步探索！
## Houdini相关流程
### 1. 对人物模型进行处理

对于导入的行走模型，我这里就采用了帧动画实现240帧一直向前走，k脚打滑的现象还挺明显...其实也可以在houdini中做个循环动画的，需要在第一帧和最后一帧之间作插值。 使用 timeshift 冻结第一帧结点命名为static_model，对于正常播放的结点命名为 motion_model。 这里冻结帧的原因是后续在解算时缝合布料的过程以及由于受重力等外力的影响需要模拟布料自由下落的功能，因此我们可以设定第一帧的布料解算到什么程度后才开始代入行走动作的解算。

### 2. 给人物做服装
服装分为三个部分，内层腰带，裙摆，长袍外套。从内到外开始制作，制作的方式其实类似MD的方法，都是将前后面片patch进行缝合。

① 腰带：

- 制作一个大小合适的长方形前侧腰带面片，需要在patch中勾选左侧和右侧分别划分为一个组。用同样的方式制作后侧的腰带，这里先不管法线是否合理，最好做完patch后展一下uv。 可以做完腰带后就先解算一下看看有什么问题。

② 裙摆：

裙摆需要注意的是这是个拖尾裙摆，在patch选择上可以修改为梯形状的。另外这里的裙摆需要和腰带进行缝合，需要将裙摆的下侧（需要具体看是哪一侧的）和腰带的下侧分别弄成一个组进行缝合。

- 这里会出现的问题是，由于腰带的顶点数和裙摆一侧的顶点个数不一定是一样的，如果不一样缝合就不会是理想的情况。可以将edgelength设置成一样的，但是这样本来腰带可以面片少一些也不得不和裙摆的密度保持一致。另外一种做法是看看腰带下侧是多少个顶点，在裙摆patch中可以设置相关的bottom  points这样就可以将二者缝合处的顶点个数相统一，但又不改变整体的网格密度。


③ 长袍外套：

选择Ring类型的patch，将open arc 设置为355，留有缝隙可以缝合。将左侧和右侧部分分别弄成一个组。

关于缝合需要注意的是 前后面片以及缝合处都不要与人物模型接触！

### 3. 解算
解算过程是需要static_model的

**merge → vellumcloth → vellumdrape → vellumprocess**

- vellumcloth里可以设置这个布料的一些物理信息，比如重力，拖拽力，密度啊等等，还有解算过程中非常重要的拉伸和弯曲性能，这些参数可以实现不同材质的一些物理特性，比如有的布料在摔动的时候看起来轻飘飘的，有的看起来很厚重。有些布料弹性很好，拉伸性能很大等。

- vellumdrape就是缝合过程的实现，里面需要设置将哪两个组进行缝合。另外可以设定freeze at frame，在某一帧冻结，也就是觉得解算的差不多了可以作为第一帧布料使用。

- vellumpostprocess这里最直观的感受是它可以合并缝合节点。在缝合的时候会出现一个缝合点在缝合结束后依旧是两个顶点，那么postprocess可以解决这个问题。在使用这个节点后可以看到信息栏里顶点数是变少了的。

上面说到前后两个腰带面片，如果是直接修改transform改变位置，那么会出现法线反了的问题，可以在后面使用polydoctor 他可以自动修改错误的法线。

以上是针对冻结帧的初步解算，后续需要motion_model以及一个地面来实现服装与人物地板的碰撞。

将两个模型merge，采用vellumcloth---vellumsolver---vellumpostprocess

- vellumsolver中可以设置解算的迭代次数，迭代次数越多，效果越好，越不容易穿模，但带来的计算时间也是越大的。可以增加风力，摩擦力等等。以及设置布料自碰撞与人物模型碰撞等相关参数。

下一步！uv。

可以把外套和裙摆划分为两个组，在分材质的时候会比较方便。

uv的问题折磨了很久，不容易导出来了，发现只有unity可以识别到它的uv，任何dcc软件都识别不到。后来才发现，需要将uv从point级别转为vertex级别才能被识别。 这一步attribpromote可以实现。

之后使用clean结点将不用的信息删除，filecache--ropabc！结束！

另外，这类的解算的结果如果使用FBX是没有动画的，貌似是因为只有点动画，当前 也只有3dsmax才能识别到动画，其他软件都不可以。即使用3dsmax再导出也不可以。  或许houdini直接导入ue/unity才是最省事的。
