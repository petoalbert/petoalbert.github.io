---
layout: post
title: "Modeling a grass field in three.js"
date: 2016-10-22 15:26:00
author: Albert Pető
---
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<a href="https://petoalbert.github.io/flora"><img src="/assets/images/flora-screenshot.png"/></a>

This post is about a three.js based application which shows a grass fild with
wind blowing in it. You can try it out in your browser [here][flora]. The source
code is on [github][source]. Here I will give a brief description about some of
the techniques which I have used in the application. A lot of things
are left out, but I plan to add them later to the post. Although the post is not
meant to be a comprehensive tutorial, please read further if you are interested
in the details of my work.

On part of the reader, I will assume a basic understanding of OpenGL and how a
shader works.

## A blade of grass

The javascript code loads an `.obj` file containing the vertices of a 2D blade
of grass, exported from a Blender model. The application uses the object loader
from a three.js [example][threejs-object-loader]. In the loaded model,
the alignment is assumed to be the following: the z axis is always zero
(i.e. the blade is in the x-y plane), and
the longitudinal axis is y, with the bottom of the grass having 0 value for
the y coordinate.

<img src="/assets/images/grass-blender.png"/>

## Instancing

Once the application has loaded the blade vertices, it needs to draw a lot of
them. The naive solution would be to make a [THREE.Mesh][threejs-mesh-class] with the
same vertices for each blade of grass, add each one to the scene and then
make them unique by changing there position and appearance. This, however, would
result in a very slow code if we really wanted to draw a lot of pieces, because
each piece would need a separate rendering call.

We can overcome this problem with instancing, which basically means that we
only use one rendering call for all of the pieces. Since every blade of grass
is based on the same vertices, those only need to be defined once, and the GPU
will use them for each blade. On the other hand, we have to associate some
unique values with each blade (rotation, position, color, etc.), so that the
geometry shader can produce different vertices and the fragment shader can
produce different colors. These values will be stored in
`THREE.InstancedBufferAttribute` objects. If you would like to read more about
OpenGL instancing, read this [great article][instancing-post], or have a look
at this [three.js example][threejs-instancing-example].

From now on, please keep in mind, that we have to use the shaders for for
every kind of transformation, beacuse the rendering loop for every blade
begins with the same set of vertices. For example, if we want to bend a specific
blade of grass in a given way, we can't just transform the vertices in the
javascript code, because that would affect every other blade of grass. We have
to do that in the shader, too.

To make each blade of grass look different, I have used the following
transformations in the geometry shader (i.e. on each vertex of a blade of
grass), in the following order:

### 1. Scaling the width and height

### 2. Bending the grass piece

This could be done in many ways. Don't forget that
when the geometry shader is working on a vertex, it doesn't know anything
about the other vertices of the blade of grass, so in a mathematical sense
we can only use a function $$f(v,a,u)$$ to calculate the transformed vertex,
where the v,a,u arguments are the actual vertex, the attributes of the
current blade of grass, and uniforms.

In my model, every blade has a constant
curvature, i.e. its y and z coordinates fit on a circular arc, which is
calculated like this in the shader (simplified version):
{% highlight glsl %}
// curve is different for each blade of grass and describes the curvature
// r is the radius of the circular arc
// len is the height of the grass piece
float r = len/(curve);
float angle = vPosition.y/r;
vPosition.z = cos(angle)*r-r;
vPosition.y = sin(angle)*r;
{% endhighlight %}
<!--* this is only used to correct syntax highlighting in the editor-->

The shader uses the mathematical identity $$\theta = s/r$$, for a circular
arc of length $$s$$ and radius $$r$$, where $$\theta$$ is the subtended angle
in radians.
For a vertex of a given blade of grass, $$s$$ is the $$y$$ coordinate of the
vertex, and $$r$$ is constant to the whole blade. The below picture shows
arcs with the same length, but with different radius, in red:

<img class="center" width="50%" src="/assets/images/curvature.png"/>

### 3. Rotating the blade along the y axis

### 4. Translating the blade in the x-z plane

Besides the geometry transformations, each blade has a slightly different color,
which is set in the fragment shader.

## Wind

To make each piece look more realistic, we need to add some movement to them,
which, in the case of this application, consist of two parts: constant movement
and gusts of wind.

In both cases, the movement is present by changing the curvature of the
blade with respect to time. This is accomplished by extending the
[previously presented snippet](#bending-the-grass-piece) from the geometry
shader:

{% highlight glsl %}
// curve is different for each blade of grass and describes the curvature
// r is the radius of the circular arc
// len is the height of the grass piece
float r = len/(curve+windEffects); // notice that we added windEffects
float angle = vPosition.y/r;
vPosition.z = cos(angle)*r-r;
vPosition.y = sin(angle)*r;
{% endhighlight %}
<!--* this is only used to correct syntax highlighting in the editor-->

### Constant movement

This movement is a very low-amplitude movement, but even this little movement
helps get a more natural feel, where not everything is 100% static.
Each blade will have a different amplitude and frequency, to make
it look natural and not too synchronized.
{% highlight glsl %}
// amplitude and freq are unique for each blade of grass
float r = len/(curve+amplitude*sin(freq*time));
{% endhighlight %}
<!--* this is only used to correct syntax highlighting in the editor-->

### Wind gust

The scene also has short-term wind gusts, about which we have the following
assumptions:

1. A wind gust has the same direction everywhere and is parallel to the
   x-z (horizontal) plane
2. A gust has its effects for a finite amount of time, and has a starting point
   and starting time. These assumptions are needed because we only want to
   model the gust near the "player's" position.
3. A gust has a speed, which is the same everywhere
4. A wind gust is only in effect for a finite amount of time, and there can
   only be one active wind gust at any time.

For every point in the OpenGL space, the shader needs to know when the blow
will reach that point, or if it has reached that.
Depending on this state, the shader will derive the strength of the gust,
which will gradually increase until the gust reaches that point, and will
gradually decrease after. It is also important for the movement to start
and end slowly, otherwise it may look unnatural. Taking into account these
criteria, I have chosen the [gaussian function][gaussian-wikipedia].

<img class="center" width="80%" src="/assets/images/gaussian.png"/>

The above picture shows a basic implementation of the gust strength at a
given point. However, I think that the settling phase can usually last longer
and contain some spring-like back-and-forth movement, so I made it longer and
multiplied it with a cosine function:

<img class="center" width="80%" src="/assets/images/gaussian-modified.png"/>

This is the simplified part of the geometry shader that computes the wind strength:
{% highlight glsl %}
float wind()
{
  // The transitional stage is governed by one half of a Gaussian function,
  // while the maximum deflection stage is constant (-1.0 or 1.0)
  if (t < windRiseTime) {
    return exp(-pow(abs(t-windRiseTime),2.0)/(2.0*pow(spread,2.0)));
  } else if (t > (windRiseTime+len)) {
    float x = t-windRiseTime-len;
    // Do a little more than a half cosine in the settling phase
    float f = 0.55*2.0*PI/windSettlingTime;
    return cos(f*x)*exp(-pow(x,2.0)/(2.0*pow(settlingSpread,2.0)));
  } else {
    return 1.0;
  }
}
{% endhighlight %}
<!--* this is only used to correct syntax highlighting in the editor-->

Another important thing is the blade's rotation about its longitudinal (y) axis.
If it does not align with the gust direction, then, besides the bending,
the gust will also rotate the piece:

{% highlight glsl %}
/*
 * Return the angle to which the wind in its strongest state will rotate
 * the tip of the grass piece.
 */
float rotateInWind()
{
  // The angle difference in the wind direction and the blade's rotation
  float diff = angleDiff();
  // The wind never rotates more than 90 degrees
  if (diff < -PI/2.0 && diff > -PI) {
    diff = PI+diff;
  } else if (diff < -PI && diff > -3.0*PI/2.0) {
    diff = PI+diff;
  } else if (diff < -3.0*PI/2.0) {
    diff = 2.0*PI+diff;
  }
  // The lowest height on the grass piece at which it can twist
  float twistHeight = unitHeight*0.3;
  // The amount that this vertex will twist
  float twistCoefficient = (position.y-twistHeight)/unitHeight;

  if (position.y < twistHeight) {
    return 0.0;
  } else if (twistCoefficient < 1.0) {
    return diff*twistCoefficient;
  } else {
    return diff;
  }

}
{% endhighlight %}
<!--* this is only used to correct syntax highlighting in the editor-->

Finally, the blade's curvature is modified according to the following line:
{% highlight glsl %}
// the weighting 0.5 is experimentally determined
float r = len/(curve+0.5*windStrength+amplitude*sin(freq*time));
{% endhighlight %}
<!--* this is only used to correct syntax highlighting in the editor-->

Here, I have left out the details of how the wind travels throght the field,
i.e. how the position influences the arguments of the gaussian curve.

## Making the field infinite

The last thing is to be able to move in the grass. The code for the controls
(WASD + mouse) is reused from a [three.js example][threejs-controls], so I
will not write about them.

The more important problem from our perspective is to always draw the blades
of grass around the "player's" current position, so that the field looks
infinite. I will note here that the original blade's are spread out around
the $$(0,0)$$ point in the x-z plane in a square shape, with width $$a$$.
Now, if the player's position in the x-z plane is $$(p_x,p_z)$$, we have to
draw the pieces in a square with width $$a$$, centered at $$(p_x,p_z)$$.
We suppose that the player can't see further than $$a/2$$ distance, because
there is fog.

The problem is solved in the geometry shader: if a blade in its original
position can't be seen from the players position (its distance is more than
$$a/2$$), we translate the blade in the x-z plane by
$$(t_x\cdot a,t_z \cdot a)$$ to the square centered around the player's
position, where $$t_x$$ and $$t_z$$ are appropiate integers:

{% highlight glsl %}
/*
 * Offset the grass piece with integer multiples of spread in the x and z directions
 * to be in the near the players current position.
 */
void calculateViewOffset() {
  vec3 playerPositionInPlane = vec3(playerPosition.x,0,playerPosition.z);
  vec3 diff = playerPositionInPlane - offset;
  viewOffset = offset;
  float dlen = length(diff);
  // spread is the side of the square in which the blades are laid out
  if (dlen > spread/2.0) {
    viewOffset.x += round(diff.x/spread)*spread;
    viewOffset.z += round(diff.z/spread)*spread;
  }
}
{% endhighlight %}
<!--* this is only used to correct syntax highlighting in the editor-->

[flora]: https://petoalbert.github.io/flora
[source]: https://github.com/petoalbert/flora
[threejs-object-loader]: https://threejs.org/examples/#webgl_loader_obj
[threejs-mesh-class]: https://threejs.org/docs/index.html#Reference/Objects/Mesh
[instancing-post]: http://learnopengl.com/#!Advanced-OpenGL/Instancing
[threejs-instancing-example]: https://threejs.org/examples/#webgl_buffergeometry_instancing
[gaussian-wikipedia]: https://en.wikipedia.org/wiki/Gaussian_function
[threejs-controls]: https://threejs.org/examples/#misc_controls_pointerlock
