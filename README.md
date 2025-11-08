<!doctype html>
<html lang="vi">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>EON-Ω Universe — 3D Simulator (Solar System Complete)</title>
  <style>
    html,body{height:100%;margin:0;background:#000;overflow:hidden;font-family:Arial,Helvetica,sans-serif}
    #info{position:absolute;left:12px;top:12px;z-index:2;color:#ddd;background:rgba(0,0,0,0.35);padding:10px;border-radius:8px}
    #footer{position:absolute;left:12px;bottom:12px;color:#aaa;font-size:12px}
    #tooltip{
      position:absolute;color:#fff;background:rgba(0,0,0,0.7);padding:4px 8px;border-radius:4px;
      pointer-events:none;font-size:13px;display:none;
      transition: opacity 0.15s ease;
      opacity:0;
    }
    canvas{display:block}
  </style>
</head>
<body>
  <div id="info">
    <strong>EON-Ω Universe (Solar System Complete)</strong><br>
    Phím: 1: Starfield, 2: Galaxy, 3: Planets, 4: Nebula, 5: Black Hole<br>
    Chuột: kéo xoay, cuộn phóng to/thu nhỏ
  </div>
  <div id="tooltip"></div>
  <div id="footer">Mô phỏng không gian vũ trụ — Lưu file và mở trong trình duyệt</div>

  <script src="https://cdn.jsdelivr.net/npm/dat.gui@0.7.9/build/dat.gui.min.js"></script>

  <script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.module.js"
    }
  }
  </script>

  <script type="module">
    import * as THREE from 'three';
    import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.152.2/examples/jsm/controls/OrbitControls.js';
    import { EffectComposer } from 'https://cdn.jsdelivr.net/npm/three@0.152.2/examples/jsm/postprocessing/EffectComposer.js';
    import { RenderPass } from 'https://cdn.jsdelivr.net/npm/three@0.152.2/examples/jsm/postprocessing/RenderPass.js';
    import { ShaderPass } from 'https://cdn.jsdelivr.net/npm/three@0.152.2/examples/jsm/postprocessing/ShaderPass.js';
    import { FXAAShader } from 'https://cdn.jsdelivr.net/npm/three@0.152.2/examples/jsm/shaders/FXAAShader.js';

    let scene, camera, renderer, controls, composer;
    let width = window.innerWidth, height = window.innerHeight;
    let clock = new THREE.Clock();
    let groupGalaxy, groupPlanets, starField, nebulaGroup, blackHoleGroup;
    let raycaster = new THREE.Raycaster();
    let mouse = new THREE.Vector2();
    const tooltip = document.getElementById('tooltip');

    let params = {
      starCount: 30000,
      galaxyArms: 3,
      galaxyParticles: 50000,
      showStarfield: true,
      showGalaxy: true,
      showPlanets: true,
      showNebula: true,
      showBlackHole: true,
      timeScale: 1
    };

    init();
    animate();

    function init(){
      scene = new THREE.Scene();
      scene.fog = new THREE.FogExp2(0x000010, 0.00006);

      camera = new THREE.PerspectiveCamera(60, width/height, 0.1, 50000);
      camera.position.set(0, 250, 800);

      renderer = new THREE.WebGLRenderer({antialias:true});
      renderer.setSize(width, height);
      renderer.setPixelRatio(window.devicePixelRatio || 1);
      document.body.appendChild(renderer.domElement);

      controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;
      controls.dampingFactor = 0.05;

      composer = new EffectComposer(renderer);
      composer.addPass(new RenderPass(scene, camera));
      const fxaaPass = new ShaderPass(FXAAShader);
      fxaaPass.uniforms['resolution'].value.set(1 / window.innerWidth, 1 / window.innerHeight);
      fxaaPass.renderToScreen = true;
      composer.addPass(fxaaPass);

      const hemi = new THREE.HemisphereLight(0x9999ff, 0x080820, 0.6);
      scene.add(hemi);

      const mainLight = new THREE.PointLight(0xffffff, 1.2, 0, 2);
      mainLight.position.set(0,0,0);
      scene.add(mainLight);

      createStarField();
      createGalaxy();
      createPlanets();
      createNebula();
      createBlackHole();

      const gui = new dat.GUI();
      gui.add(params, 'showStarfield').name('Starfield').onChange(v=> starField.visible=v);
      gui.add(params, 'showGalaxy').name('Galaxy').onChange(v=> groupGalaxy.visible=v);
      gui.add(params, 'showPlanets').name('Planets').onChange(v=> groupPlanets.visible=v);
      gui.add(params, 'showNebula').name('Nebula').onChange(v=> nebulaGroup.visible=v);
      gui.add(params, 'showBlackHole').name('Black Hole').onChange(v=> blackHoleGroup.visible=v);
      gui.add(params, 'timeScale', 0, 3, 0.01).name('Time Scale');

      window.addEventListener('resize', onWindowResize);
      window.addEventListener('keydown', onKeyDown);
      window.addEventListener('mousemove', onMouseMove);
    }

    function createStarField(){
      const geometry = new THREE.BufferGeometry();
      const n = params.starCount;
      const positions = new Float32Array(n * 3);
      const sizes = new Float32Array(n);
      for(let i=0;i<n;i++){
        const r = 800 + Math.pow(Math.random(), 3) * 6000;
        const theta = Math.random() * Math.PI * 2;
        const phi = Math.acos(2*Math.random()-1);
        positions[i*3] = r * Math.sin(phi) * Math.cos(theta);
        positions[i*3+1] = r * Math.sin(phi) * Math.sin(theta);
        positions[i*3+2] = r * Math.cos(phi);
        sizes[i] = 0.5 + Math.random()*2.5;
      }
      geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
      geometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));
      const material = new THREE.PointsMaterial({color:0xffffff, size:1.6, transparent:true, opacity:0.9, depthWrite:false, blending:THREE.AdditiveBlending});
      starField = new THREE.Points(geometry, material);
      starField.name = 'starfield';
      scene.add(starField);
    }

    function createGalaxy(){
      groupGalaxy = new THREE.Group();
      groupGalaxy.name = 'galaxy';
      const particles = params.galaxyParticles;
      const geometry = new THREE.BufferGeometry();
      const positions = new Float32Array(particles*3);
      const colors = new Float32Array(particles*3);
      const colorInside = new THREE.Color(0xfff1c9);
      const colorOutside = new THREE.Color(0x8080ff);
      for(let i=0;i<particles;i++){
        const arm = i % params.galaxyArms;
        const armAngle = (arm / params.galaxyArms) * Math.PI * 2;
        const radius = Math.pow(Math.random(), 1.2) * 600;
        const spin = radius * 0.4;
        const angle = armAngle + spin + (Math.random()-0.5)*0.5;
        const distanceJitter = Math.random() * 60;
        positions[i*3] = Math.cos(angle)*(radius+distanceJitter);
        positions[i*3+1] = (Math.random()-0.5)*20*(1-radius/600);
        positions[i*3+2] = Math.sin(angle)*(radius+distanceJitter);
        const t = radius/600;
        const c = colorInside.clone().lerp(colorOutside, t);
        colors[i*3] = c.r; colors[i*3+1] = c.g; colors[i*3+2] = c.b;
      }
      geometry.setAttribute('position', new THREE.BufferAttribute(positions,3));
      geometry.setAttribute('color', new THREE.BufferAttribute(colors,3));
      const material = new THREE.PointsMaterial({size:2.2, vertexColors:true, transparent:true, opacity:0.95, blending:THREE.AdditiveBlending, depthWrite:false});
      const points = new THREE.Points(geometry, material);
      groupGalaxy.add(points);
      const bulgeGeom = new THREE.SphereGeometry(40,32,16);
      const bulgeMat = new THREE.MeshBasicMaterial({color:0xfff4e0, transparent:true, opacity:0.9});
      const bulge = new THREE.Mesh(bulgeGeom, bulgeMat);
      groupGalaxy.add(bulge);
      scene.add(groupGalaxy);
    }

    function createPlanets(){
      groupPlanets = new THREE.Group();
      groupPlanets.name='planets';
      const sunGeo = new THREE.SphereGeometry(28, 32, 16);
      const sunMat = new THREE.MeshBasicMaterial({color:0xffee88});
      const sun = new THREE.Mesh(sunGeo, sunMat);
      sun.name='Sun';
      groupPlanets.add(sun);

      const planetSpecs = [
        {name:'Mercury', r: 4.8, dist: 50, speed: 0.047, colorA:'#bfbfbf', colorB:'#999999'},
        {name:'Venus', r: 12, dist: 80, speed: 0.035, colorA:'#e6d1a3', colorB:'#c9b186'},
        {name:'Earth', r: 12.7, dist: 110, speed: 0.03, colorA:'#5dd39e', colorB:'#235d4a'},
        {name:'Mars', r: 6.8, dist: 150, speed: 0.024, colorA:'#d66849', colorB:'#8b3a2e'},
        {name:'Jupiter', r: 70, dist: 220, speed: 0.013, colorA:'#d6b48a', colorB:'#8b4b2e'},
        {name:'Saturn', r: 58, dist: 300, speed: 0.009, colorA:'#f0e0b0', colorB:'#c8b080'},
        {name:'Uranus', r: 25, dist: 380, speed: 0.006, colorA:'#7fc8f8', colorB:'#4586b3'},
        {name:'Neptune', r: 24, dist: 450, speed: 0.005, colorA:'#4f74ff', colorB:'#2c3f99'}
      ];

      planetSpecs.forEach(spec=>{
        const tex = createPlanetTexture(spec.colorA,spec.colorB,spec.r*10+40);
        const mat = new THREE.MeshStandardMaterial({map:tex, roughness:1, metalness:0});
        const geo = new THREE.SphereGeometry(spec.r,32,24);
        const mesh = new THREE.Mesh(geo, mat);
        mesh.userData = {dist: spec.dist, speed: spec.speed, angle: Math.random()*Math.PI*2, name: spec.name};
        mesh.position.set(spec.dist,0,0);
        mesh.name = spec.name;
        groupPlanets.add(mesh);

        // orbit
        const points = [];
        const segments = 128;
        for(let s=0;s<=segments;s++){
          const a = (s/segments)*Math.PI*2;
          points.push(new THREE.Vector3(Math.cos(a)*spec.dist,0,Math.sin(a)*spec.dist));
        }
        const orbitGeom = new THREE.BufferGeometry().setFromPoints(points);
        const orbitMat = new THREE.LineBasicMaterial({color:0x444466, transparent:true, opacity:0.4});
        const orbitLine = new THREE.LineLoop(orbitGeom, orbitMat);
        groupPlanets.add(orbitLine);
      });

      scene.add(groupPlanets);
    }

    function createPlanetTexture(colorA, colorB, size){
      const cvs = document.createElement('canvas');
      cvs.width=cvs.height=size;
      const ctx = cvs.getContext('2d');
      const g = ctx.createLinearGradient(0,0,size,size);
      g.addColorStop(0,colorA); g.addColorStop(1,colorB);
      ctx.fillStyle = g; ctx.fillRect(0,0,size,size);
      for(let i=0;i<1200;i++){
        ctx.globalAlpha = 0.02+Math.random()*0.06;
        ctx.fillStyle = (Math.random()>0.5)?'rgba(255,255,255,0.6)':'rgba(0,0,0,0.07)';
        const x=Math.random()*size,y=Math.random()*size;
        ctx.beginPath();ctx.arc(x,y,Math.random()*6,0,Math.PI*2);ctx.fill();
      }
      const tex = new THREE.CanvasTexture(cvs); tex.needsUpdate=true; return tex;
    }

    function createNebula(){
      nebulaGroup = new THREE.Group();
      nebulaGroup.name='nebula';
      function makeSprite(size,color,opacity){
        const cvs=document.createElement('canvas'); cvs.width=cvs.height=size;
        const ctx=cvs.getContext('2d');
        const grd=ctx.createRadialGradient(size/2,size/2,0,size/2,size/2,size/2);
        grd.addColorStop(0,color); grd.addColorStop(1,'rgba(0,0,0,0)');
        ctx.fillStyle=grd; ctx.fillRect(0,0,size,size);
        const tex=new THREE.CanvasTexture(cvs);
        const mat=new THREE.SpriteMaterial({map:tex,transparent:true,opacity:opacity,blending:THREE.AdditiveBlending,depthWrite:false});
        return new THREE.Sprite(mat);
      }
      const colors=['rgba(160,80,255,0.9)','rgba(255,120,110,0.9)','rgba(80,200,255,0.9)'];
      for(let i=0;i<70;i++){
        const s=400+Math.random()*1400;
        const c=colors[i%colors.length];
        const sprite=makeSprite(s,c,0.06+Math.random()*0.18);
        sprite.position.set((Math.random()-0.5)*3000,(Math.random()-0.5)*600,(Math.random()-0.5)*3000);
        sprite.scale.set(s,s,1);
        nebulaGroup.add(sprite);
      }
      scene.add(nebulaGroup);
    }

    function createBlackHole(){
      blackHoleGroup = new THREE.Group();
      blackHoleGroup.name='blackhole';
      const bh = new THREE.Mesh(new THREE.SphereGeometry(22,32,16),new THREE.MeshBasicMaterial({color:0x000000}));
      blackHoleGroup.add(bh);
      const size=1024; const cvs=document.createElement('canvas'); cvs.width=cvs.height=size; const ctx=cvs.getContext('2d');
      const g=ctx.createRadialGradient(size/2,size/2,0,size/2,size/2,size/2);
      g.addColorStop(0.0,'rgba(255,240,200,0.0)'); g.addColorStop(0.5,'rgba(255,200,80,0.8)'); g.addColorStop(0.85,'rgba(120,40,10,0.6)'); g.addColorStop(1.0,'rgba(0,0,0,0)');
      ctx.fillStyle=g; ctx.fillRect(0,0,size,size); const discTex=new THREE.CanvasTexture(cvs); discTex.needsUpdate=true;
      const discGeo=new THREE.RingGeometry(28,120,128);
      const discMat=new THREE.MeshBasicMaterial({map:discTex,transparent:true,blending:THREE.AdditiveBlending,side:THREE.DoubleSide});
      const disc=new THREE.Mesh(discGeo,discMat); disc.rotateX(Math.PI/2); disc.position.set(-1200,0,-600);
      blackHoleGroup.add(disc);
      const haloGeo=new THREE.SphereGeometry(160,32,16);
      const haloMat=new THREE.MeshBasicMaterial({color:0x224466,transparent:true,opacity:0.08,blending:THREE.AdditiveBlending});
      const halo=new THREE.Mesh(haloGeo,haloMat); halo.position.copy(disc.position);
      blackHoleGroup.add(halo);
      const jetGeom=new THREE.BufferGeometry(); const n=800; const pos=new Float32Array(n*3);
      for(let i=0;i<n;i++){ pos[i*3]=disc.position.x+(Math.random()-0.5)*40; pos[i*3+1]=disc.position.y+(Math.random()-0.5)*200; pos[i*3+2]=disc.position.z+(Math.random()-0.5)*40;}
      jetGeom.setAttribute('position', new THREE.BufferAttribute(pos,3));
      const jetMat=new THREE.PointsMaterial({size:3,color:0xffcc88,transparent:true,opacity:0.9,blending:THREE.AdditiveBlending});
      const jets=new THREE.Points(jetGeom,jetMat); blackHoleGroup.add(jets);
      scene.add(blackHoleGroup);
    }

    function animate(){
      requestAnimationFrame(animate);
      const dt = clock.getDelta()*params.timeScale;
      if(groupGalaxy) groupGalaxy.rotation.y += 0.002*dt*60;
      if(nebulaGroup) nebulaGroup.children.forEach((c,i)=>{c.rotation.z+=0.0002*(i%5+1)*dt*60;});
      if(groupPlanets){
        groupPlanets.children.forEach(obj=>{
          if(obj.userData && obj.userData.dist){
            obj.userData.angle += obj.userData.speed*0.01*dt*60;
            const a = obj.userData.angle;
            obj.position.x = Math.cos(a)*obj.userData.dist;
            obj.position.z = Math.sin(a)*obj.userData.dist;
          }
        });
      }

      updateTooltip();
      controls.update();
      composer.render();
    }

    function onWindowResize(){
      width=window.innerWidth; height=window.innerHeight;
      camera.aspect=width/height; camera.updateProjectionMatrix();
      renderer.setSize(width,height); composer.setSize(width,height);
    }

    function onKeyDown(e){
      switch(e.key){
        case '1': params.showStarfield = !params.showStarfield; starField.visible=params.showStarfield; break;
        case '2': params.showGalaxy = !params.showGalaxy; groupGalaxy.visible=params.showGalaxy; break;
        case '3': params.showPlanets = !params.showPlanets; groupPlanets.visible=params.showPlanets; break;
        case '4': params.showNebula = !params.showNebula; nebulaGroup.visible=params.showNebula; break;
        case '5': params.showBlackHole = !params.showBlackHole; blackHoleGroup.visible=params.showBlackHole; break;
      }
    }

    function onMouseMove(e){
      mouse.x = (e.clientX / window.innerWidth)*2 - 1;
      mouse.y = -(e.clientY / window.innerHeight)*2 + 1;
    }

    function updateTooltip(){
      raycaster.setFromCamera(mouse, camera);
      const intersects = raycaster.intersectObjects(groupPlanets.children, true);
      if(intersects.length>0){
        const obj = intersects[0].object;
        if(obj.name){
          tooltip.style.display='block';
          tooltip.style.opacity = '1';
          tooltip.style.left = ((mouse.x + 1)/2 * window.innerWidth + 10) + 'px';
          tooltip.style.top = ((-mouse.y + 1)/2 * window.innerHeight + 10) + 'px';
          tooltip.textContent = obj.name;
          return;
        }
      }
      tooltip.style.opacity = '0';
      setTimeout(()=>{tooltip.style.display='none';},150);
    }
  </script>
</body>
</html>
