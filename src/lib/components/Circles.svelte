<script lang="ts">
	import { onMount } from 'svelte';
	import Circle from './Circle.svelte';

	interface CircleData {
		id: number;
		x: number;
		y: number;
		dx: number;
		dy: number;
		num: number;
		size: number;
		color: string;
		ignoreCollision: number;
		toRemove?: boolean; // toRemove 속성 추가
	}

	interface Particle {
		x: number;
		y: number;
		color: string;
		size: number;
		speedX: number;
		speedY: number;
		life: number;
		rotation: number;
		rotationSpeed: number;
	}

	let gameState: 'idle' | 'running' = 'idle';
	let circles: CircleData[] = [];
	let particles: Particle[] = [];

	let container: HTMLElement;
	let containerWidth: number;
	let containerHeight: number;

	const colors = ['#e60000', '#e67300', '#e6e600', '#00b300', '#0000e6', '#4b0082', '#8600b3'];

	let nextId = 0;
	const getNextId = () => nextId++;

	let timer = 0;
	let gameRunning = false;
	let animationId: number | null = null;
	let timerInterval: number | null = null;

	let gameOver = false;
	let winningColor = '';
	let totalSum = 0;

	onMount(() => {
		containerWidth = container.clientWidth;
		containerHeight = container.clientHeight;
		window.addEventListener('resize', handleResize);

		return () => {
			window.removeEventListener('resize', handleResize);
			stopGame();
		};
	});

	const handleResize = () => {
		containerWidth = container.clientWidth;
		containerHeight = container.clientHeight;
		if (gameState === 'running') {
			resetGame();
		}
	};

	const startGame = () => {
		if (gameState === 'idle' || gameOver) {
			gameState = 'running';
			gameOver = false;
			initializeCircles();
			gameRunning = true;
			timer = 0;
			if (animationId !== null) {
				cancelAnimationFrame(animationId);
				animationId = null;
			}
			animate();
			startTimer();
		}
	};

	const resetGame = () => {
		stopGame();
		circles = [];
		particles = [];
		gameState = 'idle';
		timer = 0;
		gameOver = false;
		winningColor = '';
		totalSum = 0;
		if (animationId !== null) {
			cancelAnimationFrame(animationId);
			animationId = null;
		}
	};

	const stopGame = () => {
		gameRunning = false;
		if (animationId !== null) {
			cancelAnimationFrame(animationId);
			animationId = null;
		}
		if (timerInterval !== null) {
			clearInterval(timerInterval);
			timerInterval = null;
		}
	};

	const startTimer = () => {
		if (timerInterval === null) {
			timerInterval = setInterval(() => {
				if (gameRunning) {
					timer++;
				} else {
					stopGame();
				}
			}, 1000);
		}
	};

	const initializeCircles = () => {
		const circleSize = Math.min(containerWidth, containerHeight) * 0.15;
		circles = Array(8)
			.fill(null)
			.map(() => {
				const num = Math.floor(Math.random() * 100);
				return {
					id: getNextId(),
					num,
					size: circleSize,
					x: Math.random() * (containerWidth - circleSize) + circleSize / 2,
					y: Math.random() * (containerHeight - circleSize) + circleSize / 2,
					dx: (Math.random() - 0.5) * 8, // 속도를 2배로 증가 (4에서 8로)
					dy: (Math.random() - 0.5) * 8, // 속도를 2배로 증가 (4에서 8로)
					color: colors[num % 7],
					ignoreCollision: 0
				};
			});
	};

	const checkCollision = (
		circle1: CircleData,
		circle2: CircleData,
		forceCollision: boolean = false
	): CircleData[] => {
		if (circle1.ignoreCollision > 0 || circle2.ignoreCollision > 0) {
			return [circle1, circle2];
		}

		const dx = circle2.x - circle1.x;
		const dy = circle2.y - circle1.y;
		const distance = Math.sqrt(dx * dx + dy * dy);

		if (distance < (circle1.size + circle2.size) / 2 || forceCollision) {
			if (circle1.color !== circle2.color) {
				// 다른 색깔의 공들이 충돌한 경우
				const biggerCircle = circle1.num > circle2.num ? { ...circle1 } : { ...circle2 };
				const smallerCircle = circle1.num > circle2.num ? { ...circle2 } : { ...circle1 };

				createParticles(smallerCircle.x, smallerCircle.y, smallerCircle.color);

				biggerCircle.num += smallerCircle.num;
				biggerCircle.color = colors[biggerCircle.num % colors.length];

				return [biggerCircle];
			} else {
				// 같은 색깔의 공들이 충돌한 경우
				const angle = Math.atan2(dy, dx);
				const sin = Math.sin(angle);
				const cos = Math.cos(angle);

				// 충돌 지점으로 이동
				const x1 = 0;
				const y1 = 0;
				const x2 = dx * cos + dy * sin;
				const y2 = dy * cos - dx * sin;

				// 속도 회전
				const vx1 = circle1.dx * cos + circle1.dy * sin;
				const vy1 = circle1.dy * cos - circle1.dx * sin;
				const vx2 = circle2.dx * cos + circle2.dy * sin;
				const vy2 = circle2.dy * cos - circle2.dx * sin;

				// 충돌 후 속도
				const newVx1 = vx2;
				const newVx2 = vx1;

				// 속도 회전 복구
				circle1.dx = newVx1 * cos - vy1 * sin;
				circle1.dy = vy1 * cos + newVx1 * sin;
				circle2.dx = newVx2 * cos - vy2 * sin;
				circle2.dy = vy2 * cos + newVx2 * sin;

				// 위치 조정 (겹침 방지)
				const absV = Math.abs(newVx1) + Math.abs(newVx2);
				const overlap = (circle1.size + circle2.size) / 2 - Math.abs(x2 - x1);
				circle1.x -= overlap * (newVx1 / absV);
				circle2.x += overlap * (newVx2 / absV);

				return [circle1, circle2];
			}
		}
		return [circle1, circle2];
	};

	const createParticles = (x: number, y: number, color: string) => {
		for (let i = 0; i < 8; i++) {
			particles.push({
				x,
				y,
				color,
				size: Math.random() * 10 + 10, // 크기를 더 크게 조정
				speedX: Math.random() * 8 - 4, // 속도 범위를 넓힘
				speedY: Math.random() * 8 - 4,
				life: 60, // 수명을 늘림
				rotation: Math.random() * 360,
				rotationSpeed: Math.random() * 10 - 5
			});
		}
	};

	const updateParticles = () => {
		particles = particles
			.filter((p) => p.life > 0)
			.map((p) => ({
				...p,
				x: p.x + p.speedX,
				y: p.y + p.speedY,
				rotation: p.rotation + p.rotationSpeed,
				life: p.life - 1
			}));
	};

	const animate = () => {
		if (!gameRunning && !gameOver) return;

		let newCircles: CircleData[] = [];

		// 두 개의 공만 남았을 때 속도 조정
		if (circles.length === 2 && circles[0].color !== circles[1].color) {
			const [circle1, circle2] = circles;
			const smallerCircle = circle1.num < circle2.num ? circle1 : circle2;
			const biggerCircle = circle1.num < circle2.num ? circle2 : circle1;

			// 속도 증가를 더 신중하게 적용
			const speedIncrease = 1.5; // 1.5배로 줄임
			smallerCircle.dx =
				Math.sign(smallerCircle.dx) * Math.min(Math.abs(smallerCircle.dx) * speedIncrease, 10);
			smallerCircle.dy =
				Math.sign(smallerCircle.dy) * Math.min(Math.abs(smallerCircle.dy) * speedIncrease, 10);
		}

		for (let i = 0; i < circles.length; i++) {
			let circle = { ...circles[i] };

			// 다음 위치 계산
			let nextX = circle.x + circle.dx;
			let nextY = circle.y + circle.dy;

			// 경계 체크 및 위치 조정
			if (nextX - circle.size / 2 <= 0 || nextX + circle.size / 2 >= containerWidth) {
				circle.dx *= -1;
				nextX = Math.max(circle.size / 2, Math.min(containerWidth - circle.size / 2, nextX));
			}
			if (nextY - circle.size / 2 <= 0 || nextY + circle.size / 2 >= containerHeight) {
				circle.dy *= -1;
				nextY = Math.max(circle.size / 2, Math.min(containerHeight - circle.size / 2, nextY));
			}

			// 위치 업데이트
			circle.x = nextX;
			circle.y = nextY;

			// ignoreCollision 카운트 감소
			if (circle.ignoreCollision > 0) {
				circle.ignoreCollision--;
			}

			for (let j = i + 1; j < circles.length; j++) {
				const collisionResult = checkCollision(circle, circles[j]);
				if (collisionResult.length === 1) {
					circle = collisionResult[0];
					circles[j] = { ...circles[j], toRemove: true };
				} else {
					circle = collisionResult[0];
					circles[j] = collisionResult[1];
				}
			}

			if (!circle.toRemove) {
				newCircles.push(circle);
			}
		}

		updateParticles();

		circles = newCircles;

		// 게임 종료 조건 체크
		const remainingColors = new Set(circles.map((circle) => circle.color));
		if (remainingColors.size === 1 && !gameOver) {
			gameOver = true;
			winningColor = Array.from(remainingColors)[0];
			totalSum = circles.reduce((sum, circle) => sum + circle.num, 0);
		}

		if (animationId !== null) {
			cancelAnimationFrame(animationId);
		}
		animationId = requestAnimationFrame(animate);
	};
</script>

<div class="game-container">
	<div class="canvas-container" bind:this={container}>
		<div class="title-overlay">CIRCLES</div>
		{#each circles as circle (circle.id)}
			<div
				class="circle-wrapper"
				style="left: {circle.x - circle.size / 2}px; top: {circle.y -
					circle.size / 2}px; width: {circle.size}px; height: {circle.size}px;"
			>
				<Circle num={circle.num} size={circle.size} color={circle.color} />
			</div>
		{/each}
		{#each particles as particle}
			<div
				class="particle"
				style="
					left: {particle.x}px;
					top: {particle.y}px;
					background-color: {particle.color};
					width: {particle.size}px;
					height: {particle.size}px;
					transform: rotate({particle.rotation}deg);
				"
			></div>
		{/each}
		{#if gameOver}
			<div class="game-over-layer">
				<div class="game-over-message">
					<Circle num={0} size={50} color={winningColor} />
					<small>Total Scores</small>
					<strong>{totalSum}</strong>
				</div>
			</div>
		{/if}
	</div>
	<div class="button-container">
		<button class="start-button" on:click={startGame}>
			<svg
				xmlns="http://www.w3.org/2000/svg"
				width="24"
				height="24"
				viewBox="0 0 24 24"
				fill="white"
			>
				<path d="M8 5v14l11-7z" />
			</svg>
			Start
		</button>
		<button class="reset-button" on:click={resetGame}>
			<svg
				xmlns="http://www.w3.org/2000/svg"
				width="24"
				height="24"
				viewBox="0 0 24 24"
				fill="white"
			>
				<path
					d="M17.65 6.35C16.2 4.9 14.21 4 12 4c-4.42 0-7.99 3.58-7.99 8s3.57 8 7.99 8c3.73 0 6.84-2.55 7.73-6h-2.08c-.82 2.33-3.04 4-5.65 4-3.31 0-6-2.69-6-6s2.69-6 6-6c1.66 0 3.14.69 4.22 1.78L13 11h7V4l-2.35 2.35z"
				/>
			</svg>
			Reset
		</button>
	</div>
	<div class="rules">
		<h3>게임 규칙:</h3>
		<ol>
			<li>Start 버튼을 눌러 게임을 시작합니다.</li>
			<li>숫자가 적힌 공들이 화면에 나타나 움직입니다.</li>
			<li>같은 색의 공들이 충돌하면 서로 팅겨져 나갑니다.</li>
			<li>다른 색의 공들이 충돌하면 큰 숫자의 공이 작은 숫자의 공을 흡수합니다.</li>
			<li>흡수 후 공의 숫자가 증가하고, 색상이 변경될 수 있습니다.</li>
			<li>한 가지 색상의 공만 남으면 게임이 종료됩니다.</li>
			<li>Reset 버튼을 눌러 언제든지 게임을 초기화할 수 있습니다.</li>
		</ol>
	</div>
</div>

<style>
	.game-container {
		display: flex;
		flex-direction: column;
		align-items: center;
		margin: 50px auto;
		max-width: 800px;
	}

	@media (max-width: 768px) {
		.game-container {
			margin: 10px 10px 100px 10px;
			width: calc(100% - 20px);
		}
	}

	.canvas-container {
		position: relative;
		width: 100%;
		height: 600px;
		max-height: 100vh;
		overflow: hidden;
		box-shadow: 0 0 20px rgba(38, 250, 119, 0.5);
		border-radius: 10px;
	}

	.circle-wrapper {
		position: absolute;
		transition:
			left 0.1s linear,
			top 0.1s linear;
	}

	.timer {
		position: absolute;
		bottom: 10px;
		left: 50%;
		transform: translateX(-50%);
		background-color: rgba(0, 0, 0, 0.5);
		color: white;
		padding: 5px 10px;
		border-radius: 5px;
		font-size: 14px;
	}

	.button-container {
		display: flex;
		justify-content: center;
		width: 100%;
		margin-top: 20px;
		z-index: 500;
		position: relative;
	}

	.start-button,
	.reset-button {
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 10px 20px;
		font-size: 16px;
		color: white;
		border: none;
		border-radius: 5px;
		cursor: pointer;
		transition: background-color 0.3s;
	}

	.start-button svg,
	.reset-button svg {
		margin-right: 8px;
	}

	.start-button {
		background-color: #4caf50;
	}

	.reset-button {
		background-color: #e0d6d523;
		position: absolute;
		right: 0px;
	}

	.start-button:hover {
		background-color: #45a049;
	}

	.reset-button:hover {
		background-color: #d32f2f;
	}

	@media (max-width: 768px) {
		.start-button,
		.reset-button {
			padding: 8px 16px;
			font-size: 14px;
		}

		.start-button svg,
		.reset-button svg {
			width: 20px;
			height: 20px;
		}
	}

	.rules {
		margin-top: 20px;
		text-align: left;
		max-width: 600px;
		padding: 15px;
		border-radius: 10px;
		box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
		color: #999;
	}

	.rules h3 {
		margin-top: 0;
	}

	.rules ol {
		padding-left: 20px;
	}

	.rules li {
		margin-bottom: 10px;
	}

	.game-over-layer {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background-color: rgba(0, 0, 0, 0.3); /* 배경색을 더 어둡게 변경 */
		display: flex;
		justify-content: center;
		align-items: center;
		z-index: 1000;
	}

	.game-over-message {
		background-color: rgba(0, 0, 0, 0.5);
		padding: 20px;
		border-radius: 10px;
		text-align: center;
		color: #999;
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 15px;
	}
	.game-over-message strong {
		font-size: 1.5rem;
	}

	.particle {
		position: absolute;
		opacity: 0.9;
		animation: fade-out 0.5s linear;
		clip-path: polygon(
			50% 0%,
			80% 10%,
			100% 35%,
			100% 70%,
			80% 90%,
			50% 100%,
			20% 90%,
			0% 70%,
			0% 35%,
			20% 10%
		);
	}

	@keyframes fade-out {
		0% {
			opacity: 0.9;
			transform: scale(1);
		}
		100% {
			opacity: 0;
			transform: scale(0.5);
		}
	}

	.title-overlay {
		position: absolute;
		top: 20px;
		left: 0;
		width: 100%;
		text-align: center;
		font-family: 'Protest Guerrilla', sans-serif;
		font-size: 48px;
		color: rgba(38, 250, 119, 0.5);
		pointer-events: none;
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
	}
</style>
