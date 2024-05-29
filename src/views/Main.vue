<template>
  <div id="main_render">
    <canvas @mousedown.left="startCameraReposition" @mouseup.left="stopCameraReposition" @mousewheel="zoomCamera" @mousemove="dragCamera" @mousedown.middle="startOffsetRepositioning" @mouseup.middle="stopOffsetRepositioning" id="c"></canvas>
    <div style="display:inline-block;">
        values:<br>
        a: <input type="range" v-model="test.a" step="0.02" min="-1" max="1" /> res: {{test.a}}<br> 
        b: <input type="range" v-model="test.b" step="0.02" min="-1" max="1" /> res: {{test.b}}<br>
        c: <input type="range" v-model="test.c" step="0.02" min="-1" max="1" /> res: {{test.c}}<br>
        d: <input type="range" v-model="test.d" step="0.02" min="-1" max="1" /> res: {{test.d}}<br>

        <canvas id="canvas" width="500" height="500"></canvas>
    </div>
    <div style="display:inline-block;border-left:2px solid red;padding: 5px;display:flex;">
        <div style="display:inline-block;padding: 5px;">
            sin(dir) = {{Math.sin(cameraPosition.direction)}}<br>
            sin(ang) = {{Math.sin(cameraPosition.angle)}}<br>
            cos(dir) = {{Math.cos(cameraPosition.direction)}}<br>
            cos(ang) = {{Math.cos(cameraPosition.angle)}}<br>
        </div>
        <div style="display:inline-block;padding: 5px;">
            sin(dir)*sin(dir) = {{Math.sin(cameraPosition.direction)*Math.sin(cameraPosition.direction)}}<br>
            sin(ang)*sin(ang) = {{Math.sin(cameraPosition.angle)*Math.sin(cameraPosition.angle)}}<br>
            cos(dir)*cos(dir) = {{Math.cos(cameraPosition.direction)*Math.cos(cameraPosition.direction)}}<br>
            cos(ang)*cos(ang) = {{Math.cos(cameraPosition.angle)*Math.cos(cameraPosition.angle)}}<br>
        </div>

        <div style="display:inline-block;padding: 5px;">
            sin(dir)*cos(ang) = {{Math.sin(cameraPosition.direction)*Math.sin(cameraPosition.direction)}}
        </div>
        

        sin(dir)*sin(dir) = {{Math.sin(cameraPosition.direction)*Math.sin(cameraPosition.direction)}}
    </div>
  </div>
</template>

<script>
import { create, all } from "mathjs";
const math = create(all);

export default {
    name: 'MainPage',
    data: function() {
        return {
            test: {
                a: 0.5,
                b: 0,
                c: 0,
                d: 0.5
            },
            resourcesLoaded: false,
            canvas: null,
            ctx: null,
            times: [],
            fps: 0,
            radFactor: (Math.PI / 180),
            scaleVectors: {
                x: math.matrix([1, 0, 0, 1]),
                // x: [1, 0, 0, 1],
                y: math.matrix([0, 1, 0, 1]),
                // y: [0, 1, 0, 1],
                z: math.matrix([0, 0, 1, 1])
                // z: [0, 0, 1, 1],
            },
            resources: {
                terrain: {
                    name: 'terrain',
                    src: 'terrain.jpg',
                    loaded: false,
                    imgObj: null
                },
                wood_box: {
                    name: 'wood_box',
                    src: 'wood_box.jpg',
                    loaded: false,
                    imgObj: null
                }
            },
            scale: 100,
            mouseTimestamp: null,
            velcosityX: 2,
            velcosityY: 2,
            offsetX: 0,
            offsetY: 0,
            mouseOldX: 0,
            mouseOldY: 0,
            cameraRepositioning: false,
            cameraOffsetRepositioning: false,
            cameraPosition: {
                direction: 90, //deg
                angle: 90, //(deg)
                zNear: 0.1,
                zFar: 0.2
            },
            terrain: null,
            box: {
                vertices: [
                    math.matrix([1, 1, 0, 1]),
                    math.matrix([1, 1, 1, 1]),
                    math.matrix([1, 2, 0, 1]),
                    math.matrix([1, 2, 1, 1]),
                    math.matrix([2, 1, 0, 1]),
                    math.matrix([2, 1, 1, 1]),
                    math.matrix([2, 2, 0, 1]),
                    math.matrix([2, 2, 1, 1]),
                ],
                textures: [
                    {
                        offset: [1, 1, 2, 1],
                        resource: 'wood_box'
                    }
                ]
            },
            cube: [
                math.matrix([1, 1, 1, 1]),
                math.matrix([1, 1, 2, 1]),
                math.matrix([1, 2, 1, 1]),
                math.matrix([1, 2, 2, 1]),
                math.matrix([2, 1, 1, 1]),
                math.matrix([2, 1, 2, 1]),
                math.matrix([2, 2, 1, 1]),
                math.matrix([2, 2, 2, 1]),
            ],
            camera: math.matrix([[1, 0, 0], [0 ,1, 0], [0 ,0, 1], [0, 0, 0]])
        }
    },
    mounted() {
        this.canvas = document.getElementById("c");
        this.canvas.width = window.innerWidth;
        this.canvas.height = window.innerHeight - 200;
        this.ctx = this.canvas.getContext("2d");
        this.getFps();
        this.initResources();
        this.gameLoop();
    },
    methods: {
        gameLoop: function() {
            this.drawInit();

            if (this.resourcesLoaded) {
                this.skawImage();
            }
            
            this.drawTerrain();
            this.drawVectors();
            // this.drawCube();
            this.drawBox();
            
            requestAnimationFrame(this.gameLoop);
        },
        clearCanvas: function() {
            this.ctx.clearRect(0, 0, this.ctx.width, this.ctx.height);
        },
        initResources: function() {
            Object.keys(this.resources).forEach((key) => {
                this.resources[key].imgObj = new Image();
                this.resources[key].imgObj.addEventListener('load', () => {
                    this.resources[key].loaded = true;
                });
                this.resources[key].imgObj.src = this.resources[key].src;
            });
        },
        drawInit: function() {
            this.clearCanvas();

            this.ctx.beginPath();
            this.ctx.fillStyle = "lightgray";
            this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);

            this.ctx.beginPath();
            this.ctx.fillStyle = "lightgray";
            this.ctx.fillRect(0, 0, 250, 60);
            this.ctx.fillStyle = "blue";
            this.ctx.fillText(`Canvas `, 3, 10);
            this.ctx.fillText(`Dimensions: { width: ${this.canvas.width}px, height: ${this.canvas.height}px } `, 3, 20);
            this.ctx.fillText(`cameraPosition: { direction: ${this.cameraPosition.direction}°, angle: ${this.cameraPosition.angle}° } `, 3, 30);
            var center = this.getCenter();
            this.ctx.fillText(`Center: { width: ${center.x}px, height: ${center.y}px }`, 3, 40);
            this.ctx.fillText(`FPS: ${this.fps} FPS`, 3, 50);
        },
        getCenter: function() {
            return {
                x: this.canvas.width /2 + this.offsetX,
                y: this.canvas.height /2 + this.offsetY,
            }
        },
        drawTerrain: function() {
            var center = this.getCenter();
            let size = 5;
            let gap = 0.5;

            // let rows = (size / gap) + 1;
            let terrainData = [];

            let r_i = -size
            while (r_i < size + gap) {
                terrainData.push([
                    math.matrix([-size, r_i, 0, 1]),
                    math.matrix([size, r_i, 0, 1])
                ]);
                terrainData.push([
                    math.matrix([r_i, -size, 0, 1]),
                    math.matrix([r_i, size, 0, 1])
                ]);
                r_i = r_i + gap;
            }

            terrainData.forEach((lineData) => {
                this.ctx.beginPath();
                let start_point_matrix = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), lineData[0]]);
                let end_point_matrix = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), lineData[1]]);
                this.ctx.moveTo(center.x + start_point_matrix._data[0], center.y - start_point_matrix._data[1]);
                this.ctx.lineTo(center.x + end_point_matrix._data[0], center.y - end_point_matrix._data[1]);
                this.ctx.strokeStyle = 'gray';
                this.ctx.stroke();
            });

            if (this.resources.terrain.loaded) {
                this.skawImage(this.resources.terrain, [-5, 5, 0, 1], 10*this.scale, 10*this.scale);
            }
        },
        drawVectors: function() {
            var center = this.getCenter();

            this.ctx.beginPath();
            let x_matrix = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), this.scaleVectors.x]);
            this.ctx.moveTo(center.x, center.y);
            this.ctx.lineTo(center.x + x_matrix._data[0], center.y - x_matrix._data[1]);
            this.ctx.strokeStyle = 'red';
            this.ctx.stroke();

            this.ctx.beginPath();
            let y_matrix = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), this.scaleVectors.y]);
            this.ctx.moveTo(center.x, center.y);
            this.ctx.lineTo(center.x + y_matrix._data[0], center.y - y_matrix._data[1]);
            this.ctx.strokeStyle = 'green';
            this.ctx.stroke();

            this.ctx.beginPath();
            let z_matrix = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), this.scaleVectors.z]);
            this.ctx.moveTo(center.x, center.y);
            this.ctx.lineTo(center.x + z_matrix._data[0], center.y - z_matrix._data[1]);
            this.ctx.strokeStyle = 'blue';
            this.ctx.stroke();


            ///

            // this.ctx.beginPath();
            // // this.ctx.moveTo(center.x + 70, center.y - 20);
            // // this.ctx.lineTo(center.x - 70, center.y - 20);

            // this.ctx.moveTo(center.x + (Math.cos((this.cameraPosition.direction - 90) * this.radFactor) * 70), center.y - (Math.sin((this.cameraPosition.direction - 90) * this.radFactor) * 40));
            // this.ctx.lineTo(center.x - (Math.cos((this.cameraPosition.direction - 90) * this.radFactor) * 70), center.y - (Math.sin((this.cameraPosition.direction - 90) * this.radFactor) * 40));


            // this.ctx.strokeStyle = 'green';
            // this.ctx.stroke();
        },
        drawCube: function() {
            var center = this.getCenter();
            this.cube.forEach((item) => {
                this.ctx.beginPath();
                let point_matrix = this.multiplyMatrices([this.getTranslationMatrix({x: 50, y: 50, z: 50}), this.getScaleMatrix({x: this.scale / 2, y: this.scale / 2, z: this.scale / 2}), this.getRotateAroundXMatrix(-this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), item]);
                // this.ctx.moveTo(center.x, center.y);
                this.ctx.fillStyle = 'white';
                this.ctx.fillRect(center.x + point_matrix._data[0], center.y - point_matrix._data[1], 5, 5);
                this.ctx.stroke();
            });
        },
        drawBox: function() {
            
            if (this.resources.wood_box.loaded) {
                var center = this.getCenter();
                this.box.vertices.forEach((vertex) => {
                    this.ctx.beginPath();
                    let point_matrix = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), vertex]);
                    this.ctx.fillStyle = 'black';
                    this.ctx.fillRect(center.x + point_matrix._data[0], center.y - point_matrix._data[1], 5, 5);
                    this.ctx.stroke();
                });

                this.box.textures.forEach((texture) => {
                    });
                    this.skawImage2(this.resources['wood_box'], [1, 1, 1, 1], this.scale, this.scale);
                    this.skawImage3(this.resources['wood_box'], [2, 1, 1, 1], this.scale, this.scale);
            }


        },
        skawImage: function(resource, offset, width, height) {
            let center = this.getCenter();
            var canvas = document.createElement('canvas');
            canvas.id = "virtual";

            canvas.width = 1920;
            canvas.height = 1080;
            let context = canvas.getContext('2d');

            let pointData = math.matrix(offset);
            let pointData2 = math.matrix([5, -5, 0, 1]);
            let pointData3 = math.matrix([5, 5, 0, 1]);
            let pointData4 = math.matrix([-5, 5, 0, 1]);

            let start_point_matrix1 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData]);
            let start_point_matrix2 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData2]);
            let start_point_matrix3 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData3]);
            let start_point_matrix4 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData4]);


            this.ctx.fillStyle = 'red';
            this.ctx.fillRect(center.x + start_point_matrix1._data[0], center.y - start_point_matrix1._data[1], 5, 5);
            this.ctx.fillStyle = 'green';
            this.ctx.fillRect(center.x + start_point_matrix2._data[0], center.y - start_point_matrix2._data[1], 5, 5);
            this.ctx.fillStyle = 'blue';
            this.ctx.fillRect(center.x + start_point_matrix3._data[0], center.y - start_point_matrix3._data[1], 5, 5);
            this.ctx.fillRect(center.x + start_point_matrix4._data[0], center.y - start_point_matrix4._data[1], 5, 5);

            context.setTransform(
                Math.cos(this.cameraPosition.direction * this.radFactor),
                Math.sin(this.cameraPosition.direction * this.radFactor)*Math.cos(this.cameraPosition.angle * this.radFactor),
                -Math.sin(this.cameraPosition.direction * this.radFactor),
                Math.cos(this.cameraPosition.direction * this.radFactor)*Math.cos(this.cameraPosition.angle * this.radFactor),
                center.x + start_point_matrix1._data[0],
                center.y - start_point_matrix1._data[1]
            );

            context.drawImage(resource.imgObj, 0, 0, width, height);

            this.ctx.drawImage(canvas, 0, 0)
        },
        skawImage2: function(resource, offset, width, height) {
            let center = this.getCenter();
            var canvas = document.createElement('canvas');
            canvas.id = "virtual";

            canvas.width = 1920;
            canvas.height = 1080;
            let context = canvas.getContext('2d');

            let pointData = math.matrix(offset);
            let pointData2 = math.matrix([5, -5, 0, 1]);
            let pointData3 = math.matrix([5, 5, 0, 1]);
            let pointData4 = math.matrix([-5, 5, 0, 1]);

            let start_point_matrix1 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData]);
            let start_point_matrix2 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData2]);
            let start_point_matrix3 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData3]);
            let start_point_matrix4 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData4]);

            this.ctx.fillStyle = 'red';
            this.ctx.fillRect(center.x + start_point_matrix1._data[0], center.y - start_point_matrix1._data[1], 5, 5);
            this.ctx.fillStyle = 'green';
            this.ctx.fillRect(center.x + start_point_matrix2._data[0], center.y - start_point_matrix2._data[1], 5, 5);
            this.ctx.fillStyle = 'blue';
            this.ctx.fillRect(center.x + start_point_matrix3._data[0], center.y - start_point_matrix3._data[1], 5, 5);
            this.ctx.fillRect(center.x + start_point_matrix4._data[0], center.y - start_point_matrix4._data[1], 5, 5);
            
            // context.rotate(90 * this.radFactor);
            // context.setTransform(
            //     Math.cos(this.cameraPosition.direction * this.radFactor),
            //     Math.sin(this.cameraPosition.direction * this.radFactor)*Math.cos(this.cameraPosition.angle * this.radFactor),
            //     -Math.sin(this.cameraPosition.direction * this.radFactor),
            //     Math.cos(this.cameraPosition.direction * this.radFactor)*Math.cos(this.cameraPosition.angle * this.radFactor),
            //     center.x + start_point_matrix1._data[0],
            //     center.y - start_point_matrix1._data[1]
            // );
            context.setTransform(
                Math.cos(this.cameraPosition.direction * this.radFactor),
                Math.sin(this.cameraPosition.direction * this.radFactor)*Math.cos(this.cameraPosition.angle * this.radFactor),
                0,
                Math.sin(this.cameraPosition.angle * this.radFactor),
                center.x + start_point_matrix1._data[0],
                center.y - start_point_matrix1._data[1]
            );

            context.drawImage(resource.imgObj, 0, 0, width, height);
            this.ctx.drawImage(canvas, 0, 0)
        },
        skawImage3: function(resource, offset, width, height) {
            let center = this.getCenter();
            var canvas = document.createElement('canvas');
            canvas.id = "virtual";

            canvas.width = 1920;
            canvas.height = 1080;
            let context = canvas.getContext('2d');

            let pointData = math.matrix(offset);
            let pointData2 = math.matrix([5, -5, 0, 1]);
            let pointData3 = math.matrix([5, 5, 0, 1]);
            let pointData4 = math.matrix([-5, 5, 0, 1]);

            let start_point_matrix1 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData]);
            let start_point_matrix2 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData2]);
            let start_point_matrix3 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData3]);
            let start_point_matrix4 = this.multiplyMatrices([this.getTranslationMatrix({x: 0, y: 0, z: 0}), this.getScaleMatrix({x: this.scale, y: this.scale, z: this.scale}), this.getRotateAroundXMatrix(this.cameraPosition.angle), this.getRotateAroundYMatrix(0), this.getRotateAroundZMatrix(this.cameraPosition.direction), pointData4]);

            this.ctx.fillStyle = 'red';
            this.ctx.fillRect(center.x + start_point_matrix1._data[0], center.y - start_point_matrix1._data[1], 5, 5);
            this.ctx.fillStyle = 'green';
            this.ctx.fillRect(center.x + start_point_matrix2._data[0], center.y - start_point_matrix2._data[1], 5, 5);
            this.ctx.fillStyle = 'blue';
            this.ctx.fillRect(center.x + start_point_matrix3._data[0], center.y - start_point_matrix3._data[1], 5, 5);
            this.ctx.fillRect(center.x + start_point_matrix4._data[0], center.y - start_point_matrix4._data[1], 5, 5);
            
            // context.rotate(90 * this.radFactor);
            // context.setTransform(
            //     Math.cos(this.cameraPosition.direction * this.radFactor),
            //     Math.sin(this.cameraPosition.direction * this.radFactor)*Math.cos(this.cameraPosition.angle * this.radFactor),
            //     -Math.sin(this.cameraPosition.direction * this.radFactor),
            //     Math.cos(this.cameraPosition.direction * this.radFactor)*Math.cos(this.cameraPosition.angle * this.radFactor),
            //     center.x + start_point_matrix1._data[0],
            //     center.y - start_point_matrix1._data[1]
            // );
            context.setTransform(
                Math.cos((this.cameraPosition.direction-90) * this.radFactor),
                Math.sin((this.cameraPosition.direction-90) * this.radFactor)*Math.cos(this.cameraPosition.angle * this.radFactor),
                0,
                Math.sin(this.cameraPosition.angle * this.radFactor),
                center.x + start_point_matrix1._data[0],
                center.y - start_point_matrix1._data[1]
            );

            // context.setTransform(
            //     Math.cos(this.cameraPosition.direction * this.radFactor),
            //     Math.sin(this.cameraPosition.direction * this.radFactor)*Math.cos(this.cameraPosition.angle * this.radFactor),
            //     0,
            //     Math.sin(this.cameraPosition.angle * this.radFactor),
            //     center.x + start_point_matrix1._data[0],
            //     center.y - start_point_matrix1._data[1]
            // );

            context.drawImage(resource.imgObj, 0, 0, width, height);
            this.ctx.drawImage(canvas, 0, 0)
        },
        getFps() {
            window.requestAnimationFrame(() => {
                const now = performance.now();
                while (this.times.length > 0 && this.times[0] <= now - 1000) {
                    this.times.shift();
                }
                this.times.push(now);
                this.fps = this.times.length;
                this.getFps();
            });
        },
        getRotateAroundXMatrix: function(theta) {
            theta = theta * this.radFactor;
            return math.matrix([[1, 0, 0, 0], [0, Math.cos(theta), Math.sin(theta), 0], [0, -Math.sin(theta), Math.cos(theta), 0], [0, 0, 0, 1]]);
            // return math.multiply(matrix, rotationXMatrix);
        },
        getRotateAroundYMatrix: function(theta) {
            theta = theta * this.radFactor;
             return math.matrix([[Math.cos(theta), 0, -Math.sin(theta), 0], [0, 1, 0, 0], [Math.sin(theta), 0, Math.cos(theta), 0], [0, 0, 0, 1]]);
            // return math.multiply(matrix, rotationYMatrix);
        },
        getRotateAroundZMatrix: function(theta) {
            theta = theta * this.radFactor;
             return math.matrix([[Math.cos(theta), Math.sin(theta), 0, 0], [-Math.sin(theta), Math.cos(theta), 0, 0], [0, 0, 1, 0], [0, 0, 0, 1]]);
            // return math.multiply(matrix, rotationZMatrix);
        },
        getProjectionMatrix: function() {
            return math.matrix([[(1 / this.canvas.width), 0, 0, 0], [0, (1 / this.canvas.height), 0, 0], [0, 0, -(2 / (this.cameraPosition.zFar - this.cameraPosition.zNear)), 0], [0, 0, -((this.cameraPosition.zFar + this.cameraPosition.zNear) / (this.cameraPosition.zFar - this.cameraPosition.zNear)), 1]]);
            // return math.multiply(matrix, projectionMatrix);
        },
        getTranslationMatrix: function(translation) {
            return math.matrix([[1, 0, 0, 0], [0, 1, 0, 0], [0, 0, 1, 0], [translation.x, translation.y, translation.z, 1]]);
        },
        getScaleMatrix: function(scale) {
            return math.matrix([[scale.x, 0, 0, 0], [0, scale.y, 0, 0], [0, 0, scale.z, 0], [0, 0, 0, 1]]);
        },
        multiplyMatrices: function(matrices) {
            let matrix = math.identity(4);
            for(let i = 0; i < matrices.length; i++) {
                matrix = math.multiply(matrix, matrices[i]);
            }
            return matrix;
        },
        addMatrices: function(matrices) {
            let matrix = math.identity(4);
            for(let i = 0; i < matrices.length; i++) {
                matrix = math.add(matrix, matrices[i]);
            }
            return matrix;
        },
        startCameraReposition: function() {
            this.cameraRepositioning = true;
        },
        stopCameraReposition: function() {
            this.cameraRepositioning = false;
        },
        startOffsetRepositioning: function() {
            this.cameraOffsetRepositioning = true;
        },
        stopOffsetRepositioning: function() {
            this.cameraOffsetRepositioning = false;
        },
        dragCamera: function(e) {
            if (!this.cameraRepositioning && !this.cameraOffsetRepositioning) {
                return;
            }

            var now = Date.now();
            var dt = now - this.mouseTimestamp;
            var distanceX = Math.abs(e.pageX - this.mouseOldX);
            var distanceY = Math.abs(e.pageY - this.mouseOldY);
            this.velcosityX = Math.round(distanceX / dt * 1000) * (3 / 1000);
            this.velcosityY = Math.round(distanceY / dt * 1000) * (3 / 1000);
            this.mouseTimestamp = now;

            if (!isFinite(this.velcosityX)) {
                this.velcosityX = 0.2;
            }

            if (e.pageX > this.mouseOldX) {
                if (this.cameraRepositioning) {
                    this.cameraPosition.direction = (this.cameraPosition.direction + this.velcosityX) % 360;
                } else if (this.cameraOffsetRepositioning) {
                    this.offsetX = this.offsetX + this.velcosityX;
                }
            } else if (e.pageX < this.mouseOldX) {
                if (this.cameraRepositioning) {
                    this.cameraPosition.direction = (this.cameraPosition.direction - this.velcosityX) % 360;
                } else if (this.cameraOffsetRepositioning) {
                    this.offsetX = this.offsetX - this.velcosityX;
                }
            }

            if (e.pageY < this.mouseOldY) {
                if (this.cameraRepositioning) {
                    this.cameraPosition.angle = Math.min(this.cameraPosition.angle + this.velcosityY, 90);
                } else if (this.cameraOffsetRepositioning) {
                    //
                }
            } else if (e.pageY > this.mouseOldY) {
                if (this.cameraRepositioning) {
                    this.cameraPosition.angle = this.cameraPosition.angle - this.velcosityY < 0 ? 0 : this.cameraPosition.angle - this.velcosityY;
                } else if (this.cameraOffsetRepositioning) {
                    //
                }
            }

            // console.log(directionY);

            this.mouseOldX = e.pageX;
            this.mouseOldY = e.pageY;
        },
        zoomCamera: function(e) {

            let scale = this.scale - (e.deltaY * 0.3);
            if (scale <= 10) {
                this.scale = 10;
            } else {
                this.scale = this.scale - (e.deltaY * 0.3);
            }

        }
    }
}
</script>

<style>
    #main_render {
        height: calc(100vh - 200);
        width: 100%;
    }
    #c {
        width:100%;
        height: 100%;
        border: 2px solid lightcoral;
    }
</style>