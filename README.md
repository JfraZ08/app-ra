## Créer une page HTML

- WebXR nécessite une interaction de l'utilisateur pour pouvoir démarrer une session. Créez un bouton qui appelle activateXR(). Lors du chargement de la page, l'utilisateur peut utiliser ce bouton pour lancer l'expérience de RA.

- Créez un fichier nommé index.html et ajoutez-y le code HTML suivant:

```html
<!doctype html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport"
        content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
  <title>Hello WebXR!</title>

  <!-- three.js -->
  <script src="https://unpkg.com/three@0.126.0/build/three.js"></script>
</head>
<body>

<!-- Starting an immersive WebXR session requires user interaction.
    We start this one with a simple button. -->
<button onclick="activateXR()">Start Hello WebXR</button>
<script>
async function activateXR() {
  // Add a canvas element and initialize a WebGL context that is compatible with WebXR.
  const canvas = document.createElement("canvas");
  document.body.appendChild(canvas);
  const gl = canvas.getContext("webgl", {xrCompatible: true});

  // To be continued in upcoming steps.
}
</script>
</body>
</html>
```
## Initialiser three.js

- Il ne se passe pas grand-chose lorsque vous appuyez sur le bouton "Démarrer". Pour configurer un environnement 3D, vous pouvez utiliser une bibliothèque de rendu pour afficher une scène.

- Dans cet exemple, vous allez utiliser three.js, une bibliothèque de rendu 3D JavaScript qui fournit un moteur de rendu WebGL. Three.js gère le rendu, les caméras et les graphiques de scène, ce qui facilite l'affichage de contenu 3D sur le Web.
Remarque :Bien qu'il soit possible d'utiliser une autre bibliothèque de rendu WebGL, three.js facilite la réalisation de ce tutoriel WebXR.
Créer une scène

- Un environnement 3D est généralement modélisé sous la forme d'une scène. Créez un THREE.Scene contenant des éléments de RA. Le code suivant vous permet d'examiner un cadre coloré non éclairé en RA.

- Ajoutez le code suivant en bas de la fonction activateXR():
```js
const scene = new THREE.Scene();

// The cube will have a different color on each side.
const materials = [
  new THREE.MeshBasicMaterial({color: 0xff0000}),
  new THREE.MeshBasicMaterial({color: 0x0000ff}),
  new THREE.MeshBasicMaterial({color: 0x00ff00}),
  new THREE.MeshBasicMaterial({color: 0xff00ff}),
  new THREE.MeshBasicMaterial({color: 0x00ffff}),
  new THREE.MeshBasicMaterial({color: 0xffff00})
];

// Create the cube and add it to the demo scene.
const cube = new THREE.Mesh(new THREE.BoxBufferGeometry(0.2, 0.2, 0.2), materials);
cube.position.set(1, 1, 1);
scene.add(cube);
```
## Configurer le rendu à l'aide de three.js

- Pour voir cette scène en RA, vous devez disposer d'un moteur de rendu et d'une caméra. Le moteur de rendu utilise WebGL pour dessiner votre scène à l'écran. La caméra décrit la fenêtre d'affichage à partir de laquelle la scène est vue.

- Ajoutez le code suivant en bas de la fonction activateXR():
```js
// Set up the WebGLRenderer, which handles rendering to the session's base layer.
const renderer = new THREE.WebGLRenderer({
  alpha: true,
  preserveDrawingBuffer: true,
  canvas: canvas,
  context: gl
});
renderer.autoClear = false;

// The API directly updates the camera matrices.
// Disable matrix auto updates so three.js doesn't attempt
// to handle the matrices independently.
const camera = new THREE.PerspectiveCamera();
camera.matrixAutoUpdate = false;
```
## Créer une XRSession

- Le point d'entrée vers WebXR passe par XRSystem.requestSession(). Utilisez le mode immersive-ar pour pouvoir visualiser le rendu dans un environnement réel.

- Un XRReferenceSpace décrit le système de coordonnées utilisé pour les objets du monde virtuel. Le mode 'local' convient mieux à une expérience de RA, car il offre un suivi stable et un espace de référence initial proche du spectateur.

- Pour créer XRSession et XRReferenceSpace, ajoutez le code suivant en bas de la fonction activateXR():
```js
// Initialize a WebXR session using "immersive-ar".
const session = await navigator.xr.requestSession("immersive-ar");
session.updateRenderState({
  baseLayer: new XRWebGLLayer(session, gl)
});

// A 'local' reference space has a native origin that is located
// near the viewer's position at the time the session was created.
const referenceSpace = await session.requestReferenceSpace('local');
```
## Rendre la scène

Vous pouvez maintenant effectuer le rendu de la scène. 
```js
XRSession.requestAnimationFrame() 
```
planifie un rappel qui s'exécute lorsque le navigateur est prêt à dessiner un frame.
Avertissement :Bien que les méthodes Window portent le même nom, elles ne sont pas interchangeables. Utilisez 
```js
XRSession.requestAnimationFrame() 
``` 
pendant les sessions XR.

Lors du rappel de l'image de l'animation, appelez 
```js
XRFrame.getViewerPose(); 
```
 pour obtenir la pose de l'utilisateur par rapport à l'espace de coordonnées local. Cela permet de mettre à jour la caméra de la scène, en modifiant la façon dont l'utilisateur voit le monde virtuel avant que le moteur de rendu ne dessine la scène à l'aide de la caméra mise à jour.

Ajoutez le code suivant en bas de la fonction activateXR():
```js
// Create a render loop that allows us to draw on the AR view.
const onXRFrame = (time, frame) => {
  // Queue up the next draw request.
  session.requestAnimationFrame(onXRFrame);

  // Bind the graphics framebuffer to the baseLayer's framebuffer
  gl.bindFramebuffer(gl.FRAMEBUFFER, session.renderState.baseLayer.framebuffer)

  // Retrieve the pose of the device.
  // XRFrame.getViewerPose can return null while the session attempts to establish tracking.
  const pose = frame.getViewerPose(referenceSpace);
  if (pose) {
    // In mobile AR, we only have one view.
    const view = pose.views[0];

    const viewport = session.renderState.baseLayer.getViewport(view);
    renderer.setSize(viewport.width, viewport.height)

    // Use the view's transform matrix and projection matrix to configure the THREE.camera.
    camera.matrix.fromArray(view.transform.matrix)
    camera.projectionMatrix.fromArray(view.projectionMatrix);
    camera.updateMatrixWorld(true);

    // Render the scene with THREE.WebGLRenderer.
    renderer.render(scene, camera)
  }
}
session.requestAnimationFrame(onXRFrame);
```
Exécuter Hello WebXR
Accédez au fichier WebXR sur votre appareil.