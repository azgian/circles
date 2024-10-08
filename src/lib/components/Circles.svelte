<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';
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
		isPulsing?: boolean; // isPulsing 속성 추가
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

	interface ScoreRecord {
		time: number;
		color: string;
		score: number;
		isLatest?: boolean; // 최근 게임 여부를 나타내는 속성 추가
	}

	let gameState: 'idle' | 'running' = 'idle';
	const circles = writable<CircleData[]>([]);
	let particles: Particle[] = [];

	let container: HTMLElement;
	let containerWidth: number = 0;
	let containerHeight: number = 0;

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

	let scoreRecords: ScoreRecord[] = [];

	let finalScore = 0; // 최종 스코어를 저장할 새로운 변수

	let rightWallMultiplier = 1;
	let rightWallMultiplierTimer: number | null = null;
	let isMultiplierActive = false;

	// 공의 크기를 저장할 변수 추가
	let circleSize: number;

	let fireworksActive = false;

	const INITIAL_SPEED = 8; // 초기 속도 상수 정의

	let windowHeight: number;

	let isMobile: boolean;

	onMount(() => {
		updateContainerSize();
		window.addEventListener('resize', updateContainerSize);
		updateWindowHeight();
		window.addEventListener('resize', updateWindowHeight);
		checkMobile();
		window.addEventListener('resize', checkMobile);

		return () => {
			window.removeEventListener('resize', updateContainerSize);
			window.removeEventListener('resize', updateWindowHeight);
			window.removeEventListener('resize', checkMobile);
			stopGame();
		};
	});

	function updateContainerSize() {
		if (container) {
			containerWidth = container.clientWidth;
			containerHeight = container.clientHeight;
		}
	}

	function updateWindowHeight() {
		windowHeight = window.innerHeight;
	}

	function checkMobile() {
		isMobile = window.innerWidth <= 768;
	}

	const startGame = () => {
		if (gameState === 'idle' || gameOver) {
			gameState = 'running';
			gameOver = false;
			// 게임 시작 전 컨테이너 크기 업데이트를 지연시킵니다.
			setTimeout(() => {
				updateContainerSize();
				initializeCircles();
				gameRunning = true;
				animate();
				startTimer();
				startRightWallMultiplierTimer();
			}, 100);
		}
	};

	const startRightWallMultiplierTimer = () => {
		if (rightWallMultiplierTimer !== null) {
			clearInterval(rightWallMultiplierTimer);
		}
		rightWallMultiplierTimer = window.setInterval(() => {
			if (gameRunning && !gameOver) {
				rightWallMultiplier = 10;
				isMultiplierActive = true;
				fireworksActive = true;
				setTimeout(() => {
					fireworksActive = false;
				}, 1000); // 1초 동안 폭죽 효과 유지
			}
		}, 5000);
	};

	const resetRightWallMultiplier = () => {
		rightWallMultiplier = 1;
		isMultiplierActive = false;
	};

	const resetGame = () => {
		stopGame();
		$circles = [];
		particles = [];
		gameState = 'idle';
		timer = 0;
		gameOver = false;
		winningColor = '';
		totalSum = 0;
		scoreRecords = []; // 점수 기록 초기화
		if (animationId !== null) {
			cancelAnimationFrame(animationId);
			animationId = null;
		}
		rightWallMultiplier = 1;
		if (rightWallMultiplierTimer !== null) {
			clearInterval(rightWallMultiplierTimer);
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
		if (rightWallMultiplierTimer !== null) {
			clearInterval(rightWallMultiplierTimer);
		}
	};

	const startTimer = () => {
		if (timerInterval !== null) {
			clearInterval(timerInterval);
		}
		timer = 0;
		timerInterval = window.setInterval(() => {
			timer++;
		}, 1000);
	};

	const stopTimer = () => {
		if (timerInterval !== null) {
			clearInterval(timerInterval);
			timerInterval = null;
		}
	};

	const initializeCircles = () => {
		circleSize = Math.min(containerWidth, containerHeight) * 0.125;
		$circles = Array(8)
			.fill(null)
			.map(() => {
				const num = Math.floor(Math.random() * 100);
				return {
					id: getNextId(),
					num,
					size: circleSize,
					x: Math.random() * (containerWidth - circleSize) + circleSize / 2,
					y: Math.random() * (containerHeight - circleSize) + circleSize / 2,
					dx: (Math.random() - 0.5) * INITIAL_SPEED,
					dy: (Math.random() - 0.5) * INITIAL_SPEED,
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

				// 큰 숫자에서 작은 숫자를 빼기
				biggerCircle.num -= smallerCircle.num;

				// 만약 결과가 0 이하가 되면 1로 설정
				if (biggerCircle.num <= 0) {
					biggerCircle.num = 1;
				}

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
		if ($circles.length === 2 && $circles[0].color !== $circles[1].color) {
			const [circle1, circle2] = $circles;
			const smallerCircle = circle1.num < circle2.num ? circle1 : circle2;
			const biggerCircle = circle1.num < circle2.num ? circle2 : circle1;

			// 작은 숫자 공의 속도를 1.5배로 증가
			smallerCircle.dx =
				Math.sign(smallerCircle.dx) * Math.min(Math.abs(smallerCircle.dx) * 1.5, 12);
			smallerCircle.dy =
				Math.sign(smallerCircle.dy) * Math.min(Math.abs(smallerCircle.dy) * 1.5, 12);

			// 큰 숫자 공의 속도는 그대로 유지
			biggerCircle.dx = Math.sign(biggerCircle.dx) * Math.min(Math.abs(biggerCircle.dx), 8);
			biggerCircle.dy = Math.sign(biggerCircle.dy) * Math.min(Math.abs(biggerCircle.dy), 8);
		}

		for (let i = 0; i < $circles.length; i++) {
			let circle = { ...$circles[i] };

			// 다음 위치 계산
			let nextX = circle.x + circle.dx;
			let nextY = circle.y + circle.dy;

			// 경계 체크 및 위치 조정
			if (nextX - circle.size / 2 <= 0 || nextX + circle.size / 2 >= containerWidth) {
				circle.dx *= -1;
				nextX = Math.max(circle.size / 2, Math.min(containerWidth - circle.size / 2, nextX));

				// 우측 벽에 부딪혔을 때 점수 변경 및 10배 보너스 사용
				if (nextX + circle.size / 2 >= containerWidth) {
					if (rightWallMultiplier === 10) {
						circle.num *= rightWallMultiplier;
						circle.color = colors[circle.num % colors.length];
						circle.isPulsing = true;
						resetRightWallMultiplier();

						// 0.9초 후에 펄스 효과 제거 (3번의 0.3초 애니메이션)
						setTimeout(() => {
							circle.isPulsing = false;
							circles.update((circles) =>
								circles.map((c) => (c.id === circle.id ? { ...c, isPulsing: false } : c))
							);
						}, 900);
					}
				}
			}
			if (nextY - circle.size / 2 <= 0 || nextY + circle.size / 2 >= containerHeight) {
				circle.dy *= -1;
				nextY = Math.max(circle.size / 2, Math.min(containerHeight - circle.size / 2, nextY));
				// 상하 벽 충돌 시 점수 변경 제거
			}

			// 위치 업데이트
			circle.x = nextX;
			circle.y = nextY;

			// ignoreCollision 카운트 감소
			if (circle.ignoreCollision > 0) {
				circle.ignoreCollision--;
			}

			for (let j = i + 1; j < $circles.length; j++) {
				const collisionResult = checkCollision(circle, $circles[j]);
				if (collisionResult.length === 1) {
					circle = collisionResult[0];
					$circles[j] = { ...$circles[j], toRemove: true };
				} else {
					circle = collisionResult[0];
					$circles[j] = collisionResult[1];
				}
			}

			if (!circle.toRemove) {
				newCircles.push(circle);
			}
		}

		updateParticles();

		circles.set(newCircles);

		// 게임 종료 조건 체크
		const remainingColors = new Set($circles.map((circle) => circle.color));
		if (remainingColors.size === 1 && !gameOver) {
			gameOver = true;
			winningColor = Array.from(remainingColors)[0];
			totalSum = $circles.reduce((sum, circle) => sum + circle.num, 0);
			finalScore = totalSum * $circles.length;
			stopTimer();
			stopGame();
			resetRightWallMultiplier();

			// 모든 공의 속도를 초기 속도로 리셋
			circles.update((circles) =>
				circles.map((circle) => ({
					...circle,
					dx: Math.sign(circle.dx) * INITIAL_SPEED,
					dy: Math.sign(circle.dy) * INITIAL_SPEED
				}))
			);

			// 스코어 기록 추가
			scoreRecords = [
				{ time: timer, color: winningColor, score: finalScore, isLatest: true },
				...scoreRecords.map((record) => ({ ...record, isLatest: false })).slice(0, 4)
			];
		}

		if (animationId !== null) {
			cancelAnimationFrame(animationId);
		}
		animationId = requestAnimationFrame(animate);
	};
</script>

<svelte:window on:resize={updateContainerSize} />

<div class="game-container" class:mobile={isMobile}>
	<div class="canvas-container" bind:this={container}>
		<div class="title-overlay">CIRCLES</div>
		<div class="timer">{Math.floor(timer / 60)}:{(timer % 60).toString().padStart(2, '0')}</div>
		<div class="score-records">
			{#each scoreRecords as record}
				<div class="score-record">
					<span class="score-time"
						>{Math.floor(record.time / 60)}:{(record.time % 60).toString().padStart(2, '0')}</span
					>
					<span class="score-color" style="background-color: {record.color};"></span>
					<span class="score-value">{record.score}</span>
					{#if record.isLatest}
						<span class="chevron">◀</span>
					{/if}
				</div>
			{/each}
		</div>
		{#each $circles as circle (circle.id)}
			<div
				class="circle-wrapper"
				style="left: {circle.x - circle.size / 2}px; top: {circle.y -
					circle.size / 2}px; width: {circle.size}px; height: {circle.size}px;"
			>
				<Circle
					num={circle.num}
					size={circle.size}
					color={circle.color}
					isPulsing={circle.isPulsing}
				/>
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
					<small>토탈 스코어</small>
					<div>
						<strong>{totalSum}</strong>
						<span>× {$circles.length}</span>
					</div>
					<small>최종 스코어</small>
					<strong>{finalScore}</strong>
				</div>
			</div>
		{/if}
		{#if gameState !== 'running'}
			<div class="rules">
				<h3>게임 규칙:</h3>
				<ol>
					<li>Start 버튼을 눌러 게임을 시작합니다.</li>
					<li>숫자가 적힌 공들이 화면에 나타나 움직입니다.</li>
					<li>공의 색깔은 무지개색이며 공의 숫자를 7로 나눈 나머지로 색이 정해집니다.</li>
					<li>같은 색의 공들이 충돌하면 서로 팅겨져 나갑니다.</li>
					<li>
						다른 색의 공들이 충돌하면 큰 숫자에서 작은 숫자를 뺀 후, 작은 숫자의 공은 사라집니다.
					</li>
					<li>5초마다 우측벽에 10배 보너스가 적용되어 충돌시 해당 공 숫자가 10배가 됩니다.</li>
					<li>다른 색 공 두개만 남았을 때 작은 숫자의 공의 속도가 빨라집니다.</li>
					<li>한 가지 색상의 공만 남으면 게임이 종료됩니다.</li>
					<li>Reset 버튼을 눌러 언제든지 게임을 초기화할 수 있습니다.</li>
				</ol>
			</div>
		{/if}
		{#if !gameRunning || gameOver}
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
		{/if}
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
		</button>
		<div class="right-wall-multiplier" class:active={isMultiplierActive && !gameOver}>
			<span class="multiplier-text" class:fireworks={fireworksActive}>X10</span>
		</div>
	</div>
</div>

<style>
	.game-container {
		display: flex;
		flex-direction: column;
		align-items: center;
		margin: auto;
		width: calc(100vw - 50px);
		height: calc(100vh - 50px);
		max-height: 100vh;
	}

	.canvas-container {
		position: relative;
		width: 100%;
		max-width: 800px;
		height: 100%;
		max-height: calc(100vh - 100px);
		overflow: hidden;
		box-shadow: 0 0 20px rgba(38, 250, 119, 0.5);
		border-radius: 10px;
		margin-top: 50px;
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
		left: 10px;
		font-family: 'Protest Guerrilla', sans-serif;
		font-size: 30px;
		color: rgba(255, 255, 255, 0.5);
		text-align: center;
		width: 100%;
	}

	.start-button,
	.reset-button {
		display: flex;
		align-items: center;
		justify-content: center;
		border: none;
		border-radius: 5px;
		cursor: pointer;
		transition: background-color 0.3s;
		z-index: 10;
	}

	.start-button svg {
		margin-right: 8px;
	}

	.start-button {
		background-color: #4caf50;
		padding: 10px 20px;
		font-size: 16px;
		color: white;
		position: absolute;
		bottom: 10px;
		right: 10px;
	}

	.reset-button {
		background-color: #666;
		padding: 5px;
		font-size: 12px;
		color: #ccc;
		position: absolute;
		top: 10px;
		right: 10px;
	}

	.start-button:hover {
		background-color: #45a049;
	}
	.reset-button:hover {
		background-color: #555;
	}

	@media (max-width: 768px) {
		.game-container {
			margin: 10px;
			width: calc(100vw - 20px);
			height: calc(100vh - 20px);
		}
		.canvas-container {
			height: 100%;
			margin: 0;
			max-height: calc(100vh - 150px);
		}
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
		position: absolute;
		top: 50px;
		left: 50%;
		max-width: 500px;
		margin-left: -225px;
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

	@media (max-width: 500px) {
		.rules {
			left: 0px;
			margin-left: 0px;
			padding: 15px;
		}
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
		/* z-index: 1000; */
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
		gap: 10px;
	}
	.game-over-message strong {
		font-size: 1.5rem;
	}
	.game-over-message small {
		font-size: 1rem;
		opacity: 0.8;
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

	.score-records {
		position: absolute;
		bottom: 10px;
		left: 10px;
		display: flex;
		flex-direction: column;
		align-items: flex-start;
		/* z-index: 10; */
	}

	.score-record {
		display: flex;
		align-items: center;
		margin-bottom: 5px;
		font-size: 1.25rem;
		color: rgba(255, 255, 255, 0.3);
	}

	.score-time {
		min-width: 50px; /* 시간 표시 영역 고정 */
	}

	.score-color {
		width: 20px;
		height: 20px;
		border-radius: 50%;
		margin-right: 5px;
		border: solid #fff 1px;
	}

	.score-value {
		font-weight: bold;
		margin-right: 5px;
		min-width: 45px; /* 점수 표시 영역 고정 */
		text-align: right;
	}

	.chevron {
		font-size: 1rem;
		color: #26fa77;
	}

	.right-wall-multiplier {
		position: absolute;
		top: 0;
		right: 0;
		height: 100%;
		width: 10px;
		background-color: rgba(38, 250, 119, 1);
		display: flex;
		align-items: center;
		justify-content: center;
		opacity: 0;
		transition:
			opacity 0.3s ease,
			width 0.3s ease;
	}

	.right-wall-multiplier.active {
		opacity: 1;
		animation: fade-in-out 0.5s infinite;
	}

	.multiplier-text {
		font-size: 30px;
		color: rgba(38, 250, 119, 1);
		font-family: 'Protest Guerrilla', sans-serif;
		position: absolute;
		top: 48%;
		left: -50px;
		animation: fade-in-out 0.5s infinite;
	}

	.multiplier-text.fireworks {
		animation: fireworks 1s ease-out;
	}

	@keyframes fireworks {
		0% {
			transform: scale(1);
			opacity: 1;
			text-shadow:
				0 0 0 yellow,
				0 0 0 orange,
				0 0 0 red,
				0 0 0 rgba(38, 250, 119, 0.5);
		}
		25% {
			transform: scale(1.2);
			opacity: 1;
			text-shadow:
				-2px -2px 20px yellow,
				2px -2px 20px orange,
				-2px 2px 20px red,
				2px 2px 20px rgba(38, 250, 119, 0.5);
		}
		50% {
			transform: scale(1.3);
			opacity: 1;
			text-shadow:
				-4px -4px 40px yellow,
				4px -4px 40px orange,
				-4px 4px 40px red,
				4px 4px 40px rgba(38, 250, 119, 0.5);
		}
		75% {
			transform: scale(1.1);
			opacity: 0.8;
		}
		100% {
			transform: scale(1);
			opacity: 1;
		}
	}

	@keyframes fade-in-out {
		0%,
		100% {
			opacity: 0.3;
		}
		50% {
			opacity: 1;
		}
	}
</style>
