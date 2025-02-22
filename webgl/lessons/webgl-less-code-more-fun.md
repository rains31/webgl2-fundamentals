Title: WebGL2 - Less Code, More Fun
Description: Ways to make programming WebGL less verbose
TOC: Less Code, More Fun


This post is a continuation of a series of posts about WebGL.
The first [started with fundamentals](webgl-fundamentals.html).
If you haven't read those please view them first.

WebGL programs require that you write shader programs which you have to compile and link and then
you have to look up the locations of the inputs to those shader programs. These inputs are called
uniforms and attributes and the code required to look up their locations can be wordy and tedious.

Assume we've got the <a href="webgl-boilerplate.html">typical boilerplate WebGL code for
compiling and linking shader programs</a>. Given a set of shaders like this.

vertex shader:

```
#version 300 es

uniform mat4 u_worldViewProjection;
uniform vec3 u_lightWorldPos;
uniform mat4 u_world;
uniform mat4 u_viewInverse;
uniform mat4 u_worldInverseTranspose;

in vec4 a_position;
in vec3 a_normal;
in vec2 a_texcoord;

out vec4 v_position;
out vec2 v_texCoord;
out vec3 v_normal;
out vec3 v_surfaceToLight;
out vec3 v_surfaceToView;

void main() {
  v_texCoord = a_texcoord;
  v_position = (u_worldViewProjection * a_position);
  v_normal = (u_worldInverseTranspose * vec4(a_normal, 0)).xyz;
  v_surfaceToLight = u_lightWorldPos - (u_world * a_position).xyz;
  v_surfaceToView = (u_viewInverse[3] - (u_world * a_position)).xyz;
  gl_Position = v_position;
}
```

fragment shader:

```
#version 300 es
precision highp float;

in vec4 v_position;
in vec2 v_texCoord;
in vec3 v_normal;
in vec3 v_surfaceToLight;
in vec3 v_surfaceToView;

uniform vec4 u_lightColor;
uniform vec4 u_ambient;
uniform sampler2D u_diffuse;
uniform vec4 u_specular;
uniform float u_shininess;
uniform float u_specularFactor;

out vec4 outColor;

vec4 lit(float l ,float h, float m) {
  return vec4(1.0,
              max(l, 0.0),
              (l > 0.0) ? pow(max(0.0, h), m) : 0.0,
              1.0);
}

void main() {
  vec4 diffuseColor = texture(u_diffuse, v_texCoord);
  vec3 a_normal = normalize(v_normal);
  vec3 surfaceToLight = normalize(v_surfaceToLight);
  vec3 surfaceToView = normalize(v_surfaceToView);
  vec3 halfVector = normalize(surfaceToLight + surfaceToView);
  vec4 litR = lit(dot(a_normal, surfaceToLight),
                    dot(a_normal, halfVector), u_shininess);
  outColor = vec4((
    u_lightColor * (diffuseColor * litR.y + diffuseColor * u_ambient +
                u_specular * litR.z * u_specularFactor)).rgb,
    diffuseColor.a);
}
```

You'd end up having to write code like this to look up and set all the various values to draw.

```
// At initialization time
var u_worldViewProjectionLoc   = gl.getUniformLocation(program, "u_worldViewProjection");
var u_lightWorldPosLoc         = gl.getUniformLocation(program, "u_lightWorldPos");
var u_worldLoc                 = gl.getUniformLocation(program, "u_world");
var u_viewInverseLoc           = gl.getUniformLocation(program, "u_viewInverse");
var u_worldInverseTransposeLoc = gl.getUniformLocation(program, "u_worldInverseTranspose");
var u_lightColorLoc            = gl.getUniformLocation(program, "u_lightColor");
var u_ambientLoc               = gl.getUniformLocation(program, "u_ambient");
var u_diffuseLoc               = gl.getUniformLocation(program, "u_diffuse");
var u_specularLoc              = gl.getUniformLocation(program, "u_specular");
var u_shininessLoc             = gl.getUniformLocation(program, "u_shininess");
var u_specularFactorLoc        = gl.getUniformLocation(program, "u_specularFactor");

var a_positionLoc              = gl.getAttribLocation(program, "a_position");
var a_normalLoc                = gl.getAttribLocation(program, "a_normal");
var a_texCoordLoc              = gl.getAttribLocation(program, "a_texcoord");

// Setup all the buffers and attributes (assuming you made the buffers already)
var vao = gl.createVertexArray();
gl.bindVertexArray(vao);
gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
gl.enableVertexAttribArray(a_positionLoc);
gl.vertexAttribPointer(a_positionLoc, positionNumComponents, gl.FLOAT, false, 0, 0);
gl.bindBuffer(gl.ARRAY_BUFFER, normalBuffer);
gl.enableVertexAttribArray(a_normalLoc);
gl.vertexAttribPointer(a_normalLoc, normalNumComponents, gl.FLOAT, false, 0, 0);
gl.bindBuffer(gl.ARRAY_BUFFER, texcoordBuffer);
gl.enableVertexAttribArray(a_texcoordLoc);
gl.vertexAttribPointer(a_texcoordLoc, texcoordNumComponents, gl.FLOAT, 0, 0);

// At init or draw time depending on use.
var someWorldViewProjectionMat = computeWorldViewProjectionMatrix();
var lightWorldPos              = [100, 200, 300];
var worldMat                   = computeWorldMatrix();
var viewInverseMat             = computeInverseViewMatrix();
var worldInverseTransposeMat   = computeWorldInverseTransposeMatrix();
var lightColor                 = [1, 1, 1, 1];
var ambientColor               = [0.1, 0.1, 0.1, 1];
var diffuseTextureUnit         = 0;
var specularColor              = [1, 1, 1, 1];
var shininess                  = 60;
var specularFactor             = 1;

// At draw time
gl.useProgram(program);
gl.bindVertexArray(vao);

// Setup the textures used
gl.activeTexture(gl.TEXTURE0 + diffuseTextureUnit);
gl.bindTexture(gl.TEXTURE_2D, diffuseTexture);

// Set all the uniforms.
gl.uniformMatrix4fv(u_worldViewProjectionLoc, false, someWorldViewProjectionMat);
gl.uniform3fv(u_lightWorldPosLoc, lightWorldPos);
gl.uniformMatrix4fv(u_worldLoc, worldMat);
gl.uniformMatrix4fv(u_viewInverseLoc, viewInverseMat);
gl.uniformMatrix4fv(u_worldInverseTransposeLoc, worldInverseTransposeMat);
gl.uniform4fv(u_lightColorLoc, lightColor);
gl.uniform4fv(u_ambientLoc, ambientColor);
gl.uniform1i(u_diffuseLoc, diffuseTextureUnit);
gl.uniform4fv(u_specularLoc, specularColor);
gl.uniform1f(u_shininessLoc, shininess);
gl.uniform1f(u_specularFactorLoc, specularFactor);

gl.drawArrays(...);
```

That's a lot of typing.

There are lots of ways to simplify this. One suggestion is to ask WebGL to tell us all
the uniforms, attributes and their locations and then setup functions to set them for us.
We can then pass in JavaScript objects to set our settings much more easily.
If that's clear as mud well, our code would look something like this

```
// At initialization time
var uniformSetters = twgl.createUniformSetters(gl, program);
var attribSetters  = twgl.createAttributeSetters(gl, program);

// Setup all the buffers and attributes
var attribs = {
  a_position: { buffer: positionBuffer, numComponents: 3, },
  a_normal:   { buffer: normalBuffer,   numComponents: 3, },
  a_texcoord: { buffer: texcoordBuffer, numComponents: 2, },
};
var vao = twgl.createVAOAndSetAttributes(
    gl, attribSetters, attribs);

// At init time or draw time depending on use.
var uniforms = {
  u_worldViewProjection:   computeWorldViewProjectionMatrix(...),
  u_lightWorldPos:         [100, 200, 300],
  u_world:                 computeWorldMatrix(),
  u_viewInverse:           computeInverseViewMatrix(),
  u_worldInverseTranspose: computeWorldInverseTransposeMatrix(),
  u_lightColor:            [1, 1, 1, 1],
  u_ambient:               [0.1, 0.1, 0.1, 1],
  u_diffuse:               diffuseTexture,
  u_specular:              [1, 1, 1, 1],
  u_shininess:             60,
  u_specularFactor:        1,
};

// At draw time
gl.useProgram(program);

// Bind the VAO that has all our buffers and attribute settings
gl.bindAttribArray(vao);

// Set all the uniforms and textures used.
twgl.setUniforms(uniformSetters, uniforms);

gl.drawArrays(...);
```

That seems a heck of a lot smaller, easier, and less code to me.

You can even use multiple JavaScript objects for the uniforms if it suits you. For example

```
// At initialization time
var uniformSetters = twgl.createUniformSetters(gl, program);
var attribSetters  = twgl.createAttributeSetters(gl, program);

// Setup all the buffers and attributes
var attribs = {
  a_position: { buffer: positionBuffer, numComponents: 3, },
  a_normal:   { buffer: normalBuffer,   numComponents: 3, },
  a_texcoord: { buffer: texcoordBuffer, numComponents: 2, },
};
var vao = twgl.createVAOAndSetAttributes(gl, attribSetters, attribs);

// At init time or draw time depending
var uniformsThatAreTheSameForAllObjects = {
  u_lightWorldPos:         [100, 200, 300],
  u_viewInverse:           computeInverseViewMatrix(),
  u_lightColor:            [1, 1, 1, 1],
};

var uniformsThatAreComputedForEachObject = {
  u_worldViewProjection:   perspective(...),
  u_world:                 computeWorldMatrix(),
  u_worldInverseTranspose: computeWorldInverseTransposeMatrix(),
};

var objects = [
  { translation: [10, 50, 100],
    materialUniforms: {
      u_ambient:               [0.1, 0.1, 0.1, 1],
      u_diffuse:               diffuseTexture,
      u_specular:              [1, 1, 1, 1],
      u_shininess:             60,
      u_specularFactor:        1,
    },
  },
  { translation: [-120, 20, 44],
    materialUniforms: {
      u_ambient:               [0.1, 0.2, 0.1, 1],
      u_diffuse:               someOtherDiffuseTexture,
      u_specular:              [1, 1, 0, 1],
      u_shininess:             30,
      u_specularFactor:        0.5,
    },
  },
  { translation: [200, -23, -78],
    materialUniforms: {
      u_ambient:               [0.2, 0.2, 0.1, 1],
      u_diffuse:               yetAnotherDiffuseTexture,
      u_specular:              [1, 0, 0, 1],
      u_shininess:             45,
      u_specularFactor:        0.7,
    },
  },
];

// At draw time
gl.useProgram(program);

// Setup the parts that are common for all objects

// Bind the VAO that has all our buffers and attribute settings
gl.bindAttribArray(vao);
twgl.setUniforms(uniformSetters, uniformThatAreTheSameForAllObjects);

objects.forEach(function(object) {
  computeMatricesForObject(object, uniformsThatAreComputedForEachObject);
  twgl.setUniforms(uniformSetters, uniformThatAreComputedForEachObject);
  twgl.setUniforms(uniformSetters, objects.materialUniforms);
  gl.drawArrays(...);
});
```

Here's an example using these helper functions

{{{example url="../webgl-less-code-more-fun.html" }}}

Let's take it a tiny step further. In the code above we setup a variable `attribs` with the buffers we created.
Not shown is the code to setup those buffers. For example if you want to make positions, normals and texture
coordinates you might need code like this

    // a single triangle
    var positions = [0, -10, 0, 10, 10, 0, -10, 10, 0];
    var texcoords = [0.5, 0, 1, 1, 0, 1];
    var normals   = [0, 0, 1, 0, 0, 1, 0, 0, 1];

    var positionBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.STATIC_DRAW);

    var texcoordBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, texcoordsBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(texcoords), gl.STATIC_DRAW);

    var normalBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, normalBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(normals), gl.STATIC_DRAW);

Looks like a pattern we can simplify as well.

    // a single triangle
    var arrays = {
       position: { numComponents: 3, data: [0, -10, 0, 10, 10, 0, -10, 10, 0], },
       texcoord: { numComponents: 2, data: [0.5, 0, 1, 1, 0, 1],               },
       normal:   { numComponents: 3, data: [0, 0, 1, 0, 0, 1, 0, 0, 1],        },
    };

    var bufferInfo = twgl.createBufferInfoFromArrays(gl, arrays);
    var vao = twgl.createVAOFromBufferInfo(gl, setters, bufferInfo);

Much shorter!

Here's that

{{{example url="../webgl-less-code-more-fun-triangle.html" }}}

This will even work if we have indices. `createVAOFromBufferInfo`
will set all the attributes and setup the `ELEMENT_ARRAY_BUFFER`
with your `indices` so when you bind that VAO you can call
`gl.drawElements`.

    // an indexed quad
    var arrays = {
       position: { numComponents: 3, data: [0, 0, 0, 10, 0, 0, 0, 10, 0, 10, 10, 0], },
       texcoord: { numComponents: 2, data: [0, 0, 0, 1, 1, 0, 1, 1],                 },
       normal:   { numComponents: 3, data: [0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1],     },
       indices:  { numComponents: 3, data: [0, 1, 2, 1, 2, 3],                       },
    };

    var bufferInfo = twgl.createBufferInfoFromTypedArray(gl, arrays);
    var vao = twgl.createVAOFromBufferInfo(gl, setters, bufferInfo);

and at render time we can call `gl.drawElements` instead of `gl.drawArrays`.

    ...

    // Draw the geometry.
    gl.drawElements(gl.TRIANGLES, bufferInfo.numElements, gl.UNSIGNED_SHORT, 0);

Here's that

{{{example url="../webgl-less-code-more-fun-quad.html" }}}

Finally we can go what I consider possibly too far. Given `position` almost always has 3 components (x, y, z)
and `texcoords` almost always 2, indices 3, and normals 3, we can just let the system guess the number
of components.

    // an indexed quad
    var arrays = {
       position: [0, 0, 0, 10, 0, 0, 0, 10, 0, 10, 10, 0],
       texcoord: [0, 0, 0, 1, 1, 0, 1, 1],
       normal:   [0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1],
       indices:  [0, 1, 2, 1, 2, 3],
    };

And that version

{{{example url="../webgl-less-code-more-fun-quad-guess.html" }}}

I'm not sure I personally like that style. Guessing bugs me because it can guess wrong. For example
I might choose to stick an extra set of texture coordinates in my texcoord attribute and it will
guess 2 and be wrong. Of course if it guesses wrong you can just specify it like the example above.
I guess I worry if the guessing code changes people's stuff might break. It's up to you. Some people
like things to be what they consider as simple as possible.

Why don't we look at the attributes on the shader program to figure out the number of components?
That's because it's common to supply 3 components (x, y, z) from a buffer but use a `vec4` in
the shader. For attributes WebGL will set `w = 1` automatically. But that means we can't easily
know the user's intent since what they declared in their shader might not match the number of
components they provide.

Looking for more patterns there's this

    var program = twgl.createProgramFromSources(gl, [vs, fs]);
    var uniformSetters = twgl.createUniformSetters(gl, program);
    var attribSetters  = twgl.createAttributeSetters(gl, program);

Let's simplify that too into just

    var programInfo = twgl.createProgramInfo(gl, ["vertexshader", "fragmentshader"]);

Which returns something like

    programInfo = {
       program: WebGLProgram,  // program we just compiled
       uniformSetters: ...,    // setters as returned from createUniformSetters,
       attribSetters: ...,     // setters as returned from createAttribSetters,
    }

And that's yet one more minor simplification. This will come in handy once we start using
multiple programs since it automatically keeps the setters with the program they are associated with.

{{{example url="../webgl-less-code-more-fun-quad-programinfo.html" }}}

One more, sometimes we have data that has no indices and we have to call
`gl.drawArrays`. Other times do have indices and we have to call `gl.drawElements`.
Given the data we have we can easily check which by looking at `bufferInfo.indices`.
If it exists we need to call `gl.drawElements`. If not we need to call `gl.drawArrays`.
So there's a function, `twgl.drawBufferInfo` that does that. It's used like this

    twgl.drawBufferInfo(gl, bufferInfo);

If you don't pass a 3rd parameter for the type of primitive to draw it assumes
`gl.TRIANGLES`.

Here's an example were we have a non-indexed triangle and an indexed quad. Because
we're using `twgl.drawBufferInfo` the code doesn't have to change when we
switch data.

{{{example url="../webgl-less-code-more-fun-drawbufferinfo.html" }}}

Anyway, this is the style I try to write my own WebGL programs.
For the lessons on these tutorials though I've felt like I have to use the standard **verbose**
ways so people don't get confused about what is WebGL and what is my own style. At some point
though showing all the steps gets in the way of the point so going forward some lessons will
be using this style.

Feel free to use this style in your own code. The functions `twgl.createProgramInfo`,
`twgl.createVAOAndSetAttributes`, `twgl.createBufferInfoFromArrays`, and `twgl.setUniforms`
etc... are part of a library I wrote based off these ideas. [It's called `TWGL`](https://twgljs.org).
It rhymes with wiggle and it stands for `Tiny WebGL`.

Next up, [drawing multiple things](webgl-drawing-multiple-things.html).

<div class="webgl_bottombar">
<h3>Can we use the setters directly?</h3>
<p>
For those of you familiar with JavaScript you might be wondering if you can use the setters
directly like this.
</p>
<pre class="prettyprint">
// At initialization time
var uniformSetters = twgl.createUniformSetters(program);

// At draw time
uniformSetters.u_ambient([1, 0, 0, 1]); // set the ambient color to red.
</pre>
<p>The reason this is not a good idea is because when you're working with GLSL you might
modify the shaders from time to time, often to debug. Let's say we were not seeing
anything on the screen in our program. One of the first things I do when nothing
is appearing is to simplify my shaders. For example I might change the fragment shader
to the simplest thing possible</p>
<pre class="prettyprint showlinemods">
#version 300 es
precision highp float;

in vec4 v_position;
in vec2 v_texCoord;
in vec3 v_normal;
in vec3 v_surfaceToLight;
in vec3 v_surfaceToView;

uniform vec4 u_lightColor;
uniform vec4 u_ambient;
uniform sampler2D u_diffuse;
uniform vec4 u_specular;
uniform float u_shininess;
uniform float u_specularFactor;

out vec4 outColor;

vec4 lit(float l ,float h, float m) {
  return vec4(1.0,
              max(l, 0.0),
              (l > 0.0) ? pow(max(0.0, h), m) : 0.0,
              1.0);
}

void main() {
  vec4 diffuseColor = texture2D(u_diffuse, v_texCoord);
  vec3 a_normal = normalize(v_normal);
  vec3 surfaceToLight = normalize(v_surfaceToLight);
  vec3 surfaceToView = normalize(v_surfaceToView);
  vec3 halfVector = normalize(surfaceToLight + surfaceToView);
  vec4 litR = lit(dot(a_normal, surfaceToLight),
                    dot(a_normal, halfVector), u_shininess);
  vec4 outColor = vec4((
    u_lightColor * (diffuseColor * litR.y + diffuseColor * u_ambient +
                u_specular * litR.z * u_specularFactor)).rgb,
      diffuseColor.a);
*  outColor = vec4(0,1,0,1);  // &lt;!--- just green
}
</pre>
<p>Notice I just added a line that sets <code>outColor</code> to a constant color.
Most drivers will see that none of the previous lines in the file actually contribute
to the result. As such they'll optimize out all of our uniforms. The next time we run the program
when we call <code>twgl.createUniformSetters</code> it won't create a setter for <code>u_ambient</code> so the
code above that calls <code>uniformSetters.u_ambient()</code> directly will fail with</p>
<pre class="prettyprint">
TypeError: undefined is not a function
</pre>
<p><code>twgl.setUniforms</code> solves that issue. It only sets those uniforms that actually exist</p>
</div>

