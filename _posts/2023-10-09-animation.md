# css animation
## 实现一个逐帧动画
```css
.box {
    width: 100px;
    height: 100px;
    background: img/1.png;
    /* 逐帧播放这个png，假设这个png有5帧 */
    animation: move 1s steps(5) forwards;
    /* animation: move 1s steps(5, start) infinite; */
}
@keyframes move {
    0% {
        position: 0;
    };
    100% {
        position: 100%;
    }
}
```
