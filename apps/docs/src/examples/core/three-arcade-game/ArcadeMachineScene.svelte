<script lang="ts">
	import { InteractiveObject, Three, useFrame, useTexture, useThrelte } from '@threlte/core'
	import { Environment, useCursor, useGltf } from '@threlte/extras'
	import { onMount } from 'svelte'
	import { cubicInOut } from 'svelte/easing'
	import { spring, tweened } from 'svelte/motion'
	import { derived } from 'svelte/store'
	import {
		AmbientLight,
		Color,
		CylinderGeometry,
		DirectionalLight,
		Group,
		Mesh,
		MeshStandardMaterial,
		Object3D,
		PerspectiveCamera,
		PointLight,
		Scene
	} from 'three'
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
	import { DEG2RAD } from 'three/src/math/MathUtils'
	import { gameState } from './game/state'

	const { gltf } = useGltf<{
		nodes: {
			BodyMesh: THREE.Mesh
			LeftCover: THREE.Mesh
			RightCover: THREE.Mesh
			ScreenFrame: THREE.Mesh
			joystick_base: THREE.Mesh
			joystick_stick_application: THREE.Mesh
			joystick_stick: THREE.Mesh
			joystick_cap: THREE.Mesh
			Main_Button_Enclosure: THREE.Mesh
			Main_Button: THREE.Mesh
			Screen: THREE.Mesh
		}
		materials: {
			['machine body main']: THREE.MeshStandardMaterial
			['machine body outer']: THREE.MeshStandardMaterial
			['screen frame']: THREE.MeshStandardMaterial
			['joystick base']: THREE.MeshStandardMaterial
			['joystick stick']: THREE.MeshStandardMaterial
			['joystick cap']: THREE.MeshStandardMaterial
			Screen: THREE.MeshStandardMaterial
		}
	}>('/models/ball-game/archade-machine/arcade_machine_own.glb')

	$: nodes = $gltf?.nodes
	$: materials = $gltf?.materials

	$: if ($gltf) {
		Object.entries($gltf.materials).forEach(([name, material]) => {
			if (!$gltf) return
			const n = name as keyof typeof $gltf.materials
			if (n === 'joystick cap') material.envMapIntensity = 1
			else if (n === 'joystick stick') material.envMapIntensity = 1
			else material.envMapIntensity = 0.2
		})
	}

	let basePointLightIntensity = tweened(0)

	onMount(() => {
		setTimeout(() => {
			basePointLightIntensity.set(1, {
				duration: 200
			})
		}, 1000)
	})

	const { state, gameTexture, averageScreenColor, arcadeMachineScene, orbitControls } = gameState

	let pointlight: PointLight
	let pointLightIntensity = $basePointLightIntensity
	useFrame(() => {
		if ($state === 'off') {
			pointLightIntensity = 1
		} else {
			const randomSign = Math.random() > 0.5 ? 1 : -1
			const random = -0.01 + Math.random() * 0.02 * randomSign
			pointLightIntensity = $basePointLightIntensity
		}
	})

	const scanLinesTexture = useTexture('/models/ball-game/archade-machine/textures/scanlines.png')

	let leftPressed = false
	let rightPressed = false
	let spacePressed = false

	const tipsOpacity = tweened(1, {
		duration: 200
	})

	const onKeyUp = (e: KeyboardEvent) => {
		if (e.key === 'ArrowLeft') {
			e.preventDefault()
			leftPressed = false
		} else if (e.key === 'ArrowRight') {
			e.preventDefault()
			rightPressed = false
		} else if (e.key === ' ') {
			spacePressed = false
		}
	}

	const onKeyDown = (e: KeyboardEvent) => {
		if (e.key !== 'ArrowLeft' && e.key !== 'ArrowRight' && e.key !== ' ') return
		tipsOpacity.set(0)
		if (e.key === 'ArrowLeft') {
			e.preventDefault()
			leftPressed = true
		} else if (e.key === 'ArrowRight') {
			e.preventDefault()
			rightPressed = true
		} else if (e.key === ' ') {
			spacePressed = true
		}
	}

	const rotationStick = tweened(0, {
		duration: 100
	})
	$: if (leftPressed !== rightPressed) {
		if (leftPressed) {
			rotationStick.set(-15 * DEG2RAD)
		} else {
			rotationStick.set(15 * DEG2RAD)
		}
	}
	$: if (leftPressed === rightPressed) {
		rotationStick.set(0)
	}

	const machineIsOff = derived(state, (state, set) => {
		if (state === 'off') {
			set(true)
		} else {
			set(false)
		}
	})

	const cameraTweenZ = tweened(2.1, {
		duration: 3e3,
		easing: cubicInOut
	})
	$: cameraTweenZ.set($machineIsOff ? 2.1 : 1.4)

	const { pointer } = useThrelte()

	let screenFocused = false

	const screenPos = {
		x: 0,
		y: 1.3774,
		z: 0.1447
	}

	const cameraTargetPos = spring(
		{
			x: $pointer.x * 0.1,
			y: 1.23,
			z: 0
		},
		{
			precision: 0.000001
		}
	)
	$: cameraTargetPos.set(
		screenFocused
			? {
					...screenPos,
					z: -screenPos.z
			  }
			: {
					x: $pointer.x * 0.1,
					y: 1.23,
					z: 0
			  }
	)

	const cameraPos = spring(
		{
			x: $pointer.x * ($state === 'off' ? 2 : 0.1),
			y: 1.48,
			z: $cameraTweenZ
		},
		{
			stiffness: 0.05,
			damping: 0.9,
			precision: 0.00001
		}
	)
	$: cameraPos.set(
		screenFocused
			? {
					x: screenPos.x,
					y: screenPos.y + 0.15,
					z: screenPos.z + 0.5
			  }
			: {
					x: $pointer.x * ($state === 'off' ? 0.1 : 0.1),
					y: 1.48,
					z: $cameraTweenZ
			  }
	)

	let cameraTarget: Object3D | undefined = undefined
	let camera: PerspectiveCamera | undefined = undefined
	useFrame(() => {
		if (!camera || !cameraTarget) return
		camera.lookAt(cameraTarget.position)
	})

	const { onPointerEnter, onPointerLeave } = useCursor('pointer')

	const onScreenClick = () => {
		screenFocused = !screenFocused
	}

	const backgroundColor = tweened(new Color('#020203'), {
		duration: 2.5e3
	})
	$: backgroundColor.set($machineIsOff ? new Color('#020203') : new Color('#020203'))

	const { scene, renderer } = useThrelte()
	if (renderer) renderer.physicallyCorrectLights = true
	$: scene.background = new Color($backgroundColor)

	const blueLightIntensity = tweened(2, {
		duration: 3e3
	})
	$: blueLightIntensity.set($machineIsOff ? 2 : 2)

	const redLightIntensity = tweened(1, {
		duration: 3e3
	})
	$: redLightIntensity.set($machineIsOff ? 2 : 2)

	const whiteLightIntensity = tweened(0, {
		duration: 3e3
	})
	$: whiteLightIntensity.set($machineIsOff ? 0 : 0)

	const whiteAmbientLightIntensity = tweened(1, {
		duration: 3e3
	})
	$: whiteAmbientLightIntensity.set($machineIsOff ? 1 : 0)
</script>

<svelte:window on:keydown={onKeyDown} on:keyup={onKeyUp} />

<Three type={Scene} bind:ref={$arcadeMachineScene} background={new Color(0x020203)}>
	<Environment path="/hdr/" files="shanghai_riverside_1k.hdr" />

	<Three
		position.x={$cameraTargetPos.x}
		position.y={$cameraTargetPos.y}
		position.z={$cameraTargetPos.z}
		bind:ref={cameraTarget}
		type={Object3D}
		let:ref
	/>

	{#if $orbitControls}
		<Three
			type={PerspectiveCamera}
			position.x={20}
			position.y={20}
			position.z={20}
			fov={60}
			makeDefault
			let:ref={camera}
		>
			<Three type={OrbitControls} args={[camera, renderer?.domElement]} />
		</Three>
	{:else}
		<Three
			type={PerspectiveCamera}
			position.x={$cameraPos.x}
			position.y={$cameraPos.y}
			position.z={$cameraPos.z}
			fov={30}
			makeDefault
			bind:ref={camera}
			let:ref={camera}
		/>
	{/if}

	{#if nodes && materials}
		<!-- Generated by gltfjsx -->
		<Three type={Group} rotation.y={DEG2RAD * 180}>
			<Three
				type={Mesh}
				geometry={nodes.BodyMesh.geometry}
				material={materials['machine body main']}
				position={[0.2755, 0, 0]}
			/>
			<Three
				type={Mesh}
				geometry={nodes.LeftCover.geometry}
				material={materials['machine body outer']}
				position={[0.3, 1.2099, -0.1307]}
			/>
			<Three
				type={Mesh}
				geometry={nodes.RightCover.geometry}
				material={materials['machine body outer']}
				position={[-0.3, 1.2099, -0.1307]}
				scale={[-1, 1, 1]}
			/>
			<Three
				type={Mesh}
				geometry={nodes.ScreenFrame.geometry}
				material={materials['screen frame']}
				position={[0.2755, 0.0633, 0.0346]}
			/>

			<!-- Joystick -->

			<Three
				type={Mesh}
				geometry={nodes.joystick_base.geometry}
				material={materials['joystick base']}
				position={[0.1336, 0.9611, -0.1976]}
				rotation={[-0.1939, 0, 0]}
			/>
			<Three
				type={Mesh}
				geometry={nodes.joystick_stick_application.geometry}
				material={materials['joystick base']}
				position={[0.1336, 0.9653, -0.1984]}
				rotation={[-0.1939, 0, $rotationStick]}
			>
				<Three
					type={Mesh}
					geometry={nodes.joystick_stick.geometry}
					material={materials['joystick stick']}
					position={[0, -0.0145, 0.0001]}
				>
					<Three
						type={Mesh}
						geometry={nodes.joystick_cap.geometry}
						material={materials['joystick cap']}
						position={[-0.0001, 0.1126, -0.0005]}
						material.envMapIntensity={0.5}
					/>
				</Three>
			</Three>

			<Three
				type={Mesh}
				geometry={nodes.Main_Button_Enclosure.geometry}
				material={materials['joystick base']}
				position={[-0.1143, 0.9795, -0.0933]}
				rotation={[-0.1801, 0, 0]}
				scale={0.9409}
			>
				<Three
					type={Mesh}
					geometry={nodes.Main_Button.geometry}
					material={materials['joystick cap']}
					position={[0.0001, 0.007 + (spacePressed ? -0.003 : 0), -0.0003]}
					rotation={[0.192, 0, 0]}
					scale={0.724}
				/>
			</Three>

			<!-- The screen itself gets a special treatment -->
			<Three
				type={Mesh}
				geometry={nodes.Screen.geometry}
				position={[0, 1.3774, 0.1447]}
				scale={1.0055}
				let:ref={screen}
			>
				<InteractiveObject
					object={screen}
					on:pointerenter={onPointerEnter}
					on:pointerleave={onPointerLeave}
					interactive
					on:click={onScreenClick}
				/>
				<Three
					type={MeshStandardMaterial}
					metalness={0.9}
					roughness={0.2}
					color={'#141414'}
					map={scanLinesTexture}
					metalnessMap={scanLinesTexture}
					emissiveMap={$gameTexture}
					emissive={pointLightIntensity}
					emissiveIntensity={1.2}
					envMapIntensity={0}
				/>
			</Three>
		</Three>
	{/if}

	<!-- This PointLight replicates the light emitted by the screen -->
	{#if nodes}
		<Three
			type={PointLight}
			args={['black']}
			position.y={1.376583185239323}
			position.z={-0.12185962320246482}
			intensity={25 * pointLightIntensity}
			distance={1.2}
			decay={2}
			color={$averageScreenColor}
			bind:ref={pointlight}
		/>
	{/if}

	<Three type={AmbientLight} intensity={8} color={$averageScreenColor} />
	<Three type={AmbientLight} intensity={$whiteAmbientLightIntensity} color={'white'} />

	<!-- Red light -->
	<Three
		type={DirectionalLight}
		intensity={$redLightIntensity}
		color="#F67F55"
		position.x={-2.2}
		position.y={3.5}
		position.z={2.6}
	/>

	<!-- Blue light -->
	<Three
		type={DirectionalLight}
		intensity={$blueLightIntensity}
		position.x={2.2}
		position.y={3.4}
		position.z={2.6}
		color="#2722F3"
	/>

	<!-- White light -->
	<Three
		type={DirectionalLight}
		intensity={$whiteLightIntensity}
		position.x={-1}
		position.y={2.5}
		position.z={1}
		color="white"
	/>

	<!-- Floor -->
	<Three type={Mesh}>
		<Three type={CylinderGeometry} args={[1, 1, 0.04, 64]} />
		<Three type={MeshStandardMaterial} color={'#0f0f0f'} />
	</Three>
</Three>