## 支付宝主页UI效果实现

## 头部渐变效果

头部侵入到状态栏中

头部透明度渐变度处理

```java
private int                           sumY      = 0;
private float                         duration  = 150.0f;//在0-150之间去改变头部的透明度
private ArgbEvaluator                 evaluator = new ArgbEvaluator();
private RecyclerView.OnScrollListener listener  = new RecyclerView.OnScrollListener() {
    @Override
    public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
        super.onScrolled(recyclerView, dx, dy);

        // System.out.println("recyclerView = [" + recyclerView + "], dx = [" + dx + "], dy = [" + dy + "]");

        sumY += dy;

        // 滚动的总距离相对0-150之间有一个百分比，头部的透明度也是从初始值变动到不透明，通过距离的百分比，得到透明度对应的值
        // 如果小于0那么透明度为初始值，如果大于150为不透明状态

        int bgColor = 0X553190E8;
        if (sumY < 0) {
            bgColor = 0X553190E8;
        } else if (sumY > 150) {
            bgColor = 0XFF3190E8;
        } else {
            bgColor = (int) evaluator.evaluate(sumY / duration, 0X553190E8, 0XFF3190E8);
        }

        llTitleContainer.setBackgroundColor(bgColor);
    }
};
```