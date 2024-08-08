While developing my voxel engine, I wanted to be able to orient all blocks. I wanted the orientation to include a rotation, and a flip (inverting the X, Y, or Z axes).

So I set out to find a way to implement these orientations for my game. The way I ended up doing it was by using massive match expressions (in rust). You can also use lookup tables, but the match expression is possible faster. If you were using a different language than Rust, you would be using a switch statement (or similar syntax).

I also wanted to be able to orient orientations, which means that if I have an orientation that rotates and flips the block/mesh, then I can apply it to another orientation to get a new orientation that is distinct from the original two.

[Explanation](howto.md)

Here are some videos of orientations being oriented:

### X-Axis
https://github.com/user-attachments/assets/e996879d-c822-47cd-880f-759b920a9bde
### Y-Axis
https://github.com/user-attachments/assets/223d1cf9-9239-4eec-a4a3-b249de2b542b
### Z-Axis
https://github.com/user-attachments/assets/9e28f333-8647-42bd-afb4-99da1aa50472

### And here's what flipping looks like
https://github.com/user-attachments/assets/5418b083-0979-48b7-a8f1-e47d929b24c4
