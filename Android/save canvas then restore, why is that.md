Imagine the canvas is a piece of paper, and you're tasked with drawing a picture of a robot right side up, at the bottom, and another robot upside down, slightly moved to the right, and about 40% smaller at the top. Something like this; [![Android canvas example](https://i.sstatic.net/5Bg2b.png)](https://i.sstatic.net/5Bg2b.png)

How would you start? What's easier to do first?

You would probably draw the bigger robot at the bottom first since it's right-side up and it's a lot easier to draw in the direction that feels more natural. So you've got the first one done, now how do you approach the second upside down robot?

- You could attempt to draw it as is, but that would be a bit difficult since you're upside down.

**or**

- You could rotate your paper 180°, move your starting point a bit, and start drawing at a smaller scale, and after you're all done you'd just rotate the paper back.

This is what `canvas.save()` and `canvas.restore()` do, they allow you to modify your canvas in any way that makes it *easier* for you to draw what you need. You don't need to use these methods, but they sure do simplify a lot of the process. The above would look something like

```scss
drawRobot()
canvas.save()
canvas.rotate(180)
canvas.translate(100, 0)
canvas.scale(40,40)
drawRobot()
canvas.restore()
```

If we look at the `restore()` documentation it says

> is used to remove all **modifications** to the matrix/clip state since the last save call

and to see what those modifications are we take a look at `save()` it says

> translate, scale, rotate, skew, concat or clipRect, clipPath

Well look at that, we did in fact use `translate` `rotate` and `scale` but we also did call `drawRobot()` so wouldn't calling `restore` erase our drawing? <mark>No, because it doesn't affect the drawing, **only** the modifications. So when we call `restore` it will return our canvas to the state that it was in before we started the second drawing.</mark>

# 资料

https://stackoverflow.com/questions/29040064/save-canvas-then-restore-why-is-that


