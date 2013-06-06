# Developing Kinetic Panning for the ESRI Javascript API
- Ryan
- rmordoff
- 2013/06/06
- Code
- draft

Surprisingly ESRI hasn’t implemented kinetic panning for their JS API, while it has been around for a while for their Flex and Silverlight API for some time.   From a forum post it looked like there was a demand for the feature so I figured I would give it a try and see if I could implement it myself.  So here it goes…

In all essence, it is a fairly straight forward problem to solve.

* Once the user has stopped dragging on the map, get the current velocity
* Kick off the kinetic panning
* Continue to loop through a kinetic pan function, each time decreasing the velocity until we get to 0 (or near 0)

Not too involved, but there are a few gotchas along the way.

###Panning programmatically
First thing I did was figure out how if it was possible to pan programmatically.  Digging into the ESRI API I found I was able to piggy back off of their panning functions in the NavigationManager class.  By utilizing their functions it makes our job easier.
Using the the _panInit, _panStart, _pan, and _panEnd functions we are able to get the results we are looking for.

###Calculating Initial Kinetic Panning Velocity
This is where things start to get a little tricky.  When we kick off the kinetic panning loop we need to have the starting velocity, which we calculate when the user is panning.  To get velocity we use the simple formula:
Velocity = Displacement/Time
We can’t simply take the points of the StartDragEvt and EndDragEvt to calculate this.  Users may zig zag back and forth all over the map for a number of seconds and end back at the same point they started at when the drag end event is fired.  This yields an inaccurate velocity.

The approach I took was to calculate the velocity for a specific interval (I think I settled on 50ms), and take the last velocity that was calculated.

    this._dragInterval = setInterval(lang.hitch(this, function() {
      this._setInitVelocity();
    }), this._dragIntervalDuration);
--
    _setInitVelocity : function() {
      this.xVelocity = this._calculateInitVelocity(this._lastDragEvt.screenPoint.x, this._mouseDragEvt.screenPoint.x);
      this.yVelocity = this._calculateInitVelocity(this._lastDragEvt.screenPoint.y, this._mouseDragEvt.screenPoint.y);

      this._lastDragEvt = this._mouseDragEvt;
    },

This will set the starting velocity for the kinetic panning every 50ms.  Once the user completes there panning, the last velocity set is used for the kinetic panning.

For current browsers, this seemed to work perfectly.  For ie8 and below I had issues.  Because I was using a short interval (50ms), the browser wasn’t able to keep up with the callbacks and couldn’t consistently compute the velocity.  For this reason, I had to implement a little hack for older browser to calculate the initial velocity differently.

This was done by utilizing the DragEvent and completely ditching the setInterval.  Whenever the drag event was dispatched the velocity would be set as well.  This isn’t the ideal solution since we don’t know the time interval between drag events (and time is used to calculate velocity).  So instead we estimate it; it ends up working fairly well.

    _onMouseDrag : function(e) {
      this._mouseDragEvt = e;

      // ie8 and below is too slow to calc velocity w/ setInterval,
      // use mouse drag event instead to get a starting estimate velocity
      if (this._isLegacyBrowser()) {
        this._setInitVelocity();
      }
    },
    
###The Panning Loop
This is where most of the magic happens, and it ends up being the easiest part.  First we need to set a new value for the x and y velocity.  I was going to use an easing function to do this but ended up with using a simple linear deceleration coefficient.

    // decrease velocity
    this.xVelocity *= this.deceleration;
    this.yVelocity *= this.deceleration;
  
Whenever we pan we just decrease the velocity by the deceleration coefficient and perform the pan using the internal NaviationManager pan function.

    // set new points
    this._curPanningPoint.screenPoint.x += this.xVelocity;
    this._curPanningPoint.screenPoint.y += this.yVelocity;

    // do the pan
    this.map.navigationManager._pan(this._curPanningPoint);

From there we do a final check to see if both the x and y velocity is below the min velocity threshold, if so, we end the panning.

    if (!this._continuePanning())
      this._kineticPanEnd();
-
    _continuePanning : function() {
      return (Math.abs(this.xVelocity) > this.minEndVelocity || Math.abs(this.yVelocity) > this.minEndVelocity);
    },



