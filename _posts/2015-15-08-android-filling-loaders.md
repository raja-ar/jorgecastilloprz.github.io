---
layout: post
title: Play with SVG Paths in Canvas with AndroidFillingLoaders
---

We usually don't like the internal drawing logic from the Android SDK too much. When we read about it 
we use to feel weird, as it seems to be a little bit tedious. But it is not that hard if you 
read it carefully, and if you are capable of understanding it properly, you will end up creating really 
interesting figures and animations like the following one:

<div style="text-align:center">
![Small-gif]
</div>

Isn't that cool?. The previous animation has been extracted from the [AndroidFillableLoaders library](https://github.com/JorgeCastilloPrz/AndroidFillableLoaders) 
which was published by me some days ago. The lib wants to create an interesting filling effect for a custom silhouette 
created by a given SVG Path, and it is totally open to be extended by the Android community. I will be 
using it for this code example.

# Parsing the SVG path

Standard SVG path formats are not understandable by the Android SDK out of the box, but you are always 
able to construct your own [Path](http://developer.android.com/reference/android/graphics/Path.html) 
item if you have a proper parser. 
If you take any PNG image with transparency, you will be able to export its SVG path in 
 standard format by using some available tools out there like **GIMP**. [Here](http://www.useragentman.com/blog/2013/04/26/how-to-create-svg-paths-easily-using-the-gimp/) 
you have a clear example of how to do it.

Once you have got the path, just copy the numbers that define it, which will look like the following:
```
M 2948.00,18.00
   C 2956.86,18.01 2954.31,18.45 2962.00,19.91
     3009.70,28.94 3043.56,69.15 3043.00,118.00
     3042.94,122.96 3042.06,127.15 3041.25,132.00
     3036.37,161.02 3020.92,184.46 2996.00,200.31
     2976.23,212.88 2959.60,214.26 2937.00,214.00
     2926.91,213.88 2912.06,209.70 2903.00,205.24
     2893.00,200.33 2884.08,194.74 2876.04,186.91
     2848.21,159.81 2839.19,115.93 2853.45,80.00
     2863.41,54.91 2883.01,35.57 2908.00,25.45
     2916.97,21.82 2924.84,20.75 2934.00,18.51
     2938.63,17.79 2943.32,17.99 2948.00,18.00 Z
   M 2870.76,78.00
   ...
```

This is the SVG Path that you will parse to convert it to a `Path` object which will be 
totally supported by the library. To parse it, i am using the 
[SvgPathParser](https://github.com/JorgeCastilloPrz/AndroidFillableLoaders/blob/master/library%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fjorgecastillo%2Fsvg%2FSvgPathParser.java) 
class from [romannurik's Muzei](https://github.com/romannurik/muzei) code.
There is not so much to look at here, it is just a *"made by hand"* parser which converts standard path 
movements and directions defined by the String SVG path to movements in the `Path` item using its methods 
like `path.moveTo()`, `path.lineTo()`, or `path.cubicTo()` for *Bezier* curves.
 
If you know a little bit about SVG system, it defines some movements identifiable tags like `M` or `m` for 
linear movement, `C` or `c` for curves, `H` or `h` and `V` or `v` for horizontal or vertical lines, 
`L` or `l` for generic lines ...etc. Capital letters means absolutely positioned, lower cases means 
relatively positioned.

# Lifecycle statuses

`FillableLoader` is the main view here, and it has a limited number of statuses which will 
succeed one to another in order to get the complete animation working. Statuses are just flags 
that will tell the view to get drawn by one way or another at the current drawing cycle (`onDraw()` method).
At the same time, you should know that every animation from the composition will have it's own 
duration, and that will let the View know when each step is finished, so it can change it's current 
status to the following one.
 
The `drawingState` list for the view is going to be:
* `NOT_STARTED`: This is the beginning one.
* `TRACE_STARTED`: The silhouette (dash, stroke, trace) is being drawn.
* `FILL_STARTED`: The trace drawing got complete and now we are filling the view.
* `FINISHED`: This is obviously the final state of the view.

Every time the view switches it's state, the `OnStateChangeListener` method will be called to give external 
feedback to the used in order to let him create proper reactions to the animation.

# Dash drawing

Once the view gets into it's `TRACE_STARTED` state, the surrounding dash will get drawn. I have a 
`Paint` initialized for that:
```java
dashPaint = new Paint();
    dashPaint.setStyle(Paint.Style.STROKE);
    dashPaint.setAntiAlias(true);
    dashPaint.setStrokeWidth(strokeWidth);
    dashPaint.setColor(strokeColor);
```
Everything pretty normal. But how to draw the dash? If you think about it, we will need to draw a little 
bit more of the line in every cycle. So the line will keep growing, and the space still not drawn will decrease. There is one method that will come really handy to us in the Android SDK to get this effect done. The `dashPaint.setPathEffect(new DashPathEffect(...)))` method. As its documentation says, the `DashPathEffect` will need to get an array of intervals in its constructor which must have an even number of items. The even indices of the array will specify the "on" intervals, and the odd indices specifying the "off" intervals. The second argument will be a phase value which will be used as an offset into the array, but which we will not be using for this library.

**Note:** this patheffect only affects drawing with the paint's style set to STROKE or FILL_AND_STROKE. It is ignored if the drawing is done with style == FILL.

But we are missing something here, right? The length of the dash for the current cycle that needs to get drawn. The complete code would be (into the `onDraw()` method):
```java
float phase = MathUtil.constrain(0, 1, elapsedTime * 1f / strokeDrawingDuration);
float distance = animInterpolator.getInterpolation(phase) * pathData.length;

dashPaint.setPathEffect(new DashPathEffect(new float[] { distance, pathData.length }, 0));
canvas.drawPath(pathData.path, dashPaint);
```
We will get the current percent of the total time of the "animation" and the distance of the line to get drawn will be obtained from it, using an interpolator (a `DecelerateInterpolator`) as a base for the values. The `pathData.length` has been obtained previously using the [PathMeasure](http://developer.android.com/reference/android/graphics/PathMeasure.html) class.

So here it is, we already have our dash effect getting drawn. So lets keep moving!

# Filling Drawing

So the filling drawing code will look like this:
```java
float fillPhase = MathUtil.constrain(0, 1, (elapsedTime - strokeDrawingDuration) * 1f / fillDuration);
clippingTransform.transform(canvas, fillPhase, this);
canvas.drawPath(pathData.path, fillPaint);
```
As you can see, the time phase will be the percent of time consumed for filling drawing until this very moment. To calculate that we must substract the `strokeDrawingDuration` as it was used for the dash animation.

The `transform()` method will be delegated into a [ClippingTransform](https://github.com/JorgeCastilloPrz/AndroidFillableLoaders/blob/master/library%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fjorgecastillo%2Fclippingtransforms%2FClippingTransform.java) implementation and the logic in charge to create the filling effect would reside into it.


[Small-gif]: https://raw.githubusercontent.com/JorgeCastilloPrz/AndroidFillableLoaders/master/art/demoSmall.gif