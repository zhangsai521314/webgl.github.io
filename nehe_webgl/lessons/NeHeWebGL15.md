```$xslt
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>环境贴图</title>
</head>
<body>
<canvas id="myCanvas" width="1580" height="880"></canvas>
<script src="../lib/gl-matrix-min.js"></script>

<!-- 天空盒着色器 -->
<script type="text/shader" id="vShaderSource">
	attribute vec3 a_position;
	attribute vec3 a_color;
	uniform mat4 u_matrix;
	varying vec3 v_color;
	varying vec3 v_texCoord; // 天空盒纹理坐标
	void main(){
		gl_Position = u_matrix * vec4(a_position, 1.0);
		v_color = a_color;

		// 因为天空盒的中心在原点，所以纹理坐标就是顶点坐标
		v_texCoord = a_position;
	}
</script>
<script type="text/shader" id="fShaderSource">
	precision mediump float;
	uniform samplerCube u_sampler; // 采样器
	varying vec3 v_color;
	varying vec3 v_texCoord;
	void main(){
		vec4 baseColor = vec4(v_color, 1.0); // 立方体本来的颜色
		vec4 texture = textureCube(u_sampler, v_texCoord); // 纹理颜色
		gl_FragColor = baseColor * texture;
	}
</script>

<!-- 立方体着色器 -->
<script type="text/shader" id="vShaderSource2">
	attribute vec4 a_position;
	attribute vec4 a_normal; // 顶点法线

	uniform mat4 u_matrix; // 模型视图投影矩阵
	uniform mat4 u_mMatrix; // 模型矩阵
	uniform mat4 u_normalMatrix; // 模型矩阵的逆转置矩阵

	varying vec3 v_normal;
	varying vec3 v_position;

	void main(){
		gl_Position = u_matrix * a_position;

		// 因顶点位置变换，将正确的顶点位置传给片元着色器
		v_position = vec3( u_mMatrix * a_position );

		// 因顶点位置变换，将正确的法向量传给片元着色器
		v_normal = normalize( vec3( u_normalMatrix * a_normal ) );
	}
</script>
<script type="text/shader" id="fShaderSource2">
	precision mediump float;

	uniform samplerCube u_sampler; // 采样器
	uniform vec3 u_eyePosition; // 相机的位置向量

	varying vec3 v_normal;
	varying vec3 v_position;

	void main(){
		// 计算视线向量eye
		vec3 eye = normalize( u_eyePosition - v_position );

		// 计算反射向量，即纹理坐标
		vec3 texCoord = reflect( -eye, v_normal );

		gl_FragColor = textureCube( u_sampler, texCoord );
	}
</script>



<script>
    var canvas = document.getElementById('myCanvas'); // canvas
    var gl = canvas.getContext('webgl'); // 上下文

    // 天空盒着色器程序对象
    var program = createShader(gl, document.getElementById('vShaderSource').innerHTML,
        document.getElementById('fShaderSource').innerHTML);
    program.a_position = gl.getAttribLocation(program, 'a_position');
    program.a_color = gl.getAttribLocation(program, 'a_color');
    program.u_matrix = gl.getUniformLocation(program, 'u_matrix');

    // 立方体着色器程序对象
    var program2 = createShader(gl, document.getElementById('vShaderSource2').innerHTML,
        document.getElementById('fShaderSource2').innerHTML);
    program2.a_position = gl.getAttribLocation(program2, 'a_position');
    program2.a_normal = gl.getAttribLocation(program2, 'a_normal');
    program2.u_matrix = gl.getUniformLocation(program2, 'u_matrix');
    program2.u_mMatrix = gl.getUniformLocation(program2, 'u_mMatrix');
    program2.u_normalMatrix = gl.getUniformLocation(program2, 'u_normalMatrix');
    program2.u_eyePosition = gl.getUniformLocation(program2, 'u_eyePosition');
    program2.u_sampler = gl.getUniformLocation(program2, 'u_sampler');

    // 离屏绘制尺寸
    var OFF_SCREEN_WIDTH = 512;
    var OFF_SCREEN_HEIGHT = 512;

    // 立方贴图六个面的目标{右、左、上、下、前、后}
    var cubeTextureTargets = [
        gl.TEXTURE_CUBE_MAP_POSITIVE_X,
        gl.TEXTURE_CUBE_MAP_NEGATIVE_X,
        gl.TEXTURE_CUBE_MAP_POSITIVE_Y,
        gl.TEXTURE_CUBE_MAP_NEGATIVE_Y,
        gl.TEXTURE_CUBE_MAP_POSITIVE_Z,
        gl.TEXTURE_CUBE_MAP_NEGATIVE_Z
    ];

    // 初始化天空盒buffer
    var sky = initCubeBuffer();

    // 初始化立方体buffer
    var cube = initCubeBuffer();

    // 初始化帧缓冲区
    var fbo = initFBO();

    // 初始化天空盒纹理
    var skyTexture = initSkyTexture();

    // 相机的位置
    var eyePosition = new Float32Array([2.0, 2.0, 6.0]);

    // 视图投影矩阵
    var pvMatrix = createPvMatrix({eye: eyePosition});

    // 渲染循环
    var t = 0;
    var loop = function(){
        drawFBO(t); // 在帧缓冲区中绘制天空盒
        draw(t); // 常规绘制天空盒和立方体
        t++;
        window.requestAnimationFrame(loop);
    }
    loop();


    function draw(t){
        gl.viewport(0, 0, canvas.width, canvas.height);
        gl.clearColor(0.0, 0.0, 0.0, 1.0);
        gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
        gl.enable(gl.DEPTH_TEST);

        drawSky(t);
        drawCube(t);
    }

    function drawFBO(t){
        gl.bindFramebuffer(gl.FRAMEBUFFER, fbo);

        gl.viewport(0, 0, OFF_SCREEN_WIDTH, OFF_SCREEN_HEIGHT);
        gl.clearColor(0.0, 0.0, 0.0, 1.0);
        gl.disable(gl.DEPTH_TEST);

        gl.useProgram(program);

        cubeTextureTargets.forEach(function(target, index){
            gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, target, fbo.texture, null);

            var eye = new Float32Array([0.0, 0.0, 0.0]);
            var up = vec3.create();
            var center = vec3.create();

            switch(index){
                case 0:
                    // 看右
                    up[1] = -1.0;
                    center[0] = 1.0;
                    break;
                case 1:
                    // 看左
                    up[1] = -1.0;
                    center[0] = -1.0;
                    break;
                case 2:
                    // 看上
                    up[2] = -1.0;
                    center[1] = 1.0;
                    break;
                case 3:
                    // 看下
                    up[2] = -1.0;
                    center[1] = -1.0;
                    break;
                case 4:
                    // 看前
                    up[1] = -1.0;
                    center[2] = 1.0;
                    break;
                case 5:
                    // 看后
                    up[1] = -1.0;
                    center[2] = -1.0;
                    break;
            }

            // FBO用自己的视图投影矩阵
            var pvMatrixFBO = createPvMatrix({
                eye: eye,
                center: center,
                up: up
            });

            drawSky(t, pvMatrixFBO);
        });

        gl.bindFramebuffer(gl.FRAMEBUFFER, null);
    }

    function drawCube(t){
        gl.enable(gl.CULL_FACE);
        gl.cullFace(gl.BACK);

        gl.useProgram(program2);

        gl.bindBuffer(gl.ARRAY_BUFFER, cube.vBuffer);
        gl.vertexAttribPointer(program2.a_position, 3, gl.FLOAT, false, 0, 0);
        gl.enableVertexAttribArray(program2.a_position);

        gl.bindBuffer(gl.ARRAY_BUFFER, cube.nBuffer);
        gl.vertexAttribPointer(program2.a_normal, 3, gl.FLOAT, false, 0, 0);
        gl.enableVertexAttribArray(program2.a_normal);

        var mMatrix = mat4.create();
        var pvmMatrix = mat4.create();
        mat4.scale(mMatrix, mMatrix, new Float32Array([0.1, 0.1, 0.1]));
        mat4.multiply(pvmMatrix, pvMatrix, mMatrix);
        gl.uniformMatrix4fv(program2.u_matrix, false, pvmMatrix);

        gl.uniformMatrix4fv(program2.u_mMatrix, false, mMatrix);

        var nMatrix = mat4.create();
        mat4.invert(nMatrix, mMatrix);
        mat4.transpose(nMatrix, nMatrix);
        gl.uniformMatrix4fv(program2.u_normalMatrix, false, nMatrix);

        gl.uniform3fv(program2.u_eyePosition, eyePosition);

        gl.activeTexture(gl.TEXTURE0);
        gl.bindTexture(gl.TEXTURE_CUBE_MAP, fbo.texture);

        gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, cube.iBuffer);
        gl.drawElements(gl.TRIANGLES, cube.indexCount, gl.UNSIGNED_BYTE, 0);
    }

    function drawSky(t, pvMatrixFBO){

        gl.enable(gl.CULL_FACE);
        gl.cullFace(gl.FRONT);

        var _pvMatrix = pvMatrixFBO ? pvMatrixFBO : pvMatrix;

        gl.useProgram(program);

        gl.bindBuffer(gl.ARRAY_BUFFER, sky.vBuffer);
        gl.vertexAttribPointer(program.a_position, 3, gl.FLOAT, false, 0, 0);
        gl.enableVertexAttribArray(program.a_position);

        gl.bindBuffer(gl.ARRAY_BUFFER, sky.cBuffer);
        gl.vertexAttribPointer(program.a_color, 3, gl.FLOAT, false, 0, 0);
        gl.enableVertexAttribArray(program.a_color);

        gl.activeTexture(gl.TEXTURE0);
        gl.bindTexture(gl.TEXTURE_CUBE_MAP, skyTexture);

        var mMatrix = mat4.create();
        var pvmMatrix = mat4.create();
        mat4.rotate(mMatrix, mMatrix, Math.sin(t*0.001)*Math.PI, new Float32Array([0.0, 1.0, 0.0]));
        mat4.multiply(pvmMatrix, _pvMatrix, mMatrix);
        gl.uniformMatrix4fv(program.u_matrix, false, pvmMatrix);

        gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, sky.iBuffer);
        gl.drawElements(gl.TRIANGLES, sky.indexCount, gl.UNSIGNED_BYTE, 0);
    }

    function initSkyTexture(){
        var texture = gl.createTexture();
        loadImages([
            '../resources/envir/right.jpg',
            '../resources/envir/left.jpg',
            '../resources/envir/top.jpg',
            '../resources/envir/bottom.jpg',
            '../resources/envir/front.jpg',
            '../resources/envir/back.jpg'
        ], function(images){
            gl.bindTexture(gl.TEXTURE_CUBE_MAP, texture);
            gl.texParameteri(gl.TEXTURE_CUBE_MAP, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
            cubeTextureTargets.forEach(function(target, index){
                gl.texImage2D(target, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, images[index]);
            });
            gl.useProgram(program);
            gl.uniform1i(program.u_sampler, 0);
            gl.bindTexture(gl.TEXTURE_CUBE_MAP, null);
        });
        return texture;
    }

    function initFBO(){
        var frameBuffer = gl.createFramebuffer();

        var texture = gl.createTexture();
        gl.bindTexture(gl.TEXTURE_CUBE_MAP, texture);
        cubeTextureTargets.forEach(function(target){
            gl.texImage2D(target, 0, gl.RGBA, OFF_SCREEN_WIDTH, OFF_SCREEN_HEIGHT, 0, gl.RGBA, gl.UNSIGNED_BYTE, null);
        });
        gl.texParameteri(gl.TEXTURE_CUBE_MAP, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
        frameBuffer.texture = texture;

        gl.bindTexture(gl.TEXTURE_CUBE_MAP, null);
        gl.bindFramebuffer(gl.FRAMEBUFFER, null);

        return frameBuffer;
    }

    function createPvMatrix(params){
        var params = params || {};
        var eye = params.eye || new Float32Array([0.0, 0.0, 10.0]);
        var center = params.center || new Float32Array([0.0, 0.0, 0.0]);
        var up = params.up || new Float32Array([0.0, 1.0, 0.0]);

        // 视图矩阵
        var vMatrix = mat4.create();
        mat4.lookAt(
            vMatrix, // 输出的值会赋给这个变量
            eye, // 眼睛的位置
            center, // 眼睛聚焦的位置
            up // 上方向
        );
        // 投影矩阵
        var pMatrix = mat4.create();
        mat4.perspective(
            pMatrix, // 输出的值会赋给这个变量
            70, // 眼睛视野的垂直角度(一般是120度)
            canvas.width/canvas.height, // 视口宽高比
            0.1, // 眼睛离视椎体近截面距离
            1000.0 // 眼睛离视椎体远截面距离
        );
        // 视图投影矩阵 = 投影矩阵 * 视图矩阵，相当于three.js封装的相机
        var pvMatrix = mat4.create();
        mat4.multiply(pvMatrix, pMatrix, vMatrix);
        return pvMatrix;
    }

    function initCubeBuffer(){
        // 立方体数据
        //    v6----- v5
        //   /|      /|
        //  v1------v0|
        //  | |     | |
        //  | |v7---|-|v4
        //  |/      |/
        //  v2------v3
        var vData = new Float32Array([
            10.0, 10.0, 10.0,  -10.0, 10.0, 10.0,  -10.0,-10.0, 10.0,   10.0,-10.0, 10.0,    // v0-v1-v2-v3 前
            10.0, 10.0, 10.0,   10.0,-10.0, 10.0,   10.0,-10.0,-10.0,   10.0, 10.0,-10.0,    // v0-v3-v4-v5 右
            10.0, 10.0, 10.0,   10.0, 10.0,-10.0,  -10.0, 10.0,-10.0,  -10.0, 10.0, 10.0,    // v0-v5-v6-v1 上
            -10.0, 10.0, 10.0,  -10.0, 10.0,-10.0,  -10.0,-10.0,-10.0,  -10.0,-10.0, 10.0,    // v1-v6-v7-v2 左
            -10.0,-10.0,-10.0,   10.0,-10.0,-10.0,   10.0,-10.0, 10.0,  -10.0,-10.0, 10.0,    // v7-v4-v3-v2 下
            10.0,-10.0,-10.0,  -10.0,-10.0,-10.0,  -10.0, 10.0,-10.0,   10.0, 10.0,-10.0     // v4-v7-v6-v5 后
        ]);
        var iData = new Uint8Array([
            0,  1,  2,   0,  2,  3,     // 前
            4,  5,  6,   4,  6,  7,     // 右
            8,  9,  10,  8,  10, 11,    // 上
            12, 13, 14,  12, 14, 15,    // 左
            16, 17, 18,  16, 18, 19,    // 下
            20, 21, 22,  20, 22, 23     // 后
        ]);
        var nData = new Float32Array([
            0.0, 0.0, 1.0,   0.0, 0.0, 1.0,   0.0, 0.0, 1.0,   0.0, 0.0, 1.0,  // v0-v1-v2-v3 front
            1.0, 0.0, 0.0,   1.0, 0.0, 0.0,   1.0, 0.0, 0.0,   1.0, 0.0, 0.0,  // v0-v3-v4-v5 right
            0.0, 1.0, 0.0,   0.0, 1.0, 0.0,   0.0, 1.0, 0.0,   0.0, 1.0, 0.0,  // v0-v5-v6-v1 up
            -1.0, 0.0, 0.0,  -1.0, 0.0, 0.0,  -1.0, 0.0, 0.0,  -1.0, 0.0, 0.0,  // v1-v6-v7-v2 left
            0.0,-1.0, 0.0,   0.0,-1.0, 0.0,   0.0,-1.0, 0.0,   0.0,-1.0, 0.0,  // v7-v4-v3-v2 down
            0.0, 0.0,-1.0,   0.0, 0.0,-1.0,   0.0, 0.0,-1.0,   0.0, 0.0,-1.0   // v4-v7-v6-v5 back
        ]);
        var _cData = new Float32Array([
            1.0, 0.0, 0.0,  1.0, 0.0, 0.0,  1.0, 0.0, 0.0,  1.0, 0.0, 0.0,  // 前 红
            0.0, 1.0, 0.0,  0.0, 1.0, 0.0,  0.0, 1.0, 0.0,  0.0, 1.0, 0.0,  // 右 绿
            0.0, 0.0, 1.0,  0.0, 0.0, 1.0,  0.0, 0.0, 1.0,  0.0, 0.0, 1.0,  // 上 蓝
            1.0, 1.0, 0.0,  1.0, 1.0, 0.0,  1.0, 1.0, 0.0,  1.0, 1.0, 0.0,  // 左 黄
            1.0, 0.0, 1.0,  1.0, 0.0, 1.0,  1.0, 0.0, 1.0,  1.0, 0.0, 1.0,  // 下 紫
            0.0, 1.0, 1.0,  0.0, 1.0, 1.0,  0.0, 1.0, 1.0,  0.0, 1.0, 1.0   // 后 青
        ]);
        var cData = new Float32Array([
            1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,
            1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,
            1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,
            1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,
            1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,
            1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0,  1.0, 1.0, 1.0
        ]);
        var o = {};
        o.vBuffer = createBufferForLaterUse(gl, gl.ARRAY_BUFFER, vData);
        o.iBuffer = createBufferForLaterUse(gl, gl.ELEMENT_ARRAY_BUFFER, iData);
        o.nBuffer = createBufferForLaterUse(gl, gl.ARRAY_BUFFER, nData);
        o.cBuffer = createBufferForLaterUse(gl, gl.ARRAY_BUFFER, cData);
        o.indexCount = iData.length;
        return o;
    }

    function createBufferForLaterUse(gl, type, data){
        var buffer = gl.createBuffer();
        gl.bindBuffer(type, buffer);
        gl.bufferData(type, data, gl.STATIC_DRAW);
        gl.bindBuffer(type, null);
        return buffer;
    }

    function createShader(gl, vshader, fshader){

        var loadShader = function(gl, type, source){
            var shader = gl.createShader(type);
            if (shader == null) {
                console.log('unable to create shader');
                return null;
            }
            gl.shaderSource(shader, source);
            gl.compileShader(shader);
            var compiled = gl.getShaderParameter(shader, gl.COMPILE_STATUS);
            if (!compiled) {
                var error = gl.getShaderInfoLog(shader);
                console.log('failed to compile shader: ' + error);
                gl.deleteShader(shader);
                return null;
            }
            return shader;
        };

        var createProgram = function(gl, vshader, fshader) {
            var vertexShader = loadShader(gl, gl.VERTEX_SHADER, vshader);
            var fragmentShader = loadShader(gl, gl.FRAGMENT_SHADER, fshader);
            if (!vertexShader || !fragmentShader) {
                return null;
            }
            var program = gl.createProgram();
            if (!program) {
                return null;
            }
            gl.attachShader(program, vertexShader);
            gl.attachShader(program, fragmentShader);
            gl.linkProgram(program);
            var linked = gl.getProgramParameter(program, gl.LINK_STATUS);
            if (!linked) {
                var error = gl.getProgramInfoLog(program);
                console.log('failed to link program: ' + error);
                gl.deleteProgram(program);
                gl.deleteShader(fragmentShader);
                gl.deleteShader(vertexShader);
                return null;
            }
            return program;
        };

        var program = createProgram(gl, vshader, fshader);
        if (!program) {
            console.log('failed to create program');
            return false;
        }

        return program;
    }

    function loadImages(urls, callback){
        var images = [];
        var i = 0;
        var _loop = function(url){
            var img  = new Image();
            img.onload = function(){
                images.push(img);
                if(images.length < urls.length){
                    _loop(urls[++i]);
                }else{
                    callback(images);
                }
            };
            img.src = url;
        }
        _loop(urls[0]);
    }
</script>

</body>
</html>
```