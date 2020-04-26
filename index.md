# Circle and sphere intersection optimization


## Intro

I worked on the cirlcle and shpere optimization for a game engine. Those shpaes are composed of two values, the center and the radius.
Those shapes will be used later for the physics engine.

## Intersection

the intersect for the circle and sphere is basicly the same exept we need a third dimension for the sphere.

The intersection function is :
```cpp
bool Intersects(Circle circle) const
{
	Vec2f distanceVector = center - circle.center;
	float radiusSum = radius + circle.radius;

	float distance = distanceVector.Magnitude();
   
  	return distance <= radiusSum;
}
```
So if the distance between the two circles is less or equal or the sum of the two radius it return true.

I tryed some way to implement this function, first i know that the magnitude use a square root but it is an expensive task for the processor.

## Reverse square root
Because the square root function is expensive for the processor i wanted to find another way to find the result.

This function play with the memory to return a really precise approximation(+-0.01) of the square root.
´´´cpp
float Q_rsqrt( float number )
{
    long i;
    float x2, y;
    const float threehalfs = 1.5F;

    x2 = number * 0.5F;
    y  = number;
    i  = * ( long * ) &y;    // evil floating point bit level hacking
    i  = 0x5f3759df - ( i >> 1 );               // what the fuck? 
    y  = * ( float * ) &i;
    y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//  y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration,
                                              // this can be removed
    return y;
}
´´´
> source : [The Legendary Fast Inverse Square Root](https://medium.com/hard-mode/the-legendary-fast-inverse-square-root-e51fee3b49d9)


I recommend to read [this article](https://medium.com/hard-mode/the-legendary-fast-inverse-square-root-e51fee3b49d9) about the reverse square root function to really understand the black magic behind.

#Second implementation
With this new square root function id did a new version a tested it with the [first function](https://github.com/EthanCavadia/EthanCavadia.github.io/index.md "Intersection").

##First result
I did the test between the [first function](https://github.com/EthanCavadia/EthanCavadia.github.io/index.md "Intersection") and 

## FourCircle, FourSphere

I tryed to made was to create the FourCircle class and the FourSphere class that contain an array of four circle/sphere.
and link all the value in one.


## Array of Structure of Array

AoSoA is a layout of the memory in wich data for different fields is interleaved using tiles or blocks with size equal to the SIMD vector size.
This appraoch of doing is more friendly with the Lcache and the SIMD port of the modern CPU.

### Array of Structure

The AoS is the most conventional memory layout.
```cpp
struct Circle
{
	Vec2f centerX
	float radius;
}
```

###Structure of Array

SoA is a layout separating elements of a struct into one parallel array.
```cpp
struct Circle
{
	std::array<float, N> centerX
	std::array<float, N> centerY;
	std::array<float, N> radius;
}
```
If only a specific part of the record is needed, only those parts need to be iterated over, allowing more data to fit onto a single cache line. 
The downside is requiring more cache ways when traversing data, and inefficient indexed addressing.

### Array of Structure of Array

AoSoA is a layout of the memory in wich data for different fields is interleaved using tiles or blocks with size equal to the power of 2, wich allow us to use SIMD expression.
```cpp
struct Circle
{
	std::array<float, 4> centerX
	std::array<float, 4> centerY;
	std::array<float, 4> radius;
}
```
This appraoch of doing is more friendly with the Lcache and the SIMD port of the modern CPU.

### FourCircle

![](https://github.com/EthanCavadia/EthanCavadia.github.io/blob/master/FourCircle.png)

### FourSphere

![](https://github.com/EthanCavadia/EthanCavadia.github.io/Assets/FourSphere.png)

# Trying intrinsics

I have a I7-7700HQ so i use SSE x86 intel intrinsics.

Now that i have all my value aligned i wanted to see if my functions of intersection could be faster by passing from C++ to intrinsics.
```cpp
    inline std::array<bool, 4> FourCircle::IntersectsIntrinsics(FourCircle circles)
    {
        alignas(4 * sizeof(bool))
        std::array<bool, 4> results;
        std::array<float, 4> centers;
        std::array<float, 4> radSum;
        std::array<float, 4> radSub;

        auto x1 = _mm_load_ps(centerXs.data());
        auto y1 = _mm_load_ps(centerYs.data());
        auto rad1 = _mm_load_ps(radius.data());

        auto x2 = _mm_load_ps(circles.centerXs.data());
        auto y2 = _mm_load_ps(circles.centerYs.data());
        auto rad2 = _mm_load_ps(circles.radius.data());

        x1 = _mm_sub_ps(x1, x2);
        y1 = _mm_sub_ps(y1, y2);
        rad1 = _mm_add_ps(rad1, rad2);
        rad1 = _mm_mul_ps(rad1, rad1);

        x1 = _mm_mul_ps(x1, x1);
        y1 = _mm_mul_ps(y1, y1);
        x1 = _mm_add_ps(x1, y1);

        __m128 result = _mm_cmple_ps(x1, rad1);
        unsigned condition = _mm_movemask_ps(result);

        // One or more of the distances were less-than-or-equal-to the maximum,
        // so we have something to draw
         if(condition != 0)
         {
        results[0] = (condition & 0x000F) != 0;
        results[1] = (condition & 0x00F0) != 0;
        results[2] = (condition & 0x0F00) != 0;
        results[3] = (condition & 0xF000) != 0;
        }
        
        return results;
        // The calculation in the instrincs return the good results but i suppose that the condition variables is not set correctly.
    }
```

## ps
packed single_precision floating-points. is a 4 * 32 bit floating point numbers stored as a 128-bit value.

## _mm_load_ps()
Load 128-bits composed of 4 packed single-precision (32-bit) floating-point elements) from memory into memory destination. mem_addr must be aligned on a 16-byte boundary.

With _mm_load_ps() i load my 4 float (4 * 4) to fill the 16-byte boundary.

## _mm_mul_ps()
The _mm_mul_ps function is multiplying 4 floats with 4 other floats.

## _mm_add_ps()
The _mm_add_ps function is adding two values together.

##_mm_cmple_ps()
Compare the 4 values with the "<=" operator.

## _mm_movemask_ps
Set each bit of mask of the memory destination based on the most significant bit of the corresponding packed single-precision (32-bit) floating-point element in the argument.

I used the _mm_movemask_ps() to store the result of each circle and then do a bytewise comparation.

# Result
I created a test who test 4 circle with 4 other circle but unfortunately i did not manage to found a good way to do the comparation, so that it can return the real result.

I still did a benchmark to see wich one is faster:
![](https://github.com/EthanCavadia/EthanCavadia.github.io/Assets/BM_GraphFonctionInstinsics.png)

And sadly the result is that the C++ function is faster by 1.3 time.


# Conclusion

