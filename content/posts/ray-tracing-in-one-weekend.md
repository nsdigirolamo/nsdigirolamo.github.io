---
title: "Ray Tracing in One Weekend"
date: 2024-02-14T22:53:21-05:00
summary: "This post is going to be about the ray tracer I'm writing in C++.
It's based heavily around the free online textbook series *Ray Tracing in One
Weekend* by Peter Shirley, Trevor Black, and Steve Hollasch. I've actually done
this same project in pure C and Rust, but want to try again in C++to gain a
deeper understanding of ray tracing and  principles."
---

This post is going to be about the
[ray tracer](https://github.com/nsdigirolamo/ray-tracing-playground) I'm writing
in C++. This is my third attempt now at making a ray tracer. My previous
projects on this topic can be found
[here](https://github.com/nsdigirolamo/ray-tracing-in-one-weekend) and
[here](https://github.com/nsdigirolamo/nicks-ray-tracer). All three projects are
based heavily around the free online textbook series
[*Ray Tracing in One Weekend*](https://raytracing.github.io/) by Peter Shirley,
Trevor Black, and Steve Hollasch.

I want to gain a deeper understanding into how ray tracers are created. My
previous two attempts felt like I was just following instructions rather than
actually learning anything. This time around I intend on spending more time
gaining a proper understanding of the things I'm creating, rather than just
copy/pasting from the textbook. I'm still going to be using the textbook for
feature ideas, but I am going to try to avoid just reading the code explanations
and just making a one-to-one copy of their implementation.

Most importantly, I'm going to try my best to solve problems with my own
research (googling) and only resort to the textbook if I get really stuck.

![test](/ray-tracing-in-one-weekend/testing_normals.png)

Above is one of the first images I created. Intersections with the sphere
returned the surface normals at each intersection point, and then I used those
normals to color the sphere. After some work, I managed to generate something
like the next image.

![banding.png](/ray-tracing-in-one-weekend/banding.png)

I added a diffuse material and the ability to check for plane intersections. But
 clearly, something is wrong. What's going on with all that weird banding? The
 issue has to do with my intersection code. Below is an approximation of what's
 going on:

```
if ( /** check for ray intersection */ ) {

	double distance = /** distance along the ray where the intersection occurs */

	Point intersection = ray.origin + ray.direction * distance;

	Hit hit { intersection };

	return hit;
}
```

Hopefully the above pseudocode isn't too opaque. I check to make sure an
intersection exists, and if it does I get the distance along my ray where the
intersection occurs. The issue comes along when I start recursively scattering
rays.

Once an intersection occurs, I spawn a new ray with its origin at the
intersection point. The ray will start at this origin, and then look along
whatever direction its pointing for an intersection with an object.

But the origin of my scattered ray is going to be on the very surface of an
object, so the ray will detect an intersection at its own origin! I suspect
the banding comes from some weird floating point issues, where sometimes the
origin isn't *exactly* on the surface of the plane, so the rays fail to
intersect with their own origins.

Below solves the issue. I just need to throw out any intersections below some
small minimum distance.

```
double minimum_distance = 0.0001

if ( /** check for ray intersection */ ) {

	double distance = /** distance along the ray where the intersection occurs */

	if (distance < minimum_distance) { return {}; }

	Point intersection = ray.origin + ray.direction * distance;

	Hit hit {intersection};

	return hit;
}
```

Below is the same image with the issue solved. Much nicer looking. I found the
solution to this problem after a quick google search. I found
[this reddit post](https://www.reddit.com/r/rust/comments/nacl51/weird_banding_on_custom_ray_tracer/)
with basically the same issue I was having, and the
[top comment](https://www.reddit.com/r/rust/comments/nacl51/weird_banding_on_custom_ray_tracer/gxted9h/)
explained what they thought was the issue, and that's what led me to the
solution. Ironically, if I had been reading the *Ray Tracing in One Weekend*
textbook as religiously as I was in my last two ray tracing projects, I would
have quickly found their
[solution](https://raytracing.github.io/books/RayTracingInOneWeekend.html#diffusematerials/fixingshadowacne)
to this problem. They seemed to agree with me that the issue came down to some
floating point rounding issues. Regardless, I'm happy I was able to find the
fix on my own without simply being given the solution.

![fixed_banding.png](/ray-tracing-in-one-weekend/fixed_banding.png)

The next step for me was the implement the other materials from the textbook.
I knew I needed metals and dielectrics. Below are my metals. The metals look a
bit better because I'm using 100 samples per pixel and a step depth of
50 bounces. The previous images only used 10 for both values.

![metals.png](/ray-tracing-in-one-weekend/metals.png)

And dielectrics.

![dielectrics.png](/ray-tracing-in-one-weekend/dielectrics.png)

Next was depth of field.

![depth_of_field.png](/ray-tracing-in-one-weekend/depth_of_field.png)

This image took an embarrassingly long time time code. Internally, the code
creates this blur effect with a thin disk behind the view port. The view port
exists in the 3D scene, and where-ever the view plane intersects 3D space,
that is where objects are most in focus.

For the longest time, my depth of field effect was just not working. The
entire scene was blurry instead of just the closer and further away objects.
Eventually, I tracked down the reason.

My ray tracer's camera has an origin and a viewport. The origin is where rays
originate from, and the viewport is where the rays will be directed. For each
pixel, I calculate the pixel's location in 3D space based on this viewport. So
to color a pixel in the resulting image, I just point the ray towards that
pixel's location on the viewport. And this works great if everything is always
in focus.

But when I wanted to add a defocus, the ray's origin was now no longer the same
as the camera origin. My pinhole camera became a very-wide-hole camera, and thus
the rays were no longer pointing towards the pixel location in space. All my
rays were offset by the random defocus offset, so everything was blurry! To fix
this, all I had to do was adjust for the offset and now only the things outside
of the viewport's plane are blurry, as I intentioned!
