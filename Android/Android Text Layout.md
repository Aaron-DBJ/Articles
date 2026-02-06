Let's first review what the [documentation says](http://developer.android.com/reference/android/graphics/Paint.FontMetrics.html):

- **Top** - The maximum distance above the baseline for the tallest glyph in the font at a given text size.
- **Ascent** - The recommended distance above the baseline for singled spaced text.
- **Descent** - The recommended distance below the baseline for singled spaced text.
- **Bottom** - The maximum distance below the baseline for the lowest glyph in the font at a given text size.
- **Leading** - The recommended additional space to add between lines of text.

Note that the **Baseline** is what the first four are measured from. It is **line** which forms the **base** that the text sits on, even though some characters (like g, y, j, etc.) might have parts that go below the line. It is comparable to the lines you write on in a lined notebook.

Here is a picture to help visualize these things:

<img src="https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/20241011212021.png" title="" alt="" width="667">

The **leading** is the distance between the bottom of one line and the top of the next line. In the picture above, it is the space between the orange of Line 1 and the purple of Line 2.

Android SDK提供的API：**`getLineBaseline`，`getLineBottom`，`getLineTop`返回值取决于他们所在的行，也就是是说top、baseline和bottom返回的是坐标**

但是，**`getLineAscent`，`getLineDescent`返回的值是距离，除非使用了span，每行文字返回的ascent和descent是一样的**


