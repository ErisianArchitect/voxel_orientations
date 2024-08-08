While developing my voxel engine, I wanted to be able to orient all blocks. I wanted the orientation to include a rotation, and a flip (inverting the X, Y, or Z axes).

So I set out to find a way to implement these orientations for my game. The way I ended up doing it was by using massive match expressions (in rust). You can also use lookup tables, but the match expression is possible faster. If you were using a different language than Rust, you would be using a switch statement (or similar syntax).

I also wanted to be able to orient orientations, which means that if I have an orientation that rotates and flips the block/mesh, then I can apply it to another orientation to get a new orientation that is distinct from the original two.

Here are some videos of orientations being oriented:

### X-Axis



https://github.com/user-attachments/assets/e996879d-c822-47cd-880f-759b920a9bde



<!-- ![x_axis_rotation](https://youtu.be/fRf-suuhGxs) -->
