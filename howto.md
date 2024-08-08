I'm going to use Rust code in my examples because that's what language I used to write my engine. I'll try to give explanations of things that are Rust specific where I can.

***It's very important that I mention that all of this code was written for a right-handed coordinate system (Y Up, X Right, Z backward). You will need to modify it to fit your engine's coordinate system. I apologize if that's inconvenient, but this should give you an idea of how to do it.***

# The Types

There are four types that you'll want to create for your orientations:
* `Direction`/`Face`
* `Flip`
* `Rotation`
* `Orientation`

### `Direction`/`Face`
Whether you decide to name this enum `Direction` or `Face` doesn't really matter. I chose `Direction` for my engine, but you're free to call it whatever you like.

Rust:

(Order these however you want as long as your `Up` direction is 0)
```rust
pub enum Direction {
    NegX = 4,
    NegY = 3,
    NegZ = 5,
    PosX = 1,
    PosY = 0,
    PosZ = 2,
}
```
Note: It's important that you set your default Up direction to a value of 0. The reason it's important to set your default Up direction to a value of 0 is so that the default Orientation and Rotation can have a zeroed value. It might make it a little inconvenient in other places in your code, but I managed to work around this requirement just fine.

### `Flip`
`Flip` should be a bitmask with a bit for each axis.

Rust:
```rust
pub struct Flip(pub u8);
```
C:
```C
struct Flip {
    char uint8_t;
}
```

The bits that I use for each axis are as follows (though it's not important):
* `X`: `0b001`
* `Y`: `0b010`
* `Z`: `0b100`

You'll want to have the ability to modify and query these bits to determine if an axis is flipped. `0` means unflipped, `1` means flipped.

### `Rotation`

The `Rotation` type is less simple. `Rotation` consists of an `up` direction and an `angle`. There are 24 rotations in total. 4 rotations for each face of a cube.

For my engine, the angles are:
* `0`: Up
* `1`: Right
* `2`: Down
* `3`: Left

Rust:
```rust
pub struct Rotation {
    up: Direction,
    angle: u8
}
```
C:
```C
struct Rotation {
    Direction up;
    uint8_t angle;
}
```

When you create a new rotation, it is important that the angle is a value between 0 and 4. I'll talk more about this later.

### Finally, the `Orientation` type.
The orientation type puts all the previous types together. The `Orientation` type allows for a total of 72 orientations! Don't be afraid, though. It's not that hard to manage all these orientations because there are some shortcuts that we can take. These shortcuts involve a little bit of code generation, but that shouldn't be too hard to achieve. You can also write it all by hand, but there are two tables with 1152 entries each, and you'll not want to write that by hand I imagine. These tables are important if you want to be able to remap UVs or determine where on the oriented face a UV coordinate is. You'll need that to implement Occlusion masks, which I won't be covering in this write-up.

```Rust
pub struct Orientation {
    rotation: Rotation,
    flip: Flip
}
```

#### Note:
You can also pack your Rotation struct into a single byte where the first 2 bits are the angle and the remaining bits are for the Direction (you'd only need 5 bits total). This means that you can also pack an Orientation into 8-bits (2 bits for angle, 3 bits for direction, and 3 bits for flip).

# Construction
For your `Rotation` type, you'll need a `from_up_and_forward` constructor that allows you to attempt to create a `Rotation` from an up and forward direction. If the forward direction is either the up direction or the inverse of the up direction, you should consider that an error and handle it as such.

I'll show you my implementation of `from_up_and_forward`, but keep in mind that this is for the coordinate system that my engine uses. You'll want to adapt it for your coordinate system and whatever you consider to be the `Up`, `Right`, `Down` and `Left` for each `Direction`/`Face`.
```rust
pub const fn from_up_and_forward(up: Direction, forward: Direction) -> Option<Rotation> {
    Some(Rotation::new(up, match up {
        Direction::NegX => match forward {
            Direction::NegX => return None,
            Direction::NegY => 2,
            Direction::NegZ => 3,
            Direction::PosX => return None,
            Direction::PosY => 0,
            Direction::PosZ => 1
        },
        Direction::NegY => match forward {
            Direction::NegX => 3,
            Direction::NegY => return None,
            Direction::NegZ => 2,
            Direction::PosX => 1,
            Direction::PosY => return None,
            Direction::PosZ => 0
        },
        Direction::NegZ => match forward {
            Direction::NegX => 1,
            Direction::NegY => 2,
            Direction::NegZ => return None,
            Direction::PosX => 3,
            Direction::PosY => 0,
            Direction::PosZ => return None
        },
        Direction::PosX => match forward {
            Direction::NegX => return None,
            Direction::NegY => 2,
            Direction::NegZ => 1,
            Direction::PosX => return None,
            Direction::PosY => 0,
            Direction::PosZ => 3
        },
        Direction::PosY => match forward {
            Direction::NegX => 3,
            Direction::NegY => return None,
            Direction::NegZ => 0,
            Direction::PosX => 1,
            Direction::PosY => return None,
            Direction::PosZ => 2
        },
        Direction::PosZ => match forward {
            Direction::NegX => 3,
            Direction::NegY => 2,
            Direction::NegZ => return None,
            Direction::PosX => 1,
            Direction::PosY => 0,
            Direction::PosZ => return None
        },
    }))
}
```
`from_up_and_forward` will be a neccessary function to have in order to rotate rotations and orient orientations.

# Functionality

Each type has a set of functionality that is necessary to implement orientations in their entirety. I'll start with `Flip` since that's the easiest.
I'm assuming that you've already implemented a way to query the individual bits of your `Flip` type. If you haven't implemented that yet, now would be a good time to do it. I won't give an explanation of that because it's simple bit manipulation that is easy to learn about if you don't know it already.

## `Flip` Methods

### Checking if your mesh indices need to be reversed

When you get to the stage of your implmentation where it's time to apply your orientation code to a block's mesh, you'll need a function/method to determine whether your indices need to be reversed. The easiest way is to XOR each of the axis bits in `Flip`. Here's what that looks like in my project:

```rust
pub const fn reverse_indices(self) -> bool {
    // where self is of type Flip
    self.x() ^ self.y() ^ self.z()
}
```

### Flipping a coordinate

This method for `Flip` is pretty straightforward. My version uses generics.

```rust
pub fn flip_coord<T: Copy + std::ops::Neg<Output = T>, C: Into<(T, T, T)> + From<(T, T, T)>>(self, mut value: C) -> C {
    let (mut x, mut y, mut z): (T, T, T) = value.into();
    if self.x() {
        x = -x;
    }
    if self.y() {
        y = -y;
    }
    if self.z() {
        z = -z;
    }
    C::from((x, y, z))
}
```

## `Direction` Methods

### Inverting a `Direction`

This is easy. You just choose the axis of the opposite sign.

```rust
pub const fn invert(self) -> Self {
    match self {
        Direction::NegX => Direction::PosX,
        Direction::NegY => Direction::PosY,
        Direction::NegZ => Direction::PosZ,
        Direction::PosX => Direction::NegX,
        Direction::PosY => Direction::NegY,
        Direction::PosZ => Direction::NegZ,
    }
}
```

### Flipping a `Direction`

This is also pretty easy (at least in Rust). You simply invert the `Direction` if the axis that it's on is flipped in the `Flip`.

```rust
pub const fn flip(self, flip: Flip) -> Self {
    use Direction::*;
    match self {
        NegX if flip.x() => PosX,
        NegY if flip.y() => PosY,
        NegZ if flip.z() => PosZ,
        PosX if flip.x() => NegX,
        PosY if flip.y() => NegY,
        PosZ if flip.z() => NegZ,
        _ => self
    }
}
```

This code would likely be a little more difficult to write in a language that doesn't have Rust-like match expressions.

## Rotation Methods

### Getting the Up, Down, Left, or Right  `Direction` based on the target `Direction`.
(Keep in mind that this is for my engine's coordinate system, so you'll want to figure out these values for your own engine if they don't line up with my values, but this should give you an idea of how to do it)

#### Up
```rust
/// On a non-oriented cube, each face has an "up" face. That's the face
/// whose normal points to the top of the given face's UV plane.
pub const fn up(self) -> Direction {
    use Direction::*;
    match self {
        NegX => PosY,
        NegY => PosZ,
        NegZ => PosY,
        PosX => PosY,
        PosY => NegZ,
        PosZ => PosY,
    }
}
```
#### Down
```rust
/// On a non-oriented cube, each face has a "down" face. That's the face
/// whose normal points to the bottom of the given face's UV plane.
pub const fn down(self) -> Direction {
    use Direction::*;
    match self {
        NegX => NegY,
        NegY => NegZ,
        NegZ => NegY,
        PosX => NegY,
        PosY => PosZ,
        PosZ => NegY,
    }
}
```
#### Left
```rust
/// On a non-oriented cube, each face has a "left" face. That's the face
/// whose normal points to the left of the given face's UV plane.
pub const fn left(self) -> Direction {
    use Direction::*;
    match self {
        NegX => NegZ,
        NegY => NegX,
        NegZ => PosX,
        PosX => PosZ,
        PosY => NegX,
        PosZ => NegX,
    }
}
```
#### Right
```rust
/// On a non-oriented cube, each face has a "right" face. That's the face
/// whose normal points to the right of the given face's UV plane.
pub const fn right(self) -> Direction {
    use Direction::*;
    match self {
        NegX => PosZ,
        NegY => PosX,
        NegZ => NegX,
        PosX => NegZ,
        PosY => PosX,
        PosZ => PosX,
    }
}
```

##### Getting the Up, Down, Left, Right, Forward, and Backward `Direction`

This is where things start getting a bit complicated. You see, there are two options here. You can make lookup tables, or you can use `match`/`switch`. I chose to use `match` for my implementation, but you're free to turn it into a lookup table if you want to. I'm not sure which choice is better, honestly.

You really only need to implement `Up`, `Left`, and `Forward`. The other three can be extrapolated from the first three. I personally generated code for it, but you can also just retrieve `Up`, `Left`, and `Forward` and invert them. It's likely more optimized for you to generate the code for each.

So anyway, the `Up` direction is the easiest since we already have the `Up` direction specified in the `Rotation` type. Therefore, you can extrapolate the `Down` direction from the `Up` direction and optionally generate optimized code for it.

Next you'll want to implement the `Left` and `Right` directions.

I'll just show you my code for `Left` and you can extrapolate for `Right` (by inverting the `Direction`).

```rust
pub const fn left(self) -> Direction {
    use Direction::*;
    match self.up() {
        NegX => match self.angle() {
            0 => NegZ,
            1 => PosY,
            2 => PosZ,
            3 => NegY,
            _ => unreachable!()
        }
        NegY => match self.angle() {
            0 => NegX,
            1 => PosZ,
            2 => PosX,
            3 => NegZ,
            _ => unreachable!()
        }
        NegZ => match self.angle() {
            0 => PosX,
            1 => PosY,
            2 => NegX,
            3 => NegY,
            _ => unreachable!()
        }
        PosX => match self.angle() {
            0 => PosZ,
            1 => PosY,
            2 => NegZ,
            3 => NegY,
            _ => unreachable!()
        }
        PosY => match self.angle() {
            0 => NegX,
            1 => NegZ,
            2 => PosX,
            3 => PosZ,
            _ => unreachable!()
        }
        PosZ => match self.angle() {
            0 => NegX,
            1 => PosY,
            2 => PosX,
            3 => NegY,
            _ => unreachable!()
        }
    }
}
```
(I know these huge match expressions are kinda ridiculous, but there's really no better way that I could think of to do this)

And finally, the `Forward` direction, which can be used to extrapolate for the `Backward` direction.

```rust
pub const fn forward(self) -> Direction {
    use Direction::*;
    match self.up() {
        Direction::NegX => match self.angle() {
            0 => PosY,
            1 => PosZ,
            2 => NegY,
            3 => NegZ,
            _ => unreachable!()
        }
        Direction::NegY => match self.angle() {
            0 => PosZ,
            1 => PosX,
            2 => NegZ,
            3 => NegX,
            _ => unreachable!()
        }
        Direction::NegZ => match self.angle() {
            0 => PosY,
            1 => NegX,
            2 => NegY,
            3 => PosX,
            _ => unreachable!()
        }
        Direction::PosX => match self.angle() {
            0 => PosY,
            1 => NegZ,
            2 => NegY,
            3 => PosZ,
            _ => unreachable!()
        }
        Direction::PosY => match self.angle() {
            0 => NegZ,
            1 => PosX,
            2 => PosZ,
            3 => NegX,
            _ => unreachable!()
        }
        Direction::PosZ => match self.angle() {
            0 => PosY,
            1 => PosX,
            2 => NegY,
            3 => NegX,
            _ => unreachable!()
        }
    }
}
```

At this point, you should have the ability to query which Direction is on which face.
This is maybe a little confusing, but essentially when you create a rotation, each Direction has been rotated to a new Direction. You sometimes will want to know where this Direction has gone. That's what this functionality is for. There's also functionality that can be implemented for finding the source face, which determines what face exists on what face.

And that's what we'll talk about next! The `reface` method on `Rotation`.

### Reface

`reface` is thankfully very simple. We simply use the functions we just created to line them up with the input direction.

The `reface` method for `Rotation` takes as input a `Direction` and spits out a new `Direction` which has been Rotated.

Here's what mine looks like: (You could also generate a lookup table)

```rust
/// Rotates direction.
pub const fn reface(self, direction: Direction) -> Direction {
    match direction {
        Direction::NegX => self.left(),
        Direction::NegY => self.down(),
        Direction::NegZ => self.forward(),
        Direction::PosX => self.right(),
        Direction::PosY => self.up(),
        Direction::PosZ => self.backward(),
    }
}
```

Pretty easy, right?

### Source Face

Now comes a harder part. You'll want to use the `reface` method to generate the code for `source_face`, a method for `Rotation` that will tell you where a face came from.
My code for `source_face` is a huge match expression that takes up over 200 lines. I'm not going to paste that in here, instead I'll tell you how to generate that code yourself. We'll call this bootstrapping. (You could also just use the bootstrap method as your source_face method, but it won't be optimized)

You'll want to iterate through all `Direction`s and all angles to create all 24 `Rotation`s, then you'll iterate through each `Direction` to reface it by the rotation and determine if the rotated `Direction` is equal to the input direction.

Here's what that looks like in Rust:

```rust
let directions = [
    Direction::PosY,
    Direction::PosX,
    Direction::PosZ,
    Direction::NegY,
    Direction::NegX,
    Direction::NegZ
];
let mut codegen = String::new();
writeln!(codegen, "match self.up() {{");
for up in directions {
    writeln!(codegen, "    Direction::{up:?} => match self.angle() {{");
    for angle in 0..4 {
        writeln!(codegen, "        {angle} => match direction {{");
        let rotation = Rotation::new(up, angle);
        for face in directions {
            let result = directions.into_iter().find(|&dir| face == rotation.reface(dir)).unwrap();
            writeln!(codegen, "            Direction::{face:?} => Direction::{result},");
        }
        writeln!(codegen, "        }}");
    }
    writeln!(codegen, "        _ => unreachable!()");
    writeln!(codegen, "    }}");
}
writeln!(codegen, "}}");
std::fs::write("rotation_source_face.rs", codegen).expect("Failed to write file.");
```

If you're not familiar with Rust and this is hard to understand, I apologize.

Essentially what I'm doing is I'm iterating through each possible `Rotation` by iterating the `Direction`s and the range of angles (0 <= angle < 4) and creating a new Rotation for each of them. Next we want to iterate `Direction`s again in order to get the `face` for the target direction. For each `face`, we again need to iterate through the directions to see which direction when refaced becomes the target face. The answer to this is the return value for the method.

So you should build a match expression/switch statement that is nested. You can also choose to build a lookup table or a flattened match/switch, but the flattened version is a little more complicated if you're not using Rust. From my tests, I don't think it really matters whether you use nested or flattened. I'm pretty sure the compiler will flatten it anyway.

### Rotating Coordinates

Unfortunately this part can't really be generated with code unless you come up with a really fancy algorithm. I don't know what that algorithm would look like, so instead I'll show you my implementation. This is a Rust implmentation that uses generics, so if you're not familiar with Rust it may be difficult to parse, but you should get the general idea. Yet again, you'll want to change these values based on the coordinate system that your engine uses, or otherwise use some function to convert between coordinate systems.

```rust
/// Rotates `coord`.
pub fn rotate<T: Copy + std::ops::Neg<Output = T>, C: Into<(T, T, T)> + From<(T, T, T)>>(self, coord: C) -> C {
    let (x, y, z): (T, T, T) = coord.into();
    C::from(match self.up() {
        Direction::NegX => match self.angle() {
            0 => (-y, -z, x),
            1 => (-y, -x, -z),
            2 => (-y, z, -x),
            3 => (-y, x, z),
            _ => unreachable!()
        },
        Direction::NegY => match self.angle() {
            0 => (x, -y, -z),
            1 => (-z, -y, -x),
            2 => (-x, -y, z),
            3 => (z, -y, x),
            _ => unreachable!()
        },
        Direction::NegZ => match self.angle() {
            0 => (-x, -z, -y),
            1 => (z, -x, -y),
            2 => (x, z, -y),
            3 => (-z, x, -y),
            _ => unreachable!()
        },
        Direction::PosX => match self.angle() {
            0 => (y, -z, -x),
            1 => (y, -x, z),
            2 => (y, z, x),
            3 => (y, x, -z),
            _ => unreachable!()
        },
        Direction::PosY => match self.angle() {
            0 => (x, y, z), // Default rotation, no change.
            1 => (-z, y, x),
            2 => (-x, y, -z),
            3 => (z, y, -x),
            _ => unreachable!()
        },
        Direction::PosZ => match self.angle() {
            0 => (x, -z, y),
            1 => (-z, -x, y),
            2 => (-x, z, y),
            3 => (z, x, y),
            _ => unreachable!()
        },
    })
}
```

As you can see, the code itself is pretty simple. If you need to remap this for your own coordinate system, I recommend that you build a visualizer that allows you to rotate a textured cube with quaternions to help visualize rotations and how the coordinates change. I used a note pad with axes scribbled onto it and flipped it around. That was really hard, so definitely make a visualizer first. That will be so much easier.

### Reorienting a Rotation

This method for `Rotation` is very simple if you've written the `from_up_and_forward` function and have a way to determine the `up` and `forward` directions of the `Rotation`. You will also need the `reface` method. If you skipped out on any of these, you'll want to implement them for the `reorient` method. You don't necessarily need to `reorient` method, but having it will allow you to rotate an already rotated block, which is useful if you want some tool in your game that lets players rotate blocks to their preferred orientation. Otherwise you would need to cycle through all 24 rotations, and that's inconvenient, especially when we get to `reorient` for Orientation, which has 72 variants. It will be easier if you have `reorient` implemented. There's also a `deorient` method that does the inverse of `reorient` (rotates in the opposite direction). I'll talk about that next.

```rust
pub const fn reorient(self, rotation: Self) -> Self {
    let up = self.up();
    let fwd = self.forward();
    let rot_up = rotation.reface(up);
    let rot_fwd = rotation.reface(fwd);
    let Some(rot) = Self::from_up_and_forward(rot_up, rot_fwd) else {
        unreachable!()
    };
    rot
}
```

### Deorienting a Rotation

This method for `Rotation` is the inverse of the `reorient` method. It allows you to use one `Rotation` to rotate in two directions. So you can orient a rotation then deorient it and you'll have the same rotation. If you oriented `n` times then deoriented `n` times, you'll also have the same rotation.

```rust
pub const fn deorient(self, rotation: Self) -> Self {
    let up = self.up();
    let fwd = self.forward();
    let rot_up = rotation.source_face(up);
    let rot_fwd = rotation.source_face(fwd);
    let Some(rot) = Self::from_up_and_forward(rot_up, rot_fwd) else {
        unreachable!()
    };
    rot
}
```

### Inverting a Rotation

This method for `Rotation` is also pretty simple. You'll need the `deorient` method for it to work.

```rust
pub const fn invert(self) -> Self {
    // Substitute Self::UNORIENTED with a constant for the unrotated rotation.
    Self::UNROTATED.deorient(self)
}
```

### Cycle rotations

If you're like me, you may have noticed that you could pack a `Rotation` into a single byte then decontruct it when you need to. This is what I did in my engine because it's an inexpensive operation to extract the `up` and `angle` from a byte. I'll leave that as an exercise to the reader. But in case you decide to pack `Rotation` into a single byte, you can implement the cycle method, which allows you to cycle through all 24 rotations sequentially:

```rust
pub const fn cycle(self, offset: i32) -> Rotation {
    let index = self.0 as i32;
    // I convert into i64 in case offset is i32::MAX for some reason.
    let new_index = (index as i64 + offset as i64).rem_euclid(24) as u8;
    Rotation(new_index)
}
```

### Bonus! Face Angle

This method for `Rotation` will tell you which angle a face is pointing to. In an unoriented cube, each face has an angle of 0. When a face is rotated, it will exist in the space of a different face and occupy it's angle domain. What this means is that the face takes on the face angle relative to the face it occupies.

You'll want to modify this implementation for your coordinate system.

```rust
/// Gets the angle of the source face. 
pub fn face_angle(self, face: Direction) -> u8 {
    use Direction::*;
    match (self.angle(), self.up(), face) {
        (0, NegX, NegX) => 0,
        (0, NegX, NegY) => 3,
        (0, NegX, NegZ) => 1,
        (0, NegX, PosX) => 2,
        (0, NegX, PosY) => 3,
        (0, NegX, PosZ) => 3,
        (0, NegY, NegX) => 2,
        (0, NegY, NegY) => 0,
        (0, NegY, NegZ) => 2,
        (0, NegY, PosX) => 2,
        (0, NegY, PosY) => 0,
        (0, NegY, PosZ) => 2,
        (0, NegZ, NegX) => 3,
        (0, NegZ, NegY) => 2,
        (0, NegZ, NegZ) => 0,
        (0, NegZ, PosX) => 1,
        (0, NegZ, PosY) => 0,
        (0, NegZ, PosZ) => 2,
        (0, PosX, NegX) => 2,
        (0, PosX, NegY) => 1,
        (0, PosX, NegZ) => 3,
        (0, PosX, PosX) => 0,
        (0, PosX, PosY) => 1,
        (0, PosX, PosZ) => 1,
        (0, PosY, NegX) => 0,
        (0, PosY, NegY) => 0,
        (0, PosY, NegZ) => 0,
        (0, PosY, PosX) => 0,
        (0, PosY, PosY) => 0,
        (0, PosY, PosZ) => 0,
        (0, PosZ, NegX) => 1,
        (0, PosZ, NegY) => 0,
        (0, PosZ, NegZ) => 2,
        (0, PosZ, PosX) => 3,
        (0, PosZ, PosY) => 2,
        (0, PosZ, PosZ) => 0,
        (1, NegX, NegX) => 1,
        (1, NegX, NegY) => 3,
        (1, NegX, NegZ) => 1,
        (1, NegX, PosX) => 1,
        (1, NegX, PosY) => 3,
        (1, NegX, PosZ) => 3,
        (1, NegY, NegX) => 2,
        (1, NegY, NegY) => 1,
        (1, NegY, NegZ) => 2,
        (1, NegY, PosX) => 2,
        (1, NegY, PosY) => 3,
        (1, NegY, PosZ) => 2,
        (1, NegZ, NegX) => 3,
        (1, NegZ, NegY) => 2,
        (1, NegZ, NegZ) => 1,
        (1, NegZ, PosX) => 1,
        (1, NegZ, PosY) => 0,
        (1, NegZ, PosZ) => 1,
        (1, PosX, NegX) => 1,
        (1, PosX, NegY) => 1,
        (1, PosX, NegZ) => 3,
        (1, PosX, PosX) => 1,
        (1, PosX, PosY) => 1,
        (1, PosX, PosZ) => 1,
        (1, PosY, NegX) => 0,
        (1, PosY, NegY) => 3,
        (1, PosY, NegZ) => 0,
        (1, PosY, PosX) => 0,
        (1, PosY, PosY) => 1,
        (1, PosY, PosZ) => 0,
        (1, PosZ, NegX) => 1,
        (1, PosZ, NegY) => 0,
        (1, PosZ, NegZ) => 1,
        (1, PosZ, PosX) => 3,
        (1, PosZ, PosY) => 2,
        (1, PosZ, PosZ) => 1,
        (2, NegX, NegX) => 2,
        (2, NegX, NegY) => 3,
        (2, NegX, NegZ) => 1,
        (2, NegX, PosX) => 0,
        (2, NegX, PosY) => 3,
        (2, NegX, PosZ) => 3,
        (2, NegY, NegX) => 2,
        (2, NegY, NegY) => 2,
        (2, NegY, NegZ) => 2,
        (2, NegY, PosX) => 2,
        (2, NegY, PosY) => 2,
        (2, NegY, PosZ) => 2,
        (2, NegZ, NegX) => 3,
        (2, NegZ, NegY) => 2,
        (2, NegZ, NegZ) => 2,
        (2, NegZ, PosX) => 1,
        (2, NegZ, PosY) => 0,
        (2, NegZ, PosZ) => 0,
        (2, PosX, NegX) => 0,
        (2, PosX, NegY) => 1,
        (2, PosX, NegZ) => 3,
        (2, PosX, PosX) => 2,
        (2, PosX, PosY) => 1,
        (2, PosX, PosZ) => 1,
        (2, PosY, NegX) => 0,
        (2, PosY, NegY) => 2,
        (2, PosY, NegZ) => 0,
        (2, PosY, PosX) => 0,
        (2, PosY, PosY) => 2,
        (2, PosY, PosZ) => 0,
        (2, PosZ, NegX) => 1,
        (2, PosZ, NegY) => 0,
        (2, PosZ, NegZ) => 0,
        (2, PosZ, PosX) => 3,
        (2, PosZ, PosY) => 2,
        (2, PosZ, PosZ) => 2,
        (3, NegX, NegX) => 3,
        (3, NegX, NegY) => 3,
        (3, NegX, NegZ) => 1,
        (3, NegX, PosX) => 3,
        (3, NegX, PosY) => 3,
        (3, NegX, PosZ) => 3,
        (3, NegY, NegX) => 2,
        (3, NegY, NegY) => 3,
        (3, NegY, NegZ) => 2,
        (3, NegY, PosX) => 2,
        (3, NegY, PosY) => 1,
        (3, NegY, PosZ) => 2,
        (3, NegZ, NegX) => 3,
        (3, NegZ, NegY) => 2,
        (3, NegZ, NegZ) => 3,
        (3, NegZ, PosX) => 1,
        (3, NegZ, PosY) => 0,
        (3, NegZ, PosZ) => 3,
        (3, PosX, NegX) => 3,
        (3, PosX, NegY) => 1,
        (3, PosX, NegZ) => 3,
        (3, PosX, PosX) => 3,
        (3, PosX, PosY) => 1,
        (3, PosX, PosZ) => 1,
        (3, PosY, NegX) => 0,
        (3, PosY, NegY) => 1,
        (3, PosY, NegZ) => 0,
        (3, PosY, PosX) => 0,
        (3, PosY, PosY) => 3,
        (3, PosY, PosZ) => 0,
        (3, PosZ, NegX) => 1,
        (3, PosZ, NegY) => 0,
        (3, PosZ, NegZ) => 3,
        (3, PosZ, PosX) => 3,
        (3, PosZ, PosY) => 2,
        (3, PosZ, PosZ) => 3,
        _ => unreachable!(),
    }
}
```

Alright, that covers it for `Rotation`. There may be more useful functionality that you can create for `Rotation`. In my engine, I have functionality for rotating by each axis.
To do that you just create `Rotation`s that represent a single rotation around each axis, then you use `reorient` and `deorient` to rotate around the axis. Bonus points if you use 4 `Rotation`s per axis so that you can rotate by an angle amount.

Now we'll move onto the methods for `Orientation`. These will be a lot easier since they build off of what we wrote for `Direction`, `Flip` and `Rotation`.

## `Orientaton` Methods

### Refacing a Direction

This method for `Orientation` just uses the `Rotation` and `Flip` of the `Orientation` to transform the input `Direction`.

```rust
/// `reface` can be used to determine where a face will end up after orientation.
/// First rotates and then flips the face.
pub const fn reface(self, face: Direction) -> Direction {
    let rotated = self.rotation.reface(face);
    rotated.flip(self.flip)
}
```

### Source Face

Getting the source face of an `Orientation` is easy. We do the same thing as `reface` except we use the `source_face` method for `Rotation` instead of `reface`.
It's similar to the `reface` method.

```rust
/// This determines which face was oriented to `face`. I hope that makes sense.
pub const fn source_face(self, face: Direction) -> Direction {
    let flipped = face.flip(self.flip);
    self.rotation.source_face(flipped)
}
```
Notice that it uses the `source_face` method of the rotation while `reface` uses the `reface` method.
You may notice the pattern that things are first rotated and then flipped. That's how you'll want to implement the ordering of transformations otherwise things won't make sense because flipping the Y axis will be dependent on the rotation. Rotating and then flipping means that you can logically flip a rotation.

### Reorienting an Orientation

It was actually a little difficult for me to figure this one out. I'll let you look at my code for this method on `Orientation`.

```rust
/// Apply an orientation to an orientation.
pub const fn reorient(self, orientation: Orientation) -> Self {
    let up = self.up();
    let fwd = self.forward();
    let reup = orientation.reface(up);
    let refwd = orientation.reface(fwd);
    let flip = self.flip.flip(orientation.flip);
    let flipup = reup.flip(flip);
    let flipfwd = refwd.flip(flip);
    let Some(rot) = Rotation::from_up_and_forward(flipup, flipfwd) else {
        // This is unreachable because `flipup` and `flipfwd` will always be orthongal to each other.
        unreachable!()
    };
    Orientation::new(rot, flip)
}
```

### Deorienting an Orientation

Deorient is similar to `reorient`, except it uses `source_face` instead of `reface`.

```rust
/// Remove an orientation from an orientation.
/// This is the inverse operation to [Orientation::reorient].
pub const fn deorient(self, orientation: Orientation) -> Self {
    let up = self.up();
    let fwd = self.forward();
    let reup = orientation.source_face(up);
    let refwd = orientation.source_face(fwd);
    let flip = self.flip.flip(orientation.flip);
    let flipup = reup.flip(flip);
    let flipfwd = refwd.flip(flip);
    let Some(rot) = Rotation::from_up_and_forward(flipup, flipfwd) else {
        unreachable!()
    };
    Orientation::new(rot, flip)
}
```

### Transform a coordinate

This method is really one of the most important methods for `Orientation`. It is what allows you to orient coordinates. First the coordinate is rotated using the `rotate` method on `Rotation` (which should accept the same type that you accept for your `transform` method). The next step is to take that rotated coordinate, and flip it using the `flip_coord` method on `Flip`.

Here's what that code looks like

```rust
/// If you're using this method to transform mesh vertices, make sure that you 
/// reverse your indices if the face will be flipped (for backface culling). To
/// determine if your indices need to be reversed, simply XOR each axis of the Orientation's Flip.
/// This method will rotate and then flip the coordinate.
pub fn transform<T: Copy + std::ops::Neg<Output = T>, C: Into<(T, T, T)> + From<(T, T, T)>>(self, point: C) -> C {
    let rotated = self.rotation.rotate(point);
    self.flip.flip_coord(rotated)
}
```

## Optional: Generating tables for remapping face coordinates

There are two methods that you can implement for `Orientation` that are incredibly useful if you're creative enough to know how to use them. These methods are called `map_face_coord` and `source_face_coord`. I use the `source_face_coord` to determine where a position on the unoriented face is on the oriented face. If that's hard to understand, I apologize, it's hard to explain. Being able to determine the `source_face_coord` is useful for face occlusion. In my engine, I have an `Occluder` type for face occlusion (all 6 faces can be occluded by a neighboring voxel). But here's the thing, with occlusion you don't want to make it so that the sides of stair blocks are meshed if they are occluded. You want to be able to occlude those as well. So I created occlusion masks of various sizes that you can apply to the faces of your block. There's the `Occluder`, and the `Occludee`. The `Occluder` occludes, and the `Occludee` is occluded by the `Occluder`. The `Occluder` must completely cover the `Occludee` in order for the `Occludee` to be considered occluded. That functionality is a lot more complicated to implement, and I may talk about it in another writeup, but these methods should give you the building blocks to help you with implementing your own face occluders, which will be necessary if you want blocks that aren't cube shaped.

### Map Face Coord

This method tells you where on the target face a source coordinate is. I hope that makes sense, because I sometimes get confused with which is which when thinking about it. Essentially, if your source face is rotated 90 degrees clockwise, you would expect that the bottom-left coordinate is now in the top-left. So if you were to feed `(-1, 1)` (bottom-left) into this method where the target face's source face is rotated 90 degrees clockwise, you would expect to get the output of `(-1, -1)` (top-left).

Here's my implementation:

```rust
pub fn map_face_coord<T: Copy + std::ops::Neg<Output = T>, C: Into<(T, T)> + From<(T, T)>>(self, face: Direction, uv: C) -> C {
    let table_index = map_face_coord_table_index(self.rotation, self.flip, face);
    let coordmap = orient_table::MAP_COORD_TABLE[table_index];
    coordmap.map(uv)
}
```

Yes, really. But what is this `MAP_COORD_TABLE`? Why, it's a table of 1152 elements describing how to manipulate the coordinates in order to map the coordinate.

I'll describe to you how to generate this table and also how to implement `map_face_coord_table_index`.

You'll need an enum called `AxisMap` that looks like this:

```rust
pub enum AxisMap {
    PosX,
    PosY,
    NegX,
    NegY,
}
```

You'll also need a type called `CoordMap`

```rust
pub struct CoordMap {
    x: AxisMap,
    y: AxisMap
}
```

And you'll need methods for `AxisMap` and `CoordMap` that map coordinates.

```rust
impl AxisMap {
    pub fn map<T: Copy + std::ops::Neg<Output = T>, C: Into<(T, T)>>(self, coord: C) -> T {//           He's crying because of how hard this code was to write.
        let (x, y): (T,T) = coord.into();
        match self {
            AxisMap::PosX => x,
            AxisMap::PosY => y,
            AxisMap::NegX => -x,
            AxisMap::NegY => -y,
        }
    }
}
```

```rust
impl CoordMap {
    pub fn map<T: Copy + std::ops::Neg<Output = T>, C: Into<(T, T)> + From<(T, T)>>(self, coord: C) -> C {
        let coord: (T, T) = coord.into();
        let coord = (self.x.map(coord), self.y.map(coord));
        C::from(coord)
    }
}
```

Pretty straightforward.

#### Naive Map Face Coord

The naive implementation for `map_face_coord` is complicated and was difficult to figure out.

I won't go through the trouble of explaining how it works because it's hard to explain. I think the code does a better job explaining it than I could.

```rust
fn map_face_coord_naive(orientation: Orientation, face: Direction) -> CoordMap {
    // for a more optimized implementation.
    // First get the source face
    let source_face = orientation.source_face(face);
    // next, get the up, right, down, and left for the source face and arg face.
    let face_up = face.up();
    let face_right = face.right();
    let src_up = source_face.up();
    let src_right = source_face.right();
    let src_down = source_face.down();
    let src_left = source_face.left();
    // Next, reface the src_dir faces
    let rsrc_up = orientation.reface(src_up);
    let rsrc_right = orientation.reface(src_right);
    let rsrc_down = orientation.reface(src_down);
    let rsrc_left = orientation.reface(src_left);
    // Now match up the faces
    let x_map = if face_right == rsrc_right {
        AxisMap::PosX
    } else if face_right == rsrc_up {
        AxisMap::NegY
    } else if face_right == rsrc_left {
        AxisMap::NegX
    } else {
        AxisMap::PosY
    };
    let y_map = if face_up == rsrc_up {
        AxisMap::PosY
    } else if face_up == rsrc_left {
        AxisMap::PosX
    } else if face_up == rsrc_down {
        AxisMap::NegY
    } else {
        AxisMap::NegX
    };
    CoordMap {
        x: x_map,
        y: y_map
    }
}
```

### Source Face Coord

This method will tell you where on the source face a target coordinate is. Think of it like this: You have an unoriented coordinate on a target face and you want to know where that coordinate is on the oriented source face. The method to do that is similar to the method we used for `map_face_coord`.

```rust
fn source_face_coord_naive(orientation: Orientation, face: Direction) -> CoordMap {
    // First get the source face
    let source_face = orientation.source_face(face);
    // next, get the up, right, down, and left for the source face and arg face.
    let src_up = source_face.up();
    let src_right = source_face.right();
    let face_up = face.up();
    let face_right = face.right();
    let face_down = face.down();
    let face_left = face.left();
    // Next, reface the src_dir faces
    let rsrc_up = orientation.reface(src_up);
    let rsrc_right = orientation.reface(src_right);
    // Now match up the faces
    let x_map = if rsrc_right == face_right {
        AxisMap::PosX
    } else if rsrc_right == face_down {
        AxisMap::PosY
    } else if rsrc_right == face_left {
        AxisMap::NegX
    } else {
        AxisMap::NegY
    };
    let y_map = if rsrc_up == face_up {
        AxisMap::PosY
    } else if rsrc_up == face_right {
        AxisMap::NegX
    } else if rsrc_up == face_down {
        AxisMap::NegY
    } else {
        AxisMap::PosX
    };
    CoordMap {
        x: x_map,
        y: y_map
    }
}
```

### Generating the tables

The code for generating the table is basically the same for both `map_face_coord` and `source_face_coord`. They use the naive functions we implemented previously.

```rust
fn map_face_coord_gencode() {
    const fn map_axismap(a: AxisMap) -> &'static str {
        match a {
            AxisMap::PosX => "x",
            AxisMap::PosY => "y",
            AxisMap::NegX => "-x",
            AxisMap::NegY => "-y",
        }
    }
    let directions = [
        Direction::PosY,
        Direction::PosX,
        Direction::PosZ,
        Direction::NegY,
        Direction::NegX,
        Direction::NegZ
    ];
    let output = {
        use std::fmt::Write;
        let mut output = String::new();
        writeln!(output, "const MAP_FACE_COORD_TABLE: [CoordMap; 1152] = [");
        for flipi in 0..8 { // flip
            for roti in 0..24 { // rotation
                directions.into_iter().for_each(|face| {
                    let map = map_face_coord_naive(Orientation::new(Rotation(roti as u8), Flip(flipi as u8)), face);
                    writeln!(output, "    CoordMap::new(AxisMap::{:?}, AxisMap::{:?}),", map.x, map.y);
                });
            }
        }
        writeln!(output, "];");
        output
    };
    std::fs::write("map_face_coord_table.rs").expect("Failed to write file.");
}
```

```rust
fn source_face_coord_gencode() {
    const fn map_axismap(a: AxisMap) -> &'static str {
        match a {
            AxisMap::PosX => "x",
            AxisMap::PosY => "y",
            AxisMap::NegX => "-x",
            AxisMap::NegY => "-y",
        }
    }
    let directions = [
        Direction::PosY,
        Direction::PosX,
        Direction::PosZ,
        Direction::NegY,
        Direction::NegX,
        Direction::NegZ
    ];
    let output = {
        use std::fmt::Write;
        let mut output = String::new();
        writeln!(output, "const SOURCE_FACE_COORD_TABLE: [CoordMap; 1152] = [");
        for flipi in 0..8 { // flip
            for roti in 0..24 { // rotation
                directions.into_iter().for_each(|face| {
                    let map = source_face_coord_naive(Orientation::new(Rotation(roti as u8), Flip(flipi as u8)), face);
                    writeln!(output, "    CoordMap::new(AxisMap::{:?}, AxisMap::{:?}),", map.x, map.y);
                });
            }
        }
        writeln!(output, "];");
        output
    };
    std::fs::write("source_face_coord_table.rs").expect("Failed to write file.");
}
```

And finally, `map_face_coord_table_index`

```rust
pub fn map_face_coord_table_index(rotation: Rotation, flip: Flip, face: Direction) -> usize {
    let flip = flip.0 as usize;
    let rot = rotation.0 as usize;
    let face = face as usize;
    flip * 24 * 6 + rot * 6 + face
}
```

Alright, now I've given you the building blocks to implement block orientations in your voxel project. This code was difficult to figure out and was time consuming to write in the places where I needed to explicitly write out entire match expressions by hand because I didn't have the building blocks necessary to make a naive solution.

I hope that this helps you implement voxel orientations in your engine. With this technique, you can apply 72 orientations to your blocks (and you can have blocks shaped however you like). Just keep in mind that you will need to determine whether or not your mesh's indices need to be reversed by XORing the axes bits from Flip. I discussed that in the beginning. If you don't do this step, you will have incorrect backface culling. You can also use `Orientation` to orient your normals. You shouldn't do anything with your UVs, those should stay the same. You could optionally create a system that allows you to rotate a shaped block while also keeping the UVs relatively the same. This would allow you to orient blocks while making them seamless next to each other. I don't have that implemented for my project, but it's planned.
