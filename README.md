# Donut with Math

 There has been a sudden resurgence of interest in my [“donut” code from 2006], and I’ve had a couple requests to explain this one. It’s been five years now, so it’s not exactly fresh in my memory, so I will reconstruct it from scratch, in great detail, and hopefully get approximately the same result.
 
----
[“donut” code from 2006]: https://www.a1k0n.net/2006/09/15/obfuscated-c-donut.html
This is the code:
```
             k;double sin()
         ,cos();main(){float A=
       0,B=0,i,j,z[1760];char b[
     1760];printf("\x1b[2J");for(;;
  ){memset(b,32,1760);memset(z,0,7040)
  ;for(j=0;6.28>j;j+=0.07)for(i=0;6.28
 >i;i+=0.02){float c=sin(i),d=cos(j),e=
 sin(A),f=sin(j),g=cos(A),h=d+2,D=1/(c*
 h*e+f*g+5),l=cos      (i),m=cos(B),n=s\
in(B),t=c*h*g-f*        e;int x=40+30*D*
(l*h*m-t*n),y=            12+15*D*(l*h*n
+t*m),o=x+80*y,          N=8*((f*e-c*d*g
 )*m-c*d*e-f*g-l        *d*n);if(22>y&&
 y>0&&x>0&&80>x&&D>z[o]){z[o]=D;;;b[o]=
 ".,-~:;=!*#$@"[N>0?N:0];}}/*#****!!-*/
  printf("\x1b[H");for(k=0;1761>k;k++)
   putchar(k%80?b[k]:10);A+=0.04;B+=
     0.02;}}/*****####*******!!=;:~
       ~::==!!!**********!!!==::-
         .,~~;;;========;;;:~-.
             ..,--------,*/
```      
…and the output

![Demo CountPages alpha](https://media.giphy.com/media/tfoUFS43L13Vgi4zn1/giphy.gif)

At its core, it’s a framebuffer and a Z-buffer into which I render pixels. Since it’s just rendering relatively low-resolution ASCII art, I massively cheat. All it does is plot pixels along the surface of the torus at fixed-angle increments, and does it densely enough that the final result looks solid. The “pixels” it plots are ASCII characters corresponding to the illumination value of the surface at each point: .,-~:;=!*#$@ from dimmest to brightest. No raytracing required.

So how do we do that? Well, let’s start with the basic math behind 3D perspective rendering. The following diagram is a side view of a person sitting in front of a screen, viewing a 3D object behind it.

![image](https://user-images.githubusercontent.com/86287920/205480967-f5f815ad-6a6c-45c6-99ab-171ea252d9b4.png)

To render a 3D object onto a 2D screen, we project each point (x,y,z) in 3D-space onto a plane located z’ units away from the viewer, so that the corresponding 2D position is (x’,y’). Since we’re looking from the side, we can only see the y and z axes, but the math works the same for the x axis (just pretend this is a top view instead). This projection is really easy to obtain: notice that the origin, the y-axis, and point (x,y,z) form a right triangle, and a similar right triangle is formed with (x’,y’,z’). Thus the relative proportions are maintained:

![image](https://user-images.githubusercontent.com/86287920/205480983-025d2061-6cc3-477b-8846-67d669e12e8f.png)

So to project a 3D coordinate to 2D, we scale a coordinate by the screen distance z’. Since z’ is a fixed constant, and not functionally a coordinate, let’s rename it to K1, so our projection equation becomes (x′,y′)=(K1xz,K1yz)

. We can choose K1 arbitrarily based on the field of view we want to show in our 2D window. For example, if we have a 100x100 window of pixels, then the view is centered at (50,50); and if we want to see an object which is 10 units wide in our 3D space, set back 5 units from the viewer, then K1 should be chosen so that the projection of the point x=10, z=5 is still on the screen with x’ < 50: 10K1/5 < 50, or K1 < 25.

When we’re plotting a bunch of points, we might end up plotting different points at the same (x’,y’) location but at different depths, so we maintain a z-buffer which stores the z coordinate of everything we draw. If we need to plot a location, we first check to see whether we’re plotting in front of what’s there already. It also helps to compute z-1 =1z

and use that when depth buffering because:

   - z-1 = 0 corresponds to infinite depth, so we can pre-initialize our z-buffer to 0 and have the background be infinitely far away
   - we can re-use z-1 when computing x’ and y’: Dividing once and multiplying by z-1 twice is cheaper than dividing by z twice.

Now, how do we draw a donut, AKA torus? Well, a torus is a solid of revolution, so one way to do it is to draw a 2D circle around some point in 3D space, and then rotate it around the central axis of the torus. Here is a cross-section through the center of a torus:

![image](https://user-images.githubusercontent.com/86287920/205481052-c317fae8-0fff-45d9-b2b0-e6fbc933e840.png)

So we have a circle of radius R1 centered at point (R2,0,0), drawn on the xy-plane. We can draw this by sweeping an angle — let’s call it θ — from 0 to 2π:

![image](https://user-images.githubusercontent.com/86287920/205481069-b142f8aa-58dd-4bac-bb44-a24cfae08a68.png)

Now we take that circle and rotate it around the y-axis by another angle — let’s call it φ. To rotate an arbitrary 3D point around one of the cardinal axes, the standard technique is to multiply by a rotation matrix. So if we take the previous points and rotate about the y-axis we get:

![image](https://user-images.githubusercontent.com/86287920/205481086-5c134373-26fd-4670-826c-d4a990f5a0c9.png)

But wait: we also want the whole donut to spin around on at least two more axes for the animation. They were called A and B in the original code: it was a rotation about the x-axis by A and a rotation about the z-axis by B. This is a bit hairier, so I’m not even going write the result yet, but it’s a bunch of matrix multiplies.

![image](https://user-images.githubusercontent.com/86287920/205481100-b031becd-9308-461e-88a8-0a343fbb36a2.png)

Churning through the above gets us an (x,y,z) point on the surface of our torus, rotated around two axes, centered at the origin. To actually get screen coordinates, we need to:

    - Move the torus somewhere in front of the viewer (the viewer is at the origin) — so we just add some constant to z to move it backward.
    - Project from 3D onto our 2D screen.

So we have another constant to pick, call it K2, for the distance of the donut from the viewer, and our projection now looks like:

![image](https://user-images.githubusercontent.com/86287920/205481111-49f05707-886c-4178-b0c9-b00fb24ca177.png)

K1 and K2 can be tweaked together to change the field of view and flatten or exaggerate the depth of the object.

Now, we could implement a 3x3 matrix multiplication routine in our code and implement the above in a straightforward way. But if our goal is to shrink the code as much as possible, then every 0 in the matrices above is an opportunity for simplification. So let’s multiply it out. Churning through a bunch of algebra (thanks Mathematica!), the full result is:

![image](https://user-images.githubusercontent.com/86287920/205481117-c27dced3-278f-4273-a396-6105160e43e7.png)

Well, that looks pretty hideous, but we we can precompute some common subexpressions (e.g. all the sines and cosines, and R2+R1cosθ

) and reuse them in the code. In fact I came up with a completely different factoring in the original code but that’s left as an exercise for the reader. (The original code also swaps the sines and cosines of A, effectively rotating by 90 degrees, so I guess my initial derivation was a bit different but that’s OK.)

Now we know where to put the pixel, but we still haven’t even considered which shade to plot. To calculate illumination, we need to know the surface normal — the direction perpendicular to the surface at each point. If we have that, then we can take the dot product of the surface normal with the light direction, which we can choose arbitrarily. That gives us the cosine of the angle between the light direction and the surface direction: If the dot product is >0, the surface is facing the light and if it’s <0, it faces away from the light. The higher the value, the more light falls on the surface.

The derivation of the surface normal direction turns out to be pretty much the same as our derivation of the point in space. We start with a point on a circle, rotate it around the torus’s central axis, and then make two more rotations. The surface normal of the point on the circle is fairly obvious: it’s the same as the point on a unit (radius=1) circle centered at the origin.

So our surface normal (Nx, Ny, Nz) is derived the same as above, except the point we start with is just (cos θ, sin θ, 0). Then we apply the same rotations:

![image](https://user-images.githubusercontent.com/86287920/205481131-1736fe7f-b0f0-4ba7-87db-91ebdca4a408.png)

So which lighting direction should we choose? How about we light up surfaces facing behind and above the viewer: (0,1,−1). Technically this should be a normalized unit vector, and this vector has a magnitude of √2. That’s okay – we will compensate later. Therefore we compute the above (x,y,z), throw away the x and get our luminance L = y-z.

![image](https://user-images.githubusercontent.com/86287920/205481153-c8e09b75-b58f-44a1-b5a0-05f56fcc34b1.png)

Again, not too pretty, but not terrible once we’ve precomputed all the sines and cosines.
