<template>
  <div>
    <div id="container"></div>
  </div>
</template>

<script>
/* eslint-disable */
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import Stats from "three/examples/jsm/libs/stats.module.js";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass";
import { ShaderPass } from "three/examples/jsm/postprocessing/ShaderPass";

import GUI from "lil-gui";

export default {
  name: "HelloThreeD",
  data() {
    return {
      camera: null,
      scene: null,
      renderer: null,
      mesh: null,
      model: null,
      controls: null,
      bodyMatetial: null,
      wheelMatetial: null,
      glassMaterial: null,
      lightMaterial: null,
      composer: null,
      lightOpen: false,
    };
  },
  mounted() {
    this.init();
    this.initBloomPass();
    this.initFloor();
    this.initLight();
    this.animate();
    this.initGui();
  },
  methods: {
    //初始化
    init() {
      // 创建场景对象Scene
      this.scene = new THREE.Scene();
      // 背景颜色
      this.scene.background = new THREE.Color("#f9f9f9");
      //添加坐标轴
      var axes = new THREE.AxesHelper(500); //500表示xyz轴的长度，红:x,绿:y,蓝:z
      this.scene.add(axes);
      //网格模型添加到场景中
      let geometry = new THREE.BoxGeometry(1, 1, 1);
      let material = new THREE.MeshStandardMaterial({
        color: "black",
      });
      this.mesh = new THREE.Mesh(geometry, material);

      /**
       * 相机设置
       */
      let container = document.getElementById("container");
      //创建一个摄像机对象（摄像机决定了能够在场景里看到什么）
      this.camera = new THREE.PerspectiveCamera(
        40,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
      );

      //以下是 一个正常的属性设置，比较符合常规三维软件的视角：左右为X轴，右为正；前后为Y轴，前为正；上下为Z轴，上为正。
      this.camera.position.set(12, 7, 5);

      // 定义更换的车身材质 初始值 下有方法initGui支持点击选颜色更换
      this.bodyMatetial = new THREE.MeshStandardMaterial({
        color: "red",
        metalness: 1,  //金属度
        roughness: 0.5, //粗糙度
      });
      this.glassMaterial = new THREE.MeshPhysicalMaterial({
        color: "#222222",
        metalness: 0,   //金属度
        roughness: 1,   //粗糙度
        transmission: 1.0, //透光性
        opacity:0.8,    //透明度
        transparent: true, //透明
      });
      // 定义初始的车轱辘
      this.wheelMatetial = new THREE.MeshPhongMaterial({
        color: "#ffffff",
        metalness: 1,
        roughness: 0.5,
      });

      // 加载模型 模型必须整包放在public目录
      const gltfLoader = new GLTFLoader();
      gltfLoader.load("./car.gltf", (gltf) => {
        this.model = gltf.scene;
      console.log(gltf.scene,'gltf.scene')
        this.model.traverse((item) => {
        //   // 车身有这么多的部件构成 全部写上 更换为上面定义的材质bodyMatetial
          if (
            ["Obj3d66-870414-4-94", "Obj3d66-870414-4-94_3"].includes(item.name)
          ) {
            item.material = this.bodyMatetial;
          }
        //   //车窗玻璃
          if (
            ["Obj3d66-870414-4-94_1", "Obj3d66-870414-4-94_2"].includes(
              item.name
            )
          ) {
            item.material = this.glassMaterial;
          }
        //    //轱辘
          if (["Obj3d66-870414-4-94_12"].includes(item.name)) {
            item.material = new THREE.MeshStandardMaterial({
              color: "black",
              roughness: 1,
              metalness: 0.1,
            });
          }
          if (["Obj3d66-870414-4-94_13"].includes(item.name)) {
            item.material = this.wheelMatetial;
          }
        });
        this.scene.add(this.model);
      });

    
      /**
       * 创建渲染器对象
       */
      this.renderer = new THREE.WebGLRenderer({ antialias: true });
      this.renderer.setSize(container.clientWidth, container.clientHeight);
      container.appendChild(this.renderer.domElement);

      //创建控件对象
      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
     
    },
  
    // 动画 旋转
    animate() {
      this.renderer.render(this.scene, this.camera);

      if (this.lightOpen) {
        this.composer.render();
        this.glowComposer.render();
      }

      requestAnimationFrame(this.animate);
    },

    // 添加地面
    initFloor() {
      const floorGeometry = new THREE.PlaneGeometry(20, 20);
      const material = new THREE.MeshPhysicalMaterial({
        color: "#818181",
        side: THREE.DoubleSide,
        metalness: 0,
        roughness: 0.1,
      });
      const floorMesh = new THREE.Mesh(floorGeometry, material);
      // 默认垂直的 旋转以下变成水平的
      floorMesh.rotation.x = Math.PI / 2;
      floorMesh.receiveShadow = true;
      this.scene.add(floorMesh);
    },
  //辉光效果
    initBloomPass() {
      // 场景渲染器
      this.composer = new EffectComposer(this.renderer);
      const renderPass = new RenderPass(this.scene, this.camera);
      this.composer.addPass(renderPass);
      //创建辉光效果
      this.unrealBloomPass = new UnrealBloomPass(
        new THREE.Vector2(window.innerWidth, window.innerHeight),
        1.5,
        0.4,
        0.85
      );
      this.unrealBloomPass.threshold = 1; // 辉光强度
      this.unrealBloomPass.strength = 1.5; // 辉光阈值
      this.unrealBloomPass.radius = 1; //辉光半径
      this.unrealBloomPass.renderToScreen = false; //
      // 辉光合成器
      this.glowComposer = new EffectComposer(this.renderer);
      this.glowComposer.renderToScreen = false;
      this.glowComposer.addPass(new RenderPass(this.scene, this.camera));
      this.glowComposer.addPass(this.unrealBloomPass);

      // 着色器
      let shaderPass = new ShaderPass(
        new THREE.ShaderMaterial({
          uniforms: {
            baseTexture: { value: null },
            bloomTexture: { value: this.glowComposer.renderTarget2.texture },
            tDiffuse: {
              value: null,
            },
          },
          vertexShader:
            "\t\t\tvarying vec2 vUv;\n" +
            "\n" +
            "\t\t\tvoid main() {\n" +
            "\n" +
            "\t\t\t\tvUv = uv;\n" +
            "\n" +
            "\t\t\t\tgl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );\n" +
            "\n" +
            "\t\t\t}",
          fragmentShader:
            "\t\t\tuniform sampler2D baseTexture;\n" +
            "\t\t\tuniform sampler2D bloomTexture;\n" +
            "\n" +
            "\t\t\tvarying vec2 vUv;\n" +
            "\n" +
            "\t\t\tvoid main() {\n" +
            "\n" +
            "\t\t\t\tgl_FragColor = ( texture2D( baseTexture, vUv ) + vec4( 1.0 ) * texture2D( bloomTexture, vUv ) );\n" +
            "\n" +
            "\t\t\t}",
          defines: {},
        }),
        "baseTexture"
      );

      shaderPass.renderToScreen = true;
      shaderPass.needsSwap = true;
      this.composer.addPass(shaderPass);
    },
    // 添加聚光灯和阴影
    initLight() {
      const light = new THREE.DirectionalLight(0xffffff, 1); // 柔和的白光
      light.position.set(0, 0, 10);
      this.scene.add(light);

      const light2 = new THREE.DirectionalLight(0xffffff, 1); // 柔和的白光
      light2.position.set(0, 10, 0);
      this.scene.add(light2);

      const light3 = new THREE.DirectionalLight(0xffffff, 1); // 柔和的白光
      light3.position.set(0, 0, -10);
      this.scene.add(light3);

      const light4 = new THREE.DirectionalLight(0xffffff, 1); // 柔和的白光
      light4.position.set(10, 10, 0);
      this.scene.add(light4);

      const light5 = new THREE.DirectionalLight(0xffffff, 1); // 柔和的白光
      light5.position.set(0, 10, -10);
      this.scene.add(light5);

      const light6 = new THREE.DirectionalLight(0xffffff, 1); // 柔和的白光
      light5.position.set(10, 0, 0);
      this.scene.add(light6);
    },

    // 添加改变颜色的面板 这里用到包 lil-gui 需要安装引入
    // 网址 https://lil-gui.georgealways.com/
    initGui() {
      let obj = {
        bodyColor: "#ff0000",
        glassColor: "#793e3e",
        wheelColor: "#ffffff",
      };
      // 初始化
      const gui = new GUI();
      // gui.addColor就可以出现颜色选择器
      gui
        .addColor(obj, "bodyColor")
        .name("车身颜色")
        .onChange((value) => {
          this.bodyMatetial.color.set(value);
        });
      gui
        .addColor(obj, "glassColor")
        .name("车窗颜色")
        .onChange((value) => {
          this.glassMaterial.color.set(value);
        });
      gui
        .addColor(obj, "wheelColor")
        .name("车轱辘颜色")
        .onChange((value) => {
          this.wheelMatetial.color.set(value);
        });
      //设置灯光开关
      gui
        .add(this, "lightOpen")
        .name("开关")
        .onChange((value) => {
          this.lightOpen = value;
        });
    },

  },
};
</script>
<style>
html,
body,
#app {
  margin: 0px !important;
}
#container {
  height: 100vh;
}
.btn {
  position: absolute;
  top: 0px;
  left: 0px;
}
</style>
