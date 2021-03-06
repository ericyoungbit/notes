...menustart

 - [Introduction to WebGL](#ee4f3cb275cef5c1d82ace8d34788594)
     - [Section 1: The Programmable Pipeline](#9a3040db06e663f46ca7269b3df96da5)
         - [6.1.1  The WebGL Graphics Context](#4d00c423ae698f2f9609d15a3d6df978)
         - [6.1.2  The Shader Program](#4f31b4e489e41f45a443c8b31846a270)
         - [6.1.3  Data Flow in the Pipeline](#c754edff11c3f2393931f78827b9b007)
         - [6.1.4  Values for Uniform Variables](#339ce22e55c69e7bbd8951f6e94a8586)
         - [6.1.5  Values for Attributes](#969dcaf673eeb3055c02e146d2f2900b)
         - [6.1.6  Drawing a Primitive](#82e825f128e1b88315c25d161f2545b0)
     - [Section 2: First Examples](#e305dfdab0e4cefbd5abea8eacc9c474)
     - [Section 3: GLSL](#512cc0d7b47100675a0c220edf55385a)
     - [Section 4: Image Textures](#1f7b297214b84adbccdc248c8f3a7c4e)
     - [Section 5: Implementing 2D Transforms](#6f0ba384aa3b845de4898ddb899a037c)
     - [Note](#3b0649c72650c313a357338dcdfb64ec)

...menuend


<h2 id="ee4f3cb275cef5c1d82ace8d34788594"></h2>

# Introduction to WebGL

 - WebGL 1.0 is based on OpenGL ES 2.0, a version designed for use on embedded systems 
 - OpenGL ES 1.0 was very similar to OpenGL 1.1. 
    - OpenGL ES 2.0 introduced major changes. It is actually a smaller, simpler API that puts more responsibility on the programmer. 
    - For example, functions for working with transformations, such as glRotatef and glPushMatrix, were eliminated from the API,making the programmer responsible for keeping track of transformations.
 - WebGL does not use glBegin/glEnd to generate geometry, and it doesn't use function such as `glColor*` or `glNormal*` to specify attributes of vertices. 
 - There are two sides to any WebGL program. 
    - Part of the program is written in *JavaScript*
    - The second part is written in *GLSL*


<h2 id="9a3040db06e663f46ca7269b3df96da5"></h2>

## Section 1: The Programmable Pipeline

 - OpenGL 1.1 used a *fixed-function pipeline* for graphics processing.
    - data is provided by a program , and passes through a series of processing stages.
    - the program can enable/disable some of the steps in the process, but there is no way for it to change what happends at each stage.
    - The functionality is fixed.
 - OpenGL 2.0 introduced a *programmable pipeline*. 
    - it is possible to replace certain stages in the pipeline with custom program.
    - This gives the programmer complete control over what happens at that stage. 
    - the programmability was optional
        - the complete fixed-function pipeline was still available for programs that didn't need the flexibility of programmability.
    - WebGL uses a programmable pipeline, and it is **mandatory**. 
        - There is no way to use WebGL without writing programs to implement part of the graphics processing pipeline.
 - The programs that are written as part of the pipeline are called *shaders*. 
    - For WebGL, you need to write a *vertex shader*, which is called once for each vertex in a primitive, 
        - and a *fragment shader*, which is called once for each pixel in the primitive.
    - Aside from these two programmable stages, the WebGL pipeline also contains several stages from the original fixed-function pipeline. 
 - This book covers WebGL 1.0. 
    - Version 2.0 was released in January 2017. and it is compatible with version 1.0.
    - At the end of 2017, WebGL 2.0 is available in some browsers.
    - The later versions of OpenGL have introduced **additional** programmable stages into the pipeline.

<h2 id="4d00c423ae698f2f9609d15a3d6df978"></h2>

### 6.1.1  The WebGL Graphics Context

 - To use WebGL, you need a WebGL graphics context. 
    - ( A few browsers (notably Internet Explorer and Edge) require "experimental-webgl"  )

```js
// <canvas width="800" height="600" id="webglcanvas"></canvas>
...
canvas = document.getElementById("webglcanvas");
gl = canvas.getContext("webgl") || canvas.getContext("experimental-webgl");
```

 - `gl` may be null. You'd better catch the exception

```js
function init() {
    try {
        canvas = document.getElementById("webglcanvas");
        gl = canvas.getContext("webgl") || canvas.getContext("experimental-webgl");
        if ( ! gl ) {
            throw "Browser does not support WebGL";
        }
    }
    catch (e) {
          .
          .  // report the error
          .
        return;
    }
      .
      .  // other JavaScript initialization
      .
    initGL();  // a function that initializes the WebGL graphics context
}
```

 - The init() function could be called, for example, by the onload event handler for the `<body>` element of the web page:

```
<body onload="init()">
```


<h2 id="4f31b4e489e41f45a443c8b31846a270"></h2>

### 6.1.2  The Shader Program

 - Drawing with WebGL requires a shader program. Shaders are written in the language GLSL ES 1.0.
    - The vertex shader and fragment shader are compiled separately,  and then "linked" to produce a complete shader program. 
 - To create the vertex shader.

```js
// vertexShaderSource is a JavaScript string
var vertexShader = gl.createShader( gl.VERTEX_SHADER );
gl.shaderSource( vertexShader, vertexShaderSource );
gl.compileShader( vertexShader );
```

 - Errors in the source code will cause the compilation to fail silently. You need to check for compilation errors by calling the function

```js
gl.getShaderParameter( vertexShader, gl.COMPILE_STATUS )  
```

 - which returns a boolean value to indicate whether the compilation succeeded. 
 - In the event that an error occurred, you can retrieve an error message with

```js
gl.getShaderInfoLog(vertexShader)
```

 - which returns a string containing the result of the compilation. 

 - The fragment shader can be created in the same way. 
 - With both shaders in hand, you can **create** and **link** the **program**. 

```js
var prog = gl.createProgram();
gl.attachShader( prog, vertexShader );
gl.attachShader( prog, fragmentShader );
gl.linkProgram( prog );
```

 - Even if the shaders have been successfully compiled, errors can occur when they are linked into a complete program. 
    - For example, the vertex and fragment shader can share certain kinds of variable.
    - If the two programs declare such variables with the same name but with different types, an error will occur at link time. 
    - Checking for link errors is similar to checking for compilation errors in the shaders.
 - The code for creating a shader program is always pretty much the same.
    - Here is an example function:

```js
/**
 * Creates a program for use in the WebGL context gl, and returns the
 * identifier for that program.  If an error occurs while compiling or
 * linking the program, an exception of type String is thrown.  The error
 * string contains the compilation or linking error. 
 */
function createProgram(gl, vertexShaderSource, fragmentShaderSource) {
   var vsh = gl.createShader( gl.VERTEX_SHADER );
   gl.shaderSource( vsh, vertexShaderSource );
   gl.compileShader( vsh );
   if ( ! gl.getShaderParameter(vsh, gl.COMPILE_STATUS) ) {
      throw "Error in vertex shader:  " + gl.getShaderInfoLog(vsh);
   }
   var fsh = gl.createShader( gl.FRAGMENT_SHADER );
   gl.shaderSource( fsh, fragmentShaderSource );
   gl.compileShader( fsh );
   if ( ! gl.getShaderParameter(fsh, gl.COMPILE_STATUS) ) {
      throw "Error in fragment shader:  " + gl.getShaderInfoLog(fsh);
   }
   var prog = gl.createProgram();
   gl.attachShader( prog, vsh );
   gl.attachShader( prog, fsh );
   gl.linkProgram( prog );
   if ( ! gl.getProgramParameter( prog, gl.LINK_STATUS) ) {
      throw "Link error in program:  " + gl.getProgramInfoLog(prog);
   }
   return prog;
}
```

 - There is one more step: You have to tell the WebGL context to use the program. 

```js
gl.useProgram( prog );
```

 - It is possible to create several shader programs. 
    - You can then switch from one program to another at any time by calling `gl.useProgram`, even in the middle of rendering an image. 
    - (Three.js, for example, uses a different program for each type of Material.)
 - Shaders and programs that are no longer needed can be deleted to free up the resources they consume.
    - Use the functions `gl.deleteShader(shader)` and `gl.deleteProgram(program)`.


<h2 id="c754edff11c3f2393931f78827b9b007"></h2>

### 6.1.3  Data Flow in the Pipeline

 - The basic operation in WebGL is to draw a geometric primitive. 
    - The primitives for drawing quads and polygons have been removed.
    - The remaining primitives draw points, line segments, and triangles.
    - gl.POINTS, gl.LINES, gl.LINE_STRIP, gl.LINE_LOOP, gl.TRIANGLES, gl.TRIANGLE_STRIP, and gl.TRIANGLE_FAN,
 - When WebGL is used to draw a primitive, there are two general categories of data that can be provided for the primitive. 
    - **attribute** variables
    - **uniform** variables
 - A uniform variable has a single value that is the same for the entire primitive, while the value of an attribute variable can be different for different vertices.


must be attribute | attri or uniform | must be uniform  
--- | --- | --- 
coordinates  | | 
 · | color | 
 · | normal vectors | 
 · | material properties | 
Texture coordinates | | 
 · | · | geometric transform 


 - WebGL does not come with **any** predefined attributes. 
    - In the programmable pipeline, the attributes and uniforms that are used *are entirely up to the programmer*. 
 - Attributes are just values that are passed into the vertex shader. 
    - Uniforms can be passed into the vertex shader, the fragment shader, or both. 
 - WebGL does not assign a meaning to the values. The meaning is entirely determined by what the shaders do with the values. 
 - To understand this, we need to look at what happens in the pipeline in a little more detail.
    - When drawing a primitive, the JavaScript program will specify values for any attributes and uniforms in the shader program. 
        - For each attribute, it will specify an array of values, one for each vertex. 
        - For each uniform, it will specify a single value. 
    - The values will all be sent to the GPU before the primitive is drawn.
    - When drawing the primitive, the GPU calls the vertex shader once for each vertex. 
        - The attribute values for the vertex that is to be processed are passed as input into the vertex shader. 
        - Values of uniform variables are also passed to the vertex shader. 
    - Both attributes and uniforms are represented as global variables in the shader, whose values are set before the shader is called.
 - As one of its outputs, the vertex shader must specify the coordinates of the vertex in the *clip coordinate system*.
    - It does that by assigning a value to a special variable named `gl_Position`.
    - The position is often computed by applying a transformation to the coordinates attribute in the *object coordinate system*.
 - After the positions of all the vertices in the primitive have been computed, a fixed-function stage in the pipeline *clips away* the parts of the primitive whose coordinates are outside the range of valid clip coordinates (−1 to 1 along each coordinate axis). 
    - The primitive is then rasterized; that is, it is determined which pixels lie inside the primitive. 
    - The fragment shader is then called once for each pixel that lies in the primitive. 
    - The fragment shader has access to uniform variables (but not attributes). 
        - It can also use a special variable named `gl_FragCoord` that contains the clip coordinates of the pixel. 
        - Pixel coordinates are computed by interpolating the values of gl_Position that were specified by the vertex shader. 
        - The interpolation is done by another fixed-function stage that comes between the vertex shader and the fragment shader. 
 - Other quantities besides coordinates can work in much that same way, suck like *color*. 
    - That is , 
        - the vertex shader computes a value for the quantity at each vertex of a primitive,
        - An interpolator takes the values at the vertices and computes a value for each pixel in the primitive
        - The value for a given pixel is then input into the fragment shader.
    - In GLSL, this pattern is implemented using *varying variables*.
 - A varying variable is declared both in the vertex shader and in the fragment shader.
    - The vertex shader is responsible for assigning a value to the varying variable. 
    - The interpolator takes the values from the vertex shader and computes a value for each pixel. 
    - When the fragment shader is executed for a pixel, the value of the varying variable is the interpolated value for that pixel.
    - PS. In newer versions of GLSL, the term "varying variable" has been replaced by "out variable" in the vertex shader and "in variable" in the fragment shader.
 - After all that, the job of the fragment shader is simply to specify a color for the pixel. 
    - It does that by assigning a value to a special variable named **gl_FragColor**. 
    - That value will then be used in the remaining fixed-function stages of the pipeline.
 - To summarize: 
    1. The JavaScript side of the program sends values for attributes and uniform variables to the GPU and then issues a command to draw a primitive.
    2. The GPU executes the vertex shader once for each vertex. 
        - The vertex shader can use the values of attributes and uniforms. 
        - It assigns values to gl_Position and to any varying variables that exist in the shader. 
    3. After clipping, rasterization, and interpolation, the GPU executes the fragment shader once for each pixel in the primitive. 
        - The fragment shader can use the values of varying variables, uniform variables, and gl_FragCoord. 
        - It computes a value for gl_FragColor.
    4. This diagram summarizes the flow of data:
        - ![](../imgs/cg6_webgl_gsgl_workflow.png)
        - The diagram is not complete. how are textures used?

<h2 id="339ce22e55c69e7bbd8951f6e94a8586"></h2>

### 6.1.4  Values for Uniform Variables

 - float, int, bool, vec2, vec3, vec4
 - Global variable
    - Global variable declarations in a vertex shader can be marked as *attribute*, *uniform*, or *varying*. 
        - A variable declaration with none of these modifiers defines a variable that is local to the vertex shader. 
    - Global variables in a fragment can optionally be modified with uniform or varying, or they can be declared without a modifier. 
        - A varying variable should be declared in both shaders, with the same name and type. 
 - The JavaScript side of the program needs a way to refer to particular attributes and uniform variables.

```js
//  get a reference to a uniform variable in a shader program
colorUniformLoc = gl.getUniformLocation( prog, "color" );
// The location colorUniformLoc can then be used to set the value of the uniform variable:
gl.uniform3f( colorUniformLoc, 1, 0, 0 );
gl.uniform3fv( colorUniformLoc, [ 1, 0, 0 ] );
// gl.uniform*
```

<h2 id="969dcaf673eeb3055c02e146d2f2900b"></h2>

### 6.1.5  Values for Attributes

 - Attribute is more complicated, because an attribute can take a different value for each vertex in a primitive. 
    - The basic idea is to copy the complete set of data for the attribute from JavaScript  into GPU memory.  
    - But it is non-trivial.
 - A regular JS array is not suitable for this purpose.  For efficiency , we need the data to be in a block of memory holding values in successive locations  , but JS array don't have that form. 
    - To fixe this problem, a new kind of array ,called *typed arrays* , was introduced into JS.  
 - A typed array has a fixed length, which is assigned when it is created by a constructor. 

```js
var color = new Float32Array( 12 );  // space for 12 floats
var coords = new Float32Array( [ 0,0.7, -0.7,-0.5, 0.7,-0.5 ] );
```

 - For use in WebGL, the attribute data must be transferred into a VBO (vertex buffer object).
    - A VBO is a block of memory that is accessible to the GPU. 
 - To use a VBO:

```js
// call gl.createBuffer() to create a VBO
colorBuffer = gl.createBuffer();

// Before transferring data into the VBO, you must "bind" the VBO:
// here, `gl.ARRAY_BUFFER` is called the "target"
// Only one VBO at a time can be bound to a given target.
gl.bindBuffer( gl.ARRAY_BUFFER, colorBuffer );

// copy data into that buffer, use gl.bufferData().
// 1st param is again 'target' , 2nd is the typed array
gl.bufferData(gl.ARRAY_BUFFER, colorArray, gl.STATIC_DRAW);
```
 - `gl.bufferData()`
    - All the elements of the typed array are copied into the buffer, and the size of the array determines the size of the buffer. 
        - Note that this is a  transfer of raw data bytes; WebGL does not remember whether the data represents floats or ints or some other kind of data.
    - The 3rd parameter is one of the constants gl.STATIC_DRAW, gl.STREAM_DRAW, or gl.DYNAMIC_DRAW.
        - It is a hint to WebGL about how the data will be used, and it helps WebGL to manage the data in the most efficient way. 
        - gl.STATIC_DRAW means that you intend to use the data many times without changing it. 
            - WebGL will probably store the data on the graphics card itself where it can be accessed most quickly by the graphics hardware. 
        - gl.STEAM_DRAW is for data that will be used only once, then discarded. 
            - It can be "streamed" to the card when it is needed.
        - gl.DYNAMIC_DRAW  is somewhere between the other two values; it might be used for data that will be used a couple of times and then discarded.

 - Getting attribute data into VBOs is only part of the story.
    - You also have to tell WebGL to use the VBO as the source of values for the attribute. 

```js
// get attribute location
colorAttribLoc = gl.getAttribLocation(prog, "a_color");

// enable the use of a VBO for that attribute
// it is reasonable to call this method just once, during initialization.
gl.enableVertexAttribArray( colorAttribLoc );

// tell WebGL which buffer contains the data and 
// how the bits in that buffer are to be interpreted.
// A VBO must be bound to the ARRAY_BUFFER target
gl.bindBuffer( gl.ARRAY_BUFFER, colorBuffer );
gl.vertexAttribPointer( colorAttribLoc, 3, gl.FLOAT, false, 0, 0 );
```

 - `gl.vertexAttribPointer`
    - the 2nd param is the number of values per vertex
    - the 3rd parma is the type of value 
        - Other values include gl.BYTE, gl.UNSIGNED_BYTE, gl.UNSIGNED_SHORT, and gl.SHORT for integer values
 - Here is the full set of commands:

```js
// the following 3 commands are often done in initialization
colorAttribLoc = gl.getAttribLocation( prog, "a_color" );
colorBuffer = gl.createBuffer();
gl.enableVertexAttribArray( colorAttribLoc );

gl.bindBuffer( gl.ARRAY_BUFFER, colorBuffer );
gl.vertexAttribPointer( colorAttribLoc, 3, gl.FLOAT, false, 0, 0 );
gl.bufferData( gl.ARRAY_BUFFER, colorArray, gl.STATIC_DRAW );
```

<h2 id="82e825f128e1b88315c25d161f2545b0"></h2>

### 6.1.6  Drawing a Primitive



<h2 id="e305dfdab0e4cefbd5abea8eacc9c474"></h2>

## Section 2: First Examples
<h2 id="512cc0d7b47100675a0c220edf55385a"></h2>

## Section 3: GLSL
<h2 id="1f7b297214b84adbccdc248c8f3a7c4e"></h2>

## Section 4: Image Textures
<h2 id="6f0ba384aa3b845de4898ddb899a037c"></h2>

## Section 5: Implementing 2D Transforms


<h2 id="3b0649c72650c313a357338dcdfb64ec"></h2>

## Note 

 - webGL shader 没有预定义的attributes和uniform变量，程序员自己定义使用
 - webGL 特殊变量:
    - gl_Position
    - gl_FragCoord
    - gl_FragColor

