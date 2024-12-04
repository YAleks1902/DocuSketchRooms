<template>
  <div>
    <canvas ref="canvas" width="800" height="600"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import * as THREE from 'three'
import simple from '../data/simple 1.json'
import t_shape from '../data/t_shape 1.json'
import triangle from '../data/triangle 1.json'

const canvas = ref(null)
const roomTypes = ['simple', 't_shape', 'triangle']
const roomType = ref('simple')
const currentWallIndex = ref(0)
const tolerance = 0.001

let scene, camera, renderer

const randomizeRoom = () => {
  roomType.value = roomTypes[Math.floor(Math.random() * roomTypes.length)]
  currentWallIndex.value = 0
  drawRoom()
}

const changeRoom = () => {
  let newRoomType
  do {
    newRoomType = roomTypes[Math.floor(Math.random() * roomTypes.length)]
  } while (newRoomType === roomType.value)
  roomType.value = newRoomType
  currentWallIndex.value = 0
  drawRoom()
}

const nextWall = () => {
  const corners = getRoomData()
  if (corners.length > 0) {
    currentWallIndex.value = (currentWallIndex.value + 1) % corners.length
    drawRoom()
  }
}

const getRoomData = () => {
  switch (roomType.value) {
    case 'simple':
      return simple.corners
    case 't_shape':
      return getOrderedCorners(t_shape)
    case 'triangle':
      return triangle.corners
    default:
      return []
  }
}

const getOrderedCorners = (roomData) => {
  if (roomType.value === 't_shape') {
    const ids = [
      'b4a4c5f6-59c2-15c0-d7b5-ba31e9912d76',
      '52b045fe-9753-81ca-984c-ac18d999ea7d',
      'e3875084-5db2-2c5f-10dc-70fd5fe082bf',
      '7bb551a1-3563-a01e-f0d3-dbee123189ca',
      '4f429b40-74ef-809b-5fb2-a102139f0aec',
      '0ec8a82d-67ed-3875-2722-7295090af9cf',
      '47e4aeaf-8c54-5303-acbb-c434a7fe7987',
      'df0bbbd1-fa58-6a0a-7c12-2a9f4d9658fa',
    ]
    return ids.map((id) => roomData.corners.find((c) => c.id === id)).filter(Boolean)
  }
  return roomData.corners
} // Order of the corners was causing bad rendering had to hardcode the order

const createShapeFromCorners = (corners) => {
  const shape = new THREE.Shape()
  corners.forEach((corner, index) => {
    const point = new THREE.Vector2(corner.x, corner.y)
    if (index === 0) {
      shape.moveTo(point.x, point.y)
    } else {
      shape.lineTo(point.x, point.y)
    }
  })
  shape.closePath()
  return shape
}

const clearScene = () => {
  while (scene.children.length > 0) {
    const child = scene.children[0]
    scene.remove(child)
    if (child.geometry) child.geometry.dispose()
    if (child.material) child.material.dispose()
  }
}

const drawRoom = () => {
  if (!scene || !camera || !renderer) {
    console.error('Three.js elements not initialized')
    return
  }

  clearScene()

  const corners = getRoomData()
  if (corners.length === 0) return

  const roomShape = createShapeFromCorners(corners)
  const geometry = new THREE.ShapeGeometry(roomShape)
  const fillMaterial = new THREE.MeshBasicMaterial({ color: 0xffffff, side: THREE.DoubleSide })
  const mesh = new THREE.Mesh(geometry, fillMaterial)

  geometry.computeBoundingBox()
  const bbox = geometry.boundingBox
  const centerX = (bbox.max.x + bbox.min.x) / 2
  const centerY = (bbox.max.y + bbox.min.y) / 2
  mesh.position.set(-centerX, -centerY, 0)

  scene.add(mesh)

  const outlineMaterial = new THREE.LineBasicMaterial({ color: 0x000000 })
  const edges = new THREE.EdgesGeometry(geometry)
  const outline = new THREE.LineSegments(edges, outlineMaterial)
  outline.position.copy(mesh.position)
  scene.add(outline)

  const blueLineEndpoints = drawWidthLine(roomShape, mesh.position)

  if (blueLineEndpoints) {
    drawPerpendicularRedLine(blueLineEndpoints)
  }

  renderer.render(scene, camera)
}

const drawWidthLine = (shape, offset) => {
  const points = shape.getPoints()
  if (points.length < 2) return null

  const wallStart = points[currentWallIndex.value]
  const wallEnd = points[(currentWallIndex.value + 1) % points.length]

  const midX = (wallStart.x + wallEnd.x) / 2 + offset.x
  const midY = (wallStart.y + wallEnd.y) / 2 + offset.y

  const perpVector = new THREE.Vector2(wallEnd.y - wallStart.y, wallStart.x - wallEnd.x).normalize()

  let closestIntersection = null
  let minDistance = Infinity

  points.forEach((start, i) => {
    const end = points[(i + 1) % points.length]
    const intersection = getLineIntersection(
      midX,
      midY,
      midX + perpVector.x * 1000,
      midY + perpVector.y * 1000,
      start.x + offset.x,
      start.y + offset.y,
      end.x + offset.x,
      end.y + offset.y,
    )
    if (intersection) {
      const distance = Math.hypot(intersection.x - midX, intersection.y - midY)
      if (distance < minDistance && distance > tolerance) {
        minDistance = distance
        closestIntersection = intersection
      }
    }
  })

  if (closestIntersection) {
    drawLine({ x: midX, y: midY }, closestIntersection, 0x0000ff)
    return { start: { x: midX, y: midY }, end: closestIntersection }
  } else {
    console.warn('No intersection found for blue line, using fallback')
    const fallbackEnd = { x: midX + perpVector.x * 300, y: midY + perpVector.y * 300 }
    drawLine({ x: midX, y: midY }, fallbackEnd, 0x0000ff)
    return { start: { x: midX, y: midY }, end: fallbackEnd }
  }
}

const drawPerpendicularRedLine = (blueLineEndpoints) => {
  const { start, end } = blueLineEndpoints
  const midX = (start.x + end.x) / 2
  const midY = (start.y + end.y) / 2

  const perpVector = new THREE.Vector2(start.y - end.y, end.x - start.x).normalize()
  const extendedLength = 5000

  drawLine(
    { x: midX - perpVector.x * extendedLength, y: midY - perpVector.y * extendedLength },
    { x: midX + perpVector.x * extendedLength, y: midY + perpVector.y * extendedLength },
    0xff0000,
  )
}

const getLineIntersection = (x1, y1, x2, y2, x3, y3, x4, y4) => {
  const denom = (x1 - x2) * (y3 - y4) - (y1 - y2) * (x3 - x4)
  if (Math.abs(denom) < tolerance) return null

  const intersectX = ((x1 * y2 - y1 * x2) * (x3 - x4) - (x1 - x2) * (x3 * y4 - y3 * x4)) / denom
  const intersectY = ((x1 * y2 - y1 * x2) * (y3 - y4) - (y1 - y2) * (x3 * y4 - y3 * x4)) / denom

  if (
    intersectX >= Math.min(x3, x4) - tolerance &&
    intersectX <= Math.max(x3, x4) + tolerance &&
    intersectY >= Math.min(y3, y4) - tolerance &&
    intersectY <= Math.max(y3, y4) + tolerance
  ) {
    return { x: intersectX, y: intersectY }
  }

  return null
}
//now sure about correct implementation of drawWidthLine,drawPerpendicularRedLine and getLineIntersection
// might be easeir way to cover this logic

const drawLine = (start, end, color) => {
  const material = new THREE.LineBasicMaterial({ color })
  const points = [new THREE.Vector3(start.x, start.y, 0), new THREE.Vector3(end.x, end.y, 0)]
  const geometry = new THREE.BufferGeometry().setFromPoints(points)
  scene.add(new THREE.Line(geometry, material))
}

const initThreeJS = () => {
  const canvasElement = canvas.value
  if (!canvasElement) {
    console.error('Canvas element not found')
    return
  }

  const width = canvasElement.width
  const height = canvasElement.height

  scene = new THREE.Scene()
  camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000)
  camera.position.set(0, 0, 500)

  renderer = new THREE.WebGLRenderer({ canvas: canvasElement, alpha: true })
  renderer.setSize(width, height)

  scene.add(new THREE.AmbientLight(0xffffff, 0.8))
}

onMounted(() => {
  initThreeJS()
  randomizeRoom()
})

defineExpose({
  randomizeRoom,
  changeRoom,
  nextWall,
})
</script>

<style scoped>
canvas {
  border: 1px solid black;
  margin-bottom: 20px;
}
</style>
