---
layout: post
title: Monitoring your Cloud like its 1986
date: "2018-09-22"
---


# Introduction

_The Cloud, a digital fronteir, I al-_ hang on that's the opening line from Tron...

_Cloud, the final fron-_ No that's Star Trek...

Okay so I don't have an opening line for this... so let's just get started.

Running cloud, and cloud-like systems can be fun, but no matter what kind of system you're running,
your ability to do so is only as good as the monitoring and reporting tools you have in place.

In this post we're going to explore how we can use the monitoring architecture of 2018, to produce
reports from 1986.


# The Result

This time... IN VIDEO FORMAT:

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/zeEYwzaeJ1s" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>


# The Tools

We're actually only going to need three things to do this:

* A colour graphical plotter, the older the better (but try to get one with a serial or parallel interface)
  In my case a RadioShack CGP-115
* [Go](https://golang.org/) as our programming environment
* [Prometheus](https://prometheus.io/) to collect, store and provide the data for our charts.


# The Code

The code is [available on github](https://github.com/LeoAdamek/cloud-reports)

# 1. Instructing to the Plotter

The plotter operates in two modes. Boring ASCII/JIS text mode, and graphical mode.
The graphical mode uses ASCII based commands, terminated by a `CRLF`

We'll be using the following commands:

| Command | Parameters  | Min. | Max. | Description                                      |
|:-------:|:------------|-----:|-----:|:-------------------------------------------------|
| `M`     | `X`,`Y`     | -999 |  999 | Move the cursor to the given position            |
| `D`     | `X`,`Y`     | -999 |  999 | Draw to the given position                       |
| `C`     | `Color`     |    1 |    4 | Change to the given colour                       |
| `L`     | `Style`     |    0 |   15 | Set the line style (dash-interval)               |
| `Q`     | `Direction` |    0 |    3 | Set the text direction                           |
| `S`     | `Size`      |    0 |   63 | Set the text size                                |
| `X`     | `Direction` |    0 |    1 | Draw graph axis - Direction                      |
|         | `Interval`  |    0 |  999 | Interval for ticks on axis                       |
|         | `Ticks`     |    0 |  999 | Number of ticks for axis                         |
| `P`     | `Text`      |      |      | Draw text                                        |
| `I`     |             |      |      | Set the printer's origin to the current location |


# 2. Talking to the Printer

Using a USB to Parallel adaptor on Linux, the plotter shows up as a device file at
`/dev/usb/lp0`. 
Under the default udev rules this device will be owned by root and the group  `lp`,
so as long as we're in the `lp` group, we can send data to the printer by writing to it as a file.
This is awesome. This makes our job stupidly easy.

{{< highlight go >}}
writer, err := os.OpenFile("/dev/usb/lp0", os.WRONLY|os.O_APPEND, 0660)
{{< /highlight >}}

Assuming we've got permission to do so, we now have an `io.Writer` for the printer.

Now we can send commands using the standard `io.Writer.Write([]byte) (int, error)`


{{< highlight go >}}
_, err := writer.Write([]byte("M10,10\r\n"))
{{< /highlight >}}

A simple `io.Writer` and some ad-hoc string formatting can get us talking to the printer,
and controlling it, as would've been done in BASIC. This approach has a few drawbacks,
the only way to know where the printer is at a given point in the program is to mentally
step-through the code, replaying the instructions being issued. This isn't very effective,
and will make debugging issues more difficult. It'd be very helpful for us to keep track
of the printer's state, receive a log of the commands being issued, and to ensure that
any command issued to the printer updates these.

Let's describe the printer with the following data structures:

{{< highlight go >}}

type Cursor struct {
    X,Y int
}

type Colour int

const (
    ColorBlack Colour = iota + 1
    ColorBlue
    ColorGreen
    ColorRed
)

type LineStyle int

type Printer struct {
    writer *io.Writer
    position Cursor
    colour Colour
    lineStyle LineStyle
    logger *log.Logger
}
{{< /highlight >}}

Now we can only send commands to the printer via public interfaces of the `Printer` struct,
and we can ensure that these methods correctly update the printer's state variables to match.

Now we can map the commands we're going to use to methods of the `Printer` struct so we can make use of them:

| Printer Command | Method Signature                                       |
|:---------------:|:-----------------------------------------------------------------|
| `M`             | `MoveTo(x,y int)`                                                |
| `D`             | `DrawTo(x,y int)`                                                |
| `C`             | `SetColor(c Color)`                                              |
| `L`             | `SetLineStyle(dashLen int)`                                      |
| `X`             | `DrawAxis(dir, interval, ticks int)`                             |
| `P`,`S`,`Q`     | `GraphicalWrite(text string, size int, orientation Orientation)` |
| `I`             | `SetOrigin()`                                                    |

Notice that we've combined the `P`,`S` and `Q` commands into a single function.
This function will take the parameters for all three commands, set the text properties,
print the text, and then reset them to the defaults.

For full details of the printer implementation, check the `printer` sub-package in the code.

# 3. Planning our Output

First of all, we need to plan out how we're going to construct our desired output.

The **CGP-115** has a printable width of 96mm, split into 480 steps of 0.2mm each,
the paper is on a continuous feed roll, giving an effectively infinite vertical print area,
although the maximum area the device can work on at a given time is limited to 399.6mm

In order to get more detail in the X-axis, we'll print the chart vertically.
We're working with time-series data, so our calculations will be made easier if we make our
chart's dimensions align with multiples of 60, as well as multiples of 5.
In this case we'll use a chart length of 120mm, giving a total length of 600 steps.


We'll need to allow some space on the right-hand edge for the X-axis labels, so we'll make
the Y-axis 80mm tall, allowing 16mm of space for the X-axis labels.


So we've got a planned output that looks something like this:

![Chart Mockup](/post/images/graph-plan.svg)

## Deconstructing the steps.

Now we know what we're going to output, we need to deconstruct that output, and determine
how we'll produce the output using the commands available.

We'll break things down into the following steps:

1. Draw the graph axes
2. Calculate the label at each 'tick' on the axes
3. Draw the axes labels
4. For each data series, draw it in the plot area
5. Add a title.

# 4. Modelling our Chart and Data

For this example we'll be drawing a chart of CPU usage per host over a period of 24 hours,
we'll definitely want to draw other charts. 
It'd be good to make our implementation of the chart drawing logic more generic so we can
plot any kind of time-series data.

We can describe the chart itself with the following data structure:

{{< highlight go >}}
type TimeseriesChart struct {
    Start time.Time
    End time.Time
    SampleInterval time.Duration
    Title string
    Unit string
    Series [][]float64
}
{{< /highlight >}}

This structure, and the accompanying implementation makes a few assumptions, 
which should be safe for our purposes:

* All data series are chronologically aligned
* There are no missing samples.
* The sample interval is regular and uniform over the time window.
* All data series are of the same units.
* All values in the series are positive values.
* All series will range from zero.


# 5. Producing some Output

First thing we need to do is enter graphics mode. This is done by sending an ASCII `DC2`
(Device Control 2), `0x12` followed by a terminating CRLF. We'll also reset the printer's
internal coordinates to zero so we know where we are.

{{< highlight go >}}
writer.Write([]byte{0x12, 0x0d, 0x0a})
writer.Write([]byte("I\r\n"))
{{< /highlight >}}

Now we're in graphics mode with the coordinates reset.

### Drawing the Axes

We can start drawing. As mentioned, we'll start with the chart axes.
Before we can start drawing we need to feed the paper in, as we'll be drawing the chart 'bottom-up,'
so we'll need the space to draw in.

We'll move the page 120mm down to give us enough room for the chart with the following

    printer.MoveTo(0, -600) // M0,-600
    printer.SetOrigin()     // I

This will move the page 600 steps down. Giving us room to draw the chart up the page.

Now we can draw our first axis:

    printer.DrawAxis(0, 25, 15) // X0,25,15
    
This will draw an axis horizontally on the page, with a tick every 5mm, and a total length
of 80mm.

Now we're in position to draw the X-axis:

    printer.DrawAxis(0, 25, 24) // X1,25,24
    
This draws an axis vertically, with a tick every 5mm, and a total length of 120mm.


### Labelling the Axes

Now we've drawn the axes, we need to add labels to them.

In order to label the X-axis, we'll get the duration of the chart's time window,
and divide that by the number of ticks to get the time shift per tick.

{{< highlight go >}}
timePerTick := chart.End.Sub(time.Start) / 24
{{< /highlight >}}

We now need to iterate over the ticks, and print out a label at each tick,
labelling the correct time based on `timePerTick`, the current tick index,
and the start time of the chart:

{{< highlight go >}}
for i := 0; i < 24; i++ {
    t := chart.Start.Add(time.Duration(i) * timePerTick)
    printer.MoveTo(410, 25*i)
    printer.GraphicalWrite(t.Format("15:04"), 0, TextLeftToRight)
}
{{< /highlight >}}

In order to properly scale the Y-axis, we'll need to know the maximum range of all
data series.

We'll take the maximum value of all series, and subtract the minimum value, then divide 
by the height of the chart to get the number of units for each step of the Y-axis:

{{< highlight go >}}
unitsPerStep := (chart.Max() - chart.Min())/400.0
{{< /highlight >}}

We'll leave the rounding until we draw each sample on the chart, 
in order to reduce rounding errors.

Now we can label the Y-axis in a similar way to the X-axis:

{{< highlight go >}}
for i := 0; i < 25; i++ {
    label := fmt.Sprintf("%.3f%s", unitsPerStep*25, chart.Unit)

    // With a print size of 0, each character is 6 steps (1.2mm) wide.
    // Offset by this (and a bit more) to right-align the labels to the axis
    printer.MoveTo(400 - (25*i), -6*len(label)-5 )
    
    printer.GraphicalWrite(label, 0, TextBottomToTop)
}
{{< /highlight >}}

### Plotting the data

Now we're finally ready to start putting data onto the chart.
We'll iterate over each data series, and for each one, we'll draw a line to it from the current point.

First, we need to check how long the series are, so we know how far to advance along the X-axis
for each sample:

{{< highlight go >}}
    if len(chart.Series[0]) > 600 {
        panic("Cannot plot more than 600 samples. Reduce number of samples.")
    }

    stepsPerSample := int(math.Floor( float64(len(chart.Series[0])) / 600.0 ))
{{< /highlight >}}

For each series we'll set the colour and line style based on its index in the list of series.
The plotter has 4 pen colours, and 16 line styles, giving us a maximum of 64 series before
we start running into duplicates.

{{< highlight go >}}
for i, series := range chart.Series {
    printer.SetColor(i%4)
    printer.SetLineStyle( (i/4)%16 )

    y0 := int(math.Round(series[0] / unitsPerStep))
    printer.MoveTo(y0, 0)
    
    for x, v := range series {
        y := int(math.Round(v / unitsPerStep))
        printer.DrawTo(y, x)
    }
}
{{< /highlight >}}

### Adding the Title and Presenting

The last thing we need to add to our chart is a title, so let's do that:

{{< highlight go >}}
// With a character size of 5 each character is 36 steps (6mm) wide
// Offset by -half the width of the title to center it
printer.MoveTo(12, -36*(len(chart.Title)/2))
printer.GraphicalWrite(chart.Title, 5, TextBottomToTop)
{{< /highlight >}}

And finally, we can move the page up some (and reset the origin) to present the finished chart:

{{< highlight go >}}
printer.MoveTo(0, -800)
printer.SetOrigin()
{{< /highlight >}}

